---
name: xbtfx-trading
description: >
  Use when the user wants to open, close, reverse, or modify MetaTrader 5
  positions through the XBTFX API. Always confirm symbol, side, volume, and
  SL/TP with the user before executing any trade.
version: 1.0.0
author: XBTFX
homepage: https://console.xbtfx.com
requires_env: [XBTFX_API_KEY]
requires_bins: [curl]
---

# XBTFX Trading Skill

Execute trades on MetaTrader 5 through the XBTFX REST API. This skill covers opening positions, closing (full and partial), modifying SL/TP, close-by (hedge netting), reversals, and bulk close operations.

## When to use this

Use this skill when the user wants to execute trades, close positions (full or partial), modify stop-loss or take-profit levels, reverse a position, or perform a close-by on opposing hedged positions. All operations go through the XBTFX REST API and hit a live MT5 account.

## Do not use this for

- Market commentary, strategy discussion, or signal generation
- Read-only account checks like balance, margin, or position listing (use `xbtfx-account`)
- Symbol specs or price quotes (use `xbtfx-market-data`)
- Any action without explicit user confirmation — **always confirm before executing**

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
```

Back off if `X-RateLimit-Remaining` approaches 0.

## Quick Reference

| Method | Endpoint | Description | Weight |
|--------|----------|-------------|--------|
| POST | `/v1/trade` | Open a new position (market order) | 1 |
| POST | `/v1/close` | Close a position (full or partial) | 1 |
| POST | `/v1/modify` | Update SL/TP on a position | 1 |
| POST | `/v1/close-by` | Net two opposing positions (hedging only) | 1 |
| POST | `/v1/reverse` | Close and open opposite side | 2 |
| POST | `/v1/close-all` | Close all open positions | 10 |
| POST | `/v1/close-symbol` | Close all positions for a symbol | 10 |

## Endpoints

### POST /v1/trade

Open a new market position. In netting mode, trades on the same symbol modify the existing position.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| symbol | string | Yes | Trading symbol (e.g. `EURUSD`) |
| side | string | Yes | `buy` or `sell` |
| volume | number | Yes | Lot size (e.g. `0.10`). Must respect symbol's volume_min, volume_max, and volume_step. |
| sl | number | No | Stop loss price |
| tp | number | No | Take profit price |
| comment | string | No | Max 27 characters, ASCII only. Prefixed with `[API]` in MT5. |

**Example:**

```bash
curl -X POST https://interface.xbtfx.com/v1/trade \
  -H "Authorization: Bearer $XBTFX_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: bot-trade-$(date +%s)" \
  -d '{
    "symbol": "EURUSD",
    "side": "buy",
    "volume": 0.10,
    "sl": 1.08200,
    "tp": 1.09000
  }'
```

**Response (200):**

```json
{
  "status": "filled",
  "deal": 12345678,
  "order": 87654321,
  "volume": 0.10,
  "price": 1.08550,
  "comment": "[API] "
}
```

**Retcodes:** `10008` (placed) and `10009` (done) both indicate success.

---

### POST /v1/close

Close an open position. Supports partial close by specifying volume.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| ticket | number | Yes | Position ticket to close |
| volume | number | No | Partial close volume in lots. Omit for full close. |
| comment | string | No | Max 27 characters, ASCII only |

**Example:**

```bash
curl -X POST https://interface.xbtfx.com/v1/close \
  -H "Authorization: Bearer $XBTFX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "ticket": 12345678 }'
```

**Response (200):**

```json
{
  "status": "filled",
  "deal": 12345679,
  "price": 1.08620
}
```

---

### POST /v1/modify

Update stop loss and/or take profit on an open position.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| ticket | number | Yes | Position ticket to modify |
| sl | number | No | New stop loss. Set to `0` to remove. |
| tp | number | No | New take profit. Set to `0` to remove. |

At least one of `sl` or `tp` must be provided.

**Example:**

```bash
curl -X POST https://interface.xbtfx.com/v1/modify \
  -H "Authorization: Bearer $XBTFX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "ticket": 12345678, "sl": 1.08300, "tp": 1.09500 }'
```

**Response (200):**

```json
{
  "status": "ok",
  "retcode": 10009
}
```

---

### POST /v1/close-by

Close two opposing positions against each other via MT5 CLOSE_BY. Saves spread cost on the smaller side.

**Hedging accounts only.** Returns 400 for netting accounts.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| position | number | Yes | First position ticket |
| position_by | number | Yes | Opposing position ticket (same symbol, opposite side) |
| comment | string | No | Max 27 characters, ASCII only |

The smaller volume closes fully; the larger is reduced by that amount.

**Example:**

```bash
curl -X POST https://interface.xbtfx.com/v1/close-by \
  -H "Authorization: Bearer $XBTFX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "position": 12345, "position_by": 12346 }'
