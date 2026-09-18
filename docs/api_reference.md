# API Reference — Contextor

> **Переписано заново 2026-09-18** по фактически зарегистрированным маршрутам
> (`app.openapi()` + `app.routes`), HEAD `dd36774`. Предыдущая версия описывала четыре
> эндпоинта, отдающих 404, и WebSocket по неверному пути. Архивирована.
>
> **Проблемы:** `docs/known-issues.md` · **Архитектура:** `docs/architecture.md`

---

## Базовый URL

```
http://localhost:7860        # по умолчанию
http://0.0.0.0:7860          # на что реально слушает сервер (__main__.py:110)
```

`config.yaml` содержит `server.host: 0.0.0.0` и `server.port: 7860`, но эти ключи
**не читаются** — `serve` берёт значения из аргументов CLI (`__main__.py:110-111`), а
дефолты заданы pydantic-settings (`config.py`). См. `ISSUE-017`.

Интерактивная документация доступна на `/docs` (Swagger) и `/redoc`.

**Аутентификации нет.** Все эндпоинты открыты, CORS настроен на `allow_origins=["*"]`
(`server.py:27-35`). Это создаёт критическую уязвимость — `ISSUE-001`.

---

## Полный список маршрутов

Проверено через `app.openapi()`. Всего 27 маршрутов в схеме, плюс два скрытых.

### Система

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/health` | Статус: `{"status":"healthy","model_loaded":false,"version":"0.1.0"}` |
| `GET` | `/api/v1/version` | Версия и путь к статике. **Скрыт** из OpenAPI (`server.py:71`) |
| `GET` | `/` | Web UI (`static/index.html`). **Скрыт** (`server.py:62`) |
| `GET` | `/api/v1/config` | Эффективный конфиг + абсолютный путь к файлу |
| `POST` | `/api/v1/config/reload` | Перечитать конфиг. **Тело не принимается** (`ISSUE-025`) |
| `GET` | `/api/v1/config/hardware` | Аппаратные параметры из конфига |
| `GET` | `/api/v1/settings` | Текущие настройки (реально — только `num_ctx` и значения из конфига) |
| `PATCH` | `/api/v1/settings` | Изменить `num_ctx`. При отсутствии конфига → 500 (`ISSUE-033`) |
| `GET` | `/api/v1/hardware/detect` | Определение GPU/CPU/RAM и рекомендации по моделям |
| `GET` | `/api/v1/logs` | Буфер логов |
| `DELETE` | `/api/v1/logs` | Очистить буфер логов |

`/api/v1/health` возвращает **`version: "0.1.0"`** при `pyproject.toml:7` = `0.3.0`
(`ISSUE-027`).

### Чат

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/chat` | Ответ через пайплайн |
| `WS` | `/ws` | WebSocket-чат (`server.py:42`) |

`POST /api/v1/chat` через `api/state.py` тянет legacy-пакет `engine/`, поэтому на машине
без `llama_cpp` даёт `500 {"detail":"Failed to load model: No module named 'llama_cpp'"}`
(`ISSUE` в разделе B.7 реестра).

Стриминга нет: `/ws` получает готовый ответ и режет его по пробелам
(`api/websocket.py:217-226`) — `ISSUE-012`. Тип завершающего сообщения — `end`, **не**
`done`.

### Сессии

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/sessions` | Список сессий |
| `POST` | `/api/v1/sessions` | Создать сессию |
| `GET` | `/api/v1/sessions/{session_id}` | Информация о сессии |
| `PATCH` | `/api/v1/sessions/{session_id}` | Переименовать |
| `DELETE` | `/api/v1/sessions/{session_id}` | Удалить. **Уязвим к обходу пути — `ISSUE-001`** |
| `POST` | `/api/v1/sessions/{session_id}/switch` | Переключиться на сессию |
| `GET` | `/api/v1/sessions/{session_id}/history` | История сообщений |
| `GET` | `/api/v1/session/info` | Информация об активной сессии |
| `DELETE` | `/api/v1/session` | Удалить активную сессию |

> ### ⚠️ `DELETE /api/v1/sessions/{session_id}` — обход пути
>
> `session_id` не валидируется, поэтому `DELETE /api/v1/sessions/..` вызывает
> `shutil.rmtree("storage/")` и **уничтожает всю память, все сессии и индекс**.
> Проверено: против живого uvicorn форма `%2E%2E` даёт `HTTP 200` и удаляет `storage/`.
> `session_exists()` — это просто `is_dir()` (`core/session_manager.py:372-374`),
> `delete_session()` — `rmtree` без проверки (`:302-307`).
>
> Детали и минимальная правка — `ISSUE-001`.

### Память

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/memory/stats` | Статистика: HOT-факты, хранилище, провайдер эмбеддингов |
| `GET` | `/api/v1/memory/facts` | Список фактов (WorkingMemory + storage) |
| `POST` | `/api/v1/memory/clear` | Очистить память. **Только POST**, удаление через `DELETE` даёт 405 |

**Поиска по памяти нет.** `GET /api/v1/memory/search` и `POST /api/v1/memory/fact` из
прежней документации не существуют — оба отдают 404.

