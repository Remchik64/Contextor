# Реестр проблем — Contextor

> **Единственный источник истины по проблемам проекта.** Создан 2026-09-18.
> HEAD на момент создания: `dd36774`.
>
> Заменяет разделы «Активные проблемы» и «Следующие задачи» из архивированного
> `docs/archive/HANDOFF-2026-07-11.md`, которые содержали ложные утверждения.

---

## Как пользоваться

Каждая проблема имеет идентификатор `ISSUE-NNN`. Ссылайтесь на идентификатор, а не на
номер строки — строки сдвигаются при правках.

**Метка достоверности** у каждой записи:

| Метка | Значение |
|---|---|
| ✅ | Проверено лично: запуском кода или чтением обеих сторон |
| ⚠️ | Сообщено аудитом, лично не перепроверял |
| 📄 | Дефект документации, не кода |

**Степень критичности:**

| Степень | Значение |
|---|---|
| 🔴 CRITICAL | Потеря данных, уязвимость или недостижимость основной функции |
| 🟠 HIGH | Функция не работает или выдаёт неверный результат |
| 🟡 MEDIUM | Работает частично, искажает поведение или измерения |
| 🔵 LOW | Косметика, гигиена, удобство |

---

## Сводная таблица

| ID | Степень | Область | Проблема | Достоверность |
|---|---|---|---|---|
| ISSUE-001 | 🔴 | Безопасность | `DELETE /sessions/..` удаляет весь `storage/` без аутентификации | ✅ |
| ISSUE-002 | 🔴 | Память | Не-ASCII текст уничтожается при сохранении; со 2-го раза сессия теряется целиком | ✅ |
| ISSUE-003 | 🔴 | Ядро | Клиентское `system` отключает всю инжекцию памяти | ✅ |
| ISSUE-004 | 🟠 | Ядро | CCI не может сработать → подкачка фактов из WARM недостижима | ✅ |
| ISSUE-005 | 🟠 | Ядро | `num_ctx` не управляет сжатием (захардкожено 12 сообщений) | ✅ |
| ISSUE-006 | 🟠 | Ядро | Слой графа падает каждый ход (`search_nodes` не существует) | ✅ |
| ISSUE-007 | 🟠 | Память | Anchor-факты вытесняются вопреки гарантии | ✅ |
| ISSUE-008 | 🟠 | Память | Сжатие заменяет содержимое факта служебным тегом | ✅ |
| ISSUE-009 | 🟠 | Память | Обрыв вытеснения на 30 фактах | ✅ |
| ISSUE-010 | 🟠 | Память | Факты протекают между сессиями | ⚠️ |
| ISSUE-011 | 🟠 | Ядро | Отказ бэкенда выглядит как успешный пустой ответ | ✅ |
| ISSUE-012 | 🟠 | API | `stream: true` не работает: SSE-заглушка, `stream: False` на прокси | ✅ |
| ISSUE-013 | 🟠 | Ядро | RAG: карточки рендерятся как «Unknown» | ✅ |
| ISSUE-014 | 🟠 | Ядро | RAG-индекс никогда не наполняется | ⚠️ |
| ISSUE-015 | 🟠 | Ядро | Провайдер эмбеддингов залипает на BM25 навсегда | ⚠️ |
| ISSUE-016 | 🟠 | Конфиг | Репозиторный `config.yaml` игнорируется (эффективен AppData) | ✅ |
| ISSUE-017 | 🟠 | Конфиг | Секция `adaptive_reset` не парсится; 4 ключа парсятся, но не используются | ✅ |
| ISSUE-018 | 🟠 | Упаковка | `psutil` не объявлен и отсутствует → `cpu_cores: 0`, `ram_gb: 0` | ✅ |
| ISSUE-019 | 🟠 | Упаковка | Установщики бьют в 404 + `curl` без `-f` | ✅ |
| ISSUE-020 | 🟠 | Тесты | 25 падений; `test_system_full.py` вешает сбор тестов | ✅ |
| ISSUE-021 | 🟡 | Ядро | `add_anchor` не вызывается в `_soft_reset` → координаты не сцепляются | ✅ |
| ISSUE-022 | 🟡 | Ядро | Бюджет считается токенизатором GPT-4, а не модели | ✅ |
| ISSUE-023 | 🟡 | Ядро | `model_used` сообщает модель, которая не работала | ✅ |
| ISSUE-024 | 🟡 | Ядро | Синхронный пайплайн внутри `async def` | ⚠️ |
| ISSUE-025 | 🟡 | API | Панель Settings молча выбрасывает 7 значений | ✅ |
| ISSUE-026 | 🟡 | API | `session_meta.json` затирается каждый ход, имена чатов не сохраняются | ✅ |
| ISSUE-027 | 🟡 | API | Версии расходятся: 0.3.0 против 0.1.0 | ✅ |
| ISSUE-028 | 🟡 | Упаковка | `PyYAML` не объявлен (только транзитивно) | ✅ |
| ISSUE-029 | 🟡 | Упаковка | CI отсутствует | ✅ |
| ISSUE-030 | 🟡 | Тесты | Покрытие 49%; `orchestrator.py` 23%; 0% на сервере, WS и CLI | ✅ |
| ISSUE-031 | 🟡 | Продукт | UCIP v2 / Module Mode: замысел без реализации | ✅ |
| ISSUE-032 | 🟡 | Продукт | Бенчмарки фабрикуют базовую линию | ⚠️ |
| ISSUE-033 | 🔵 | API | `PATCH /settings` → 500 при отсутствии конфига (`Path` не импортирован) | ✅ |
| ISSUE-034 | 🔵 | API | XSS через имя чата; `javascript:` в markdown | ⚠️ |
| ISSUE-035 | 🔵 | API | `deleteFact()` удаляет только из локального массива | ⚠️ |
| ISSUE-036 | 🔵 | API | `turn: null` в WebSocket (нет атрибута `turn_count`) | ⚠️ |
| ISSUE-037 | 🔵 | Ядро | `summarizer.py` мёртв, коллабораторы отсутствуют | ✅ |
| ISSUE-038 | 🔵 | Упаковка | `uvloop` объявлен без импорта; `prompts/*` в `package-data` без каталога | ✅ |
| ISSUE-039 | 🔵 | Ядро | `num_ctx`/`num_gpu` шлются в поле `options` на OpenAI-эндпоинт | ⚠️ |
| ISSUE-040 | 📄 | Документация | Ложные утверждения в README и архивированных документах | ✅ |