```

**Response (200):**

```json
{
  "status": "ok",
  "position": 12345,
  "position_by": 12346,
  "retcode": 10009
}
```

---

### POST /v1/reverse

Close a position and immediately open the opposite side with the same volume. Two-step composite operation.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| ticket | number | Yes | Position ticket to reverse |
| comment | string | No | Max 27 characters, ASCII only |

**Example:**

```bash
curl -X POST https://interface.xbtfx.com/v1/reverse \
  -H "Authorization: Bearer $XBTFX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "ticket": 12345678 }'
```

**Response (200 — full success):**

```json
{
  "close": { "status": "filled", "deal": 99001, "price": 1.08620 },
  "open": { "status": "filled", "deal": 99002, "order": 88002, "volume": 0.10, "price": 1.08625, "side": "sell" }
}
```

**Response (207 — partial failure):**

```json
{
  "close": { "status": "filled", "deal": 99001, "price": 1.08620 },
  "open": { "status": "rejected", "retcode": 10006, "message": "Trade rejected" },
  "warning": "Position was closed but reverse open failed. Re-enter manually."
}
```

If you receive a 207, the position has already been closed. Inform the user and do not retry automatically.

---

### POST /v1/close-all

Close all open positions. Sends parallel close commands (up to 20 concurrent).

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| comment | string | No | Applied to all close operations |

**Example:**

```bash
curl -X POST https://interface.xbtfx.com/v1/close-all \
  -H "Authorization: Bearer $XBTFX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Response (200):**

```json
{
  "closed": 3,
  "failed": 0,
  "total_profit": 42.50,
  "details": [
    { "ticket": 12345, "status": "filled", "deal": 99001, "profit": 15.00 },
    { "ticket": 12346, "status": "filled", "deal": 99002, "profit": 20.00 },
    { "ticket": 12347, "status": "filled", "deal": 99003, "profit": 7.50 }
  ]
}
```

---

### POST /v1/close-symbol

Close all positions for a specific symbol.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| symbol | string | Yes | Symbol to close all positions for |
| comment | string | No | Applied to all close operations |

**Response:** Same format as `/v1/close-all`.

---

## Enums

### side
| Value | Description |
|-------|-------------|
| `buy` | Long position |
| `sell` | Short position |

### status (response)
| Value | Description |
|-------|-------------|
| `filled` | Order executed successfully |
| `ok` | Modification applied |
| `rejected` | MT5 rejected the operation |

---

## Idempotency

Trade endpoints support the `Idempotency-Key` header. If you send the same key within 120 seconds, the server returns the cached response without re-executing the trade.

```
Idempotency-Key: my-bot-trade-1709500000
```

Always use idempotency keys for trade operations to prevent accidental duplicate orders on retry.

---

## Error Codes

| HTTP | Code | Description |
|------|------|-------------|
| 400 | `invalid_request` | Missing or invalid parameters |
| 400 | `invalid_symbol` | Symbol not found or not tradeable |
| 400 | `invalid_volume` | Volume outside min/max/step for the symbol |
| 404 | `position_not_found` | Ticket does not exist in your account |
| 422 | `mt5_rejected` | MT5 rejected the trade (includes retcode and message) |
| 503 | `bridge_unavailable` | No MT5 bridge connections available |
| 504 | `mt5_timeout` | Bridge did not respond within 7 seconds |

---

## Agent Behavior Guidelines

1. **Always confirm before executing trades.** Show the user the symbol, side, volume, and any SL/TP before sending. Ask for explicit confirmation.
2. **Check symbol specs first.** Before trading, call `GET /v1/symbols/:symbol` to verify volume_min, volume_max, and volume_step. Round volume to the nearest valid step.
3. **Check margin mode.** Call `GET /v1/auth/status` once at session start to know if the account is hedging or netting. Do not attempt `close-by` on netting accounts.
4. **Use idempotency keys.** Always include an `Idempotency-Key` header on trade requests. Generate a unique key per intended trade action.
5. **Handle 207 on reverse.** If a reverse returns 207, the close succeeded but the open failed. Tell the user their position was closed and they may need to re-enter manually.
6. **Never mask ticket numbers.** Tickets are not sensitive — always show them in full so the user can reference them.
7. **Mask API keys.** Never display the full API key. Show only the prefix: `xbtfx_live_a1b2...`
8. **Respect rate limits.** Check `X-RateLimit-Remaining` in response headers. If approaching 0, wait before sending more requests.
9. **Volume is in lots.** Always express volume in lots (e.g. 0.10, 1.00), not in units or contract sizes.
10. **Comments are optional.** If provided, keep under 27 characters, ASCII only. The server prepends `[API]`.
