---
phase: 7
title: Server
status: pending
effort: 1.5h
---

# Phase 7: FastAPI + WebSocket Manager

## Reference
- BITCOIN project does not have a web server (uses Telegram). This is new.

## Files to Create

### 1. `src/server/app.py` (~50 lines)

FastAPI app factory. Serves:
- `GET /` -- static dashboard (index.html)
- `GET /api/snapshot` -- REST endpoint for initial state
- `WS /ws` -- WebSocket for live updates

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
from pathlib import Path

from src.storage.cache import Cache
from src.server.websocket import WSManager
from src.config import Settings

def create_app(settings: Settings, cache: Cache, ws_manager: WSManager) -> FastAPI:
    app = FastAPI(title="HyperTrader", version="2.0.0")

    static_dir = Path(__file__).parent.parent.parent / "static"
    app.mount("/static", StaticFiles(directory=str(static_dir)), name="static")

    @app.get("/")
    async def index():
        return FileResponse(str(static_dir / "index.html"))

    @app.get("/api/snapshot")
    async def snapshot():
        """Full dashboard state for initial load."""
        return cache.get_snapshot(
            symbol=settings.coin,
            candle_interval="1h",  # default chart interval
        )

    @app.websocket("/ws")
    async def websocket_endpoint(ws: WebSocket):
        await ws_manager.connect(ws)
        try:
            # Send initial snapshot
            data = cache.get_snapshot(settings.coin, "1h")
            await ws.send_json({"type": "snapshot", "data": data})
            # Keep alive -- read messages (e.g. interval change requests)
            while True:
                msg = await ws.receive_json()
                if msg.get("type") == "set_interval":
                    interval = msg.get("interval", "1h")
                    data = cache.get_snapshot(settings.coin, interval)
                    await ws.send_json({"type": "snapshot", "data": data})
        except WebSocketDisconnect:
            ws_manager.disconnect(ws)
        except Exception:
            ws_manager.disconnect(ws)

    return app
```

### 2. `src/server/websocket.py` (~80 lines)

WebSocket connection manager. Subscribes to EventBus for live push.

```python
import asyncio
import json
from datetime import datetime, timezone
from fastapi import WebSocket
from typing import Any

from src.utils.logger import get_logger

log = get_logger(__name__)

class WSManager:
    """Manages WebSocket connections and broadcasts updates.

    Subscribes to EventBus events and pushes to all connected clients.
    Throttles broadcasts to avoid overwhelming browsers.
    """
    def __init__(self, broadcast_interval: float = 1.0):
        self._connections: list[WebSocket] = []
        self._broadcast_interval = broadcast_interval
        self._pending: dict[str, Any] = {}  # event_type -> latest data
        self._task: asyncio.Task | None = None

    async def start(self) -> None:
        """Start the broadcast loop."""
        self._task = asyncio.create_task(self._broadcast_loop())

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()

    async def connect(self, ws: WebSocket) -> None:
        await ws.accept()
        self._connections.append(ws)
        log.info(f"client connected ({len(self._connections)} total)")

    def disconnect(self, ws: WebSocket) -> None:
        if ws in self._connections:
            self._connections.remove(ws)
        log.info(f"client disconnected ({len(self._connections)} total)")

    # ── EventBus handlers ──
    # Each handler stores latest data; broadcast loop sends batched updates

    async def on_market_data(self, ctx) -> None:
        self._pending["market"] = ctx.model_dump(mode="json")

    async def on_candle_update(self, update) -> None:
        self._pending["candle"] = update.model_dump(mode="json")

    async def on_orderbook(self, ob) -> None:
        self._pending["orderbook"] = ob.model_dump(mode="json")

    async def on_news_update(self, items: list) -> None:
        self._pending["news"] = [i.model_dump(mode="json") for i in items[:20]]

    async def on_sentiment(self, data) -> None:
        self._pending["sentiment"] = data.model_dump(mode="json")

    # ── Broadcast loop ──

    async def _broadcast_loop(self) -> None:
        """Periodically broadcast pending updates to all clients."""
        while True:
            await asyncio.sleep(self._broadcast_interval)
            if not self._pending or not self._connections:
                continue

            message = {"type": "update", "data": dict(self._pending)}
            self._pending.clear()

            dead: list[WebSocket] = []
            for ws in self._connections:
                try:
                    await ws.send_json(message)
                except Exception:
                    dead.append(ws)

            for ws in dead:
                self.disconnect(ws)
```

## Event Integration

In `main.py`, after creating EventBus, Cache, and WSManager:

```python
# Cache subscribes to EventBus
bus.subscribe(MARKET_DATA, cache.on_market_data)
bus.subscribe(CANDLE_UPDATE, cache.on_candle_update)
bus.subscribe(ORDERBOOK, cache.on_orderbook)
bus.subscribe(NEWS_UPDATE, cache.on_news_update)
bus.subscribe(SENTIMENT, cache.on_sentiment)

# WSManager subscribes to EventBus
bus.subscribe(MARKET_DATA, ws_manager.on_market_data)
bus.subscribe(CANDLE_UPDATE, ws_manager.on_candle_update)
bus.subscribe(ORDERBOOK, ws_manager.on_orderbook)
bus.subscribe(NEWS_UPDATE, ws_manager.on_news_update)
bus.subscribe(SENTIMENT, ws_manager.on_sentiment)
```

## Key Design: Throttled Broadcast

Problem: Market data updates every 10s, orderbook every 30s. Sending each event individually would flood browsers.

Solution: WSManager buffers latest state per event type. Broadcast loop runs every 1s, sends batched `{"type": "update", "data": {...}}` with all pending updates. Client merges into its state.

## WS Message Protocol

### Server -> Client

```json
// Initial snapshot (on connect)
{"type": "snapshot", "data": {"market": {...}, "orderbook": {...}, "candles": [...], "news": [...], "sentiment": {...}}}

// Live updates (every 1s if data changed)
{"type": "update", "data": {"market": {...}}}
{"type": "update", "data": {"market": {...}, "orderbook": {...}}}
{"type": "update", "data": {"candle": {...}}}
```

### Client -> Server

```json
// Change chart interval
{"type": "set_interval", "interval": "4h"}
```

## TODO
- [ ] Create `src/server/websocket.py`
- [ ] Create `src/server/app.py`
- [ ] Verify: `uvicorn src.server.app:app --factory` starts

## Success Criteria
- Dashboard loads at `http://localhost:8080`
- WS connects and receives initial snapshot
- Live updates arrive every ~1s when data changes
- Dead connections cleaned up automatically
- `set_interval` message switches candle data
