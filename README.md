# XBTFX Skills Hub

**Give any AI agent the ability to trade forex, crypto, metals, indices, stocks, and energies on your XBTFX MT5 account.**

The [XBTFX Trading API](https://docs.xbtfx.com/trading-api/) provides programmatic access to your XBTFX MetaTrader 5 trading account. Skills Hub packages that API into structured skill files that any AI agent can pick up and use — execute trades, manage positions, stream live quotes, and query account data through natural language.

> This API is for XBTFX trading accounts only. Create an account at [my.xbtfx.com](https://my.xbtfx.com) and manage API keys at [console.xbtfx.com](https://console.xbtfx.com).

## What Can Agents Do?

| Capability | Examples |
|------------|---------|
| **Trade** | Open/close positions, set SL/TP, partial close, reverse, close-by |
| **Monitor** | Check balance, equity, margin, open positions, pending orders |
| **Analyze** | Get symbol specs, swap rates, margin requirements, trading sessions |
| **Stream** | Subscribe to live bid/ask quotes via WebSocket |
| **Review** | Query trade history by period or date range |

### Supported Markets

All instruments available on your XBTFX MT5 account:

- **Forex** — 50+ major, minor, and exotic pairs
- **Crypto** — BTC, ETH, SOL, and more vs USD/USDT
- **Metals** — Gold, Silver, Platinum
- **Indices** — Nasdaq, S&P 500, Dow Jones, DAX, and more
- **Stocks** — US equities (Apple, Tesla, Netflix, etc.)
- **Energies** — WTI Crude, Brent, Natural Gas

## Quick Start

1. Create an XBTFX account at [my.xbtfx.com](https://my.xbtfx.com)
2. Get an API key from [console.xbtfx.com](https://console.xbtfx.com)
3. Point your agent at a skill file:

```bash
# Claude Code
claude --skill ./skills/xbtfx/xbtfx-trading/SKILL.md

# Or use the MCP server for automatic tool discovery
claude mcp add xbtfx-trading -e XBTFX_API_KEY=your_key -- npx @xbtfx/mcp-trading
```

## Works With

| Platform | Integration |
|----------|-------------|
| **Claude Code** | `--skill SKILL.md` or MCP server |
| **Claude Desktop** | MCP server via `claude_desktop_config.json` |
| **Cursor** | MCP server via `.cursor/mcp.json` |
| **OpenAI Codex** | MCP server via `codex mcp add` |
| **Windsurf** | MCP server via settings |
| **LangChain / CrewAI** | Ingest SKILL.md as tool description |
| **Custom agents** | Parse structured markdown or use MCP |

## Skills

| Skill | What it teaches the agent | Key Endpoints |
|-------|---------------------------|---------------|
| [Trading](skills/xbtfx/xbtfx-trading/SKILL.md) | Open, close, modify, and reverse positions | `POST /v1/trade`, `/v1/close`, `/v1/modify`, `/v1/close-by`, `/v1/reverse`, `/v1/close-all` |
| [Account](skills/xbtfx/xbtfx-account/SKILL.md) | Query balance, positions, orders, and history | `GET /v1/account`, `/v1/positions`, `/v1/orders`, `/v1/history` |
| [Market Data](skills/xbtfx/xbtfx-market-data/SKILL.md) | Symbol specs, swap rates, margins, live quotes | `GET /v1/symbols`, `/v1/symbols/:symbol` |
| [WebSocket](skills/xbtfx/xbtfx-websocket/SKILL.md) | Stream real-time prices and depth of market | `wss://ws.xbtfx.com/v1/ws` |

## What is a Skill?

A skill is a structured markdown file (`SKILL.md`) that teaches an AI agent how to use a specific API capability. Each skill contains endpoint references, parameter specs, response formats, error codes, and behavioral guidelines so agents can operate safely and correctly.

## Example Agent Use Cases

### Price-aware trading

> "Buy 0.05 XAUUSD if gold is below 2300, with a 200 pip stop loss"

The agent calls `get_symbol` to check volume constraints, reads the live bid/ask, evaluates the condition, and executes the trade with SL — all from one natural language instruction.

### Portfolio monitoring

> "Show me all my open positions and total P&L"

The agent calls `get_positions` and `get_account`, summarizes exposure by symbol, and reports unrealized profit across the portfolio.

### Risk management

> "Close any position that's down more than $50"

The agent fetches positions, filters by profit threshold, and closes each losing position individually — confirming with you before executing.

## Safety

- **Login isolation** — Each API key is bound to one MT5 account. The server enforces this; agents cannot access other accounts.
- **Idempotency** — Trade endpoints accept an `Idempotency-Key` header to prevent duplicate executions on retry.
- **Rate limits** — 600 weight per minute per API key. Agents receive rate limit headers on every response.
- **Confirmation pattern** — MCP trading tools are marked as `destructiveHint: true`, prompting the AI client to confirm with the user before executing.

## Authentication

All skills use Bearer token authentication:

```
Authorization: Bearer xbtfx_live_<32 hex chars>
```

API keys are created at [console.xbtfx.com](https://console.xbtfx.com). Each key is bound to one MT5 account and shown once at creation — store it securely.

## Base URL

```
https://interface.xbtfx.com
```

## Related

- [XBTFX Trading API Docs](https://docs.xbtfx.com/trading-api/) — Full API reference
- [MCP Server](https://github.com/XBTFX/xbtfx-mcp-server) — Automatic tool discovery for MCP clients
- [API Examples](https://github.com/XBTFX/xbtfx-api-examples) — Python, JavaScript, Go, curl

## License

MIT
