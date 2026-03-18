# HyperTrader v2 — Этап 1: Сбор данных

## Цель
Собирать рыночные данные ETH с Hyperliquid + новости + сентимент.
Показывать всё в веб-дашборде в реальном времени.
Архитектура должна быть готова к Stage 2+ (AI-анализ, сигналы, торговля).

## Что собираем

| Источник | Данные | Метод | Частота |
|---|---|---|---|
| Hyperliquid REST | Цена, funding, OI, объём | POST /info `metaAndAssetCtxs` | 10 сек |
| Hyperliquid REST | Свечи OHLCV (5m, 15m, 1h, 4h) | POST /info `candleSnapshot` + WS | при старте + по закрытию |
| Hyperliquid REST | Стакан (l2Book, 20 уровней) | POST /info `l2Book` | 30 сек |
| RSS | Новости (CoinDesk, CoinTelegraph, Decrypt, TheBlock) | GET | 5 мин |
| alternative.me | Fear & Greed Index | GET | 1 час |

## Что показываем в дашборде

- Цена ETH live
- Свечной график (Chart.js)
- Стакан: bid/ask стенки, спред, имбаланс
- Funding rate + annualized
- Open Interest
- 24h объём
- Последние новости
- Fear & Greed

## Архитектура

```
Collectors (async) ──► EventBus ──► Cache (typed, in-memory)
                           │                │
                           │                ▼
                           │        Server (FastAPI + WS) ──► Browser
                           │
                           └──► [Stage 2+: Persistence, AI, Signals]
```

Один процесс, один event loop.
Event bus — in-process pub/sub для развязки компонентов.
Потребители подписываются на события — при добавлении AI/signals не трогаем коллекторы.

## Структура

```
HYPERTRADER/
├── src/
│   ├── __init__.py
│   ├── main.py                 # Entry point: init, start, shutdown
│   ├── config.py               # Pydantic Settings + YAML loader
│   ├── models/
│   │   ├── __init__.py
│   │   ├── market.py           # AssetContext, Candle, OrderBook, FundingRate
│   │   └── news.py             # NewsItem, NewsSource, SentimentSnapshot
│   ├── collectors/
│   │   ├── __init__.py
│   │   ├── base.py             # BaseCollector (ABC: start/stop/health_check)
│   │   ├── market.py           # MarketDataCollector (metaAndAssetCtxs)
│   │   ├── candles.py          # CandleCollector (REST + WS)
│   │   ├── orderbook.py        # OrderbookCollector (l2Book)
│   │   ├── news.py             # NewsCollector (RSS via feedparser)
│   │   └── sentiment.py        # SentimentCollector (Fear & Greed)
│   ├── storage/
│   │   ├── __init__.py
│   │   └── cache.py            # Typed Cache with TTL, deque for candles
│   ├── events/
│   │   ├── __init__.py
│   │   └── bus.py              # EventBus: subscribe/emit, async callbacks
│   ├── server/
│   │   ├── __init__.py
│   │   ├── app.py              # FastAPI app factory
│   │   └── websocket.py        # WS manager: broadcast snapshots
│   └── utils/
│       ├── __init__.py
│       ├── logger.py           # Structured logging
│       └── rate-limiter.py     # Token bucket for Hyperliquid rate limits
├── static/
│   └── index.html              # Dashboard (Chart.js CDN)
├── config/
│   └── settings.yaml           # All settings
├── data/                       # Runtime data
├── pyproject.toml
└── requirements.txt
```

## Стек
- Python 3.12, asyncio, aiohttp
- Pydantic + pydantic-settings (models, config)
- FastAPI + uvicorn + WebSocket
- feedparser (RSS)
- Chart.js (CDN)
- PyYAML (config)

## Референс
Код из `../BITCOIN/src/` — рабочие коллекторы, модели, кэш. Адаптировать, не копировать слепо.
