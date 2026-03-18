---
phase: 8
title: Dashboard
status: pending
effort: 1.5h
---

# Phase 8: Dashboard (static/index.html)

## Overview

Single HTML file with Chart.js (CDN). Connects to WS, renders live data. Dark theme.

## File to Create

### `static/index.html` (~200 lines)

## Layout

```
+---------------------------------------------------+
|  ETH $3,450.50  (+2.3%)    F&G: 45 (Fear)         |
+---------------------------------------------------+
|                                                    |
|            Candlestick Chart (Chart.js)            |
|            [5m] [15m] [1h] [4h] intervals          |
|                                                    |
+---------------------------------------------------+
|  Orderbook          |  Market Info    |  News      |
|  Bid | Ask          |  Funding: 0.01%|  - Title 1 |
|  3449.5 | 3450.5    |  OI: 567K ETH  |  - Title 2 |
|  3449.0 | 3451.0    |  Vol: $1.2B    |  - Title 3 |
|  ...                |  Spread: 0.03% |  ...       |
+---------------------------------------------------+
```

## Implementation Steps

### 1. HTML Structure
- Dark theme via inline CSS (no build step)
- CSS Grid for layout
- Responsive (works on mobile)

### 2. CDN Dependencies
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-chart-financial@0.2"></script>
<script src="https://cdn.jsdelivr.net/npm/luxon@3"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-adapter-luxon@1"></script>
```

Note: `chartjs-chart-financial` provides candlestick chart type.

### 3. JavaScript Logic (~100 lines)

```javascript
// WebSocket connection
const ws = new WebSocket(`ws://${location.host}/ws`);

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    if (msg.type === "snapshot") {
        renderAll(msg.data);
    } else if (msg.type === "update") {
        mergeUpdate(msg.data);
    }
};

// State
let state = { market: null, orderbook: null, candles: [], news: [], sentiment: null };
let currentInterval = "1h";
let chart = null;

function renderAll(data) {
    state = { ...state, ...data };
    renderPrice();
    renderChart();
    renderOrderbook();
    renderMarketInfo();
    renderNews();
    renderSentiment();
}

function mergeUpdate(data) {
    // Merge each present key
    if (data.market) { state.market = data.market; renderPrice(); renderMarketInfo(); }
    if (data.orderbook) { state.orderbook = data.orderbook; renderOrderbook(); }
    if (data.candle) { updateCandleInChart(data.candle); }
    if (data.news) { state.news = data.news; renderNews(); }
    if (data.sentiment) { state.sentiment = data.sentiment; renderSentiment(); }
}
```

### 4. Chart Rendering

```javascript
function renderChart() {
    if (chart) chart.destroy();
    const ctx = document.getElementById("chart").getContext("2d");
    chart = new Chart(ctx, {
        type: "candlestick",
        data: {
            datasets: [{
                label: "ETH",
                data: state.candles.map(c => ({
                    x: new Date(c.open_time).getTime(),
                    o: c.open, h: c.high, l: c.low, c: c.close,
                })),
            }],
        },
        options: {
            scales: { x: { type: "time" } },
            plugins: { legend: { display: false } },
        },
    });
}

function updateCandleInChart(candleUpdate) {
    // If same interval, update last candle or append
    if (candleUpdate.interval !== currentInterval) return;
    const c = candleUpdate.candle;
    const point = { x: new Date(c.open_time).getTime(), o: c.open, h: c.high, l: c.low, c: c.close };
    const dataset = chart.data.datasets[0].data;
    if (dataset.length > 0 && dataset[dataset.length - 1].x === point.x) {
        dataset[dataset.length - 1] = point;
    } else {
        dataset.push(point);
        if (dataset.length > 200) dataset.shift();
    }
    chart.update("none"); // no animation
}
```

### 5. Interval Selector

```javascript
function setInterval(interval) {
    currentInterval = interval;
    ws.send(JSON.stringify({ type: "set_interval", interval }));
    // Highlight active button
    document.querySelectorAll(".interval-btn").forEach(b => b.classList.remove("active"));
    document.querySelector(`[data-interval="${interval}"]`).classList.add("active");
}
```

### 6. Orderbook Rendering

Simple table. Top 10 levels per side. Color-coded bars for size.

```javascript
function renderOrderbook() {
    const ob = state.orderbook;
    if (!ob) return;
    // Render bids (green) and asks (red) as mirrored tables
    // Show spread at center
    // Show imbalance bar
}
```

### 7. Fear & Greed Display

Circular gauge or colored badge:
- 0-24: Extreme Fear (dark red)
- 25-44: Fear (orange)
- 45-55: Neutral (yellow)
- 56-74: Greed (light green)
- 75-100: Extreme Greed (green)

### 8. News Feed

Scrollable list, max 20 items. Each shows:
- Source badge (colored by source)
- Title (clickable link)
- Time ago (e.g. "2h ago")

## CSS Theme (Dark)

```css
:root {
    --bg: #0d1117;
    --surface: #161b22;
    --border: #30363d;
    --text: #e6edf3;
    --text-dim: #8b949e;
    --green: #3fb950;
    --red: #f85149;
    --accent: #58a6ff;
}
```

## TODO
- [ ] Create `static/index.html`
- [ ] Verify: chart renders with mock data
- [ ] Verify: WS connects and updates live
- [ ] Verify: interval switching works
- [ ] Test responsive layout on mobile viewport

## Success Criteria
- Dashboard loads and displays all sections
- Candlestick chart renders correctly
- Orderbook shows live bid/ask levels
- News feed updates without page refresh
- Fear & Greed displays with color coding
- Interval buttons switch chart data