---

## 🔴 CRITICAL

### ISSUE-001 — `DELETE /api/v1/sessions/..` уничтожает весь `storage/`

✅ Проверено дважды, в изолированных каталогах.

```
DELETE /api/v1/sessions/%2E%2E   →  HTTP 200  {"status":"deleted","session_id":".."}
canary после запроса: DESTROYED — storage/ уничтожен
```

**Причина.** `session_id` берётся из URL без валидации (`api/session.py:62-85`),
`session_exists()` — это просто `is_dir()` (`core/session_manager.py:372-374`), а
`delete_session()` делает `shutil.rmtree(self._base_dir / session_id)`
(`core/session_manager.py:302-307`). Для `session_id = ".."` путь равен `storage/`.

**Достижимость.** Сырой `..` парсер URL схлопывает, но `%2E%2E` — нет, а uvicorn
декодирует процент-энкодинг до маршрутизации. Проверено против живого сервера через
`curl --path-as-is`: 200 и уничтожение `storage/`.

**Усугубляет:**
- `server.py:27-35` — `allow_origins=["*"]`, `DELETE` в разрешённых методах;
- аутентификации нет вообще (в `openapi.json` нет `securitySchemes`);
- `__main__.py:110` и `config.yaml:47` — сервер слушает `0.0.0.0`.

**Правка.** Валидировать `session_id` (`^[A-Za-z0-9_-]{1,64}$`) и проверять
`resolved.is_relative_to(self._base_dir)` в `delete_session`, `session_exists`,
`switch_to`, `rename_session`. Сузить CORS.

**Затрагивает прокси:** да — прокси стоит между инструментами и моделью.

---

