---
name: xbtfx-market-data
description: >
  Use when the user needs symbol specifications, contract details, or
  live bid/ask quotes from MT5 via the XBTFX API.
  This skill is read-only.
version: 1.0.0
author: XBTFX
homepage: https://console.xbtfx.com
requires_env: [XBTFX_API_KEY]
requires_bins: [curl]
---

# XBTFX Market Data Skill

Query available trading symbols, contract specifications, and live quotes from MetaTrader 5 through the XBTFX REST API. Use this before trading to validate symbols and check volume constraints.

## When to use this

Use this skill when the user needs instrument details — available symbols, volume constraints (min/max/step), contract sizes, tick sizes, or a current bid/ask snapshot. All endpoints are read-only.

## Do not use this for

- Account info, positions, or history (use `xbtfx-account`)
- Trade execution (use `xbtfx-trading`)
- Continuous live price streaming (use `xbtfx-websocket`)

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
| GET | `/v1/symbols` | List all symbols with live quotes | 2 |
| GET | `/v1/symbols/:symbol` | Full contract spec for one symbol | 1 |

## Endpoints

### GET /v1/symbols

List all available trading symbols with current bid/ask prices. Served from in-memory cache, refreshed hourly for specs and real-time for quotes.

**Parameters:** None

**Example:**

```bash
curl -s https://interface.xbtfx.com/v1/symbols \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "symbols": [
    {
      "name": "EURUSD",
      "digits": 5,
      "contract_size": 100000.0,
      "point": 0.00001,
      "trade_mode": 4,
      "swap_long": -0.50,
      "swap_short": -0.30,
      "spread": 0,
      "bid": 1.09500,
      "ask": 1.09510
    },
    {
      "name": "NDXUSD",
      "digits": 2,
      "contract_size": 1.0,
      "point": 0.01,
      "trade_mode": 4,
      "swap_long": -2.15,
      "swap_short": -1.85,
      "spread": 100,
      "bid": 21543.50,
      "ask": 21544.50
    }
  ],
  "count": 2
}
```

**Note:** This returns a summary for all symbols. Use `/v1/symbols/:symbol` for full contract details including volume constraints.

---

### GET /v1/symbols/:symbol

Full contract specification for a single symbol, including volume limits, margin parameters, and current quote.

**Path Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| symbol | string | Yes | Symbol name, case-sensitive (e.g. `EURUSD`, `NDXUSD`) |

**Example:**

```bash
curl -s https://interface.xbtfx.com/v1/symbols/EURUSD \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

**Response (200):**

```json
{
  "symbol": "EURUSD",
  "description": "Euro vs US Dollar",
  "currency_base": "EUR",
  "currency_profit": "USD",
  "currency_margin": "USD",
  "digits": 5,
  "point": 0.00001,
  "contract_size": 100000.0,
  "trade_mode": "Full",
  "tradeable": true,
  "volume_min": 0.01,
  "volume_max": 100.00,
  "volume_step": 0.01,
  "stops_level": 0,
  "spread": 10,
  "bid": 1.09500,
  "ask": 1.09510
}
```

**Field descriptions:**

| Field | Description |
|-------|-------------|
| symbol | Symbol name |
| description | Human-readable description |
| currency_base | Base currency |
| currency_profit | Profit calculation currency |
| currency_margin | Margin calculation currency |
| digits | Price decimal places |
| point | Minimum price change |
| contract_size | Contract size in base currency units |
| trade_mode | `Full` = fully tradeable |
| tradeable | Whether trading is currently allowed |
| volume_min | Minimum order volume in lots |
| volume_max | Maximum order volume in lots |
| volume_step | Volume increment step in lots |
| stops_level | Minimum distance for SL/TP from current price (in points). 0 = no restriction. |
| spread | Current spread in points |
| bid | Current bid price |
| ask | Current ask price |

---

## Volume Validation

Before placing a trade, always check the symbol's volume constraints:

```
volume_min <= your_volume <= volume_max
(your_volume - volume_min) % volume_step == 0
```

**Example:** If `volume_min=0.10`, `volume_max=50.00`, `volume_step=0.10`:
- `0.10` — valid
- `0.15` — invalid (not a multiple of step)
- `0.20` — valid
- `55.00` — invalid (exceeds max)

Round to the nearest valid step:
```
rounded = Math.round(volume / volume_step) * volume_step
clamped = Math.max(volume_min, Math.min(volume_max, rounded))
```

---

## Error Codes

| HTTP | Code | Description |
|------|------|-------------|
| 400 | `invalid_symbol` | Symbol not found or not tradeable |
| 401 | `unauthorized` | Missing or invalid API key |
| 429 | `rate_limit_exceeded` | Weight budget exhausted |

---

## Agent Behavior Guidelines

1. **Always check symbol specs before trading.** Call `GET /v1/symbols/:symbol` to get volume_min, volume_max, and volume_step. Validate and round the user's requested volume.
2. **Cache symbol specs in memory.** They change infrequently (hourly refresh). Avoid calling `/v1/symbols` repeatedly within a session.
3. **The full list costs 2 weight.** A single symbol lookup costs 1. If you only need a few symbols, query them individually.
4. **Symbol names are case-sensitive.** Use the exact name returned by the API (e.g. `EURUSD` not `eurusd`).
5. **Spread is in points, not pips.** For a 5-digit pair like EURUSD, 1 pip = 10 points. A spread of 10 points = 1.0 pips.
6. **Show bid/ask clearly.** When displaying quotes, always show both bid and ask. The bid is the sell price; the ask is the buy price.
7. **Check `tradeable` before trading.** If `tradeable` is `false`, the symbol is currently closed (e.g. weekend, market holiday). Inform the user.
