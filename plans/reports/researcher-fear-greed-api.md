---
name: Fear & Greed Index API Research Report
description: Comprehensive investigation of Alternative.me Fear & Greed Index API and competitive alternatives for crypto market sentiment data
type: research
date: 2026-03-18
---

# Fear & Greed Index API Research Report

## Executive Summary

Alternative.me's Fear & Greed Index API is the most widely-used, free, publicly-documented crypto sentiment source. It provides daily-updated sentiment data (0-100 scale) without authentication. However, the API has **no official status monitoring, undocumented downtime patterns, and generic 60 req/min rate limits**. For production use, CFGI.io and CoinMarketCap offer more reliable alternatives with higher update frequency and better SLA visibility, though at cost.

---

## 1. Alternative.me Fear & Greed Index API

### Endpoint & Request Format

**Base URL:** `https://api.alternative.me/`
**Endpoint:** `/fng/`
**Method:** GET (no authentication required)

#### Request Parameters
- `limit` [int]: Number of results to return
  - Default: `1` (latest only)
  - `0` = all historical data available
- `format` [string]: `json` or `csv`
  - Default: `json`
- `date_format` [string]: Timezone for timestamps
  - `us` = MM/DD/YYYY
  - `cn` = YYYY/MM/DD (China)
  - `kr` = YYYY/MM/DD (Korea)
  - `world` = DD/MM/YYYY (rest of world)
  - Default: unixtime (seconds since epoch)

#### Example Requests
```
GET https://api.alternative.me/fng/
GET https://api.alternative.me/fng/?limit=10
GET https://api.alternative.me/fng/?limit=0&format=json
GET https://api.alternative.me/fng/?limit=30&date_format=us
```

### Response Format

**Full JSON Response Structure:**
```json
{
  "name": "Fear and Greed Index",
  "data": [
    {
      "value": "40",
      "value_classification": "Fear",
      "timestamp": "1551157200",
      "time_until_update": "68499"
    }
  ],
  "metadata": {
    "error": null
  }
}
```

#### Response Fields
| Field | Type | Description | Values |
|-------|------|-------------|--------|
| `value` | string | Sentiment index (0=extreme fear, 100=extreme greed) | 0-100 |
| `value_classification` | string | Human-readable classification | "Extreme Fear", "Fear", "Neutral", "Greed", "Extreme Greed" |
| `timestamp` | string | Unix timestamp (seconds) | Format varies by `date_format` param |
| `time_until_update` | string | Seconds until next daily update | Numeric seconds |
| `error` | null/string | Error message if present | null on success |

### Update Frequency

**Daily updates** — new value published once per 24 hours. Exact time not publicly documented. `time_until_update` field indicates seconds remaining until next update.

### Rate Limits

- **60 requests per minute** (enforced over 10-minute rolling window)
- No authentication required (no API key needed)
- **HTTP 429** ("Too Many Requests") returned when limit exceeded
- Contact `support@alternative.me` for higher quotas

### Historical Data Availability

**Full coverage since inception** — Use `limit=0` to fetch all available data. No date range filtering supported in API; filtering must be done client-side.

**Data Structure:** Each point is a daily snapshot at a fixed time (likely UTC midnight).

### Error Handling & Downtime Patterns

#### Known Error Responses
| Scenario | HTTP Code | Response |
|----------|-----------|----------|
| Rate limit hit | 429 | Unclear (not documented) |
| Server error | 500-503 | Likely plain error or empty JSON |
| No data available | 200 | `{"data": [], "metadata": {"error": null}}` |

#### Reliability Status
- **No public status page** — Alternative.me does not publish uptime metrics or incident history
- **No documented SLA** — No availability guarantees
- **GitHub issues:** Multiple community wrappers exist but no official repository with issue tracking
- **Downtime patterns:** Anecdotal reports from integrations suggest **occasional 5-30min outages**, but no systematic data available

#### Recommended Handling
```python
# Pseudocode for production use
async def fetch_fng(retries=3, backoff_base=2):
    for attempt in range(retries):
        try:
            response = await http_get(
                'https://api.alternative.me/fng/?limit=1&format=json',
                timeout=5
            )
            if response.status == 429:
                wait = backoff_base ** attempt
                await asyncio.sleep(wait)
                continue
            elif response.status in [500, 502, 503]:
                wait = backoff_base ** attempt
                await asyncio.sleep(wait)
                continue
            elif response.status == 200:
                return response.json()
        except (timeout, connection_error):
            if attempt < retries - 1:
                await asyncio.sleep(backoff_base ** attempt)

    return None  # or cached fallback
```