### ISSUE-002 — Не-ASCII текст уничтожается при сохранении памяти

✅ Проверено экспериментом.

| Место | Запись | Чтение |
|---|---|---|
| `core/memory/working_memory.py` | L249 без `encoding` | L264 `utf-8, errors='replace'` |
| `core/memory/storage.py` | L551 без `encoding` | L560 `utf-8, errors='replace'` |

Дефолтная кодировка хоста — `cp1251`. Файл содержит `b'\xff\xea\xee\xf0\xfc'` (cp1251
для «якорь»), читается как utf-8 → `'�����'`, все символы заменены на U+FFFD.

**Цепочка.** Испорченный U+FFFD попадает в память → следующее сохранение падает на
`UnicodeEncodeError: 'charmap' codec can't encode character '\ufffd'` → исключение
глотается (`core/session.py:112`) → **сохранение молча не происходит**. Так как
`session_meta.json` пишется последним (`core/session.py:101`), теряются и `storage.json`,
и `chat_history.json`. Перезагрузка даёт 0 фактов вместо 2.

**Происхождение.** `032f0ff` перевёл чтение на utf-8, не тронув запись; `ce7ec97`
добавил `errors='replace'` — падение стало тихой порчей.

**Правка.** `encoding="utf-8"` в оба `write_text`, убрать `errors='replace'` из чтения.

**Затрагивает прокси:** да — вся память не-ASCII теряется.

---

### ISSUE-003 — Клиентское `system`-сообщение отключает всю инжекцию памяти

✅ Проверено чтением.

```python
# core/orchestrator.py:444
system_prompt = system or self._build_system_prompt(intent, context_cards, graph_entities)
```
```python
# core/orchestrator.py:736-739 — та же логика повторно
if system_override: system = system_override
else:               system = self._build_system_prompt(...)
```

Если клиент прислал `system`, `_build_system_prompt` не вызывается: в промпт не попадают
ни координаты сессии (`:705-708`), ни горячие факты (`:711`), ни карточки кода, ни граф.
Память при этом обновляется. Все OpenAI-совместимые клиенты `system` присылают.

**Следствие.** Для сценария внешнего клиента память отключена по построению.

**Правка.** Не давать клиентскому `system` подменять сборку контекста: объединять, а не
заменять.

**Затрагивает прокси:** да, критично.

---

## 🟠 HIGH

### ISSUE-004 — CCI не может сработать → подкачка фактов из WARM недостижима

✅ Проверено запуском.

```python
# core/memory/cci.py:207-210
if final_score < self.threshold and self._history:
    prev_score = self._history[-1].coherence_score
    if prev_score >= self.threshold:
        final_score = self.threshold + 0.01
```

Первый ход всегда даёт `1.0` (`cci.py:160-169`) и сохраняется, поэтому все последующие
принудительно поднимаются до `threshold + 0.01`. Замер на пяти намеренно несвязанных
темах: все `coherent=True`, счёт зажат в `0.16`.

**Следствие.** `needs_context_restore()` всегда `False`, а это единственное условие вызова
`MemoryStorage.retrieve()` (`orchestrator.py:351`). Факты из долговременного хранилища
никогда не подкачиваются.

**Важно:** цепочка координат (координатор → `MetaCoordinator` → промпт генератора)
работает независимо и **не** затронута. Сломан именно контур фактов.

**Правка.** Удалить блок dampening.

---

### ISSUE-005 — `num_ctx` не управляет сжатием

✅ Проверено чтением.

```python
# core/orchestrator.py:307
if len(self._chat_history) > self._context_window_size:   # 12 сообщений, захардкожено
```

Решение о сбросе основано на **количестве сообщений**, а не на токенах. `num_ctx`
применяется только как опция окна движка (`dual_model.py:199`, `utility_worker.py:74-85`),
а на прокси-пути OpenAI захардкожен обратно в `4096` (`api/system.py:388`).

**Следствие.** 12 коротких реплик вызывают сброс, израсходовав ~4% буфера на 16000;
12 сообщений с кодом переполняют окно до срабатывания триггера. Пользователь задаёт
буфер, который ничем не управляет.

