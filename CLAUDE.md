# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project

HyperTrader v2 — real-time ETH market data collector + web dashboard.
See `PLAN.md` for full spec.

## Stack

- Python 3.12, asyncio, aiohttp
- Pydantic + pydantic-settings (models, config validation)
- FastAPI + uvicorn + WebSocket
- feedparser (RSS)
- Chart.js (CDN)
- Single process, single event loop

## Agent Orchestration

### Available Agents (`.claude/agents/`)

| Agent | Model | Role |
|---|---|---|
| `planner` | opus | Архитектура, декомпозиция задач, создание планов в `./plans/` |
| `researcher` | haiku | Исследование технологий, API, best practices. Запускать параллельно по темам |
| `fullstack-developer` | sonnet | Реализация кода по плану. Строго следует плану planner'а |
| `code-simplifier` | opus | Рефакторинг и упрощение после реализации |
| `tester` | haiku | Написание и запуск тестов. Без моков — только реальные вызовы |
| `code-reviewer` | — | Ревью качества кода (read-only) |
| `debugger` | sonnet | Диагностика багов, анализ логов, профилирование |

### Кейс 1: Новая фича (основной flow)

```
1. planner          — создаёт план + TODO-фазы в ./plans/
   ↕ параллельно
   researcher ×N    — исследует вопросы для planner'а
2. fullstack-developer — пишет код по плану (фаза за фазой)
3. code-simplifier     — упрощает написанное
4. tester              — пишет и гоняет тесты
   ↓ тесты падают? → назад к п.2, фиксим, повторяем с п.4
5. code-reviewer       — финальное ревью
```

### Кейс 2: Баг / проблема в рантайме

```
1. debugger         — анализирует баг, логи, воспроизводит
2. planner          — если фикс нетривиальный, планирует решение
3. fullstack-developer — реализует фикс
4. tester           — прогоняет тесты
   ↓ тесты падают? → назад к п.3
5. code-reviewer    — ревью фикса
```

### Кейс 3: Исследование / выбор технологии

```
1. researcher ×2-3 параллельно — каждый исследует свой вариант
2. planner                     — анализирует результаты, даёт рекомендацию
```

### Кейс 4: Рефакторинг существующего кода

```
1. code-reviewer       — находит проблемы качества
2. planner             — план рефакторинга
3. code-simplifier     — выполняет рефакторинг
4. tester              — прогоняет тесты
```

### Кейс 5: Быстрая правка (мелочь, < 20 строк)

Без агентов. Правим напрямую, проверяем вручную.

### Правила оркестрации

- При делегации ВСЕГДА указывать: work context path, reports path (`./plans/reports/`), plans path (`./plans/`)
- Параллельно запускать ТОЛЬКО независимые задачи (разные файлы, нет конфликтов)
- Каждый агент завершается полностью перед передачей следующему в цепочке
- Если tester репортит падения — ОБЯЗАТЕЛЬНО фиксить и перезапускать, не игнорировать

## Skills (`.claude/skills/`)

- **backend-development** — FastAPI, async, API-дизайн, performance, security, testing
- **databases** — PostgreSQL, MongoDB, ETL, миграции, оптимизация

## Rules

- YAGNI, KISS, DRY
- Keep files under 200 lines — modularize if exceeded
- Use kebab-case for file names
- Async everywhere — no blocking calls
- All Hyperliquid API calls via `aiohttp` POST to `https://api.hyperliquid.xyz/info`
- WebSocket for pushing data to browser
- Pydantic models for all data structures — no raw dicts
- Event bus for decoupling collectors from consumers

## Structure

```
HYPERTRADER/
├── src/
│   ├── main.py              # Entry point
│   ├── config.py            # Pydantic Settings + YAML
│   ├── models/              # Pydantic models (market, news)
│   ├── collectors/          # BaseCollector + 5 collectors
│   ├── storage/             # Typed Cache with TTL
│   ├── events/              # In-process EventBus
│   ├── server/              # FastAPI + WebSocket
│   └── utils/               # Logger, RateLimiter
├── static/index.html        # Dashboard (Chart.js)
├── config/settings.yaml     # All settings
└── data/                    # Runtime data
```

## Reference Code

`../BITCOIN/src/` contains working implementations of collectors, models, cache, config.
Adapt patterns, don't copy blindly. Key differences from BITCOIN:
- No ArcticDB / MarketStore (yet) — in-memory cache only for Stage 1
- EventBus instead of direct cache writes
- Simpler config (no AI, risk, execution sections yet)
- Single coin (ETH) focus
