# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background tasks: market data, snapshots       │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues in production, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

### Single-User Simplification

All tables carry a user identifier (see §7), but the code does not abstract over users. A single constant `DEFAULT_USER_ID = "default"` is used everywhere; do not build services or repositories parameterised by user.

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── app/
│       ├── market/           # Market data subsystem (complete)
│       └── db/               # Schema definitions, seed data, initialization
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints using the contracts in §8. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. All Python code lives in the `app` package. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/app/db/`** contains schema SQL definitions and seed logic. The backend initializes the database during FastAPI lifespan startup (see §7).
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- `OPENROUTER_API_KEY` is required: if it is absent or empty and `LLM_MOCK` is not `true`, the backend logs a clear error and refuses to start
- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (see §9) and never calls OpenRouter
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

The market data subsystem is complete and lives in `backend/app/market/` (see `planning/MARKET_DATA_SUMMARY.md`). The items marked **(to add)** below are small extensions that the remaining work needs.

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Tickers without seed data (e.g., `PYPL`) start at a random price between $50 and $300 with default volatility
- **(to add)** Held tickers without seed data start at their position's `avg_cost` on backend startup, instead of a random price, so a restart doesn't cause a large fake P&L swing. This needs `start()` to accept optional initial prices (e.g. `start(tickers, initial_prices: dict[str, float] | None = None)`). Seeded tickers still restart at their seed price; the resulting small P&L jump is accepted.
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all tracked tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- **(to add)** A poll runs in a worker thread, so it can finish after `remove_ticker()` has been called. Before writing to the cache, `_poll_once` must drop any snapshot whose ticker is no longer in the tracked set. Otherwise a late poll re-inserts the removed ticker with a stale price, and it becomes tradeable again.

### Tracked Tickers

The set of tickers the market data source tracks is **watchlist ∪ open positions**:

- Adding a ticker to the watchlist → `source.add_ticker()`
- Removing a ticker from the watchlist → `source.remove_ticker()` **only if** there is no open position in it
- Selling a position to zero → `source.remove_ticker()` **only if** the ticker is not on the watchlist
- At startup, the source starts with watchlist ∪ positions loaded from the database

This keeps held positions priced (for valuation and selling) even after they are removed from the watchlist.

Each "only if" rule is a check-then-act across the database and the data source, so it must run under the same process-wide lock as trades (see §8 Business Rules). Otherwise a concurrent buy could land between the check and `remove_ticker()`, leaving a held position with no price.

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- **(to add)** The cache also records each ticker's **open price**: the first price it saw for that ticker (the seed price for the simulator, the first poll result for Massive). This is the baseline for "daily change %". It resets when the backend restarts, which is acceptable for this project. It is stored in the ticker's cache entry, so `cache.remove()` discards it. Removing and later re-adding a ticker starts a fresh baseline, which is intended.
- SSE streams, portfolio valuation and trade execution all read from this cache

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The server sends `retry: 1000` first, then checks the cache every ~500ms. If anything changed, it sends **one event containing every tracked ticker**, keyed by ticker:

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.52, "previous_price": 190.41, "timestamp": 1761040000.12, "change": 0.11, "change_percent": 0.0578, "direction": "up", "open_price": 190.0, "day_change_percent": 0.2737}, "GOOGL": {...}}
```

- `timestamp` is Unix seconds (float). `direction` is `"up" | "down" | "flat"`. `change` and `change_percent` are tick-over-tick.
- **(to add)** `open_price` and `day_change_percent` (relative to `open_price`, as a percent) to `PriceUpdate.to_dict()`
- The stream includes every tracked ticker, which may include held tickers that are not on the watchlist. The frontend uses `/api/watchlist` to decide which rows to show in the watchlist panel.
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Automatic Initialization

During FastAPI lifespan startup, the backend checks the SQLite database. If the file doesn't exist or tables are missing, it creates the schema (`CREATE TABLE IF NOT EXISTS`) and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

Every table except `users_profile` has a `user_id` column defaulting to `"default"`. In `users_profile`, the primary key `id` is the user identifier. It is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). A background task records one every 30 seconds, always, including when there are no positions (value = cash, a flat line). One is also recorded immediately after each trade. Rows are not pruned.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — the `actions` object from the chat response, including each action's status and error; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Summary

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates (format in §6) |
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a market order |
| GET | `/api/portfolio/history` | Most recent 500 portfolio snapshots (for P&L chart) |
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |
| POST | `/api/chat` | Send a message, receive the complete response with executed actions |
| GET | `/api/chat/history` | The last 20 chat messages (to restore the chat panel on page load) |
| GET | `/api/health` | Health check (for Docker/deployment) |

