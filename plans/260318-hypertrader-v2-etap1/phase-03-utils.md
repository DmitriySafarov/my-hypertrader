---
phase: 3
title: Utils
status: pending
effort: 30m
---

# Phase 3: Utilities

## Reference
- `../BITCOIN/src/utils/rate_limiter.py` -- token bucket, copy nearly as-is
- `../BITCOIN/src/utils/logger.py` -- structured logging

## Files to Create

### 1. `src/utils/logger.py` (~30 lines)

Simple structured logging with `logging` stdlib. No external deps.

```python
import logging
import sys

def setup_logging(level: str = "INFO") -> None:
    """Configure root logger. Call once at startup."""
    logging.basicConfig(
        level=getattr(logging, level.upper(), logging.INFO),
        format="%(asctime)s | %(levelname)-7s | %(name)s | %(message)s",
        datefmt="%H:%M:%S",
        stream=sys.stdout,
    )

def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)
```

### 2. `src/utils/rate-limiter.py` (~40 lines)

Token bucket. Adapted from BITCOIN -- identical logic.

```python
import asyncio
import time

class RateLimiter:
    """Token bucket rate limiter for Hyperliquid API.

    Budget: 1200 weight/min max. We use 1100 as safety margin.
    Most endpoints: 20 weight. l2Book: 2 weight.
    """
    def __init__(self, budget_per_min: float = 1100.0):
        self._max = budget_per_min
        self._rate = budget_per_min / 60.0  # tokens per second
        self._tokens = budget_per_min
        self._lock = asyncio.Lock()
        self._last_refill: float | None = None

    async def acquire(self, weight: int = 20) -> None:
        """Wait until `weight` tokens available, then consume them."""
        while True:
            async with self._lock:
                self._refill()
                if self._tokens >= weight:
                    self._tokens -= weight
                    return
                wait = (weight - self._tokens) / self._rate
            await asyncio.sleep(wait)

    def _refill(self) -> None:
        now = time.monotonic()
        if self._last_refill is None:
            self._last_refill = now
            return
        elapsed = now - self._last_refill
        self._tokens = min(self._max, self._tokens + elapsed * self._rate)
        self._last_refill = now
```

## Key Point
- RateLimiter is shared across all Hyperliquid collectors (market, candles, orderbook)
- Instantiated once in `main.py`, passed to each collector

## TODO
- [ ] Create `src/utils/logger.py`
- [ ] Create `src/utils/rate-limiter.py`
- [ ] Verify import: `python -c "from src.utils.rate_limiter import RateLimiter"`

## Success Criteria
- `setup_logging("DEBUG")` configures root logger
- `RateLimiter(1100).acquire(20)` returns immediately on first call
