# Архитектура Contextor

> **Переписано заново 2026-09-18** по фактическому коду, HEAD `dd36774`.
> Предыдущая версия начиналась словами «Честное описание архитектуры… как есть, без
> маркетинга» и при этом содержала ложные утверждения: ChromaDB как хранилище WARM,
> несуществующий слой COLD (`core/archive.py`), «cosine similarity» для CCI,
> несуществующий Dashboard, «465 тестов проходят». Архивирована.
>
> **Проблемы:** `docs/known-issues.md` · **Хронология:** `docs/JOURNAL.md` ·
> **Проработка прокси-режима:** `docs/proxy/`

Здесь описано **как есть**. Где функция не работает — это указано. Где чего-то нет —
тоже указано.

---

## 1. Что это такое по факту кода

Contextor — локальный оркестратор, который управляет контекстным окном LLM через
иерархическую память и периодический сброс контекста.

Проблема: у LLM конечное окно. При длинном разговоре окно переполняется, и модель
деградирует — забывает начало, путает решения. Contextor пытается не обрезать историю, а
**сжимать её в компактную «координату»** и подмешивать эту координату в промпт.

В коде это реализовано как **самостоятельный чат-сервер со своим буфером**. Замысел
прокси перед чужим движком (Module Mode) в коде отсутствует.

---

## 2. Топология по факту

```
┌──────────────────────────────────────────────────────────────┐
│  FastAPI (server.py, 255 строк)                               │
│    /            → static/index.html                           │
│    /api/v1/*    → 27 маршрутов (server.py:38)                  │
│    /v1/*        → OpenAI-совместимые (server.py:39)            │
│    /ws          → WebSocket (server.py:42)                     │
└───────┬──────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────┐
│  OrchestratorPipeline (core/orchestrator.py, 928 строк)       │
│    intent → retrieve → graph → prompt → generate → memory      │
└───────┬──────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────┐   ┌────────────────────────────────┐
│  DualModelRouter          │   │  Память                        │
│  (core/dual_model.py)     │   │  WorkingMemory — список в RAM  │
│   coordinator  2B         │   │  MemoryStorage — словарь в RAM │
│   generator    9B         │   │  MetaCoordinator — координаты  │
│   utility      9B         │   │  + зеркало на диск (JSON)      │
└───────┬──────────────────┘   └────────────────────────────────┘
        │ HTTP /v1/chat/completions
┌───────▼──────────────────────────────────────────────────────┐
│  Ollama (внешний процесс, по умолчанию localhost:11434)        │
└──────────────────────────────────────────────────────────────┘
```

**Хранилище — не база данных.** `MemoryStorage` держит факты в обычном словаре в
оперативной памяти (`core/memory/storage.py:205`) и зеркалит его в один JSON-файл
(`:551`). ChromaDB используется **только** для индекса карточек кода
(`core/retriever.py:68-75`), который никогда не наполняется. Утверждение прежней
документации «WARM = ChromaDB + SentenceTransformer» неверно.

---

## 3. Пара «координатор + генератор» — ядро замысла

Это главный механизм, и он **реализован**. Порядок вызовов внутри одного хода
(`core/orchestrator.py`):

| Строка | Действие |
|---|---|
| `:332` | текущий запрос добавляется в историю |
| `:335` | `cci_tracker.evaluate(query)` — оценка связности |
| `:342` | `_should_soft_reset(score)` — решение о сбросе |
| **`:345`** | **`_soft_reset()`** — координатор строит координату |
| `:234` | история обрезается до последних 6 сообщений |
| `:364` | определение intent (регулярки) |
| `:382` | поиск карточек кода |
| `:401` | поиск по графу (падает, ошибка глушится — `ISSUE-006`) |
| `:412-438` | utility-воркер для web_search / read_document |
| **`:444`** | `system_prompt = system or _build_system_prompt(...)` |
| `:451` | `_build_messages()` — сборка `messages[]` |
| **`:460`** | **`_generate()` → генератор** |
| `:469` | ответ добавляется в историю |
| `:492` | сохранение сессии |
| `:521-575` | tagger → рабочая память → оптимизатор |

Ключевое: **координата создаётся до сборки промпта**, поэтому генератор получает её в
том же ходу, в котором контекст был сброшен.

