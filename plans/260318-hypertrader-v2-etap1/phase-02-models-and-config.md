---
phase: 2
title: Models and Config
status: pending
effort: 1.5h
---

# Phase 2: Pydantic Models + Config

## Reference
- `../BITCOIN/src/models/market.py` -- adapt Candle, OrderBookLevel, OrderBookSnapshot, AssetContext, NewsItem, SentimentSnapshot
- `../BITCOIN/src/models/config.py` -- adapt Settings with YAML loader
- `../BITCOIN/config/settings.yaml` -- adapt for ETH-only, simpler config

## Files to Create

### 1. `src/models/market.py` (~80 lines)

Key models with `float` (not Decimal -- Stage 1 simplification):

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class Candle(BaseModel):
    symbol: str
    interval: str
    open_time: datetime
    close_time: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float
    trades_count: int

    @classmethod
    def from_hyperliquid(cls, data: dict) -> "Candle":
        """Parse Hyperliquid candleSnapshot response item.
        Fields: t, T, s, i, o, h, l, c, v, n -- all prices as strings."""
        ...

class OrderBookLevel(BaseModel):
    price: float
    size: float
    orders_count: int

class OrderBookSnapshot(BaseModel):
    symbol: str
    bids: list[OrderBookLevel]
    asks: list[OrderBookLevel]
    timestamp: datetime
    spread: float | None = None
    spread_pct: float | None = None
    bid_total: float | None = None
    ask_total: float | None = None
    imbalance: float | None = None

    def compute_derived(self) -> None:
        """Calculate spread, totals, imbalance from bids/asks."""
        ...

class AssetContext(BaseModel):
    """Parsed from metaAndAssetCtxs response for ETH."""
    symbol: str
    mark_price: float
    mid_price: float | None = None
    oracle_price: float
    funding_rate: float          # hourly rate, e.g. 0.0001
    open_interest: float         # in ETH
    day_volume: float            # 24h notional in USDC
    prev_day_price: float
    premium: float | None = None
    price_change_pct: float | None = None
    timestamp: datetime

class CandleUpdate(BaseModel):
    """Wrapper for EventBus candle events."""
    symbol: str
    interval: str
    candle: Candle
    is_closed: bool = False
```

### 2. `src/models/news.py` (~40 lines)

```python
from pydantic import BaseModel
from datetime import datetime
from enum import Enum
from typing import Optional

class NewsSource(str, Enum):
    COINDESK = "coindesk"
    COINTELEGRAPH = "cointelegraph"
    DECRYPT = "decrypt"
    THEBLOCK = "theblock"

class NewsItem(BaseModel):
    title: str
    url: str | None = None
    source: NewsSource
    published_at: datetime
    summary: str | None = None

class SentimentData(BaseModel):
    """Fear & Greed Index snapshot."""
    value: int                  # 0-100
    classification: str         # "Extreme Fear", "Fear", etc.
    timestamp: datetime
```

### 3. `src/config.py` (~90 lines)

Simplified vs BITCOIN -- no trading/AI/signal configs. YAML + env var override.

```python
from pydantic import BaseModel
from pydantic_settings import BaseSettings
from pathlib import Path
from yaml import safe_load
from typing import Optional

class RSSFeedConfig(BaseModel):
    url: str
    name: str

class CollectorConfig(BaseModel):
    candle_intervals: list[str] = ["5m", "15m", "1h", "4h"]
    candle_history_count: int = 500      # candles to fetch on startup per interval
    market_poll_sec: int = 10
    orderbook_poll_sec: int = 30
    news_poll_sec: int = 300
    sentiment_poll_sec: int = 3600
    orderbook_levels: int = 20

class ServerConfig(BaseModel):
    host: str = "0.0.0.0"
    port: int = 8080
    ws_broadcast_sec: float = 1.0        # throttle WS broadcasts

class Settings(BaseSettings):
    coin: str = "ETH"
    log_level: str = "INFO"
    hyperliquid_url: str = "https://api.hyperliquid.xyz/info"
    rate_limit_budget: float = 1100.0    # weight/min (conservative vs 1200 max)

    collector: CollectorConfig = CollectorConfig()
    server: ServerConfig = ServerConfig()
    rss_feeds: list[RSSFeedConfig] = []

    class Config:
        env_file = ".env"
        env_prefix = "HT_"

    @classmethod
    def load(cls, path: str | Path = "config/settings.yaml") -> "Settings":
        """Load from YAML, overlay with env vars."""
        ...
```

### 4. `config/settings.yaml`

```yaml
coin: "ETH"
log_level: "INFO"

collector:
  candle_intervals: ["5m", "15m", "1h", "4h"]
  candle_history_count: 500
  market_poll_sec: 10
  orderbook_poll_sec: 30
  news_poll_sec: 300
  sentiment_poll_sec: 3600

server:
  host: "0.0.0.0"
  port: 8080

rss_feeds:
  - url: "https://www.coindesk.com/arc/outboundfeeds/rss/"
    name: "CoinDesk"
  - url: "https://cointelegraph.com/rss"
    name: "CoinTelegraph"
  - url: "https://decrypt.co/feed"
    name: "Decrypt"
  - url: "https://www.theblock.co/rss.xml"
    name: "The Block"
```

## Key Adaptations from BITCOIN

| BITCOIN | HyperTrader | Why |
|---------|-------------|-----|
| `Decimal` for prices | `float` | No trading in Stage 1; simpler JSON serialization |
| `AssetMeta`, `VenueFunding`, `PredictedFunding` | Removed | Not needed for dashboard |
| Complex `Settings.load()` with 15+ sections | Simple load with 3 sections | YAGNI |
| `CollectorConfig.coins: list[str]` | `Settings.coin: str` | Single asset (ETH) |
| `NewsSource` enum includes cryptopanic, telegram | Only RSS sources | YAGNI |
| `NewsSentiment` enum | Removed | No AI sentiment in Stage 1 |

## TODO
- [ ] Create `src/models/market.py`
- [ ] Create `src/models/news.py`
- [ ] Create `src/config.py`
- [ ] Create `config/settings.yaml`
- [ ] Verify: `python -c "from src.config import Settings; s = Settings.load(); print(s.coin)"`

## Success Criteria
- All models instantiate with valid data
- `Settings.load()` reads YAML correctly
- `Candle.from_hyperliquid()` parses raw API dict
- `OrderBookSnapshot.compute_derived()` calculates spread/imbalance