### Conventions

- JSON in, JSON out. Timestamps in REST responses are ISO 8601 strings (UTC).
- Errors use FastAPI's default shape `{"detail": "Human-readable message"}`, with status codes `400` (business rule or value validation), `404` (not found), `409` (conflict), and `422` (request body structure).
- `422` is used only for structural problems that Pydantic catches: missing fields, wrong types, and `side` not in `"buy" | "sell"` / `action` not in `"add" | "remove"` (use `Literal` types). Do **not** put `pattern=` or `gt=` constraints on ticker or quantity fields; those checks are done manually in the handler and return `400`.
- Tickers are normalised to uppercase and must match `^[A-Z]{1,5}$` after normalisation. Otherwise → `400`. This applies to request bodies and to the `{ticker}` path parameter on `DELETE /api/watchlist/{ticker}`.

### Business Rules (shared by manual trades and AI trades)

- `quantity` must be a finite number (`math.isfinite`; reject `inf`/`NaN` with `400`), is then rounded to 4 decimal places, and must then be > 0 (so `0.00001` → `0` → `400`). The trade value `quantity × price` must also be finite. These checks run before any state changes, and apply equally to AI-requested trades.
- A trade is rejected (`400`) if the price cache has no price for the ticker. In practice you can only trade tickers that are on the watchlist or already held.
- **Buy**: requires `cash_balance >= quantity × price`. New `avg_cost = (old_qty × old_avg + qty × price) / (old_qty + qty)`.
- **Sell**: requires held quantity ≥ `quantity`. `avg_cost` is unchanged. If the remaining quantity is below `1e-6`, the position row is deleted.
- Realized P&L is not tracked or displayed. Only unrealized P&L is shown.
- Each trade appends to `trades`, updates `positions` and `users_profile.cash_balance`, and records a portfolio snapshot, all in **one SQLite transaction**. If any step fails, the whole trade rolls back. Tracked-ticker changes (`source.remove_ticker()` after a sell to zero) run only **after** the commit succeeds. Watchlist add/remove follows the same pattern: commit the DB change first, then call `add_ticker()`/`remove_ticker()`.
- All trade execution and all watchlist mutations, manual or AI, run under one process-wide `asyncio.Lock`, including the tracked-ticker check-and-remove in §6. This means concurrent requests cannot race on cash, quantity, or which tickers are priced.

### `GET /api/portfolio`

```json
{
  "cash_balance": 8100.0,
  "positions_value": 1905.2,
  "total_value": 10005.2,
  "unrealized_pnl": 5.2,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 10,
      "avg_cost": 190.0,
      "current_price": 190.52,
      "market_value": 1905.2,
      "unrealized_pnl": 5.2,
      "unrealized_pnl_percent": 0.2737
    }
  ]
}
```

If a held ticker temporarily has no cached price, `current_price` falls back to `avg_cost`.

### `POST /api/portfolio/trade`

Request: `{"ticker": "AAPL", "side": "buy", "quantity": 10}`

Response `200`:
```json
{
  "trade": {"id": "uuid", "ticker": "AAPL", "side": "buy", "quantity": 10, "price": 190.5, "executed_at": "2026-10-21T14:00:00Z"},
  "cash_balance": 8095.0
}
```

Errors: `400` for insufficient cash, insufficient shares, no price available, or an invalid ticker/quantity.

### `GET /api/portfolio/history`

Returns the most recent 500 snapshots in ascending time order:
```json
{"snapshots": [{"total_value": 10000.0, "recorded_at": "2026-10-21T14:00:00Z"}]}
```

### `GET /api/watchlist`

```json
{
  "tickers": [
    {"ticker": "AAPL", "price": 190.52, "previous_price": 190.41, "direction": "up", "open_price": 190.0, "day_change_percent": 0.2737}
  ]
}
```

Price fields are `null` until the first price arrives for a newly added ticker. With the Massive source, a symbol Massive doesn't recognise never gets a price, so its price fields stay `null` and the frontend shows a placeholder (e.g. `—`). It can't be traded, because there is no cached price.

### `POST /api/watchlist`

Request: `{"ticker": "PYPL"}` → `201` `{"ticker": "PYPL"}`

Errors: `400` for an invalid format, `409` if already on the watchlist. There is no existence check against the data source: the simulator prices any well-formed ticker, and with Massive unknown symbols simply never receive a price.

### `DELETE /api/watchlist/{ticker}`