**Правка.** Считать заполнение в токенах и сравнивать с `num_ctx`.

---

### ISSUE-006 — Слой графа падает каждый ход

✅ Проверено запуском.

```python
# core/orchestrator.py:401,607
results = self.graph_builder.search_nodes(entity, limit=3)
```
```
hasattr(GraphBuilder, 'search_nodes')  →  False
hasattr(GraphBuilder, 'search')        →  True
```

`AttributeError` глотается на `:404-405` (warning) и `:609-610` (голый `pass`). Граф
никогда не строился: `build_from_directory` не имеет вызовов, `storage/graph.json`
отсутствует.

**Правка.** `search_nodes` → `search`.

---

### ISSUE-007 — Anchor-факты вытесняются вопреки гарантии

✅ Проверено чтением.

`cleanup()` защищает якоря (`working_memory.py:155`), но `_evict_to_budget()`
(`:380-389`) берёт `min(self._facts, key=lambda f: f.attention_weight)` **без проверки
`is_anchor`**. Комментарий на `:154` утверждает «Anchor facts НИКОГДА не evict».

**Правка.** Пропускать `is_anchor` в `_evict_to_budget()`.

---

### ISSUE-008 — Сжатие заменяет содержимое факта служебным тегом

✅ Проверено чтением.

```python
# core/memory/storage.py:479
fact.content = fact.source if fact.source else fact.content[:50]
```

Для фактов из tagger источник — `tagger_turn_N` (`orchestrator.py:530,538`). Решение с
обоснованием превращается в строку `tagger_turn_7`. Дополнительно `SUMMARIZED`
(`:472-473`) режет текст наивным `split('.')` по первым двум предложениям.

---

### ISSUE-009 — Обрыв вытеснения на 30 фактах

✅ Проверено чтением.

```python
# core/memory/working_memory.py:161-163
if len(self._facts) < 30:
    kept.append(fact); continue
```

До 30 фактов не вытесняется ничего, после — сбрасывается пачкой. Причина падения
`test_memory.py::test_cold_facts_evicted_to_storage`. `COLD_THRESHOLD` в этом пути мёртв.

---

### ISSUE-010 — Факты протекают между сессиями

⚠️ Сообщено двумя независимыми аудитами; лично не перепроверял.

`switch_session` чистит `WorkingMemory`, но не `memory_storage` (`orchestrator.py:809-814`);
`working_memory.clear()` выталкивает старые факты в storage (`working_memory.py:231-236`),
а `MemoryStorage._load()` **сливает**, а не заменяет (`storage.py:561-562`). Факты сессии A
оказываются в файле сессии B.

---

### ISSUE-011 — Отказ бэкенда выглядит как успешный пустой ответ

✅ Проверено чтением и запросом.

```python
# core/dual_model.py:288-290
except Exception as e:
    logger.error(f"[generator] Call failed: {e}")
    return "", 0, 0
```

`run()` добавляет пустое assistant-сообщение (`orchestrator.py:469`) и продолжает.
У `OrchestrationResult` (`:22-36`) нет поля ошибки, а `coherence_score` по умолчанию
`1.0` — провальный ход рапортует о полной связности.

**Проверено запросом:** `POST /v1/chat/completions` без доступной модели →
`HTTP 200 {"content":"","completion_tokens":0}`.

---

### ISSUE-012 — `stream: true` не работает нигде

✅ Проверено чтением.

| Место | Что |
|---|---|
| `api/system.py:322-341` | SSE-заглушка, в докстроке прямо `Fake SSE streaming` — режет готовый ответ на слова |
| `api/system.py:388` | на прокси-пути захардкожено `"stream": False` |
| `api/websocket.py:217-226` | WebSocket имитирует стрим, разбивая готовый ответ по пробелам с `asyncio.sleep(0)` |
| `core/orchestrator.py:580` | `run_stream()` — ноль вызовов во всём дереве |

Время до первого токена равно времени полной генерации.

---

### ISSUE-013 — RAG: карточки рендерятся как «Unknown»

✅ Проверено чтением обеих сторон.