---

## 2. Alternative Fear & Greed Index APIs

### CoinMarketCap Fear & Greed Index

**Endpoint:** `GET https://pro-api.coinmarketcap.com/v3/fear-and-greed/historical`
**Authentication:** API key required (free Basic tier available)

#### Features
- **Update Frequency:** Daily (same as Alternative.me)
- **Historical Data:** Supported via `start` and `limit` parameters
- **Update Depth:** Uses multiple weighting sources:
  - Bitcoin & Ethereum volatility indices
  - Options Put/Call ratios
  - Social trend keyword searches
  - Complex weighting algorithm (proprietary)

#### Rate Limits (Free Tier)
- **30 calls/minute** (Basic plan)
- **10,000 monthly credits**
- Paid tiers: 60 calls/min (Hobbyist) up to custom limits (Enterprise)
- Rate limit resets every 60 seconds

#### Pros & Cons
**Pros:**
- More sophisticated index calculation (multiple data sources)
- Better reliability (established API infrastructure)
- Free tier available
- Official API with better documentation

**Cons:**
- Requires API key registration
- Same daily update frequency as Alternative.me
- Lower free tier rate limit (30 vs 60 req/min on alternative.me)
- Proprietary weighting methodology

---

### CFGI.io (Crypto Fear Greed Index)

**Endpoint:** Developer API at `cfgi.io`
**Authentication:** API key required

#### Features
- **Update Frequency:** Every 15 minutes (4x faster than Alternative.me)
- **Token Coverage:** 52+ cryptocurrencies (Bitcoin, Ethereum, + 50 altcoins)
- **Algorithms:** 10 distinct sentiment indicators
  - Price sentiment analysis
  - Volatility assessment
  - Volume analysis
  - Technical indicators
  - Social media sentiment
  - Whale movement tracking
  - Order book pressure analysis

#### Historical Data
- **Full history since 2021**
- All 52+ tokens
- Client-side filtering by date available

#### Capabilities
- **Unlimited API calls** (per documentation)
- **Realtime webhooks** for sentiment changes
- **Multi-currency queries** with algorithm selection
- **Full historical data access**

#### Rate Limits & Pricing
- Specific rate limits **not publicly documented**
- Registration required for API access
- ~50 active developers on platform (as of last doc update)
- Pricing tiers exist but not detailed publicly

#### Pros & Cons
**Pros:**
- Much faster update cadence (15 min vs 24 hours)
- Covers 52+ tokens vs just major ones
- 10 different algorithms to choose from
- Unlimited calls (no per-minute throttling)
- Webhooks for real-time updates

**Cons:**
- Undocumented rate limits (may hit undocumented quotas)
- Requires API key registration
- Pricing not transparent
- Smaller platform (50 active devs) = less community validation
- API documentation minimal

---

### CoinGlass Fear & Greed Index

**Endpoint:** `GET https://open-api-v4.coinglass.com/api/index/fear-greed-history`
**Authentication:** Required (`CG-API-KEY` header)

#### Features
- **Focus:** Market sentiment via volatility, volume, momentum
- **Update Frequency:** Not explicitly documented (likely daily like Alternative.me)
- **Data Included:** Bitcoin-centric (may cover other tokens)

#### Rate Limits
- **Not publicly documented** — varies by API tier
- Authentication required (unlike Alternative.me)

#### Pros & Cons
**Pros:**
- Part of broader CoinGlass ecosystem (futures data, liquidation maps, etc.)
- Professional-grade data platform

**Cons:**
- Requires authentication
- Rate limits unknown
- Documentation sparse
- Less sentiment-specific vs purpose-built Fear & Greed APIs
- No free tier clearly stated

---

## 3. Summary Comparison Table

| Feature | Alternative.me | CoinMarketCap | CFGI.io | CoinGlass |
|---------|---|---|---|---|
| **Free Tier** | ✅ Yes | ✅ Yes (30 req/min) | ⚠️ Registration required | ❌ Not clear |
| **Auth Required** | ❌ No | ✅ API Key | ✅ API Key | ✅ API Key |
| **Update Frequency** | Daily | Daily | **15 min** | Daily (est.) |
| **Token Coverage** | 1-5 major | 1-5 major | **52+** | ~1-5 |
| **Rate Limit (req/min)** | 60 | 30 (free) | **Unlimited** (?) | Unknown |
| **Historical Data** | ✅ Full (all-time) | ✅ Via params | ✅ Since 2021 | Likely yes |
| **Status/SLA** | ❌ None | ✅ Professional | ❌ None | ⚠️ Limited |
| **Community Adoption** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Production Readiness** | ⚠️ Basic | ✅ High | ⚠️ Medium | ⚠️ Medium |

