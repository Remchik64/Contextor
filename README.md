# 🧠 Contextor

> **Local AI with hierarchical memory — управление контекстным окном вместо его обрезки**

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.11%2B-green.svg)](https://python.org)
[![GitHub](https://img.shields.io/badge/GitHub-Remchik64%2FContextor-black.svg)](https://github.com/Remchik64/Contextor)
[![Support on Boosty](https://img.shields.io/badge/Support%20on%20Boosty-orange.svg)](https://boosty.to/rem64)
[![Support via YooMoney](https://img.shields.io/badge/Support-ЮMoney-blueviolet.svg)](https://yoomoney.ru/to/4100118846255337)

![Contextor Admin Panel](docs/images/admin-panel.png)

---

## ⚠️ Текущее состояние проекта

Этот README описывает **замысел**. Прежде чем пользоваться — прочитайте фактическое
состояние, потому что часть механизмов не работает:

| Документ | О чём |
|---|---|
| [`docs/JOURNAL.md`](docs/JOURNAL.md) | Что реально работает, что нет, хронология |
| [`docs/known-issues.md`](docs/known-issues.md) | 40 проверенных проблем с `file:line` |
| [`docs/architecture.md`](docs/architecture.md) | Архитектура по факту кода |
| [`docs/installation.md`](docs/installation.md) | Установка и обходные пути |

**Коротко:** 392 теста, 367 проходит, 25 падает; покрытие 49%. Сервер поднимается и
отдаёт 27 маршрутов. Связка «координатор → генератор» реализована и работает.
Но: **установщики ссылаются на переименованный репозиторий и отдают 404**, не-ASCII
память повреждается при перезапуске, а `DELETE /api/v1/sessions/..` уничтожает весь
`storage/` без аутентификации.

Установка: используйте [ручной способ](docs/installation.md), не установщик.

---

## Проблема, которую решает Contextor

У любой LLM конечное контекстное окно. Когда разговор становится длинным, окно
переполняется — и модель не обрывается, а **деградирует**: забывает начало, путает
ранее принятые решения, начинает выдумывать.

Обычные подходы: обрезать историю (знания теряются) или ждать падения. Contextor
пробует третье — **управлять контекстом, а не обрезать его**.

---

## Как это устроено

### Иерархическая память

Факты хранятся в двух уровнях и на диске:

| Уровень | Что | Где физически |
|---|---|---|
| **HOT** | активные факты | список в оперативной памяти |
| **WARM** | остальные факты | словарь в памяти + зеркало в JSON |
| **ARCHIVED** | сжатые факты | значение уровня сжатия |

> В предыдущей версии README здесь был третий уровень «Anchor» и утверждалось, что WARM
> хранится в ChromaDB с эмбеддингами SentenceTransformer. Оба утверждения неверны:
> «Anchor» — это булев флаг `is_anchor` на факте (`fact.py:53`), а WARM — обычный JSON-файл
> (`storage.py:551`). ChromaDB используется только для индекса карточек кода, который
> никогда не наполняется.

### Сброс контекста через координату

Основная идея:

1. окно заполняется;
2. **координатор** — маленькая модель (~2B) — создаёт **координату**: сжатый снимок
   того, что важно сохранить;
3. история обрезается до последних нескольких сообщений;
4. координата попадает в системный промпт;
5. генератор продолжает разговор, уже не помня историю дословно.

Порядок в коде именно такой, и он работает: координата создаётся
(`orchestrator.py:345`) **до** сборки промпта (`:444`), поэтому генератор получает её в
том же ходу, в котором контекст был сброшен. Детали — [`docs/architecture.md`](docs/architecture.md).

### Память не растёт с длиной сессии

Каждые 4 координаты сворачиваются в одну мета-координату, старые уходят в архив. В промпт
всегда попадают только мета-координата и последняя активная
(`meta_coordinator.py:144-167`), поэтому размер памяти постоянен при любой длине беседы.

### Две модели, которые не знают друг о друге

| Роль | Размер | Задача |
|---|---|---|
| Coordinator | ~2B | координаты, извлечение фактов |
| Generator | ~9B | только ответ пользователю |

Вызываются независимыми HTTP-запросами без общего состояния (`dual_model.py:180-219`).
Смысл разделения — экономия: большая модель включается только на генерации.

---

## Производительность

> **Следующие числа не подтверждены.** В прежней версии README они подавались как факт.
> Бенчмарки в репозитории их не измеряют: `benchmarks/runner.py` не подключён ни к тестам,
> ни к CLI, ни к CI, а его «базовая линия» задана захардкоженными нулями
> (`ISSUE-032` в [`docs/known-issues.md`](docs/known-issues.md)).

Что можно измерить честно и как это сделать — в
[`docs/known-issues.md`](docs/known-issues.md) и [`docs/proxy/03-context-budget.md`](docs/proxy/03-context-budget.md).

---

## Установка

### Требования

Python 3.11+, [Ollama](https://ollama.com). GPU необязателен.

### Ручная установка (рекомендуется)

```bash
# 1. Ollama
curl -fsSL https://ollama.com/install.sh | sh      # Linux
# Windows/macOS: https://ollama.com/download

# 2. Код
git clone https://github.com/Remchik64/Contextor.git
cd Contextor

# 3. Окружение
python -m venv .venv
source .venv/bin/activate        # Linux/macOS
.\.venv\Scripts\activate         # Windows

# 4. Зависимости
pip install -e .
pip install psutil pyyaml        # не объявлены, но нужны

# 5. Модели (имена из config.yaml)
ollama pull qwen3.5:2b
ollama pull qwen3.5:9b
ollama pull nomic-embed-text

# 6. Запуск
python -m contextor serve --port 7860
```

Откройте `http://localhost:7860`.

> **Установщики `install.bat` / `install.sh` сейчас не работают:** они скачивают из
> `Remchik64/Contextor-pro` — репозиторий переименован, и этот URL отдаёт **404**
> (проверено запросом). Подробности — [`docs/installation.md`](docs/installation.md).

### Конфигурация — важная особенность

Файл `config.yaml` **в корне проекта может не читаться**: `%APPDATA%\Contextor\config.yaml`
имеет приоритет над ним (`config_loader.py:118-153`). Какой файл используется фактически,
показывает `GET /api/v1/config`. Чтобы задать свой явно:

```bash
export PURE_INTELLECT_CONFIG="$PWD/config.yaml"     # имя переменной — историческое
```

Часть ключей конфигурации не работает вовсе — полный список в
[`ISSUE-017`](docs/known-issues.md).

---

## CLI

```bash
contextor serve --port 7860     # запустить сервер
contextor model list            # список моделей
contextor model download <key>  # скачать модель
```

Команд ровно две: `serve` и `model`. Упоминавшаяся ранее `contextor config --show-path`
не существует.

---

## API

OpenAI-совместимый интерфейс:

```bash
curl http://localhost:7860/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"contextor","messages":[{"role":"user","content":"Привет"}]}'
```

Полный список — 27 маршрутов плюс WebSocket `/ws` — в
[`docs/api_reference.md`](docs/api_reference.md).

> При использовании внешним клиентом помните: клиентское `system`-сообщение отключает
> инжекцию памяти (`ISSUE-003`), а `stream: true` не стримит (`ISSUE-012`).

---

## Разработка

```bash
git clone https://github.com/Remchik64/Contextor.git
cd Contextor
python -m venv .venv && source .venv/bin/activate
pip install -e .
pip install pytest pytest-asyncio pytest-cov

# тесты
python -m pytest tests/ -q --ignore=tests/test_system_full.py
# 367 passed, 25 failed — известное состояние (ISSUE-020)
```

`tests/test_system_full.py` через pytest запускать нельзя: это не тест, а скрипт с живыми
HTTP-запросами на импорте, из-за которого сбор тестов зависает.

Документация для разработчиков начинается с [`docs/README.md`](docs/README.md).

---

## Замысел дальше: прокси-режим

Основное направление развития — превратить Contextor из самостоятельного чата в
**прокси перед чужим движком**: пользователь поднимает `llama.cpp` со своей моделью, а
Contextor становится её памятью и управляет контекстным окном. Тогда Contextor не тратит
VRAM на генератор и заодно освобождает память под веса модели.

Проработка — в [`docs/proxy/`](docs/proxy/). **В коде этого пока нет** (провайдер
`llamacpp` закомментирован в `engines/provider.py:217-218`).

---

## Лицензия

Copyright 2025 **Яраев Ренат Жавдетович**

Licensed under the **Apache License, Version 2.0**.

Разрешает: коммерческое использование, изменение и распространение, патентное
использование, приватное использование.

Условия: сохранение лицензии и копирайта, указание внесённых изменений, атрибуция автора.

Полные условия — [LICENSE](LICENSE).

---

<div align="center">

**Contextor** — Your context, orchestrated.

*Built with ❤️ by Ренат Яраев (Remchik64)*

[GitHub](https://github.com/Remchik64/Contextor) · [Issues](https://github.com/Remchik64/Contextor/issues) · [License](LICENSE) · [VK](https://vk.com/remchik64) · [Boosty](https://boosty.to/rem64) · [ЮMoney](https://yoomoney.ru/to/4100118846255337)

📧 renataraev51@gmail.com · remch2013@yandex.ru

</div>
