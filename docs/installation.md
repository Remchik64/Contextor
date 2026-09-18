# Установка — Contextor

> **Переписано заново 2026-09-18**, HEAD `dd36774`. Предыдущая версия ссылалась на URL
> репозитория, который переименован и отдаёт **404** (проверено запросом), и упоминала
> команду CLI, которой не существует. Архивирована.
>
> **Проблемы:** `docs/known-issues.md`

---

## Требования

| Компонент | Минимум | Рекомендуется |
|---|---|---|
| Python | 3.11+ (`pyproject.toml:11`) | 3.13 |
| RAM | 8 GB | 16 GB |
| VRAM | 0 (режим CPU) | 8+ GB (режим GPU) |
| Место на диске | 5 GB | 20+ GB под модели |
| ОС | Windows 10 / Ubuntu 20.04 / macOS 12+ | — |

**Обязательно:** [Ollama](https://ollama.com) — внешний runtime для моделей. Без него
сервер запустится, но не сможет генерировать ответы: деградация корректная (эмбеддинги
уходят в BM25-фallback), но ответы будут пустыми.

---

## ⚠️ Сначала прочитайте: установщики сейчас не работают

Проверено запросом к GitHub:

```
HTTP 404  https://raw.githubusercontent.com/Remchik64/Contextor-pro/main/start.bat
OK   200  https://raw.githubusercontent.com/Remchik64/Contextor/main/start.bat
```

Репозиторий переименован из `Contextor-pro` в `Contextor`, и **редиректа на старое имя
нет**. Все ссылки в `install.bat` и `install.sh` ведут на несуществующий URL.

Дополнительно в `install.bat:113` используется `curl -s -L` **без флага `-f`**: curl
возвращает код 0 даже при HTTP 404, поэтому проверка `if errorlevel 1` на `:114` не
срабатывает, резервная ветка (`:116-121`) мертва, и **тело 404-ответа записывается как
лаунчер**.

**Вывод:** используйте ручную установку (способ 2). Проблема — `ISSUE-019`.

---

## Способ 1 — Установщик (после исправления URL)

Если ссылки в скриптах будут исправлены на `Remchik64/Contextor`:

**Windows:**

```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/Remchik64/Contextor/main/install.bat -OutFile install.bat
.\install.bat
```

**Linux / macOS:**

```bash
curl -fsSL https://raw.githubusercontent.com/Remchik64/Contextor/main/install.sh | bash
```

Перед запуском проверьте, что в скачанном файле нет `Contextor-pro`:

```bash
grep -n "Contextor-pro" install.bat install.sh   # должно быть пусто
```

Обратите внимание: `install.bat:138` и `install.sh:241` предлагают скачать модель
`qwen2.5:3b`, которой **нет в `config.yaml`** (там `qwen3.5:2b` и `qwen3.5:9b`).

---

## Способ 2 — Ручная установка (рекомендуется)

### Шаг 1. Ollama

```bash
# Linux
curl -fsSL https://ollama.com/install.sh | sh
# Windows / macOS — установщик с https://ollama.com/download
```

Проверка:

```bash
ollama --version
ollama serve          # если служба не запущена автоматически
```

> Наличие клиента `ollama` **не означает**, что сервер работает. Встроенная проверка
> `hardware_detector.py:102` считает `ollama_available = shutil.which("ollama") is not None`
> — только наличие бинарника. Она может показать `true` при недоступном сервере
> (`ISSUE-018`).

### Шаг 2. Код

```bash
git clone https://github.com/Remchik64/Contextor.git
cd Contextor
```

### Шаг 3. Виртуальное окружение

```bash
python -m venv .venv
# Windows
.\.venv\Scripts\activate
# Linux / macOS
source .venv/bin/activate
```

### Шаг 4. Зависимости

```bash
pip install -e .
```

> **Может не установиться сразу.** Известные проблемы:
>
> - **`psutil` не объявлен** в `pyproject.toml`, хотя импортируется в
>   `utils/hardware_detector.py:16`. Без него `cpu_cores` и `ram_gb` будут `0` на
>   Windows (`ISSUE-018`). Установите вручную: `pip install psutil`.
> - **`PyYAML` не объявлен**, приходит только транзитивно через `chromadb`
>   (`ISSUE-028`). Установите вручную: `pip install pyyaml`.
> - **`requests` нужен для `tests/test_system_full.py`**, но не объявлен.
> - `sentence_transformers` и `torch` **не объявлены и не нужны**: они импортируются
>   лениво внутри функций (`storage.py:96-97`), с откатом на Ollama. Без них
>   семантические эмбеддинги недоступны, поиск идёт через BM25 — это штатное поведение.
> - `llama-cpp-python` объявлен только как extra `cuda` и нужен лишь для CLI-команды
>   `model`, а не для работы сервера.

### Шаг 5. Модели

Имена берутся из `config.yaml`:

```bash
ollama pull qwen3.5:2b          # координатор
ollama pull qwen3.5:9b          # генератор и utility
ollama pull nomic-embed-text    # эмбеддер
```

> `VERIFY`: доступность конкретных тегов в вашей версии Ollama проверяется через
> `ollama list`. Если `qwen3.5:*` недоступны, используйте любые другие и укажите их в
> конфиге — имена нигде не зашиты жёстко, кроме `ImportanceTagger`
> (`memory/tagger.py:27`, там `qwen3.5:2b` захардкожен).

### Шаг 6. Запуск

```bash
python -m contextor serve --port 7860
# или, если консольный скрипт в PATH:
contextor serve --port 7860
```

CLI имеет **ровно две команды** (`__main__.py`): `serve` и `model`. Команд `config`,
`doctor` и подобных нет — их упоминание в прежней документации было ошибкой.

Проверка:

```bash
curl http://127.0.0.1:7860/api/v1/health
# {"status":"healthy","model_loaded":false,"version":"0.1.0"}
```

Web UI: `http://localhost:7860`

---

## Конфигурация — важная особенность

Файл `config.yaml` **в корне проекта может не читаться вообще.** Порядок поиска
(`engines/config_loader.py:118-153`):

1. переменная окружения `PURE_INTELLECT_CONFIG`
2. **`%APPDATA%\Contextor\config.yaml`** (Windows)
3. `~/.config/contextor/config.yaml` (Linux/macOS)
4. текущий рабочий каталог
5. корень проекта
6. `/etc/contextor/config.yaml`

Первое найденное значение выигрывает. На проверенной машине эффективен файл в AppData
размером **72 байта**, а репозиторный `config.yaml` не читается. `PATCH /api/v1/settings`
пишет именно в найденный файл.

**Как узнать, какой файл реально используется:**

```bash
curl http://127.0.0.1:7860/api/v1/config
# поле config_file содержит абсолютный путь
```

**Как заставить использовать нужный файл** — переменной окружения, у неё высший приоритет:

```bash
# Windows PowerShell
$env:PURE_INTELLECT_CONFIG = "$PWD\config.yaml"
# Linux / macOS
export PURE_INTELLECT_CONFIG="$PWD/config.yaml"
```

Имя переменной осталось от прежнего названия проекта «Чистый Интеллект» — это не ошибка
в документации, так в коде.

### Какие ключи действительно работают

| Ключ | Работает |
|---|---|
| `models.coordinator.*`, `models.generator.*` | ✅ |
| `memory.num_ctx` | ✅ |
| `memory.meta_coordinate_every` | ✅ |
| `memory.adaptive_reset.*` | ❌ не парсится вообще (`ISSUE-017`) |
| `memory.context_window_messages`, `keep_after_reset`, `working_memory_tokens`, `max_storage_facts` | ❌ парсятся, но в логике не используются |
| `models.utility.*`, `server.*`, `ollama.*` | ❌ не читаются |

Практическое следствие: изменить порог CCI или жёсткий лимит ходов через конфиг
**невозможно** — они захардкожены в `orchestrator.py:92-94`. То же с `server.port`:
он берётся из аргумента CLI, а не из конфига.

---

## Проверка установки

```bash
# 1. Сервер поднимается и отдаёт 27 маршрутов
curl http://127.0.0.1:7860/openapi.json | python -c "import json,sys; print(len(json.load(sys.stdin)['paths']), 'routes')"

# 2. Аппаратное определение
curl http://127.0.0.1:7860/api/v1/hardware/detect
# ожидайте gpu/vram_gb; cpu_cores и ram_gb будут 0 без psutil

# 3. Модели Ollama видны
curl http://127.0.0.1:7860/api/v1/ollama/models

# 4. Тесты (без интеграционного скрипта — он вешает сбор)
pip install pytest pytest-asyncio
python -m pytest tests/ -q --ignore=tests/test_system_full.py
# ожидание: 367 passed, 25 failed — это известное состояние, см. ISSUE-020
```

`tests/test_system_full.py` **не запускайте через pytest**: это не тест, а скрипт с ~20
живыми HTTP-запросами на импорте и без guard'а `__main__`. Из-за него `pytest tests/`
зависает. Отдельно от pytest он тоже непригоден: бьёт в `/api/v1/orchestrate`, которого
не существует (404).

---

## Известные проблемы установки

| Проблема | Обходной путь |
|---|---|
| Установщики бьют в 404 (`Contextor-pro`) | Ручная установка |
| `curl -f` отсутствует → тело 404 как лаунчер | Использовать `start.bat` из репозитория вручную |
| `psutil` не объявлен | `pip install psutil` |
| `PyYAML` не объявлен | `pip install pyyaml` |
| `cpu_cores: 0`, `ram_gb: 0` на Windows | Следствие отсутствия `psutil` |
| Репозиторный `config.yaml` игнорируется | Задать `PURE_INTELLECT_CONFIG` |
| `adaptive_reset` в конфиге ничего не меняет | Значения захардкожены в `orchestrator.py:92-94` |
| Русский текст в памяти портится при перезапуске | Нет обходного пути — `ISSUE-002` |
| `pytest tests/` зависает | `--ignore=tests/test_system_full.py` |
| `/api/v1/chat` даёт 500 про `llama_cpp` | Не использовать; для чата идти через WebSocket или `/v1/chat/completions` |

---

## Удаление

```bash
pip uninstall contextor
rm -rf storage/                    # память и сессии
rm -rf ~/.config/contextor/        # Linux/macOS
# Windows: %APPDATA%\Contextor\
ollama rm qwen3.5:2b qwen3.5:9b nomic-embed-text
```

> **Осторожно:** не выполняйте `DELETE /api/v1/sessions/..` в попытке удалить одну
> сессию — этот запрос уничтожает весь каталог `storage/` (`ISSUE-001`).

---

## Обвязка `.dsh-dev/` (только для sandbox DSH)

Если проект запускается внутри ограниченного sandbox DSH, используйте
`.\.dsh-dev\dev.ps1` вместо прямого вызова `python`. Sandbox отклоняет доступ к каталогам
с ACL `0o700` — именно так `tempfile.mkdtemp()` создаёт временные папки, что ломает
`ensurepip`, `pip` и pytest `tmp_path`. Дополнительно системный `TEMP` закрыт на запись.
Скрипт задаёт `PYTHONPATH` и перенаправляет `TEMP` внутрь проекта.

Вне sandbox каталог не нужен и исключён через `.git/info/exclude`.
