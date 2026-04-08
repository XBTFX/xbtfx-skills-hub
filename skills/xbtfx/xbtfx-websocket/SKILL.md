---
name: xbtfx-websocket
description: >
  Use when the user needs real-time streaming data — live quotes, position
  updates, order events, or deal notifications — via the XBTFX WebSocket API.
  Read-only; does not execute trades.
version: 1.0.0
author: XBTFX
homepage: https://console.xbtfx.com
requires_env: [XBTFX_API_KEY]
requires_bins: [curl, websocat]
---

# XBTFX WebSocket Skill

Stream real-time market data and account events from MetaTrader 5 through the XBTFX WebSocket API. Subscribe to live quotes, position changes, order events, and deal notifications.

## When to use this

Use this skill when the user needs a persistent real-time feed — live tick quotes, order-book depth, position open/close/update events, or deal notifications. The WebSocket is read-only and event-driven.

## Do not use this for

- One-off price lookups or symbol specs (use `xbtfx-market-data`)
- Trade execution or position modification (use `xbtfx-trading`)
- Account snapshots like balance or history (use `xbtfx-account`)

## Endpoint

```
wss://ws.xbtfx.com/v1/ws
```

> `wss://interface.xbtfx.com/v1/ws` also works as an alias.

## Authentication

The first message after connecting **must** be an auth message:

```json
{ "action": "auth", "token": "xbtfx_live_<your key>" }
```

**Success response:**

```json
{ "action": "auth_ok", "login": 50234, "tier": "standard" }
```

**Failure response (then disconnect):**

```json
{ "action": "auth_error", "message": "Invalid API key" }
```

You have 10 seconds to authenticate after connecting, or the server disconnects.

## Connection Example

Using `websocat`:

```bash
echo '{"action":"auth","token":"'"$XBTFX_API_KEY"'"}' | \
  websocat wss://ws.xbtfx.com/v1/ws
```

Using Python:

```python
import asyncio, json, websockets, os

async def main():
    uri = "wss://ws.xbtfx.com/v1/ws"
    async with websockets.connect(uri) as ws:
        # Authenticate
        await ws.send(json.dumps({
            "action": "auth",
            "token": os.environ["XBTFX_API_KEY"]
        }))
        auth = json.loads(await ws.recv())
        print(f"Authenticated: login {auth['login']}")

        # Subscribe to quotes
        await ws.send(json.dumps({
            "action": "subscribe",
            "channel": "quotes",
            "symbols": ["EURUSD", "GBPUSD"]
        }))

        # Subscribe to account events
        await ws.send(json.dumps({
            "action": "subscribe",
            "channel": "account"
        }))

        # Listen
        async for msg in ws:
            data = json.loads(msg)
            print(data)

asyncio.run(main())
```

## Channels

### Quote Subscription

Subscribe to live bid/ask quotes for specific symbols.

**Subscribe:**

```json
{ "action": "subscribe", "channel": "quotes", "symbols": ["EURUSD", "GBPUSD", "NDXUSD"] }
```

**Subscribe with depth of market:**

```json
{ "action": "subscribe", "channel": "quotes", "symbols": ["EURUSD"], "depth": true }
```

**Unsubscribe:**

```json
{ "action": "unsubscribe", "channel": "quotes", "symbols": ["GBPUSD"] }
```

**Quote message (basic):**

```json
{
  "channel": "quote",
  "symbol": "EURUSD",
  "bid": 1.08550,
  "ask": 1.08553,
  "time": "2025-03-05T12:30:00Z"
}
```

**Quote message (with depth):**

```json
{
  "channel": "quote",
  "symbol": "EURUSD",
  "bid": 1.09500,
  "ask": 1.09510,
  "spread": 0.00010,
  "time": 1709500000,
  "depth": {
    "bids": [[1.09495, 2000000], [1.09490, 3000000]],
    "asks": [[1.09515, 2000000], [1.09520, 3000000]]
  }
}
```

