---
title: "HyperTrader v2 - Etap 1: Data Collection + Dashboard"
description: "Real-time ETH market data collection from Hyperliquid + RSS news + Fear&Greed, served via FastAPI WebSocket dashboard"
status: pending
priority: P1
effort: 12h
branch: main
tags: [hypertrader, data-collection, dashboard, etap-1]
created: 2026-03-18
---

# HyperTrader v2 - Etap 1: Data Collection + Dashboard

## Architecture

```
                          +-----------+
                          |  EventBus |
                          +-----+-----+
                                |
          emit("market_data")   |   emit("candle_update")
          emit("orderbook")     |   emit("news_update")
          emit("sentiment")     |
                                |
   +----------------------------+----------------------------+
   |            |               |              |             |
+--+--+   +----+----+   +------+-----+  +-----+--+  +------+----+
|Market|   | Candle  |   | Orderbook  |  |  News  |  | Sentiment |
|Coll. |   | Coll.   |   | Collector  |  | Coll.  |  | Collector |
+------+   +---------+   +------------+  +--------+  +-----------+
   |            |               |              |             |
   +----------------------------+----------------------------+
                                |
                          +-----+-----+
                          |   Cache   |  (typed, TTL, deque for candles)
                          +-----+-----+
                                |
                          +-----+-----+
                          |  Server   |  FastAPI + WS Manager
                          +-----+-----+
                                |
                          +-----+-----+
                          |  Browser  |  Chart.js dashboard
                          +-----------+
```

### Event Flow (key design decision)

1. Collector fetches data -> creates Pydantic model -> emits event on EventBus
2. EventBus handler (registered by Cache) receives event -> updates typed cache
3. EventBus handler (registered by WS Manager) receives event -> broadcasts to connected browsers
4. Future: AI/Signal handlers subscribe to same events without touching collectors

### Events

| Event Name       | Payload Type        | Emitter            | Consumers              |
|------------------|---------------------|--------------------|------------------------|
| `market_data`    | `AssetContext`      | MarketDataCollector| Cache, WSManager       |
| `candle_update`  | `CandleUpdate`      | CandleCollector    | Cache, WSManager       |
| `orderbook`      | `OrderBookSnapshot` | OrderbookCollector | Cache, WSManager       |
| `news_update`    | `list[NewsItem]`    | NewsCollector      | Cache, WSManager       |
| `sentiment`      | `SentimentData`     | SentimentCollector | Cache, WSManager       |

### Key Decisions

1. **float, not Decimal** -- Dashboard-only stage, no trading. float is simpler, faster for JSON serialization. Switch to Decimal in Stage 2 when precision matters for orders.
2. **EventBus** -- Simple async pub/sub. No external deps. Decouples collectors from consumers.
3. **No persistence** -- In-memory only for Stage 1. EventBus makes adding persistence trivial later.
4. **Single aiohttp.ClientSession** per collector -- shared within collector, not across collectors.
5. **CandleCollector: REST-only** for Stage 1 -- WS subscription adds complexity for multi-timeframe. REST polling on interval close is simpler and sufficient.

## Phases

| # | Phase | Files | Status | Effort |
|---|-------|-------|--------|--------|
| 1 | [Project Setup](phase-01-project-setup.md) | pyproject.toml, requirements.txt, dirs, __init__.py | **done** | 30m |
| 2 | [Models + Config](phase-02-models-and-config.md) | models/market.py, models/news.py, config.py, settings.yaml | pending | 1.5h |
| 3 | [Utils](phase-03-utils.md) | utils/logger.py, utils/rate-limiter.py | pending | 30m |
| 4 | [EventBus](phase-04-events.md) | events/bus.py | pending | 1h |
| 5 | [Cache](phase-05-cache.md) | storage/cache.py | pending | 1h |
| 6 | [Collectors](phase-06-collectors.md) | collectors/base.py + 5 collectors | pending | 4h |
| 7 | [Server](phase-07-server.md) | server/app.py, server/websocket.py | pending | 1.5h |
| 8 | [Dashboard](phase-08-dashboard.md) | static/index.html | pending | 1.5h |
| 9 | [Main Entry](phase-09-main.md) | main.py | pending | 30m |

## Dependencies

```
Phase 1 (setup)
  -> Phase 2 (models + config)
    -> Phase 3 (utils)
      -> Phase 4 (EventBus)
        -> Phase 5 (cache)
          -> Phase 6 (collectors) -- depends on 2,3,4,5
            -> Phase 7 (server) -- depends on 4,5
              -> Phase 8 (dashboard)
                -> Phase 9 (main) -- depends on all
```

## Reference Code

All patterns adapted from `../BITCOIN/src/`. Key adaptations:
- Remove ArcticDB/MarketStore (no persistence in Stage 1)
- Remove pandas dependency (not needed without persistence)
- Remove multi-coin tracking (ETH only)
- Add EventBus integration (BITCOIN writes directly to cache)
- Simplify config (remove trading/AI/signal configs)