```python
# core/orchestrator.py:679-684 — что читает код
entity = getattr(card, 'entity', None)
name = getattr(entity, 'name', 'Unknown') or 'Unknown'
fpath = getattr(entity, 'file_path', 'Unknown') or 'Unknown'
```
```python
# core/retriever.py:16-26 — что реально есть в RetrievalResult
card_id, entity_name, entity_type, file_path,
start_line, end_line, summary, distance, relevance_score
```

Атрибута `entity` не существует, поэтому `entity = None` и всё превращается в
`'Unknown'`. Так как `getattr` с дефолтом не бросает исключение, `except Exception:
continue` на `:692` **не срабатывает** — поломка беззвучна.

---

### ISSUE-014 — RAG-индекс никогда не наполняется

⚠️ Сообщено аудитом.

`CardGenerator.index_directory` / `index_file` (`card_generator.py:137,91`) не имеют
вызовов; `self.card_generator` создаётся (`orchestrator.py:70`) и больше не используется.
Коллекция `code_cards` в ChromaDB пуста.

---

### ISSUE-015 — Провайдер эмбеддингов залипает навсегда

⚠️ Сообщено аудитом.

`storage.py:232-233` кэширует провайдера и не перепроверяет. Одна транзиентная ошибка
Ollama при старте (через catch-all на `storage.py:169-171`) переключает процесс на BM25
до перезапуска. Семантический поиск молча выключен, видно только в
`stats()["embedding_provider"]`.

---

### ISSUE-016 — Репозиторный `config.yaml` игнорируется

✅ Проверено.

`_find_config_yaml()` (`engines/config_loader.py:118-153`) ищет в шести местах, и AppData
имеет приоритет над текущим каталогом и корнем проекта. На проверенной машине эффективен
`%APPDATA%\Contextor\config.yaml` размером **72 байта**, а репозиторный `config.yaml`
(53 строки) **не читается вообще**. `PATCH /settings` пишет туда же.

**Следствие.** Пользователь правит один файл, система читает другой.

---

### ISSUE-017 — Секция `adaptive_reset` не парсится; 4 ключа не используются

✅ Проверено.

| Ключ | Состояние |
|---|---|
| `memory.adaptive_reset.*` | **не парсится**: в `MemoryConfig` таких полей нет, оркестратор хардкодит 0.55 / 4 / 16 (`orchestrator.py:92-94`) |
| `memory.context_window_messages` | парсится, но не используется (захардкожено `_context_window_size = 12`) |
| `memory.keep_after_reset` | парсится, но не используется (захардкожено `[-6:]`, `orchestrator.py:234`) |
| `memory.working_memory_tokens` | парсится, но не используется (захардкожено `token_budget=1500`, `:75`) |
| `memory.max_storage_facts` | парсится, но не используется |
| `memory.meta_coordinate_every` | **парсится и используется** (`orchestrator.py:118,831`) |
| `memory.num_ctx` | **парсится и используется** (`dual_model.py:98,168`) |
| `models.utility`, `server:*`, `ollama:*` | не читаются (последнее маскирует `host.docker.internal` в `config.yaml:52`) |

---

### ISSUE-018 — `psutil` не объявлен и отсутствует

✅ Проверено живым выводом.

`utils/hardware_detector.py:16` импортирует `psutil` под `try/except ImportError`
(`HAS_PSUTIL`). В `pyproject.toml` он **не объявлен**. Живой результат на машине с
RTX 3060: `cpu_cores: 0`, `ram_gb: 0.0`. На Windows фолбэка нет (`/proc/meminfo` только
Linux). На машине без GPU рекомендация всегда уйдёт в ветку «CPU MINIMAL».

Дополнительно: `hardware_detector.py:102` считает `ollama_available = shutil.which("ollama")
is not None` — только наличие бинарника, не доступность сервера. При недоступном сервере
показал `true`.

---

### ISSUE-019 — Установщики не работают

✅ Проверено запросом.

```
HTTP 404  https://raw.githubusercontent.com/Remchik64/Contextor-pro/main/start.bat
OK   200  https://raw.githubusercontent.com/Remchik64/Contextor/main/start.bat
```

