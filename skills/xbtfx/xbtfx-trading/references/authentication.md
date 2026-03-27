# Authentication Reference

## Credentials

| Field | Description |
|-------|-------------|
| `api_key` | Your XBTFX API key in format `xbtfx_live_<32 hex chars>` |

API keys are created at [console.xbtfx.com](https://console.xbtfx.com). Each key is bound to exactly one MT5 login. The server enforces this — you cannot access other accounts.

## Base URL

```
https://interface.xbtfx.com
```

## Request Headers

Every request must include:

```
Authorization: Bearer xbtfx_live_<your key>
Content-Type: application/json
```

Optional headers:

```
Idempotency-Key: <unique-string>    # For trade endpoints, prevents duplicate execution (120s TTL)
User-Agent: xbtfx-trading/1.0.0 (Skill)
```

## Key Properties

| Property | Description |
|----------|-------------|
| Permissions | `trade,read` (full access) or `read` (read-only; trade endpoints return 403) |
| Margin Mode | `hedging` or `netting` — set at creation, matches MT5 account config |
| Tier | `standard`, `premium`, or `institutional` — determines rate limits |

## Credential Security

- **Never** log, display, or transmit the full API key
- **Always** mask keys in output: show only prefix (e.g. `xbtfx_live_a1b2...`)
- **Never** pass API keys as query parameters — use the Authorization header only
- Store keys in environment variables or a secrets file, never in code

### Environment Variable (recommended)

```bash
export XBTFX_API_KEY="xbtfx_live_a1b2c3d4e5f6..."
```

### Secrets File

```bash
# ~/.xbtfx/credentials
XBTFX_API_KEY=xbtfx_live_a1b2c3d4e5f6...
```

File permissions should be `600` (owner read/write only).

## Verifying Your Key

```bash
curl -s https://interface.xbtfx.com/v1/auth/status \
  -H "Authorization: Bearer $XBTFX_API_KEY"
```

Response:
```json
{
  "login": 1234567,
  "tier": "standard",
  "status": "active",
  "permissions": "trade,read",
  "margin_mode": "hedging"
}
```

## Rate Limits

Weight budget: **600 per minute** per API key. Max open positions: **200**.

WebSocket subscription limits vary by tier:

| Tier | Max WS Subscriptions |
|------|---------------------|
| Standard | 20 |
| Premium | 100 |
| Institutional | Unlimited |

Response headers:

```
X-RateLimit-Budget: 600
X-RateLimit-Used: 42
X-RateLimit-Remaining: 558
X-RateLimit-Weight: 1
```

## Error Responses

```json
{
  "error": "unauthorized",
  "message": "Invalid or missing API key"
}
```

| HTTP | Error | Meaning |
|------|-------|---------|
| 401 | `unauthorized` | Missing or invalid API key |
| 403 | `forbidden` | Key lacks required permission |
| 429 | `rate_limit_exceeded` | Weight budget exhausted |
