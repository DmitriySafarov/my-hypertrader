---
title: Hyperliquid REST API Research Report
author: Researcher
date: 2026-03-18
---

# Hyperliquid REST API for ETH Market Data Collection

## Executive Summary

Hyperliquid provides a robust REST API for fetching real-time market data. All endpoints use **POST** requests to `https://api.hyperliquid.xyz/info` with JSON bodies. The API uses a weighted rate limiting system: 1200 weight per minute per IP, with most info endpoints costing 20 weight each (= ~60 requests/minute).

---

## 1. Mid Price / Mark Price (metaAndAssetCtxs)

**Purpose:** Get current mark price, mid price, and other perpetual asset metadata.

### Request

```json
{
  "type": "metaAndAssetCtxs",
  "dex": ""
}
```

**Parameters:**
- `type` (required): `"metaAndAssetCtxs"`
- `dex` (optional): Perp dex name, defaults to empty string (first perp dex)

### Response

Returns an array with two elements: `[universe_metadata, asset_contexts]`

**Relevant fields from `asset_contexts` array** (each asset is an object):
```json
{
  "name": "ETH",
  "markPx": "3450.50",
  "midPx": "3450.45",
  "oraclePx": "3450.48",
  "dayNtlVlm": "1234567890.50",
  "funding": "0.0001",
  "premium": "0.00005",
  "openInterest": "567890.12",
  "impactPxs": ["3449.50", "3451.50"],
  "prevDayPx": "3440.00"
}
```

**Key Fields:**
- `markPx`: Mark price (string, decimal format)
- `midPx`: Mid price = (best bid + best ask) / 2
- `oraclePx`: Oracle price
- `dayNtlVlm`: 24-hour notional volume in USDC
- `funding`: Current hourly funding rate (decimal, e.g., 0.0001 = 0.01%)
- `openInterest`: Current open interest in base asset quantity
- `impactPxs`: [buy_impact_price, sell_impact_price]

**Rate Limit Weight:** 20

---

## 2. Funding Rate

**Option A: Current Funding (via metaAndAssetCtxs)**

Use the `funding` field from response above.

**Option B: Historical Funding Rates (fundingHistory)**

```json
{
  "type": "fundingHistory",
  "coin": "ETH",
  "startTime": 1710777600000,
  "endTime": 1710864000000
}
```

**Parameters:**
- `type` (required): `"fundingHistory"`
- `coin` (required): Asset symbol ("ETH", "BTC", "SOL", etc.)
- `startTime` (required): Start time in milliseconds (inclusive)
- `endTime` (optional): End time in milliseconds (inclusive), defaults to now

### Response

```json
[
  {
    "time": 1710780000000,
    "funding": "0.0001",
    "premium": "0.00005"
  },
  {
    "time": 1710783600000,
    "funding": "0.00012",
    "premium": "0.00006"
  }
]
```

**Fields:**
- `time`: Unix timestamp in milliseconds when rate was applied
- `funding`: Funding rate for that period (decimal)
- `premium`: Price premium component used in calculation

**Rate Limit Weight:** 20

---

## 3. Open Interest

**Via metaAndAssetCtxs** (see section 1)

The `openInterest` field contains current ETH open interest in base asset quantity.

**Example:** `"openInterest": "567890.12"` = 567,890.12 ETH

**Rate Limit Weight:** 20 (for metaAndAssetCtxs)

---

## 4. 24-Hour Volume

**Via metaAndAssetCtxs** (see section 1)

The `dayNtlVlm` field contains 24-hour notional volume in USDC.

**Example:** `"dayNtlVlm": "1234567890.50"` = $1,234,567,890.50

**Rate Limit Weight:** 20

---

## 5. OHLCV Candles

**Endpoint:** `candleSnapshot`

### Request

```json
{
  "type": "candleSnapshot",
  "req": {
    "coin": "ETH",
    "interval": "1h",
    "startTime": 1710777600000,
    "endTime": 1710864000000
  }
}
```

