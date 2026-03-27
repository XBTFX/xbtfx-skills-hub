---
name: xbtfx-account
description: >
  Use when the user needs MT5 account state — balance, equity, margin,
  open positions, pending orders, or trade history — via the XBTFX API.
  This skill is read-only; it never modifies positions.
version: 1.0.0
author: XBTFX
homepage: https://console.xbtfx.com
requires_env: [XBTFX_API_KEY]
requires_bins: [curl]
---

# XBTFX Account Skill

Read account state from MetaTrader 5 through the XBTFX REST API. This skill covers account balance and equity, open positions, pending orders, and trade history.

## When to use this

Use this skill when the user asks about account balance, equity, margin, leverage, open positions, pending orders, margin mode, or trade history. All endpoints are read-only.

## Do not use this for

- Placing, modifying, or closing trades (use `xbtfx-trading`)
- Symbol specs or live quotes (use `xbtfx-market-data`)
- Streaming real-time data (use `xbtfx-websocket`)

## Base URL

```
https://interface.xbtfx.com
```

## Authentication

See [references/authentication.md](references/authentication.md).

All requests require:
```
Authorization: Bearer xbtfx_live_<your key>
Content-Type: application/json
```

## Rate Limits

600 weight per minute per API key. Most endpoints cost **1 weight**. Composite operations (`close-all`, `close-symbol`) cost **10 weight**. Every response includes rate-limit headers:

```
X-RateLimit-Budget: 600
X-RateLimit-Used: 42
X-RateLimit-Remaining: 558
X-RateLimit-Weight: 1
```

Back off if `X-RateLimit-Remaining` approaches 0.

## Quick Reference

| Method | Endpoint | Description | Weight |
|--------|----------|-------------|--------|
| GET | `/v1/auth/status` | Verify key, see permissions and margin mode | 1 |
| GET | `/v1/account` | Balance, equity, margin, leverage | 1 |
| GET | `/v1/positions` | List open positions | 1 |
| GET | `/v1/orders` | List pending orders | 1 |
| GET | `/v1/history` | Trade deal history | 2 |

## Endpoints

### GET /v1/auth/status

Verify your API key and see account configuration.

**Parameters:** None

**Example:**

```bash
curl -s https://interface.xbtfx.com/v1/auth/status \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "login": 1234567,
  "tier": "standard",
  "status": "active",
  "permissions": "trade,read",
  "margin_mode": "hedging"
}
```

Call this once at session start to learn the account's margin mode and permissions.

---

### GET /v1/account

Real-time account balance, equity, and margin. Served from in-memory cache (2s TTL) with real-time push updates.

**Parameters:** None

**Example:**

```bash
curl -s https://interface.xbtfx.com/v1/account \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "login": 1234567,
  "balance": 10250.00,
  "equity": 10180.50,
  "margin": 500.00,
  "margin_free": 9680.50,
  "profit": -69.50,
  "leverage": 100,
  "margin_mode": "hedging"
}
```

**Field descriptions:**

| Field | Description |
|-------|-------------|
| balance | Account balance (deposits + closed P&L) |
| equity | Balance + unrealized P&L of open positions |
| margin | Margin currently used by open positions |
| margin_free | Available margin for new positions (equity - margin) |
| profit | Total unrealized P&L of all open positions |
| leverage | Account leverage ratio (e.g. 100 = 1:100) |

---

### GET /v1/positions

List all open positions. Served from in-memory cache with real-time push updates from MT5.

**Query Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| symbol | string | No | Filter by symbol (e.g. `EURUSD`) |

**Example:**