---

## 4. Recommendations for HyperTrader

### Primary Choice: **Alternative.me** (Current Plan Baseline)

**Rationale:**
- ✅ No auth required (simplest to integrate)
- ✅ Free with no hidden quotas
- ✅ 60 req/min sufficient for single-process polling
- ✅ Massive community adoption (battle-tested)
- ✅ Full historical data available for backtesting

**Implementation Pattern:**
```python
# Fetch daily sentiment, cache aggressively
async def poll_fear_greed():
    try:
        data = await aiohttp.get(
            'https://api.alternative.me/fng/?limit=1&format=json'
        )
        return data.json()['data'][0]
    except Exception as e:
        # Fall back to cached value + manual retry
        return cache.get('last_fng', default={})
```

**Limitations to Handle:**
- No official downtime/SLA — cache aggressively
- Daily-only updates (not suitable for intraday signals)
- No token-level sentiment (Bitcoin only)

### Fallback: **CoinMarketCap** (If Alternative.me Unreliable)

**When to switch:**
- If Alternative.me has >3 unplanned 30min+ outages per month
- If you need higher confidence in data freshness

**Effort to switch:** Low (both are daily REST endpoints; swap URL + add API key to config)

### Future Enhancement: **CFGI.io** (For Advanced Strategy)

**When to use:**
- When extending to multi-token strategies
- When intraday sentiment changes matter (15-min updates)
- When you want algorithmic flexibility (10 different indicators)

**Cost:** Unknown (may require paid plan)
**Effort:** Medium (new API integration + webhook infrastructure)

---

## 5. Implementation Checklist for HyperTrader

### Phase 1: Alternative.me Integration (Current)
- [ ] Add `/collectors.py` function for FNG polling
- [ ] Cache mechanism: store last value + timestamp in `cache.py`
- [ ] Daily polling (once per 24h, offset from update window)
- [ ] Error handling: 3-retry exponential backoff
- [ ] Dashboard widget: display current FNG value + classification
- [ ] Historical view: fetch full history on startup for charts

### Phase 2: Reliability Hardening
- [ ] Add fallback to CoinMarketCap if Alternative.me fails 3x in a row
- [ ] In-memory "last known good" value for display during outages
- [ ] Alert on staleness (if data >36h old)
- [ ] Log all API failures to detect patterns

### Phase 3: Enhanced Sentiment (Future)
- [ ] Evaluate CFGI.io for multi-token sentiment
- [ ] Webhook-based push updates (vs polling)
- [ ] Combine FNG with on-chain metrics (Whale Wallet Activity, etc.)

---

## 6. Unresolved Questions & Gaps

1. **Alternative.me Uptime:** No public monitoring = must assume 95-98% uptime. How critical is downtime tolerance for HyperTrader?
2. **CFGI.io Rate Limits:** "Unlimited calls" claim lacks verification. What's the actual quota?
3. **CoinMarketCap Sentiment Algorithm:** Exact weighting of sources not disclosed. Suitable for your use case?
4. **Error Response Formats:** Alternative.me API error response structure not documented. Need to test edge cases (500s, 429s).
5. **Historical Data Completeness:** Are there gaps in Alternative.me historical data? Need to validate backtest data integrity.

---

## Sources

- [Alternative.me Crypto Fear & Greed Index](https://alternative.me/crypto/fear-and-greed-index/)
- [Alternative.me Crypto API Documentation](https://alternative.me/crypto/api/)
- [CoinMarketCap Fear & Greed Index](https://coinmarketcap.com/charts/fear-and-greed-index/)
- [CoinMarketCap API Documentation](https://coinmarketcap.com/api/documentation/v1/)
- [CFGI.io Developer API](https://cfgi.io/)
- [CoinGlass Fear & Greed Index API](https://docs.coinglass.com/reference/cryptofear-greedindex)
- [GitHub: fear-and-greed-crypto Python Wrapper](https://github.com/rhettre/fear-and-greed-crypto)
- [GitHub: crypto-feargreed-mcp MCP Server](https://github.com/kukapay/crypto-feargreed-mcp)
- [HTTP 429 Rate Limiting Best Practices - DEV Community](https://dev.to/robertobutti/how-to-handle-api-rate-limits-and-http-429-errors-in-an-easy-and-reliable-way-14e6)
- [CoinMarketCap API Pricing Plans](https://coinmarketcap.com/api/pricing/)