**Parameters:**
- `type` (required): `"candleSnapshot"`
- `coin` (required): Asset symbol ("ETH", "BTC", etc.)
- `interval` (required): Candle interval
- `startTime` (required): Start time in milliseconds (inclusive)
- `endTime` (required): End time in milliseconds (inclusive)

**Supported Intervals:**
- `"1m"`, `"3m"`, `"5m"`, `"15m"`, `"30m"` (minute intervals)
- `"1h"`, `"2h"`, `"4h"`, `"8h"`, `"12h"` (hour intervals)
- `"1d"`, `"3d"` (day intervals)
- `"1w"` (weekly)
- `"1M"` (monthly)

### Response

```json
[
  {
    "t": 1710777600000,
    "T": 1710781199999,
    "o": "3440.50",
    "h": "3460.00",
    "l": "3435.00",
    "c": "3450.50",
    "v": "12345.67",
    "n": 5432,
    "s": "ETH",
    "i": "1h"
  },
  {
    "t": 1710781200000,
    "T": 1710784799999,
    "o": "3450.50",
    "h": "3465.00",
    "l": "3448.00",
    "c": "3455.25",
    "v": "9876.54",
    "n": 4123,
    "s": "ETH",
    "i": "1h"
  }
]
```

**Fields:**
- `t`: Candle open time (milliseconds)
- `T`: Candle close time (milliseconds)
- `o`: Open price (string)
- `h`: High price (string)
- `l`: Low price (string)
- `c`: Close price (string)
- `v`: Volume (base asset quantity, string)
- `n`: Number of trades in candle
- `s`: Symbol ("ETH")
- `i`: Interval ("1h")

**Gotchas:**
- Only the most recent **5000 candles** are available for any interval
- For `"1m"` interval: ~3.5 days of history
- For `"1d"` interval: ~13.7 years of history
- Time parameters must be within available history or you'll get empty results
- All prices and volumes are returned as **strings** (need to parse to float)

**Rate Limit Weight:** 20 + additional weight per 60 items returned
- Example: 61-120 items = additional weight 2, total = 22

---

## 6. L2 Order Book (l2Book)

**Purpose:** Get up to 20 levels of bids and asks (order book snapshot).

### Request

```json
{
  "type": "l2Book",
  "coin": "ETH",
  "nSigFigs": null
}
```

**Parameters:**
- `type` (required): `"l2Book"`
- `coin` (required): Asset symbol ("ETH", "BTC", etc.)
- `nSigFigs` (optional): Aggregate levels to N significant figures
  - Valid values: `2`, `3`, `4`, `5`, or `null` (no aggregation)
  - Default: `null` (full precision)
- `mantissa` (optional, only if nSigFigs=5): Number of mantissa digits
  - Valid values: `1`, `2`, or `5`
  - Only allowed when `nSigFigs=5`

### Response

```json
{
  "coin": "ETH",
  "time": 1710780000000,
  "levels": [
    [
      {"px": "3449.50", "sz": "12.34", "n": 5},
      {"px": "3449.00", "sz": "56.78", "n": 12},
      {"px": "3448.50", "sz": "34.56", "n": 8}
    ],
    [
      {"px": "3450.50", "sz": "23.45", "n": 7},
      {"px": "3451.00", "sz": "67.89", "n": 15},
      {"px": "3451.50", "sz": "45.67", "n": 9}
    ]
  ]
}
```

**Response Structure:**
- `coin`: Asset symbol
- `time`: Snapshot timestamp (milliseconds)
- `levels`: Array with exactly 2 sub-arrays:
  - `levels[0]`: Bids (buy orders, descending price)
  - `levels[1]`: Asks (sell orders, ascending price)

**Level Fields:**
- `px`: Price (string, decimal format)
- `sz`: Size in base asset (string, decimal format)
- `n`: Number of orders at this price level (integer)

**Gotchas:**
- Returns **at most 20 levels per side** (bids and asks)
- If fewer levels exist, returns what's available
- Bids sorted descending (highest price first)
- Asks sorted ascending (lowest price first)
- All prices and sizes are **strings** (parse to float)
- `nSigFigs` aggregation reduces granularity but compresses response
- Snapshot is taken at exact `time` - not real-time streaming