Репозиторий переименован, редиректа на старое имя **нет**. Установщики бьют именно туда:
`install.bat:86,113`, `install.sh:149,241`.

Плюс `install.bat:113` использует `curl -s -L` **без `-f`**: curl возвращает 0 даже при
HTTP 404, поэтому guard `if errorlevel 1` на `:114` не срабатывает, резервная ветка
(`:116-121`) мертва, и тело 404 записывается как лаунчер. Дополнительно `install.bat:138`
и `install.sh:241` предлагают модель `qwen2.5:3b`, которой нет в конфиге.

---

### ISSUE-020 — 25 падений тестов; `test_system_full.py` вешает сбор

✅ Проверено прогоном: **392 собрано, 367 passed, 25 failed** (89 с).

| Причина | Кол-во | Вердикт |
|---|---:|---|
| Имена моделей (`qwen2.5` → `qwen3.5`) | 11 | тесты устарели |
| Ключ реестра `qwen3.5-2b` при Qwen2.5-3B внутри (`engine/registry.py:4-8`) | 1 | устаревший тест + **реальная несогласованность данных** |
| Устаревший импорт из `api.routes` (схемы в `api.system:309-319`) | 6 | тесты устарели, код исправен |
| Кодировка (ISSUE-002) | 4 | **реальный баг** |
| Испорченный meta-файл (тест пишет без `encoding`) | 1 | баг теста + глушение на `session_manager.py:111` |
| Обрыв на 30 фактах (ISSUE-009) | 1 | **реальный баг** |
| Тест требует неустановленный `torch` | 1 | упаковка |

