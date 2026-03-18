---
phase: 6
title: Collectors
status: pending
effort: 4h
---

# Phase 6: BaseCollector + 5 Collectors

## Reference
- `../BITCOIN/src/collectors/base.py` -- ABC pattern
- `../BITCOIN/src/collectors/market_data_collector.py` -- metaAndAssetCtxs
- `../BITCOIN/src/collectors/candle_collector.py` -- candleSnapshot + WS
- `../BITCOIN/src/collectors/orderbook_collector.py` -- l2Book
- `../BITCOIN/src/collectors/news_collector.py` -- RSS via feedparser
- `../BITCOIN/src/collectors/sentiment_collector.py` -- Fear & Greed

## Files to Create

### 1. `src/collectors/base.py` (~25 lines)

Identical to BITCOIN pattern:

```python
from abc import ABC, abstractmethod
from src.utils.logger import get_logger

class BaseCollector(ABC):
    def __init__(self, name: str):
        self.name = name
        self.is_running = False
        self._log = get_logger(name)

    @abstractmethod
    async def start(self) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def health_check(self) -> bool: ...
```

### 2. `src/collectors/market.py` (~90 lines)

**Emits:** `MARKET_DATA` event with `AssetContext` payload.

Adapted from BITCOIN `market_data_collector.py`:
- Remove MarketStore (no persistence)
- Remove multi-coin loop (ETH only -- but still filter by `settings.coin`)
- Remove pandas
- Add EventBus emit

```python
class MarketDataCollector(BaseCollector):
    def __init__(self, settings: Settings, bus: EventBus, rate_limiter: RateLimiter):
        super().__init__("market_data")
        self._settings = settings
        self._bus = bus
        self._limiter = rate_limiter
        self._session: aiohttp.ClientSession | None = None
        self._task: asyncio.Task | None = None

    async def start(self) -> None:
        self._session = aiohttp.ClientSession()
        self.is_running = True
        await self._fetch()  # immediate first fetch
        self._task = asyncio.create_task(self._poll_loop())

    async def stop(self) -> None:
        self.is_running = False
        if self._task: self._task.cancel()
        if self._session: await self._session.close()

    async def health_check(self) -> bool:
        return self.is_running

    async def _poll_loop(self) -> None:
        while self.is_running:
            await asyncio.sleep(self._settings.collector.market_poll_sec)
            try:
                await self._fetch()
            except asyncio.CancelledError:
                return
            except Exception as e:
                self._log.error(f"poll error: {e}")

    async def _fetch(self) -> None:
        await self._limiter.acquire(weight=20)
        body = {"type": "metaAndAssetCtxs"}
        async with self._session.post(
            self._settings.hyperliquid_url, json=body,
            timeout=aiohttp.ClientTimeout(total=15)
        ) as resp:
            if resp.status != 200:
                self._log.error(f"HTTP {resp.status}")
                return
            data = await resp.json()

        if not isinstance(data, list) or len(data) != 2:
            self._log.error("bad response format")
            return

        meta, ctxs = data
        universe = meta.get("universe", [])
        coin = self._settings.coin

        for i, asset_meta in enumerate(universe):
            if asset_meta["name"] != coin or i >= len(ctxs):
                continue
            ctx = ctxs[i]
            mark = float(ctx.get("markPx", 0))
            prev = float(ctx.get("prevDayPx", 0))

            asset = AssetContext(
                symbol=coin,
                mark_price=mark,
                mid_price=float(ctx["midPx"]) if ctx.get("midPx") else None,
                oracle_price=float(ctx.get("oraclePx", 0)),
                funding_rate=float(ctx.get("funding", 0)),
                open_interest=float(ctx.get("openInterest", 0)),
                day_volume=float(ctx.get("dayNtlVlm", 0)),
                prev_day_price=prev,
                premium=float(ctx["premium"]) if ctx.get("premium") else None,
                price_change_pct=((mark - prev) / prev * 100) if prev else None,
                timestamp=datetime.now(timezone.utc),
            )
            await self._bus.emit(MARKET_DATA, asset)
            break
```