### Две модели независимы

`DualModelRouter` (`core/dual_model.py`) вызывает обе через отдельные HTTP-запросы и не
передаёт между ними состояния:

```python
# core/dual_model.py:221-248 — координатор
def coordinate(self, messages, temperature=0.1, max_tokens=512):
    content, pt, ct = self._call_ollama(messages=messages,
                                        model=self.coordinator_model, ...)

# core/dual_model.py:250-290 — генератор
def generate(self, messages, temperature=0.7, max_tokens=2048):
    model = self.generator_model if self._check_generator_available() else self.coordinator_model
```

Координатор — **чистая функция** от переданных сообщений: он не помнит предыдущих
вызовов. Генератор о существовании координатора не знает. Оба ходят в
`{ollama_url}/v1/chat/completions` (`:180-219`).

### Как координата попадает к генератору

```python
# core/orchestrator.py:704-708
meta_context = self._meta_coordinator.get_context_for_prompt()
if meta_context:
    parts.append("\n## Координаты сессии:")
    parts.append(meta_context)

# core/orchestrator.py:710-714
memory_context = self.working_memory.get_context(max_tokens=400)
if memory_context:
    parts.append("\n## Контекст из памяти:")
    parts.append(memory_context)
```

Координата попадает в **системный промпт**, а не первым сообщением — прежняя
документация утверждала обратное.

### Как память не растёт с длиной сессии

`MetaCoordinator` (`core/memory/meta_coordinator.py`) решает задачу постоянного размера:

1. каждый сброс → `add_coordinate()` (`:97`), координата идёт в `_active`;
2. при `len(_active) >= 4` → `needs_meta()` истинно (`:111`);
3. оркестратор строит мета-координату, `consolidate()` (`:113-142`) архивирует активные
   в `coordinate_archive/batch_<timestamp>.json` и оставляет одну мета;
4. `get_context_for_prompt()` (`:144-167`) отдаёт **мету + последнюю активную**.

Размер памяти, уходящей в промпт, постоянен при любой длине беседы — в докстроке
заявлено ~600 токенов. Обратная сторона: до 3 промежуточных координат в промпт не
попадают.

### Отказ координатора

```python
# core/orchestrator.py:196-198 — скриптовый fallback
key_messages = [m for m in chat_history if m["role"] == "user"][-3:]
return "Контекст разговора: " + " | ".join(m["content"][:100] for m in key_messages)
```

Если координатор вернул пусто (`dual_model.py:246-248`), координата собирается
конкатенацией последних трёх пользовательских сообщений. Модель для этого не нужна.

### Поведение сброса на практике

Триггеры (`core/orchestrator.py:285-315`): `turns >= 16` → безусловно;
`len(chat_history) > 12` → переполнение истории; `cci < 0.55 и turns >= 4` → по связности.
Все три значения **захардкожены**.

Так как переполнение проверяется по 12 **сообщениям**, а не по токенам, при сжатии
истории до 6 сообщений порог снова достигается через 3 хода — то есть «адаптивный» сброс
на практике работает как расписание с периодом 4 хода (`ISSUE-005`).

### Ключевое ограничение

```python
# core/orchestrator.py:444
system_prompt = system or self._build_system_prompt(intent, context_cards, graph_entities)
```

Если клиент прислал `system`-сообщение (а так делают все OpenAI-совместимые клиенты),
`_build_system_prompt` **не вызывается** — координаты, факты, карточки и граф в промпт не
попадают. `ISSUE-003`.

---

## 4. Подсистема памяти

| Компонент | Файл | Строк | Назначение | Состояние |
|---|---|---:|---|---|
| `Fact` | `memory/fact.py` | 141 | единица знания: `content`, `attention_weight`, `is_anchor` | Работает |
| `WorkingMemory` | `memory/working_memory.py` | 396 | список фактов в RAM, бюджет токенов, decay | Работает, с дефектами |
| `MemoryStorage` | `memory/storage.py` | 588 | словарь фактов в RAM + зеркало на диск | Работает, с дефектами |
| `Scorer` | `memory/scorer.py` | 170 | вес внимания | Работает |
| `Optimizer` | `memory/optimizer.py` | 235 | сжатие и архивация холодных фактов | **Уничтожает содержимое** (`ISSUE-008`) |
| `ImportanceTagger` | `memory/tagger.py` | 248 | извлечение фактов моделью | Залипает на regex-fallback (`ISSUE-015`) |
| `CCITracker` | `memory/cci.py` | 279 | метрика связности | **Не может сработать** (`ISSUE-004`) |
| `MetaCoordinator` | `memory/meta_coordinator.py` | 254 | жизненный цикл координат | Работает |

