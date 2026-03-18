---
phase: 9
title: Main Entry Point
status: pending
effort: 30m
---

# Phase 9: Main Entry Point

## File to Create

### `src/main.py` (~80 lines)

Orchestrates all components: init, wire EventBus, start collectors, run server, graceful shutdown.

```python
import asyncio
import signal
import uvicorn

from src.config import Settings
from src.events import MARKET_DATA, CANDLE_UPDATE, ORDERBOOK, NEWS_UPDATE, SENTIMENT
from src.events.bus import EventBus
from src.storage.cache import Cache
from src.server.websocket import WSManager
from src.server.app import create_app
from src.collectors.market import MarketDataCollector
from src.collectors.candles import CandleCollector
from src.collectors.orderbook import OrderbookCollector
from src.collectors.news import NewsCollector
from src.collectors.sentiment import SentimentCollector
from src.utils.logger import setup_logging, get_logger
from src.utils.rate_limiter import RateLimiter

log = get_logger(__name__)


async def run() -> None:
    # 1. Load config
    settings = Settings.load()
    setup_logging(settings.log_level)
    log.info(f"starting HyperTrader v2 | coin={settings.coin}")

    # 2. Create core components
    bus = EventBus()
    cache = Cache()
    rate_limiter = RateLimiter(budget_per_min=settings.rate_limit_budget)
    ws_manager = WSManager(broadcast_interval=settings.server.ws_broadcast_sec)

    # 3. Wire EventBus subscriptions
    #    Cache consumes all events (stores state)
    bus.subscribe(MARKET_DATA, cache.on_market_data)
    bus.subscribe(CANDLE_UPDATE, cache.on_candle_update)
    bus.subscribe(ORDERBOOK, cache.on_orderbook)
    bus.subscribe(NEWS_UPDATE, cache.on_news_update)
    bus.subscribe(SENTIMENT, cache.on_sentiment)

    #    WSManager consumes all events (broadcasts to browsers)
    bus.subscribe(MARKET_DATA, ws_manager.on_market_data)
    bus.subscribe(CANDLE_UPDATE, ws_manager.on_candle_update)
    bus.subscribe(ORDERBOOK, ws_manager.on_orderbook)
    bus.subscribe(NEWS_UPDATE, ws_manager.on_news_update)
    bus.subscribe(SENTIMENT, ws_manager.on_sentiment)

    # 4. Create collectors
    collectors = [
        MarketDataCollector(settings, bus, rate_limiter),
        CandleCollector(settings, bus, rate_limiter),
        OrderbookCollector(settings, bus, rate_limiter),
        NewsCollector(settings, bus),
        SentimentCollector(settings, bus),
    ]

    # 5. Create FastAPI app
    app = create_app(settings, cache, ws_manager)

    # 6. Start everything
    await ws_manager.start()
    for c in collectors:
        await c.start()
        log.info(f"collector started: {c.name}")

    # 7. Run uvicorn
    config = uvicorn.Config(
        app,
        host=settings.server.host,
        port=settings.server.port,
        log_level=settings.log_level.lower(),
    )
    server = uvicorn.Server(config)

    # 8. Graceful shutdown on SIGINT/SIGTERM
    shutdown_event = asyncio.Event()

    def _signal_handler():
        log.info("shutdown signal received")
        shutdown_event.set()

    loop = asyncio.get_running_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, _signal_handler)

    # Run server in background, wait for shutdown
    server_task = asyncio.create_task(server.serve())

    await shutdown_event.wait()

    # 9. Stop collectors
    log.info("stopping collectors...")
    for c in reversed(collectors):
        try:
            await c.stop()
        except Exception as e:
            log.error(f"error stopping {c.name}: {e}")

    await ws_manager.stop()
    server.should_exit = True
    await server_task
    log.info("HyperTrader stopped")


def main():
    asyncio.run(run())


if __name__ == "__main__":
    main()
```

## Startup Sequence

```
1. Load config/settings.yaml
2. Setup logging
3. Create: EventBus, Cache, RateLimiter, WSManager
4. Wire: bus.subscribe() for Cache + WSManager
5. Create 5 collectors (pass bus, limiter)
6. Start WSManager broadcast loop
7. Start collectors (each does initial fetch + starts poll loop)
8. Start uvicorn (FastAPI serves dashboard + WS)
9. Wait for SIGINT/SIGTERM
10. Stop collectors (reverse order)
11. Stop WSManager
12. Stop uvicorn
```

## Shutdown Behavior
- Signal handler sets `shutdown_event`
- Collectors stop in reverse order (sentiment first, market last)
- Each collector cancels its poll task, closes aiohttp session
- WSManager stops broadcast loop
- Uvicorn exits gracefully

## TODO
- [ ] Create `src/main.py`
- [ ] Verify: `python -m src.main` starts all components
- [ ] Verify: CTRL+C triggers graceful shutdown
- [ ] Verify: dashboard accessible at http://localhost:8080

## Success Criteria
- Application starts without errors
- All 5 collectors polling
- Dashboard loads and receives live data via WS
- CTRL+C stops everything cleanly (no orphaned tasks)
- Log output shows startup/shutdown sequence