Depth entries are `[price, volume]` pairs, sorted best-to-worst.

**Server-side throttle:** Max 5 quote updates per second per symbol per client.

---

### Account Channel

Subscribe to position, order, and deal events for your account.

**Subscribe:**

```json
{ "action": "subscribe", "channel": "account" }
```

**Position events:**

```json
{ "channel": "position", "event": "open", "ticket": 98765, "symbol": "EURUSD", "side": "buy", "volume": 1.00, "price_open": 1.09500 }
```

```json
{ "channel": "position", "event": "update", "ticket": 98765, "profit": 10.00, "price_current": 1.09510 }
```

```json
{ "channel": "position", "event": "close", "ticket": 98765, "symbol": "EURUSD", "profit": 100.00 }
```

**Order events:**

```json
{ "channel": "order", "event": "add", "ticket": 11111, "symbol": "EURUSD", "type": "buy_limit", "volume": 1.00, "price": 1.08500 }
```

```json
{ "channel": "order", "event": "remove", "ticket": 11111 }
```

**Deal events:**

```json
{ "channel": "deal", "ticket": 12345, "symbol": "EURUSD", "side": "buy", "volume": 1.00, "price": 1.09500, "profit": 0.00 }
```

---

### System Messages

The server may send system-level messages:

**Rate limit warning:**

```json
{ "channel": "system", "event": "rate_limit", "message": "Approaching subscription limit" }
```

---

## Keepalive

Send a ping every 30 seconds to keep the connection alive:

**Client sends:**

```json
{ "action": "ping" }
```

**Server responds:**

```json
{ "action": "pong" }
```

If no ping is received for 90 seconds, the server disconnects.

---

## Rate Limits

The WebSocket itself is not weight-budgeted like REST, but the underlying API key shares the same 600-weight-per-minute budget if REST calls are also in flight. Subscription and message limits are enforced separately — see the table below.

## Limits

| Limit | Value |
|-------|-------|
| Max connections per API key | 10 |
| Max subscriptions per connection | 1,000 |
| Quote throttle | 5 updates/sec per symbol per client |
| Auth timeout | 10 seconds after connect |
| Idle timeout | 90 seconds without ping |

Tier-based subscription limits:

| Tier | Max Subscriptions |
|------|------------------|
| Standard | 20 |
| Premium | 100 |
| Institutional | Unlimited |

---

## Error Codes

WebSocket errors are sent as JSON messages before disconnecting:

```json
{ "action": "error", "code": "subscription_limit", "message": "Maximum subscriptions reached" }
```

| Code | Description |
|------|-------------|
| `auth_error` | Invalid or expired API key |
| `subscription_limit` | Exceeded max subscriptions for your tier |
| `connection_limit` | Too many concurrent connections |
| `rate_limited` | Sending messages too fast |

---

## Agent Behavior Guidelines

1. **Authenticate immediately.** Send the auth message within 10 seconds of connecting.
2. **Implement ping/pong.** Send `{"action":"ping"}` every 30 seconds. Set a timer; don't rely on incoming messages to trigger it.
3. **Subscribe only to symbols you need.** Each subscription counts against your tier limit. Unsubscribe when you're done with a symbol.
4. **Handle reconnection.** WebSocket connections can drop. Implement automatic reconnection with exponential backoff (1s, 2s, 5s, 10s, 30s). Re-authenticate and re-subscribe after reconnecting.
5. **Don't use WebSocket for trading.** The WebSocket is read-only (quotes + events). Use REST endpoints for trade execution.
6. **Account channel gives real-time position updates.** Use this instead of polling `GET /v1/positions` for live tracking.
7. **Depth data is optional.** Only subscribe with `"depth": true` if you need order book depth. It increases message volume significantly.
8. **Never log the API key.** The auth message contains the full key. Ensure your logging excludes WebSocket auth frames.