### Слои: что есть на самом деле

В коде:

| Слой документации | Что в коде |
|---|---|
| HOT | `WorkingMemory._facts` — список в RAM (`working_memory.py:49`) |
| WARM | `MemoryStorage._facts` — словарь в RAM + JSON-зеркало (`storage.py:205,551`) |
| Anchor | **не слой**, а булев флаг `Fact.is_anchor` (`fact.py:53`) на факте в том же HOT-списке |
| COLD | значение `CompressionLevel.ARCHIVED` (`fact.py:22`) |

Слоя «Anchor» как отдельного хранилища не существует. Гарантия «никогда не удаляется» не
выполняется: `_evict_to_budget()` (`working_memory.py:380-389`) выбирает факт по
минимальному весу без проверки `is_anchor` (`ISSUE-007`).

### Где живут данные

```
storage/
├── chromadb/            # индекс карточек кода — никогда не наполняется
└── sessions/
    └── <session_id>/
        ├── working_memory.json     # HOT-факты
        ├── storage.json            # WARM-факты
        ├── chat_history.json       # история беседы
        ├── session_meta.json       # метаданные
        └── coordinate_archive/     # архивированные координаты (batch_*.json)
```

Путь `storage/sessions` **захардкожен** относительно текущего каталога
(`orchestrator.py:107,112`). `settings.storage_dir` и `settings.chroma_dir` существуют, но
используются для одной строки лога (`server.py:242`).

`session_meta.json` пишут два разных класса (`core/session.py:63` и
`core/session_manager.py:94`), причём `save()` перезаписывает его без `display_name` —
имена чатов не сохраняются (`ISSUE-026`).

### Контур подкачки фактов не работает

`MemoryStorage.retrieve()` имеет ровно один вызов вне тестов — `orchestrator.py:357`,
внутри `if coherence_result.needs_context_restore():`. Предикат всегда `False`
(`ISSUE-004`), поэтому факты из хранилища в окно никогда не подкачиваются.

Это **не** затрагивает цепочку координат из раздела 3 — она работает независимо.

---

## 5. Граф и RAG

| Компонент | Файл | Строк | Состояние |
|---|---|---:|---|
| `KnowledgeGraph` | `core/graph.py` | 128 | Не используется |
| `GraphBuilder` | `core/graph_builder.py` | 137 | `build_from_directory` без вызовов; `storage/graph.json` отсутствует |
| `Retriever` | `core/retriever.py` | 199 | Вызывается, но коллекция пуста |
| `CardGenerator` | `core/card_generator.py` | 214 | Создаётся (`orchestrator.py:70`) и не используется |
| `ContextAssembler` | `core/assembler.py` | 162 | Достижим только из мёртвого `run_stream()` |

Два дефекта, делающих эти подсистемы бесполезными:

```python
# core/orchestrator.py:401 — метода не существует
results = self.graph_builder.search_nodes(entity, limit=3)
# GraphBuilder.search_nodes → False ; GraphBuilder.search → True  (ISSUE-006)
```
```python
# core/orchestrator.py:679 — карточки читаются не с теми полями
entity = getattr(card, 'entity', None)          # а в RetrievalResult поля плоские
name = getattr(entity, 'name', 'Unknown')       # → всегда 'Unknown'  (ISSUE-013)
```

---

## 6. Триада моделей

| Роль | Модель по умолчанию | Где используется |
|---|---|---|
| Coordinator | `qwen3.5:2b` | `DualModelRouter.coordinate()` (`dual_model.py:234`); `ImportanceTagger` (`tagger.py:27` — жёстко зашито) |
| Generator | `qwen3.5:9b` | `DualModelRouter.generate()` (`:262`) |
| Utility | `qwen3.5:9b` | `core/utility_worker.py` — web_search, read_document |
| Embedder | `nomic-embed-text` | `storage.py:154` (через HTTP Ollama) |

