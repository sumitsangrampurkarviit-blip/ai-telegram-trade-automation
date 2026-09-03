# AI Telegram Trade Automation

Paper-first beta software that turns Telegram trading messages into structured
actions, validates them in deterministic code, and simulates execution. It is
designed to show beta testers exactly what arrived, what the AI understood,
what safety checks passed, and what the paper executor did.

## What it does

- Interprets natural-language trade instructions with an OpenAI-compatible LLM.
- Sends only structured JSON actions into the safety layer.
- Normalizes common aliases such as Gold, XAU, and XAUUSD.
- Maintains paper positions and resolves context-aware instructions such as
  “SL to cost”, “Close half”, and “Close gold”.
- Stores every manual/Telegram message and result for an audit trail.
- Blocks live execution by default. There is no live broker adapter in this beta.

## Architecture

Telegram adapter or manual inspector → LLM interpreter → deterministic
validator → trade resolver → paper executor → PostgreSQL → dashboard.

The LLM never calls a broker, chooses volume, invents prices, or overrides
risk rules. The server owns validation and execution.

## Run locally

This repository uses pnpm and the Replit-managed PostgreSQL database.

```bash
pnpm install
pnpm --filter @workspace/db run push
pnpm --filter @workspace/api-server run dev
```

The dashboard is started by the `artifacts/telegram-trade-automation: web`
workflow. The API is available under `/api`.

## Environment variables

Copy `.env.example` as a reference. Set these in the workspace environment,
not in source control:

- `LLM_API_KEY` — key for your chosen compatible provider.
- `LLM_PROVIDER` — `gemini`, `groq`, `openrouter`, or `openai`.
- `LLM_BASE_URL` and `LLM_MODEL` — optional overrides for any compatible API.
- `TRADING_MODE=paper` — required for the beta.
- `LIVE_TRADING_ENABLED=false` — keep false; live execution is not implemented.
- `DEFAULT_VOLUME=0.01` — fixed paper size; the AI cannot select it.
- `PAPER_MARKET_PRICE=3355` — deterministic price used for a MARKET demo.
- `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, `TELEGRAM_SESSION_NAME`, and
  `TELEGRAM_SOURCE_CHANNEL` — optional listener settings.

Free-tier providers still require a free provider account and API key. Gemini,
Groq, and OpenRouter are compatible choices; quotas and model names can change,
so set `LLM_BASE_URL` and `LLM_MODEL` when needed.

## Manual testing

Open `/test` in the dashboard and try these in order:

1. `BUY GOLD NOW` / `SL 3345` / `TP 3370`
2. `SL to cost`
3. `Close half`
4. `Close gold`
5. `Good morning everyone`

To test duplicate protection, send the same message twice with the same
Telegram message ID. The second result is marked `DUPLICATE` and cannot execute.

## Telegram setup

Create Telegram API credentials at my.telegram.org, keep them in workspace
secrets, and run `python scripts/telegram_login.py` in the environment where
the Telethon session will live. The beta remains fully usable without Telegram
through `/test`.

## MT5 later

The execution interface is deliberately isolated from the pipeline. A future
Windows/VPS adapter can implement the broker methods behind the same boundary.
Live mode must remain opt-in and require `TRADING_MODE=live`,
`LIVE_TRADING_ENABLED=true`, and a verified broker adapter.

## Known limitations

- The Telegram adapter is a safe boundary and setup/status surface; a Telethon
  worker still needs to be connected for live channel ingestion.
- Paper MARKET orders use the configured deterministic demo price rather than a
  market data quote.
- Only XAUUSD and BTCUSD aliases are recognized initially.
- There is no authentication or multi-user workspace isolation in this beta.
- Arbitrary language requires a configured LLM key; without one, demo mode only
  recognizes the documented sample messages.