### 3. `src/collectors/candles.py` (~120 lines)

**Emits:** `CANDLE_UPDATE` event with `CandleUpdate` payload.

Simplified vs BITCOIN:
- REST-only for Stage 1 (no WebSocket -- simpler, sufficient for dashboard)
- No persistence (no MarketStore, no pandas)
- Fetch history on start, then poll each interval on its period

```python
class CandleCollector(BaseCollector):
    """Fetches candle history on start, then polls periodically.

    Stage 1: REST only. Each interval polled on its own period:
    - 5m  -> poll every 60s (catch close)
    - 15m -> poll every 60s
    - 1h  -> poll every 60s
    - 4h  -> poll every 60s
    All share same poll loop for simplicity. Each poll fetches
    last 2 candles (to catch the just-closed one).
    """

    INTERVAL_MS = {"5m": 300_000, "15m": 900_000, "1h": 3_600_000, "4h": 14_400_000}

    def __init__(self, settings: Settings, bus: EventBus, rate_limiter: RateLimiter):
        super().__init__("candle_collector")
        self._settings = settings
        self._bus = bus
        self._limiter = rate_limiter
        self._session: aiohttp.ClientSession | None = None
        self._task: asyncio.Task | None = None
        self._last_open_times: dict[str, int] = {}  # interval -> last seen open_time_ms

    async def start(self) -> None:
        self._session = aiohttp.ClientSession()
        self.is_running = True
        await self._fetch_history()
        self._task = asyncio.create_task(self._poll_loop())

    async def _fetch_history(self) -> None:
        """Fetch initial candle history for all intervals."""
        coin = self._settings.coin
        count = self._settings.collector.candle_history_count
        now_ms = int(datetime.now(timezone.utc).timestamp() * 1000)

        for interval in self._settings.collector.candle_intervals:
            interval_ms = self.INTERVAL_MS.get(interval, 300_000)
            start_ms = now_ms - (count * interval_ms)
            try:
                candles = await self._fetch_candles(coin, interval, start_ms, now_ms)
                for c in candles:
                    await self._bus.emit(CANDLE_UPDATE, CandleUpdate(
                        symbol=coin, interval=interval, candle=c, is_closed=True
                    ))
                self._log.info(f"history loaded: {interval} ({len(candles)} candles)")
            except Exception as e:
                self._log.error(f"history error {interval}: {e}")

    async def _poll_loop(self) -> None:
        """Poll all intervals every 60s. Detect closed candles via open_time change."""
        while self.is_running:
            await asyncio.sleep(60)
            coin = self._settings.coin
            now_ms = int(datetime.now(timezone.utc).timestamp() * 1000)
            for interval in self._settings.collector.candle_intervals:
                try:
                    # Fetch last 3 candles to detect close
                    interval_ms = self.INTERVAL_MS.get(interval, 300_000)
                    start_ms = now_ms - (3 * interval_ms)
                    candles = await self._fetch_candles(coin, interval, start_ms, now_ms)
                    if not candles:
                        continue
                    latest = candles[-1]
                    ot = int(latest.open_time.timestamp() * 1000)
                    prev_ot = self._last_open_times.get(interval)
                    is_closed = prev_ot is not None and ot != prev_ot
                    self._last_open_times[interval] = ot

                    await self._bus.emit(CANDLE_UPDATE, CandleUpdate(
                        symbol=coin, interval=interval,
                        candle=latest, is_closed=is_closed,
                    ))
                except asyncio.CancelledError:
                    return
                except Exception as e:
                    self._log.error(f"poll error {interval}: {e}")

    async def _fetch_candles(self, coin: str, interval: str,
                              start_ms: int, end_ms: int) -> list[Candle]:
        await self._limiter.acquire(weight=20)
        body = {"type": "candleSnapshot", "req": {
            "coin": coin, "interval": interval,
            "startTime": start_ms, "endTime": end_ms,
        }}
        async with self._session.post(
            self._settings.hyperliquid_url, json=body,
            timeout=aiohttp.ClientTimeout(total=30)
        ) as resp:
            if resp.status != 200:
                return []
            data = await resp.json()
        if not isinstance(data, list):
            return []
        return [Candle.from_hyperliquid(d) for d in data]

    # stop, health_check -- same pattern as MarketDataCollector
```