**Rate Limit Weight:** 2 (low cost!)

---

## Rate Limits & Gotchas

### Weight System
- **Global limit:** 1200 weight per minute per IP address
- **Info endpoint weights:**
  - `l2Book`, `allMids`, `clearinghouseState`, `orderStatus`, `spotClearinghouseState`, `exchangeStatus`: **weight 2**
  - Most other info endpoints (metaAndAssetCtxs, candleSnapshot, fundingHistory): **weight 20**
  - `userRole`: **weight 60**
  - Some endpoints add weight per items returned (e.g., candleSnapshot adds weight per 60 items)

### Effective Request Rates
- Most info endpoints: 1200 / 20 = **60 requests/minute** (~1 per second)
- `l2Book`: 1200 / 2 = **600 requests/minute** (~10 per second)
- Conservative approach: **1 request/second** per info endpoint

### Address-Based Throttling
- For authenticated operations (not relevant for data collection), rate limit is "1 request per 1 USDC traded"
- Initial buffer: 10,000 requests
- When throttled: 1 request per 10 seconds

### Other Gotchas
- All numeric values (prices, volumes, sizes) are returned as **strings**, not floats
- Timestamps are always in **milliseconds**, not seconds
- No authentication required for public info endpoints
- HTTP **POST only** (not GET with query params)
- Content-Type header: `application/json`
- Empty result arrays are valid responses (e.g., if time range has no data)
- **Candle data retention:** Only 5000 most recent candles per interval
- **Order book:** Snapshot only, not real-time streaming (use WebSocket for streaming)

---

## Implementation Notes for HyperTrader

### Recommended Request Pattern
1. **metaAndAssetCtxs** (weight 20): Get markPx, midPx, funding, openInterest, dayNtlVlm in one call
2. **l2Book** (weight 2): Get order book separately (cheaper, can call more frequently)
3. **candleSnapshot** (weight 20+): Fetch candles on-demand or on a longer interval

### Best Practices
- Use `l2Book` frequently for order book updates (costs only 2 weight)
- Use `metaAndAssetCtxs` for periodic snapshots (20 weight for all data in one call is efficient)
- Cache `metaAndAssetCtxs` for 10-30 seconds if polling (each call = 20 weight)
- For candles: Request in batches; store locally once fetched
- Parse all numeric strings to float immediately upon receipt
- Handle empty response arrays gracefully

### Connection Details
- **Mainnet:** `https://api.hyperliquid.xyz/info`
- **Testnet:** `https://api.hyperliquid-testnet.xyz/info`
- **Method:** POST
- **Content-Type:** `application/json`
- **Response Format:** JSON
- **No authentication** required for public data endpoints

---

## Summary Table

| Endpoint | Type | Request Weight | Rate Limit | Purpose |
|----------|------|---|---|---|
| metaAndAssetCtxs | POST | 20 | ~60/min | Mark price, mid price, funding, OI, volume |
| l2Book | POST | 2 | ~600/min | Order book (20 levels each side) |
| candleSnapshot | POST | 20+ | ~60/min | OHLCV candles with configurable intervals |
| fundingHistory | POST | 20 | ~60/min | Historical funding rates |

---

## Unresolved Questions

None at this time. All core endpoints, request/response formats, rate limits, and gotchas are documented.

---

## Sources

- [Hyperliquid API Documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api)
- [Info Endpoint Reference](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint)
- [Perpetuals Data Fields](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals)
- [Rate Limits and User Limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits)
- [metaAndAssetCtxs Reference (Chainstack)](https://docs.chainstack.com/reference/hyperliquid-info-meta-and-asset-ctxs)
- [l2Book Reference (Chainstack)](https://docs.chainstack.com/reference/hyperliquid-info-l2-book)
- [candleSnapshot Reference (Chainstack)](https://docs.chainstack.com/reference/hyperliquid-info-candle-snapshot)
- [fundingHistory Reference (Chainstack)](https://docs.chainstack.com/reference/hyperliquid-info-funding-history)