**Обе основные модели грузятся резидентно.** При старте `server.py:204-228` считает, влезут
ли обе с 25% запасом, и если да — грузит с `keep_alive: -1`:

```python
# server.py:224
json={"model": model, "prompt": "", "keep_alive": -1}
```

Механизм для обратного — выгрузка координатора и возврат по требованию — написан
(`utils/swap_manager.py:113,132` `acquire_coordinator` / `release_coordinator`) и
**не вызывается ниоткуда**. Из модуля используются только `acquire_utility` /
`release_utility`, и то лишь в `core/utility_worker.py:125,140`.

### Имя модели в ответе недостоверно

```python
# core/orchestrator.py:753-764
def _generate(self, messages, model_key, temperature, max_tokens):
    text, p, c = self._router.generate(messages=messages, ...)
    # model_key принимается и не передаётся в роутер
```

`_select_model()` возвращает ключи реестра (`:651,653`), а генерирует модель из конфига
роутера. `OrchestrationResult.model_used` сообщает не ту модель (`ISSUE-023`).

---

## 7. Пайплайн ответа и стриминг

Стриминга нет ни в одном из четырёх мест (`ISSUE-012`):

| Путь | Что происходит |
|---|---|
| WebSocket `/ws` | `pipeline.run()` целиком, затем ответ режется по пробелам с `asyncio.sleep(0)` (`api/websocket.py:217-226`) |
| OpenAI SSE | `_sse_stream()` с докстрокой `Fake SSE streaming` (`api/system.py:322-341`) |
| OpenAI прокси-путь | захардкожено `"stream": False` (`api/system.py:388`) |
| `run_stream()` | Написан (`orchestrator.py:580`), ноль вызовов |

Синхронный `pipe.run()` вызывается внутри `async def` (`api/system.py:412`), блокируя цикл
событий на всё время генерации (`ISSUE-024`).

---

## 8. Конфигурация

Две независимые системы:

| Система | Источник | Что читает |
|---|---|---|
| `engines/config_loader.py` | `config.yaml` | модели, `num_ctx`, часть ключей памяти |
| `config.py` (pydantic-settings) | env `CONTEXTOR_*` + `.env` | `host`, `port`, `ollama_url` |

**Порядок поиска `config.yaml`** (`config_loader.py:118-153`): env `PURE_INTELLECT_CONFIG` →
`%APPDATA%\Contextor\config.yaml` → `~/.config/contextor/` → текущий каталог → корень
проекта → `/etc`. AppData имеет приоритет над репозиторием, поэтому **репозиторный
`config.yaml` может не читаться вообще** (`ISSUE-016`).

**Живые ключи:** `models.*`, `memory.num_ctx`, `memory.meta_coordinate_every`.
**Мёртвые:** вся секция `memory.adaptive_reset.*` (не парсится), `memory.context_window_messages`,
`memory.keep_after_reset`, `memory.working_memory_tokens`, `memory.max_storage_facts`,
`models.utility`, `server.*`, `ollama.*`. Полный разбор — `ISSUE-017`.

Отдельная деталь: имя переменной окружения `PURE_INTELLECT_CONFIG` осталось от прежнего
названия проекта «Чистый Интеллект» и не соответствует префиксу `CONTEXTOR_` в `config.py:92`.

---

## 9. HTTP-поверхность

27 маршрутов в OpenAPI, плюс два скрытых (`include_in_schema=False`), WebSocket и
статический каталог. Полное описание — `docs/api_reference.md`. Здесь — группы:

| Группа | Префикс | Назначение |
|---|---|---|
| Система | `/api/v1` | health, config, settings, hardware, logs, version |
| Чат | `/api/v1/chat` | ответ через пайплайн |
| Сессии | `/api/v1/sessions/*`, `/session/*` | список, создание, переключение, история |
| Память | `/api/v1/memory/*` | stats, facts, clear |
| Модели | `/api/v1/models/*`, `/ollama/models` | статус, переключение, скачивание, удаление |
| Диагностика | `/api/v1/cci/stats`, `/dual-model/stats` | метрики |
| OpenAI | `/v1/models`, `/v1/chat/completions` | совместимый API |
| WebSocket | `/ws` | чат-поток |