`204` on success, `404` if the ticker is not on the watchlist. Removing a ticker does not affect positions in it (see Tracked Tickers in §6).

### `POST /api/chat`

Request: `{"message": "Buy 5 shares of NVDA"}`

Response `200`:
```json
{
  "message": "Adding PYPL to your watchlist and placing orders to buy 5 NVDA and 100 TSLA.",
  "actions": {
    "watchlist_changes": [
      {"ticker": "PYPL", "action": "add", "status": "executed"}
    ],
    "trades": [
      {"ticker": "NVDA", "side": "buy", "quantity": 5, "status": "executed", "price": 875.2},
      {"ticker": "TSLA", "side": "buy", "quantity": 100, "status": "failed", "error": "Insufficient cash"}
    ]
  }
}
```

`status` is `"executed"` or `"failed"`. `error` is present only when an action failed, and `price` only on executed trades. If the LLM call fails, the response is still `200` (see §9).

### `GET /api/chat/history`

Returns the last 20 messages from `chat_messages` in ascending time order. `actions` is `null` for user messages.
```json
{
  "messages": [
    {"id": "uuid", "role": "user", "content": "Buy 5 shares of NVDA", "actions": null, "created_at": "2026-10-21T14:00:00Z"},
    {"id": "uuid", "role": "assistant", "content": "Placing an order to buy 5 NVDA.", "actions": {"watchlist_changes": [], "trades": [{"ticker": "NVDA", "side": "buy", "quantity": 5, "status": "executed", "price": 875.2}]}, "created_at": "2026-10-21T14:00:01Z"}
  ]
}
```

### `GET /api/health`

