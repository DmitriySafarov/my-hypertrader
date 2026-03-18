---
phase: 4
title: EventBus
status: pending
effort: 1h
---

# Phase 4: EventBus

## Overview

In-process async pub/sub. Core architectural component that decouples collectors from consumers.

**No external dependencies.** Pure asyncio.

## File to Create

### `src/events/bus.py` (~60 lines)

```python
import asyncio
from collections import defaultdict
from typing import Any, Callable, Awaitable
from src.utils.logger import get_logger

log = get_logger(__name__)

# Type alias for event handlers
Handler = Callable[[Any], Awaitable[None]]

class EventBus:
    """Async in-process pub/sub.

    Usage:
        bus = EventBus()
        bus.subscribe("market_data", my_handler)   # async def my_handler(data): ...
        await bus.emit("market_data", asset_ctx)    # calls all subscribed handlers
    """
    def __init__(self) -> None:
        self._handlers: dict[str, list[Handler]] = defaultdict(list)

    def subscribe(self, event: str, handler: Handler) -> None:
        """Register async handler for event type."""
        self._handlers[event].append(handler)
        log.debug("subscribed", event=event, handler=handler.__qualname__)

    def unsubscribe(self, event: str, handler: Handler) -> None:
        """Remove handler. No-op if not found."""
        try:
            self._handlers[event].remove(handler)
        except ValueError:
            pass

    async def emit(self, event: str, data: Any) -> None:
        """Emit event to all subscribed handlers.

        Handlers run concurrently via gather.
        One failing handler does not block others.
        """
        handlers = self._handlers.get(event, [])
        if not handlers:
            return
        results = await asyncio.gather(
            *(h(data) for h in handlers),
            return_exceptions=True,
        )
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                log.error(
                    "handler_error",
                    event=event,
                    handler=handlers[i].__qualname__,
                    error=str(result),
                )
```

## Event Contract

All events carry a **Pydantic model** as payload. This gives:
- Type safety (handlers know what they receive)
- Easy serialization (`.model_dump()` for JSON)
- Schema evolution (add optional fields without breaking)

### Event Names (string constants)

Define in `src/events/__init__.py`:
```python
# Event name constants
MARKET_DATA = "market_data"
CANDLE_UPDATE = "candle_update"
ORDERBOOK = "orderbook"
NEWS_UPDATE = "news_update"
SENTIMENT = "sentiment"
```

## Integration Pattern

### Collector emits:
```python
# Inside MarketDataCollector._fetch_and_update():
asset_ctx = AssetContext(...)
await self._bus.emit(MARKET_DATA, asset_ctx)
```

### Cache subscribes:
```python
# In main.py setup:
bus.subscribe(MARKET_DATA, cache.on_market_data)
bus.subscribe(ORDERBOOK, cache.on_orderbook)
bus.subscribe(CANDLE_UPDATE, cache.on_candle_update)
bus.subscribe(NEWS_UPDATE, cache.on_news_update)
bus.subscribe(SENTIMENT, cache.on_sentiment)
```

### WS Manager subscribes:
```python
# In main.py setup:
bus.subscribe(MARKET_DATA, ws_manager.on_market_data)
bus.subscribe(CANDLE_UPDATE, ws_manager.on_candle_update)
# ... etc
```

## Why Not Just Direct Cache Writes?

The BITCOIN project has collectors writing directly to cache (`self.cache.set_asset_context(...)`). This works but couples collectors to cache API. With EventBus:

1. Adding a new consumer (AI, signals, persistence) = just `bus.subscribe()`
2. Collectors don't know about cache, WS, or any consumer
3. Testing collectors = mock the bus, check emitted events

## TODO
- [ ] Create `src/events/bus.py`
- [ ] Add event constants to `src/events/__init__.py`
- [ ] Verify: instantiate EventBus, subscribe handler, emit event

## Success Criteria
- `await bus.emit("test", data)` calls all subscribed handlers
- Failed handler does not prevent other handlers from running
- Handler errors are logged, not raised