`tests/test_system_full.py` — не тест, а скрипт с ~20 живыми HTTP-запросами **на импорте**
(нет guard'а `__main__`), поэтому `pytest tests/` из `README.md:285` **зависает**.
Бьёт в `/api/v1/orchestrate`, которого не существует (404).

---

## 🟡 MEDIUM

### ISSUE-021 — Координаты не сцепляются

✅ `add_anchor` вызывается только на `orchestrator.py:528` (ветка tagger), но **не** в
`_soft_reset()`, хотя докстрока `:201-205` обещает «Сохраняет координату как anchor fact».
Поэтому вход «ПРЕДЫДУЩИЕ КООРДИНАТЫ» (`:152-162`) наполняется WM-якорями, а не
координатами. Функцию сцепки частично выполняет `MetaCoordinator.consolidate()`.

### ISSUE-022 — Бюджет считается токенизатором GPT-4

✅ `utils/tokenizer.py:6` использует `cl100k_base`. Для Qwen/Llama на русском и на коде
расхождение существенное. Для механизма, чья суть в бюджете, это означает переполнение
окна при `fill`, близком к порогу. Готовая, но неиспользуемая альтернатива —
`fit_messages_budget()` (`utils/tokenizer.py:22`, ноль вызовов).

### ISSUE-023 — `model_used` сообщает модель, которая не работала

✅ `_select_model()` возвращает ключи реестра (`orchestrator.py:651,653`), а `_generate()`
принимает `model_key` и **не передаёт его в роутер** (`:753-764`): в
`self._router.generate()` уходят только `messages`, `temperature`, `max_tokens`.

### ISSUE-024 — Синхронный пайплайн внутри `async def`

⚠️ `api/system.py:412` вызывает синхронный `pipe.run(...)` внутри `async def`, блокируя
цикл событий на всё время генерации. Блокировка есть только на создании (`state.py:50-58`),
поэтому параллельные ходы гоняются за `_chat_history` / `_turn` / WM.

### ISSUE-025 — Панель Settings молча выбрасывает значения

✅ `app.js:1178-1200` отправляет 7 значений (`coordinator_model`, `generator_model`,
`cci_threshold`, `max_turns_without_reset`, `min_turns_between_resets`, `hot_facts_max`)
в `POST /api/v1/config/reload`, **хендлер которого не принимает тело**
(`api/system.py:147-148`). Значения отбрасываются, интерфейс рапортует успех. Реально
работает только `num_ctx` (через `PATCH /settings`). `loadSettings()` (`:1153-1156`)
читает ключи, которых сервер не возвращает, поэтому поля всегда показывают захардкоженные
`0.7 / 20 / 5` вместо `0.55 / 16 / 4`.

### ISSUE-026 — `session_meta.json` затирается каждый ход

✅ Файл пишут два класса (`core/session.py:63` и `core/session_manager.py:94`), причём
`save()` перезаписывает его пятью ключами **без `display_name`**. Метод авто-именования
`_auto_name_session_if_first` (`orchestrator.py:919`) не вызывается никогда.

### ISSUE-027 — Версии расходятся

✅ `pyproject.toml:7` и `server.py:23` — `0.3.0`; `/api/v1/health` (`api/system.py:26`) и
`contextor --version` (`__main__.py:12`) — `0.1.0`; установщики — `v0.1`.
`src/contextor/__init__.py` пуст, `__version__` отсутствует.

### ISSUE-028 — `PyYAML` не объявлен

✅ Импортируется в `engines/config_loader.py:11`, `api/system.py:95`,
`core/memory/storage.py:58` — под `try/except ImportError`. Приходит только транзитивно
через `chromadb`. При изменении зависимости `config.yaml` будет молча игнорироваться
целиком.

### ISSUE-029 — CI отсутствует

✅ `.github/` содержит только `FUNDING.yml`. Ни один тест не запускается автоматически,
`ruff` и `mypy` объявлены, но не исполняются.

### ISSUE-030 — Покрытие 49%

✅ `core/orchestrator.py` — 23% (418 инструкций). **0%**: `server.py`, `api/websocket.py`,
`core/summarizer.py`, `core/utility_worker.py`, `utils/swap_manager.py`, `__main__.py`.
Фикстура `mock_pipeline` в `test_orchestrator.py` не используется ни одним тестом, и ни
один тест не вызывает `pipeline.run()`.

### ISSUE-031 — UCIP v2 / Module Mode: замысел без реализации

✅ Grep по `src/` не находит ни `UCIP`, ни `Context Package`, ни 4-слойной структуры.
Существует один слой `[RELEVANT CODE CONTEXT]` (`retriever.py:187`). Проработка — в
`docs/proxy/`.

### ISSUE-032 — Бенчмарки фабрикуют базовую линию

⚠️ `benchmarks/runner.py:113-134`: `latency_ms` замеряет время двух сравнений, а
`context_tokens=0`, `keyword_recall=0.0`, `memory_facts_count=0` — захардкоженные
литералы. Модель не вызывается. Ни одно из «85% fewer tokens / 100% recall / 5ms per fact»
этим кодом не проверяется. ✅ Дополнительно проверено: `benchmarks/` не подключён ни к
`src/`, ни к `tests/`, не входит в дистрибутив.

---

## 🔵 LOW

| ID | Проблема | Достоверность |
|---|---|---|
| ISSUE-033 | `api/system.py:112` использует `Path("config.yaml")`, но `pathlib.Path` в модуле не импортирован → `PATCH /settings` даёт 500, если конфиг не найден | ✅ |
| ISSUE-034 | XSS через имя чата: `app.js:101` подставляет имя в inline-JS внутри HTML-атрибута, а `esc()` даёт `&#39;`, который HTML-парсер декодирует обратно до парсинга JS. `renderMarkdown` (`:376`) пропускает `javascript:`-ссылки | ⚠️ |
| ISSUE-035 | `deleteFact()` (`app.js:766-772`) удаляет только из локального массива и рапортует успех | ⚠️ |
| ISSUE-036 | `api/websocket.py:233` читает `getattr(pipeline_obj.session, 'turn_count', None)` — такого атрибута нет, клиент получает `turn: null` | ⚠️ |
| ISSUE-037 | `core/summarizer.py` не импортируется нигде, а его коллабораторы (`archive.get_pairs`, `set_summary`, `trim_pairs`) в репозитории отсутствуют (`core/archive.py` удалён в `5aa4777`) | ✅ |
| ISSUE-038 | `uvloop` объявлен (`pyproject.toml:31`), но не импортируется; глобы `prompts/*` в `package-data` (`:70-71`) указывают на несуществующий `src/contextor/prompts/` | ✅ |
| ISSUE-039 | `dual_model.py:198-201` шлёт `num_ctx` и `num_gpu` в поле `options` на OpenAI-совместимый `/v1/chat/completions`. Нативная опция Ollama на этом эндпоинте может игнорироваться | ⚠️ |

---

## 📄 ISSUE-040 — Ложные утверждения в документации

✅ Проверено против кода.

| Утверждение | Источник | Факт |
|---|---|---|
| «85% fewer tokens», «100% recall», «5ms/fact» | `README.md:3,146-152` | Не измеряются ничем (`ISSUE-032`) |
| «370+ tests» | `README.md:206` | 392 собрано, 25 падают |
| «465 тестов проходят стабильно» | `architecture.md:276` (архив) | 367 / 25 |
| WARM = «ChromaDB + SentenceTransformer» | `architecture.md`, README | JSON-файл + опциональный Ollama-эмбеддер; ChromaDB только для карточек кода |
| «Anchor — отдельный слой» | `README.md:23-27` | Булев флаг `Fact.is_anchor` в том же HOT-списке |
| «CCI: cosine similarity» | `architecture.md:58-61` (архив) | BM25 по ключевым словам |
| Dashboard с метриками GPU/CCI | `architecture.md:114` (архив) | Пять вкладок, Dashboard отсутствует |
| `GET /memory/search`, `POST /memory/fact` | `api_reference.md:123,153` | 404 |
| `WS /ws/chat`, тип `done` | `api_reference.md:213-239` | Реально `/ws` и `end` |
| CLI `contextor config --show-path` | `installation.md:126` | Такой команды нет |
| `core/archive.py` как COLD | `architecture.md:224` (архив) | Файла нет |
| `docs/ROADMAP.md` | `HANDOFF.md:180` | Файла нет |
| «~12K строк, ~6K тестов» | `HANDOFF.md:19` | 9748 и 5037 |

**Статус.** Документы, содержавшие ложные утверждения, перемещены в `docs/archive/` с
предупреждающими шапками. `architecture.md`, `api_reference.md`, `installation.md`
переписаны заново по коду. Корневой `README.md` исправлен.

---

## Порядок работ

Порядок определяется не количеством, а тем, что разблокирует остальное.

1. **ISSUE-001** — безопасность. Пока это не закрыто, любой результат можно потерять
   внешним запросом.
2. **ISSUE-002** — память не выживает перезапуск. Закрывает 4 падения тестов.
3. **ISSUE-003** — без этого память не работает для внешних клиентов вообще.
4. **ISSUE-006** (одна строка) → **ISSUE-004** (одна строка) — оживляют граф и контур
   фактов.
5. **ISSUE-005** — превращает `num_ctx` в работающий механизм; ядро продуктовой идеи.
6. **ISSUE-016**, **ISSUE-017** — чтобы конфигурация вообще что-то значила.
7. **ISSUE-018**, **ISSUE-019** — чтобы проект можно было установить.
8. **ISSUE-020** — починить 18 устаревших тестов (это рассинхрон, а не поломка).

Далее — по `docs/proxy/05-roadmap.md`.

---

## Удалённые утверждения (не воспроизводить)

Пять утверждений из первичных отчётов аудита при перепроверке **не подтвердились**.
Приведены здесь, чтобы они не вернулись в документацию:

| Утверждение | Что на самом деле |
|---|---|
| «`_select_model()` отбрасывается» | Передаётся в `_generate`, но `_generate` не форвардит его в роутер (ISSUE-023) |
| «UI зовёт несуществующий `/api/v1/version`» | Существует: `server.py:71`, скрыт через `include_in_schema=False` |
| «В `app.js:627` испорченная кириллица» | Корректный UTF-8: `toast('WebSocket не подключён', 'error')` |
| «19 падений из-за имён моделей» | Фактически 12 (11 имён + 1 ключ реестра) |
| «`%2E%2E` безопасен (404)» | 404 только при подаче пути в ASGI напрямую. Против живого uvicorn — 200 и удаление `storage/` |