Эндпоинты `GET /memory/search`, `POST /memory/fact`, `DELETE /memory/clear` (реально
только POST), `WS /ws/chat` из прежней документации **не существуют**.

---

## 10. Инвентарь кода

54 файла, 9748 строк в `src/`. Крупнейшие:

| Файл | Строк | Роль |
|---|---:|---|
| `core/orchestrator.py` | 928 | главный пайплайн, сброс, координаты |
| `core/memory/storage.py` | 588 | WARM-хранилище |
| `api/system.py` | 457 | системный API + OpenAI-совместимый |
| `utils/hardware_detector.py` | 401 | определение GPU и рекомендации |
| `core/session_manager.py` | 400 | управление сессиями |
| `core/memory/working_memory.py` | 396 | HOT-буфер |
| `api/websocket.py` | 358 | WebSocket |
| `core/dual_model.py` | 345 | маршрутизация моделей |
| `api/models_api.py` | 310 | управление моделями |
| `core/intent.py` | 308 | определение намерения |
| `engines/provider.py` | 297 | фабрика провайдеров |
| `engines/config_loader.py` | 292 | загрузка конфигурации |
| `core/memory/cci.py` | 279 | метрика связности |
| `server.py` | 255 | FastAPI-приложение |
| `core/memory/meta_coordinator.py` | 254 | жизненный цикл координат |

### Мёртвый код

Есть в дереве, не вызывается ниоткуда:

- `core/summarizer.py` (121) — **не импортируется нигде**; коллабораторы
  (`archive.get_pairs`, `set_summary`, `trim_pairs`) отсутствуют, `core/archive.py` удалён
  в `5aa4777`. Содержит скриптовый (`_simple_compress`) и модельный (`_llm_compress`) пути
  построения памяти;
- `core/orchestrator.py:580` `run_stream()`, `:919` `_auto_name_session_if_first()`;
- `core/orchestrator.py:70` `self.card_generator`, `:83` `self._scorer`;
- `utils/tokenizer.py:22` `fit_messages_budget()` — готовая обрезка под бюджет токенов;
- `utils/swap_manager.py:113,132` `acquire_coordinator` / `release_coordinator`;
- `core/assembler.py:154` `assemble_and_respond()`;
- `core/intent.py:164-199` `detect_llm()` — требует `model_manager.loaded_model`, который не загружается;
- `engines/ollama.py` `OllamaEngine`, `engines/provider.py` `ProviderFactory` — вне живого пути;
- `engine/` (ед. ч., 213 строк) — legacy llama-cpp стек; достигается из `__main__.py:8` и
  косвенно через `api/state.py:7`, то есть по HTTP тоже достижим.

---

## 11. Чего в коде нет

Перечислено, чтобы это не появилось в документации снова как существующее.

| Заявлено в архивной документации | Факт |
|---|---|
| UCIP v2, 4-слойный Context Package | Отсутствует. Есть один слой `[RELEVANT CODE CONTEXT]` (`retriever.py:187`) |
| Module Mode, Decision Engine, Context Surgery | Отсутствует полностью |
| Dashboard с метриками GPU/CCI | Отсутствует. Вкладок пять: Chat, Memory, Models, Settings, Logs |
| `core/archive.py` как слой COLD | Файла нет |
| `docs/ROADMAP.md` | Файла нет |
| CLI `contextor config --show-path` | Команд только две: `model`, `serve` |
| «85% fewer tokens», «100% recall», «5ms/fact» | Ничем не измеряются (`ISSUE-032`) |
| «465 тестов проходят» | 367 passed, 25 failed |

---

## 12. Границы применимости

Что проект делает: управляет контекстным окном **своей** беседы в собственном чат-сервере,
сжимая историю в координаты через вторую, маленькую модель.

Что он не делает: не выступает прокси перед чужим движком, не обслуживает внешние
приложения прозрачно, не работает с `llama.cpp` (провайдер `llamacpp` закомментирован в
`engines/provider.py:217-218`), не хранит семантический индекс фактов, не стримит ответ.

Проработка перехода к прокси-режиму — в `docs/proxy/`.
