---
phase: 5
title: Typed Cache
status: pending
effort: 1h
---

# Phase 5: Typed Cache with TTL

## Reference
- `../BITCOIN/src/storage/cache.py` -- typed methods, TTL, candle deque

## File to Create

### `src/storage/cache.py` (~120 lines)

Adapted from BITCOIN. Key changes:
- Remove multi-coin complexity (ETH only, but keep symbol param for future)
- Remove ArcticDB/MarketStore refs
- Remove `ContextReport`, `ready_coins`, `loading_progress` (Stage 2+)
- Add EventBus handler methods (`on_market_data`, `on_orderbook`, etc.)
- Add `get_snapshot()` for building full dashboard payload

```python
from collections import deque
from datetime import datetime, timedelta, timezone
from typing import Any, Optional

from src.models.market import AssetContext, Candle, CandleUpdate, OrderBookSnapshot
from src.models.news import NewsItem, SentimentData
from src.utils.logger import get_logger

log = get_logger(__name__)

class Cache:
    """In-memory typed cache with TTL support.

    Stores latest market data, orderbook, candles, news, sentiment.
    Provides EventBus handler methods and a snapshot builder for WS broadcast.
    """
    CANDLE_BUFFER_SIZE = 500
    NEWS_MAX_ITEMS = 50

    def __init__(self) -> None:
        self._store: dict[str, Any] = {}
        self._ttl: dict[str, datetime] = {}
        self._candles: dict[str, deque[Candle]] = {}  # key: "{symbol}_{interval}"
        self._news: list[NewsItem] = []
        self._sentiment: SentimentData | None = None

    # ── Generic TTL store ──

    def _set(self, key: str, value: Any, ttl_sec: int | None = None) -> None:
        self._store[key] = value
        if ttl_sec is not None:
            self._ttl[key] = datetime.now(timezone.utc) + timedelta(seconds=ttl_sec)

    def _get(self, key: str) -> Any | None:
        if key not in self._store:
            return None
        if key in self._ttl and datetime.now(timezone.utc) > self._ttl[key]:
            del self._store[key]
            del self._ttl[key]
            return None
        return self._store[key]

    # ── Typed accessors ──

    def get_asset_context(self, symbol: str) -> AssetContext | None:
        return self._get(f"ctx_{symbol}")

    def get_orderbook(self, symbol: str) -> OrderBookSnapshot | None:
        return self._get(f"ob_{symbol}")

    def get_candles(self, symbol: str, interval: str) -> list[Candle]:
        key = f"{symbol}_{interval}"
        dq = self._candles.get(key)
        return list(dq) if dq else []

    def get_news(self) -> list[NewsItem]:
        return list(self._news)

    def get_sentiment(self) -> SentimentData | None:
        return self._sentiment

    # ── EventBus handlers (async, called by bus.emit) ──

    async def on_market_data(self, ctx: AssetContext) -> None:
        self._set(f"ctx_{ctx.symbol}", ctx)

    async def on_orderbook(self, ob: OrderBookSnapshot) -> None:
        self._set(f"ob_{ob.symbol}", ob)

    async def on_candle_update(self, update: CandleUpdate) -> None:
        key = f"{update.symbol}_{update.interval}"
        if key not in self._candles:
            self._candles[key] = deque(maxlen=self.CANDLE_BUFFER_SIZE)
        dq = self._candles[key]
        candle = update.candle
        # Update in-place if same open_time, else append
        if dq and int(dq[-1].open_time.timestamp()) == int(candle.open_time.timestamp()):
            dq[-1] = candle
        else:
            dq.append(candle)

    async def on_news_update(self, items: list[NewsItem]) -> None:
        """Merge new items, dedup by URL, keep latest NEWS_MAX_ITEMS."""
        existing_urls = {n.url for n in self._news if n.url}
        for item in items:
            if item.url and item.url not in existing_urls:
                self._news.append(item)
                existing_urls.add(item.url)
        # Sort by date, trim
        self._news.sort(key=lambda n: n.published_at, reverse=True)
        self._news = self._news[:self.NEWS_MAX_ITEMS]

    async def on_sentiment(self, data: SentimentData) -> None:
        self._sentiment = data

    # ── Snapshot for WS broadcast ──

    def get_snapshot(self, symbol: str, candle_interval: str = "1h") -> dict:
        """Build full dashboard state for WS broadcast."""
        ctx = self.get_asset_context(symbol)
        ob = self.get_orderbook(symbol)
        candles = self.get_candles(symbol, candle_interval)

        return {
            "market": ctx.model_dump(mode="json") if ctx else None,
            "orderbook": ob.model_dump(mode="json") if ob else None,
            "candles": [c.model_dump(mode="json") for c in candles[-200:]],
            "news": [n.model_dump(mode="json") for n in self._news[:20]],
            "sentiment": self._sentiment.model_dump(mode="json") if self._sentiment else None,
        }
```

## Key Design Choices

| Decision | Why |
|----------|-----|
| Separate `_candles` dict with deque | Candles need deque semantics (bounded, update-in-place) different from TTL store |
| `_news` as sorted list | Need dedup + sort by date; TTL not useful for news (managed by max count) |
| `get_snapshot()` returns plain dict | Ready for `json.dumps()` in WS broadcast; no extra serialization step |
| `model_dump(mode="json")` | Pydantic v2: converts datetime to ISO string, handles all types |

## TODO
- [ ] Create `src/storage/cache.py`
- [ ] Verify: create Cache, call handler methods, check `get_snapshot()`

## Success Criteria
- `on_candle_update()` with same open_time updates in-place (not duplicates)
- `on_news_update()` deduplicates by URL
- `get_snapshot()` returns JSON-serializable dict
- `get_candles()` returns list sorted by time