### 4. `src/collectors/orderbook.py` (~70 lines)

**Emits:** `ORDERBOOK` event with `OrderBookSnapshot` payload.

Adapted from BITCOIN. Key: l2Book costs only 2 weight.

```python
class OrderbookCollector(BaseCollector):
    def __init__(self, settings: Settings, bus: EventBus, rate_limiter: RateLimiter):
        super().__init__("orderbook")
        ...

    async def _fetch(self) -> None:
        await self._limiter.acquire(weight=2)  # l2Book = 2 weight!
        body = {"type": "l2Book", "coin": self._settings.coin}
        async with self._session.post(...) as resp:
            data = await resp.json()

        levels = data.get("levels", [])
        if len(levels) < 2:
            return

        n = self._settings.collector.orderbook_levels
        bids = [OrderBookLevel(price=float(b["px"]), size=float(b["sz"]),
                               orders_count=b["n"]) for b in levels[0][:n]]
        asks = [OrderBookLevel(price=float(a["px"]), size=float(a["sz"]),
                               orders_count=a["n"]) for a in levels[1][:n]]

        snapshot = OrderBookSnapshot(
            symbol=self._settings.coin,
            bids=bids, asks=asks,
            timestamp=datetime.fromtimestamp(data["time"] / 1000, tz=timezone.utc),
        )
        snapshot.compute_derived()
        await self._bus.emit(ORDERBOOK, snapshot)
```

### 5. `src/collectors/news.py` (~80 lines)

**Emits:** `NEWS_UPDATE` event with `list[NewsItem]` payload.

Adapted from BITCOIN `news_collector.py`:
- Use aiohttp to fetch RSS (feedparser is sync, so fetch content async then parse)
- Process top 5 entries per feed
- Dedup by URL within batch

```python
class NewsCollector(BaseCollector):
    SOURCE_MAP = {
        "coindesk": NewsSource.COINDESK,
        "cointelegraph": NewsSource.COINTELEGRAPH,
        "decrypt": NewsSource.DECRYPT,
        "theblock": NewsSource.THEBLOCK,
    }

    def __init__(self, settings: Settings, bus: EventBus):
        super().__init__("news")
        self._settings = settings
        self._bus = bus
        self._seen_urls: set[str] = set()
        ...

    async def _fetch_all(self) -> None:
        """Fetch all RSS feeds concurrently, emit batch."""
        items: list[NewsItem] = []
        async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=15)) as session:
            tasks = [self._fetch_feed(session, feed) for feed in self._settings.rss_feeds]
            results = await asyncio.gather(*tasks, return_exceptions=True)
            for result in results:
                if isinstance(result, list):
                    items.extend(result)

        if items:
            await self._bus.emit(NEWS_UPDATE, items)

    async def _fetch_feed(self, session: aiohttp.ClientSession,
                           feed: RSSFeedConfig) -> list[NewsItem]:
        async with session.get(feed.url) as resp:
            content = await resp.text()
        parsed = feedparser.parse(content)
        if parsed.bozo:
            self._log.warning(f"bozo feed {feed.name}: {type(parsed.bozo_exception).__name__}")

        items = []
        source = self.SOURCE_MAP.get(feed.name.lower(), NewsSource.COINDESK)
        for entry in parsed.entries[:5]:
            url = getattr(entry, "link", None)
            if not url or url in self._seen_urls:
                continue
            self._seen_urls.add(url)
            pub = getattr(entry, "published_parsed", None)
            if not pub or len(pub) < 6:
                continue
            items.append(NewsItem(
                title=getattr(entry, "title", ""),
                url=url,
                source=source,
                published_at=datetime(*pub[:6], tzinfo=timezone.utc),
                summary=getattr(entry, "summary", None),
            ))
        return items
```

