# XBTFX Skills Hub

**Open skills for AI agents to trade via the XBTFX Trading API.**

The XBTFX Trading API provides access to trade forex, crypto, metals, indices, stocks, and energies on XBTFX MetaTrader 5 trading accounts. Skills Hub gives AI agents a structured way to learn and use the API: execute trades, manage positions, stream live quotes, and query account data through natural language.

> **Note:** This API is for XBTFX trading accounts only. Create an account at [my.xbtfx.com](https://my.xbtfx.com) and manage API keys at [console.xbtfx.com](https://console.xbtfx.com).

## What is a Skill?

A skill is a structured markdown file (`SKILL.md`) that teaches an AI agent how to use a specific API capability. Skills contain endpoint references, parameter specs, authentication details, and behavioral guidelines so agents can operate safely and correctly.

Skills work with any agent framework: **Claude Code**, **LangChain**, **CrewAI**, **OpenClaw**, or your own custom implementation.

## Quick Start

1. Get an API key from [console.xbtfx.com](https://console.xbtfx.com)
2. Point your agent at the relevant `SKILL.md` file
3. Start trading

```
# Example: give Claude Code the trading skill
claude --skill ./skills/xbtfx/xbtfx-trading/SKILL.md
```

## Skills

```
skills/xbtfx/
  xbtfx-trading/      Open, close, modify, and reverse positions
  xbtfx-account/      Balance, equity, margin, and trade history
  xbtfx-market-data/  Symbols, contract specs, and live quotes
  xbtfx-websocket/    Real-time price feeds and position updates
```

| Skill | Description | Key Endpoints |
|-------|-------------|---------------|
| [Trading](skills/xbtfx/xbtfx-trading/SKILL.md) | Execute trades on MT5 | `POST /v1/trade`, `/v1/close`, `/v1/modify`, `/v1/close-by`, `/v1/reverse`, `/v1/close-all` |
| [Account](skills/xbtfx/xbtfx-account/SKILL.md) | Query account state and history | `GET /v1/account`, `/v1/positions`, `/v1/orders`, `/v1/history` |
| [Market Data](skills/xbtfx/xbtfx-market-data/SKILL.md) | Symbol specs and quotes | `GET /v1/symbols`, `/v1/symbols/:symbol` |
| [WebSocket](skills/xbtfx/xbtfx-websocket/SKILL.md) | Real-time streaming | `wss://ws.xbtfx.com/v1/ws` |

## Repository Structure

```
xbtfx-skills-hub/
├── README.md
├── skills/
│   └── xbtfx/
│       ├── xbtfx-trading/
│       │   ├── SKILL.md
│       │   └── references/
│       │       └── authentication.md
│       ├── xbtfx-account/
│       │   ├── SKILL.md
│       │   └── references/
│       │       └── authentication.md
│       ├── xbtfx-market-data/
│       │   ├── SKILL.md
│       │   └── references/
│       │       └── authentication.md
│       └── xbtfx-websocket/
│           ├── SKILL.md
│           └── references/
│               └── authentication.md
```

## Authentication

All skills use the same authentication method: a Bearer token in the `Authorization` header.

```
Authorization: Bearer xbtfx_live_<32 hex chars>
```

API keys are created at [console.xbtfx.com](https://console.xbtfx.com). Each key is bound to one MT5 account and cannot access other accounts. Keys are shown once at creation — store them securely.

See [authentication reference](skills/xbtfx/xbtfx-trading/references/authentication.md) for full details.

## Margin Modes

Each API key has a margin mode set at creation, matching the MT5 account:

| Mode | Behavior | Close-By |
|------|----------|----------|
| **Hedging** | Multiple independent positions per symbol | Available |
| **Netting** | One aggregated position per symbol | Not available (400) |

Check `GET /v1/auth/status` to confirm your key's margin mode before trading.

## Rate Limits

Weight-based: **600 weight per minute** per API key. Most endpoints cost 1 weight. Composite operations (`close-all`, `close-symbol`) cost 10.

Every response includes rate limit headers:
```
X-RateLimit-Budget: 600
X-RateLimit-Used: 42
X-RateLimit-Remaining: 558
```

## Base URL

```
https://interface.xbtfx.com
```

## Contributing

We welcome skill contributions from the community. Each skill should:

1. Live in its own directory under `skills/`
2. Contain a `SKILL.md` with YAML frontmatter and structured instructions
3. Include a `references/authentication.md` if the skill requires authentication
4. Follow the patterns established by existing skills

## License

MIT