```bash
curl -s "https://interface.xbtfx.com/v1/positions" \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

```bash
# Filter by symbol
curl -s "https://interface.xbtfx.com/v1/positions?symbol=EURUSD" \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "positions": [
    {
      "ticket": 12345678,
      "symbol": "EURUSD",
      "side": "buy",
      "volume": 0.10,
      "price_open": 1.08550,
      "price_current": 1.08620,
      "sl": 1.08200,
      "tp": 1.09000,
      "profit": 7.00,
      "swap": -0.50
    }
  ],
  "count": 1
}
```

**Position fields:**

| Field | Description |
|-------|-------------|
| ticket | Unique position identifier — use this for close/modify operations |
| symbol | Trading symbol |
| side | `buy` or `sell` |
| volume | Position size in lots |
| price_open | Entry price |
| price_current | Current market price |
| sl | Stop loss price (0 if not set) |
| tp | Take profit price (0 if not set) |
| profit | Unrealized P&L in account currency |
| swap | Accumulated swap charges |

---

### GET /v1/orders

List pending orders (limit, stop, stop-limit). Served from cache.

**Parameters:** None

**Example:**

```bash
curl -s https://interface.xbtfx.com/v1/orders \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "orders": [
    {
      "ticket": 87654321,
      "symbol": "GBPUSD",
      "type": "buy_limit",
      "volume": 0.50,
      "price": 1.26000,
      "sl": 1.25500,
      "tp": 1.27000
    }
  ],
  "count": 1
}
```

**Order types:**

| Type | Description |
|------|-------------|
| `buy_limit` | Buy at or below specified price |
| `sell_limit` | Sell at or above specified price |
| `buy_stop` | Buy when price rises to specified level |
| `sell_stop` | Sell when price falls to specified level |
| `buy_stop_limit` | Places buy limit when price hits stop level |
| `sell_stop_limit` | Places sell limit when price hits stop level |

---

### GET /v1/history

Trade deal history. Hits the MT5 bridge directly (not cached).

Use **either** the `period` shortcut **or** `from`+`to` date range, not both.
Custom date ranges are limited to 90 days.

**Query Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| period | string | No | Preset period (see enum below) |
| from | string | No | Start date `YYYY-MM-DD` (use with `to`) |
| to | string | No | End date `YYYY-MM-DD` (use with `from`) |

**Period enum:**

| Value | Description |
|-------|-------------|
| `today` | Current day |
| `last_3_days` | Last 3 days |
| `last_week` | Last 7 days |
| `last_month` | Last 30 days |
| `last_3_months` | Last 90 days |
| `last_6_months` | Last 6 months |
| `all` | Full account history |

**Example:**

```bash
curl -s "https://interface.xbtfx.com/v1/history?period=last_week" \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

```bash
# Custom date range
curl -s "https://interface.xbtfx.com/v1/history?from=2025-03-01&to=2025-03-15" \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "deals": [
    {
      "ticket": 99001122,
      "symbol": "EURUSD",
      "side": "buy",
      "entry": "in",
      "volume": 0.10,
      "price": 1.08550,
      "profit": 0.00,
      "time": "2025-01-15T10:30:00",
      "order": 88001122,
      "position": 12345678
    }
  ],
  "count": 1
}
```

**Entry types:**

| Value | Description |
|-------|-------------|
| `in` | Position opened |
| `out` | Position closed |
| `in_out` | Close and open (netting reversal) |
| `out_by` | Closed by opposing position (close-by) |

---

## Error Codes

| HTTP | Code | Description |
|------|------|-------------|
| 401 | `unauthorized` | Missing or invalid API key |
| 403 | `forbidden` | Key lacks `read` permission |
| 404 | `position_not_found` | Ticket does not exist in your account |
| 429 | `rate_limit_exceeded` | Weight budget exhausted |
| 503 | `bridge_unavailable` | No MT5 bridge connections available |
| 504 | `mt5_timeout` | Bridge did not respond within 7 seconds |

---

## Agent Behavior Guidelines

1. **Call `/v1/auth/status` first.** At the start of any session, verify the key works and learn the margin mode. Cache this — it doesn't change.
2. **Use positions for current state, history for past trades.** Don't query history to find open positions — use `/v1/positions`.
3. **Positions are cached server-side.** They update in real-time via MT5 push messages. Polling every few seconds is fine and costs only 1 weight.
4. **History hits the bridge directly.** It's slower and costs 2 weight. Don't poll it frequently.
5. **Present P&L clearly.** Always show profit values with the correct sign. Negative profit means the position is at a loss.
6. **Show tickets in full.** Position tickets are not sensitive data. Always display them so the user can reference them for close/modify operations.
7. **Format currency values.** Show balances and P&L with 2 decimal places for fiat currencies.