### 6. `src/collectors/sentiment.py` (~60 lines)

**Emits:** `SENTIMENT` event with `SentimentData` payload.

Nearly identical to BITCOIN `sentiment_collector.py`, adapted:

```python
class SentimentCollector(BaseCollector):
    URL = "https://api.alternative.me/fng/?limit=1&format=json"

    def __init__(self, settings: Settings, bus: EventBus):
        super().__init__("sentiment")
        self._settings = settings
        self._bus = bus
        ...

    async def _fetch(self) -> None:
        async with self._session.get(self.URL) as resp:
            if resp.status != 200:
                self._log.warning(f"API returned {resp.status}")
                return
            data = await resp.json()

        if not isinstance(data, dict) or "data" not in data or not data["data"]:
            self._log.error("unexpected response format")
            return

        item = data["data"][0]
        value = int(item.get("value", 0))
        if not 0 <= value <= 100:
            self._log.warning(f"value out of range: {value}")
            return

        sentiment = SentimentData(
            value=value,
            classification=item.get("value_classification", ""),
            timestamp=datetime.now(timezone.utc),
        )
        await self._bus.emit(SENTIMENT, sentiment)
```

## Event Flow Summary

```
MarketDataCollector._fetch()
  -> AssetContext(...)
  -> bus.emit(MARKET_DATA, ctx)
     -> cache.on_market_data(ctx)    # stores in cache
     -> ws_manager.on_market_data(ctx)  # broadcasts to browsers

CandleCollector._poll_loop()
  -> Candle.from_hyperliquid(data)
  -> bus.emit(CANDLE_UPDATE, CandleUpdate(...))
     -> cache.on_candle_update(update)
     -> ws_manager.on_candle_update(update)

OrderbookCollector._fetch()
  -> OrderBookSnapshot(...)
  -> bus.emit(ORDERBOOK, snapshot)
     -> cache.on_orderbook(snapshot)
     -> ws_manager.on_orderbook(snapshot)

NewsCollector._fetch_all()
  -> [NewsItem(...), ...]
  -> bus.emit(NEWS_UPDATE, items)
     -> cache.on_news_update(items)
     -> ws_manager.on_news_update(items)

SentimentCollector._fetch()
  -> SentimentData(...)
  -> bus.emit(SENTIMENT, data)
     -> cache.on_sentiment(data)
     -> ws_manager.on_sentiment(data)
```

## Rate Budget Estimate (per minute)

| Collector | Weight | Frequency | Weight/min |
|-----------|--------|-----------|------------|
| Market    | 20     | every 10s | 120        |
| Candles   | 20 x 4 intervals | every 60s | 80 |
| Orderbook | 2      | every 30s | 4          |
| **Total** |        |           | **204**    |

Budget: 1100/min. Usage: ~204/min = **18.5%** -- plenty of headroom.

## TODO
- [ ] Create `src/collectors/base.py`
- [ ] Create `src/collectors/market.py`
- [ ] Create `src/collectors/candles.py`
- [ ] Create `src/collectors/orderbook.py`
- [ ] Create `src/collectors/news.py`
- [ ] Create `src/collectors/sentiment.py`
- [ ] Verify each collector starts/stops cleanly

## Success Criteria
- All 5 collectors start, poll, and emit events
- Rate limiter prevents exceeding budget
- News collector handles bozo feeds gracefully
- Candle collector detects closed candles via open_time change
- All collectors handle network errors without crashing