`200` `{"status": "ok"}`

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the last 20 messages from the `chat_messages` table, including each assistant message's `actions` so the LLM can see the results of earlier actions, including failures
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes the requested actions: **watchlist changes first, then trades, each in the order given**. Each action is independent, so one failure does not roll back or skip the others. Under the simulator, a ticker added in this step is priced immediately, so "add X and buy X" works in one turn. Under Massive, the new ticker has no price until the next poll, so a same-turn trade in it fails with "no price available". This is expected.
7. Stores the user message, and the assistant message with its `actions` results, in `chat_messages`
8. Returns the response described in §8 (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

Chat turns are serialized: steps 1–7 run under a separate process-wide `asyncio.Lock` dedicated to chat, so two concurrent messages cannot load the same history or store messages out of order. The trade/watchlist mutation lock (§8) is taken only around each action in step 6, never across the LLM network call. The frontend also disables the send button while a chat response is pending.

There is exactly one LLM call per user message. The LLM writes its `message` before the actions run, so it cannot describe failures in that same message. Instead:
- The response's `actions` carry the per-action `status`/`error`, and the frontend shows failures inline next to the message
- The LLM sees those results in the history on the next turn

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation and business rules as manual trades (§8)
- `watchlist_changes` (optional): Array of watchlist modifications; `action` is `"add"` or `"remove"`

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

### Failure Handling

If the LLM call times out, OpenRouter returns an error, or the response cannot be parsed as the schema, `/api/chat` still returns `200` with:
- a fixed apologetic `message` (e.g., "Sorry, I couldn't process that request. Please try again.")
- empty actions: `{"watchlist_changes": [], "trades": []}`

Both the user message and the fallback assistant message are stored, and the stored `actions` uses the same empty object. Every assistant message, fallback or not, always has both keys present, even when they are empty lists. The error is logged server-side.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Phrase trade confirmations as intent ("Placing an order to buy…"), since actions run after the message is written
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. The resulting actions still go through the real execution path. This enables:
- Fast, free, reproducible E2E tests
- CI/CD pipelines

Mock rules: the user message is trimmed, then matched against each regex below as a **whole-message, case-insensitive** match (`re.fullmatch(..., re.IGNORECASE)`), and the first match wins. `<qty>` is echoed as the user typed it, and `<TICKER>` is echoed uppercased.

| Regex | Mock response |
|---|---|
| `buy\s+(\d+(?:\.\d+)?)\s+([a-z]{1,5})` | `message: "Buying <qty> <TICKER>."`, one buy trade |
| `sell\s+(\d+(?:\.\d+)?)\s+([a-z]{1,5})` | `message: "Selling <qty> <TICKER>."`, one sell trade |
| `add\s+([a-z]{1,5})` | `message: "Adding <TICKER> to your watchlist."`, one watchlist add |
| `remove\s+([a-z]{1,5})` | `message: "Removing <TICKER> from your watchlist."`, one watchlist remove |
| anything else | `message: "This is a mock response from FinAlly."`, no actions |

Natural phrasings such as "Buy 5 shares of NVDA" deliberately fall through to the default response. E2E tests use the exact forms, e.g. `buy 1 AAPL`.

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout and no client-side routes. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change % (`day_change_percent` from SSE), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using `/api/portfolio/history`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill. API errors (`detail`) are shown inline.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response, and the send button disabled until the response arrives. Executed trades and watchlist changes are shown inline as confirmations, and failed actions inline with their error.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Data Flow

- On load: fetch `/api/portfolio`, `/api/watchlist`, `/api/portfolio/history` and `/api/chat/history`, and open the SSE stream
- After any mutation (trade, watchlist add/remove, chat response with actions): re-fetch `/api/portfolio` and `/api/watchlist`. After a trade, also re-fetch `/api/portfolio/history`.
- Live values are computed client-side from SSE prices: header total value = `cash_balance + Σ(quantity × live price)`, plus per-position current price, P&L and heatmap colors
- Price precedence for every portfolio display (header, positions table, heatmap): latest SSE `price` for the ticker → `current_price` from the last `/api/portfolio` fetch → `avg_cost`. This covers held tickers with no live price, e.g. at Massive startup or after failed polls.
- Re-fetch `/api/portfolio/history` periodically (e.g., every 30s) so the P&L chart picks up background snapshots

### Charting

- **Main price chart and P&L chart**: [Lightweight Charts](https://github.com/tradingview/lightweight-charts) (canvas-based, built for financial data)
- **Sparklines**: a small hand-rolled inline SVG component (no library)
- **Heatmap**: a hand-rolled treemap (simple squarified or slice-and-dice layout using absolutely positioned divs). Each cell exposes `data-ticker` and `data-pnl="positive|negative|flat"` attributes for testing.

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`. Derive connection status from EventSource events: `open` → connected; `error` with `readyState === CONNECTING` → reconnecting; `readyState === CLOSED` → disconnected.
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- In production all API calls go to the same origin (`/api/*`) — no CORS needed
- **Local development**: the frontend prefixes all API/SSE URLs with `process.env.NEXT_PUBLIC_API_BASE_URL`. This is empty in production builds, and set to `http://localhost:8000` in `frontend/.env.development`. The backend enables CORS for `http://localhost:3000` only, so `next dev` on :3000 can talk to FastAPI on :8000.
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000. API routers (including SSE) are registered **before** mounting `StaticFiles(directory="static", html=True)` at `/`, so `/api/*` is never shadowed. No SPA fallback is needed, because the app has a single page.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: thin PowerShell equivalents that mirror the bash scripts line-for-line.

All scripts should be idempotent — safe to run multiple times. There is no top-level `docker-compose.yml`; the start scripts are the only supported launcher.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface, `open_price`/`day_change_percent` are correct, a Massive poll that completes after `remove_ticker()` does not re-insert the ticker
- Portfolio: trade execution logic, weighted avg cost on buys, avg cost unchanged on sells, position row deleted on full sell, quantity rounding, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, trading a ticker with no price, selling at a loss)
- Tracked tickers: removing a held ticker from the watchlist keeps it tracked, and selling an unwatched position to zero stops tracking it. Held tickers without seed data start at `avg_cost` on startup.
- Validation: ticker/quantity value errors return `400`, structural errors return `422`, quantity is rounded before the `> 0` check, and `inf`/`NaN` quantities are rejected for both manual and AI trades
- Atomicity: a failure partway through a trade (e.g. snapshot insert raises) leaves cash, positions and trades unchanged
- Chat concurrency: two simultaneous `/api/chat` requests are processed one after the other, and messages are stored in order
- LLM: structured output parsing handles all valid schemas, malformed responses produce the fallback message, action ordering (watchlist before trades), partial failures report per-action status, every mock-mode rule
- API routes: correct status codes, response shapes per §8, error handling
- Startup: refuses to start without `OPENROUTER_API_KEY` unless `LLM_MOCK=true`

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Connection status indicator state transitions (connected → reconnecting → disconnected)
- Watchlist CRUD operations
- Portfolio display calculations (live total value from SSE prices, and the SSE → REST → `avg_cost` price fallback)
- Chat message rendering, loading state, and inline display of failed actions

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap cells render with the correct `data-pnl` attributes, P&L chart has data points
- AI chat (mocked): send `buy 1 AAPL`, receive a response, trade execution appears inline and the position appears. Reload the page and the conversation is restored.

SSE reconnection is not E2E-tested. EventSource retry is browser-native, and the indicator logic is covered by frontend unit tests.