### Модели

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/models/status` | Статус моделей |
| `POST` | `/api/v1/models/switch` | Переключить модель |
| `POST` | `/api/v1/models/download` | Скачать модель |
| `GET` | `/api/v1/models/download/check/{model_name}` | Проверить наличие модели |
| `DELETE` | `/api/v1/models/{model_name}` | Удалить модель с диска. **Без аутентификации** |
| `GET` | `/api/v1/ollama/models` | Список моделей Ollama (CORS-прокси) |

### Диагностика

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/cci/stats` | Статистика метрики связности |
| `GET` | `/api/v1/dual-model/stats` | Статистика вызовов координатора и генератора |

### OpenAI-совместимый API

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/v1/models` | Список моделей в формате OpenAI |
| `POST` | `/v1/chat/completions` | Чат в формате OpenAI |

`openai_router` объявлен с `prefix="/v1"` (`api/system.py:344`) и подключается без
дополнительного префикса (`server.py:39`).

---

## POST /v1/chat/completions

Два режима, выбираемых по полю `model` (`api/system.py:381-382`):

| `model` | Режим |
|---|---|
| `contextor`, `contextor-code`, `contextor-fast` | Полный пайплайн с памятью |
| любое другое значение | Прозрачный прокси в Ollama, **без памяти** |

**Запрос:**

```json
{
  "model": "contextor",
  "messages": [{"role": "user", "content": "Привет"}],
  "temperature": 0.7,
  "max_tokens": 2000,
  "stream": false
}
```

**Ответ при `stream: false`** (проверено запросом):

```json
{
  "id": "chatcmpl-1db4e9f42346",
  "object": "chat.completion",
  "created": 1789756471,
  "model": "contextor",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": ""},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 2, "completion_tokens": 0, "total_tokens": 2},
  "system_fingerprint": "contextor-v1"
}
```

Обратите внимание на `content: ""` — при недоступной модели эндпоинт возвращает
**HTTP 200 с пустым ответом**, а не ошибку (`ISSUE-011`).

**Известные отклонения от спецификации OpenAI:**

| Поле | Проблема |
|---|---|
| `stream: true` | Не работает. На прокси-пути захардкожено `"stream": False` (`:388`); на основном пути SSE — заглушка, режущая готовый ответ (`:322-341`) — `ISSUE-012` |
| `usage` | Выдуман: `result.tokens_completion or len(response_text.split()) * 2` (`:419-420`) — `ISSUE` C.3 |
| `model` в ответе | Возвращается то, что прислал клиент, а не модель, которая реально работала (`ISSUE-023`) |
| `system` в `messages` | Приводит к тому, что память **не инжектируется** (`:444`) — `ISSUE-003` |
| `GET /v1/models` | Перечисляет `contextor` и `contextor-code`, но обработчик принимает ещё и `contextor-fast` — он в список не попадает |

---

## GET /v1/models

```json
{
  "object": "list",
  "data": [
    {"id": "contextor", "object": "model", "created": 1714000000, "owned_by": "contextor",
     "description": "Local AI with hierarchical memory"},
    {"id": "contextor-code", "object": "model", "created": 1714000000, "owned_by": "contextor",
     "description": "Contextor with Code Module"}
  ]
}
```

Второй элемент ссылается на «Code Module», удалённый в коммите `5aa4777`. Ответ
статический (`api/system.py:347-368`), реальные модели Ollama здесь не отражаются —
для них есть `/api/v1/ollama/models`.

---

## WebSocket /ws

Путь — `/ws`, **не** `/ws/chat`. Регистрируется через
`app.add_api_websocket_route` (`server.py:42`).

Сервер отправляет завершающее сообщение типа `end` (`api/websocket.py:245,288`), не `done`.
Клиент обрабатывает именно `end` (`static/js/app.js:537`).

Клиентское поле `session_id` **не читается** — активная сессия берётся из серверного
состояния (`api/websocket.py:60-79`).

Автопереподключение на клиенте реализовано: `ws.onclose → setTimeout(connectWS, 4000)`
(`app.js:424-428`).

---

## Что было удалено из прежней версии документа

Приведено, чтобы эти эндпоинты не вернулись в документацию.

| Прежнее утверждение | Факт |
|---|---|
| `GET /api/v1/memory/search` | 404 — не существует |
| `POST /api/v1/memory/fact` | 404 — не существует |
| `DELETE /api/v1/memory/clear` | 405 — реально только `POST` |
| `WS /ws/chat` | Такого пути нет, есть `/ws` |
| Тип сообщения `done` | Реально `end` |
| `contextor config --show-path` | CLI имеет только команды `model` и `serve` |
| «Любой OpenAI-клиент работает из коробки» | Работает, но память применяется только при `model` из списка `contextor*`, а `system`-сообщение отключает инжекцию памяти |
| «`stream: true` стримит» | Не стримит |

---

## Как проверить список маршрутов самостоятельно

```powershell
.\.dsh-dev\dev.ps1 -m contextor serve --port 7871
# затем в другом окне:
Invoke-WebRequest http://127.0.0.1:7871/openapi.json -UseBasicParsing | Select-Object -ExpandProperty Content
```

Или без запуска сервера:

```powershell
.\.dsh-dev\dev.ps1 -c "from contextor.server import app; print('\n'.join(sorted(app.openapi()['paths'])))"
```

`.dsh-dev` — обвязка для запуска внутри sandbox DSH, см. `docs/JOURNAL.md`.
