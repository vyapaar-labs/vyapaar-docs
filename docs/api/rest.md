# REST Read API and `vyapaar serve`

`vyapaar serve` exposes a full HTTP/WebSocket API. It boots `vyapaar-runtime-api-rest` as a standalone server over a directory of Parquet bar files plus optional profile, portfolio, journal, and token stores.

```bash
vyapaar serve --data-dir ./bars                      # 127.0.0.1:8000
vyapaar serve --data-dir ./bars --addr 127.0.0.1:0   # pick a free port
vyapaar serve --data-dir ./bars --profiles-dir ./profiles --portfolios-dir ./portfolios \
    --journals-dir ./journals --tokens-file ./tokens.toml
```

The data directory holds `SYMBOL.EXCHANGE.parquet` files and is loaded **once** at start-up; a bad directory fails before anything listens. The first stdout line is `listening on http://<addr>` with the address actually bound.

| Flag | Default | Meaning |
|------|---------|---------|
| `--data-dir` | required | Directory of Parquet bar files (`SYMBOL.EXCHANGE.parquet`) |
| `--addr` | `127.0.0.1:8000` | Listen address |
| `--cors-origin <origin>` | none | Allow this browser origin (repeatable) |
| `--profiles-dir` | `data/profiles` | Directory holding one JSON profile per user (`/profile` routes) |
| `--portfolios-dir` | `data/portfolios` | Directory holding one JSON portfolio per user (`/portfolio` routes) |
| `--journals-dir` | `data/journals` | Directory holding one sub-directory per backtest/run journal |
| `--tokens-file` | none | Bearer-token file (TOML); enables auth when set |

## Security posture

- **Loopback by default.** If `--addr` is not loopback, `vyapaar serve` prints a warning on stderr: the API has no auth or TLS unless `--tokens-file` is set, so anyone who can reach it can read and write data.
- **No TLS, no rate limiting by default.**
- **No CORS by default** (no CORS headers are sent). Opt in with `--cors-origin https://app.example` (repeatable). `*` and values that are not valid header values are rejected at start-up.
- **Auth** (optional): when `--tokens-file` is set, bearer tokens are required on mutating routes (`POST`, `DELETE`, `PUT`, `PATCH`). Read routes remain public.

## Routes

All routes below are implemented. Routes marked `501` answer `not_implemented` and have no typed client method.

| Method | Route | Notes |
|--------|-------|-------|
| `GET` | `/capabilities` | API capabilities manifest |
| `GET` | `/health` | `{"status": "ok"}` |
| `GET` | `/schema` / `/schema/openapi/v{1,2}.json` | OpenAPI 3.1 document |
| `GET` | `/docs` | Swagger UI |
| `GET` | `/instruments` | Filters `exchange`, `symbol`; id order |
| `GET` | `/instruments/{id}` | `{id}` is `SYMBOL.EXCHANGE`, split on the last dot |
| `GET` | `/quotes` | `symbols` (comma list), `venue`, `as_of` (inclusive) |
| `GET` | `/bars/{id}` | `tf`, `from`, `to`; half-open `[from, to)`, ascending `ts_event` |
| `POST` | `/bars` | Ingest bars |
| `GET` | `/depth/{id}` | `depth` 1..=50, default 5 — over bars returns 404 `market_data_unavailable` |
| `GET` | `/market-data/symbol-profile/{id}` | **Plan 0003 Phase 3** — company fundamentals / symbol profile |
| `GET` | `/market-data/calendar` | **Plan 0003 Phase 3** — economic/earnings/dividends/splits/IPO events; filter by `from`, `to`, `category` |
| `GET` | `/screener/scan` | Custom AST-filtered stock scan |
| `POST` | `/strategies/verify` | Verifies a manifest, returns the IR |
| `GET` | `/strategies` | Lists compiled strategies of this process |
| `POST` | `/strategies` | Compiles and stores a manifest; returns `id` and `IR` |
| `POST` | `/backtests` | Creates a backtest run; returns `run_id` |
| `GET` | `/backtests` | Lists backtests |
| `GET` | `/backtests/{id}` | Backtest by ID |
| `GET` | `/backtests/{id}/journal` | Backtest execution journal |
| `GET` | `/backtests/{id}/events` | SSE stream of run progress (`text/event-stream`) |
| `POST` | `/sweeps` | Creates a parameter sweep |
| `GET` | `/sweeps` | Lists sweeps |
| `GET` | `/sweeps/{id}` | Sweep by ID |
| `GET` | `/sweeps/{id}/journal` | Sweep execution journal |
| `POST` | `/orders` | Places an order (risk-gated, ADR 0018) |
| `GET` | `/orders` | Lists orders |
| `DELETE` | `/orders/{id}` | Cancels an order |
| `GET` | `/orders/{id}/fills` | Order fills |
| `POST` | `/positions/close` | Emergency close-all |
| `GET` | `/positions` | Open positions |
| `GET` | `/positions/audit` | Position audit |
| `GET` | `/trades` | Portfolio trades (alias of `/portfolio/trades`) |
| `GET` | `/portfolio` | **501** `not_implemented` (`valuation_engine_mismatch`) — live valuation engine not yet wired |
| `GET` | `/portfolio/accounts` | **501** `not_implemented` (`valuation_engine_mismatch`) |
| `GET` | `/portfolio/positions` | **501** `not_implemented` (`valuation_engine_mismatch`) |
| `GET` | `/portfolio/holdings` | **501** `not_implemented` (`valuation_engine_mismatch`) |
| `GET` | `/portfolio/trades` | Portfolio trades (implemented) |
| `GET` | `/portfolio/chart` | Equity curve |
| `GET` | `/portfolio/concentration` | Concentration risk |
| `GET` | `/portfolio/wac` | Weighted average cost |
| `GET` | `/ledger` | Double-entry ledger |
| `GET` | `/tax/report` | Tax report |
| `GET` | `/fees` | Fees |
| `GET` | `/reports/ledger` | Ledger report |
| `GET` | `/reports/tax` | Tax report |
| `GET` | `/reports/snapshots` | Portfolio snapshots |
| `GET` | `/reports/audit` | Position audit |
| `GET` | `/reports/concentration` | Concentration risk |
| `GET` | `/reports/wac` | Weighted average cost |
| `GET` | `/reports/fees` | Fees report |
| `GET` | `/reports/equity-curve` | Equity curve |
| `GET` | `/reports/fills/{order_id}` | Order fills |
| `GET` | `/trade-journal/accounts` | **Plan 0003 Phase 1** — List journal accounts |
| `POST` | `/trade-journal/accounts` | **Plan 0003 Phase 1** — Create journal account |
| `GET` | `/trade-journal/overview` | **Plan 0003 Phase 1** — Performance metrics + edge score |
| `GET` | `/trade-journal/trades` | **Plan 0003 Phase 1** — Closed trades with annotations, cursor-paginated |
| `PATCH` | `/trade-journal/trades/{id}` | **Plan 0003 Phase 1** — Set trade annotation |
| `GET` | `/trade-journal/calendar` | **Plan 0003 Phase 2** — Daily P&L heatmap (`?month=YYYY-MM`) |
| `GET` | `/trade-journal/notes` | **Plan 0003 Phase 2** — Day-keyed journal notes |
| `POST` | `/trade-journal/notes` | **Plan 0003 Phase 2** — Create/replace day-keyed note |
| `GET` | `/trade-journal/compare` | **Plan 0003 Phase 2** — Two-sided account comparison |
| `GET` | `/approval/queue` | Approval queue |
| `POST` | `/approval/approve` | Approve order |
| `POST` | `/approval/reject` | Reject order |
| `GET` | `/trading-state` | Get trading state (operator halt/resume, ADR 0018) |
| `PUT` | `/trading-state` | Put trading state |
| `GET` | `/profile` | Get profile |
| `PUT` | `/profile` | Put profile |
| `GET` | `/profile/layouts` | Get layouts |
| `PUT` | `/profile/layouts/{name}` | Put layout |
| `DELETE` | `/profile/layouts/{name}` | Delete layout |
| `GET` | `/profile/modules` | Get modules |
| `PUT` | `/profile/modules/{id}` | Put module |
| `DELETE` | `/profile/modules/{id}` | Delete module |
| `GET` | `/ws` | Real-time streaming WebSocket (`WsTick` / `WsDepth` push, `WsClientCommand` subscribe/switch) |
| `GET` | `/ws/market` | Real-time market streaming WebSocket (same handler as `/ws`) |
| `GET` | `/ws/chat` | Chat agent WebSocket (`ChatRuntime`); requires `VYAPAAR_GGUF_PATH` or `VYAPAAR_LLM_BASE_URL` — see [Runtime chat agent](../architecture/runtime_chat.md) |
| `GET` | `/journals/{id}` | Run/backtest execution journal |

`POST /orders` is risk-gated (ADR 0018): a refusal is a 422 carrying the `risk_*` code (category `risk`, not retryable); `DELETE /orders/{id}` cancels without a gate and `POST /positions/close` closes without one.

A request body that does not parse on these routes is still a 422 first.

### `GET /capabilities`

The manifest carries `crates`, `market_packs`, `toolsets`, `adapters` and `features`. `endpoints` lists every registry row as `METHOD /path`. The `not_implemented` list names routes that answer 501.

### `GET /screener/scan`

Query: `universe` (JSON array of `SYMBOL.EXCHANGE`, required), `filters` (JSON filter group; omitted matches every instrument), `tf` (default `1d`) and `as_of` (inclusive). The answer is `{rows: [{instrument_id, metrics}]}` in instrument-id order; `metrics` holds the latest value of every metric the filter reads (`null` while warming up). Predicates are evaluated in Rust over the bar dataset. Computable metrics: `open`, `high`, `low`, `close`, `volume`, `price_52_week_high`/`_low` (need 252 bars), `SMA<N>` and `RSI`. Anything else is a 422 `reason = unsupported_metric`, never an empty result; a predicate `timeframe` must equal `tf` (`unsupported_timeframe`). An unknown instrument is a 404. A Parquet data directory holds `1m` bars, so pass `tf=1m` there; the default `1d` is a 404 `market_data_unavailable`.

Limits, each a 422 `too_many_rows` with `context.limit`: 1,000 instruments in the universe, 500 matching rows, 2,000,000 bars read per scan, and 100,000 bars per instrument. Filters are bounded to 8 levels of nesting and 64 predicates (`invalid_filter`). There is no pagination.

### Strategy compilation: `POST /strategies` and `GET /strategies`

**`POST /strategies`**: compiles and stores a manifest. Request body is `{manifest}` only (unknown fields are rejected). Compilation is the same verification step as `POST /strategies/verify`; if it succeeds, the strategy is stored in an in-process catalog and returned as `{id, ir}` (the IR is a dict). The `id` is the SHA-256 of the canonical manifest JSON, so the same manifest gets the same id across processes and is stored once. A manifest already held is re-posted successfully without duplication.

The catalog is per-process, in-memory, and lost on restart; capacity is bounded at **1,000 strategies**. A new strategy past the limit is a 422 `validation_invalid_request` with `context.reason = catalog_full` and `context.limit`, but re-posting a manifest already held always succeeds.

A request body containing `code` or `source` is rejected with 422 `context.reason = source_unsupported`. An invalid manifest gets the same 422 envelope as `/strategies/verify` with `context.reason` naming the issue (e.g., `no_subscriptions`).

**`GET /strategies`**: lists all compiled strategies of this process, ordered by id. Returns `{strategies: [CompiledStrategy, ...]}` where each has `id` and `ir`.

### Trade-Performance Journal: `/trade-journal/*` (Plan 0003)

Backed by `vyapaar-ledger-performance`. These are **distinct** from the existing run/backtest journals (`/journals/{id}`, `/backtests/{id}/journal`, `/sweeps/{id}/journal`) and from `/portfolio/accounts`.

| Method | Route | Purpose |
|--------|-------|---------|
| `GET` | `/trade-journal/accounts` | List journal accounts (`JournalAccount`: id, name, provenance, pnl_convention) |
| `POST` | `/trade-journal/accounts` | Create journal account (provenance: `Manual` / `Imported` / `BrokerSynced`) |
| `GET` | `/trade-journal/overview` | Performance metrics + edge score (`PerformanceMetrics`, `EdgeScore`) |
| `GET` | `/trade-journal/trades` | Closed trades with annotations, cursor-paginated (`?from=&to=&limit=&cursor=`) |
| `PATCH` | `/trade-journal/trades/{id}` | Set trade annotation (`tags`, `rating` 1–5, `review`, `planned_risk`) |
| `GET` | `/trade-journal/calendar` | Daily P&L heatmap (`?month=YYYY-MM`) |
| `GET` | `/trade-journal/notes` | Day-keyed journal notes (`?account=&limit=`) |
| `POST` | `/trade-journal/notes` | Create/replace day-keyed note |
| `GET` | `/trade-journal/compare` | Two-sided account comparison (`?accountA=&accountB=` or `?account=&mode=previous`) |

All monetary values are serialized as strings (Decimal exactness preserved).

### Market Reference Data: `/market-data/*` (Plan 0003 Phase 3)

Backed by `vyapaar-data` (symbol profile, economic calendar) — currently serves **illustrative sample data** pending the `vyapaar-data` → Rust FFI bridge. The owning gaps are tracked in `vyapaar-data/docs/gaps/`: `0002-company-fundamentals-symbol-profile.md` (symbol profile ingestion) and `0001-economic-calendar-ingestion.md` / `0003-calendar-stubbed-data.md` (calendar). Sample-data stubs in `vyapaar-runtime` cite these gaps in code comments.

| Method | Route | Purpose |
|--------|-------|---------|
| `GET` | `/market-data/symbol-profile/{id}` | `SymbolProfileDto`: symbol, name, description, sector, industry, market_cap, exchange, country, ipo_date, etc. |
| `GET` | `/market-data/calendar` | `CalendarEventDto`: instrument, scheduled_at, category, event_name, impact_level, actual, forecast, prior |

### Streaming & Chat: `/ws`, `/ws/market`, `/ws/chat`, SSE

* `/ws` and `/ws/market` share one command-driven handler: the server pushes `WsTick` (instrument, bid/ask/last, prev close) and `WsDepth {bids, asks}` frames as data arrives; the client sends `WsClientCommand` subscribe/switch actions. No polling.
* `/backtests/{id}/events` is an SSE stream replayed from the run's line-indexed `events.ndjson`; pass `resume_from_seq=N` to resume after a reconnect without gaps or duplicates.
* `/ws/chat` connects the [runtime chat agent](../architecture/runtime_chat.md) (`ChatRuntime`). It needs an LLM backend in the environment — `VYAPAAR_GGUF_PATH` (local Candle-GGUF, takes priority) or `VYAPAAR_LLM_BASE_URL`/`VYAPAAR_LLM_MODEL`/`VYAPAAR_LLM_API_KEY` (OpenAI/Ollama-compatible); without either, the socket answers with a clear `chat not configured` message. Sessions are in-memory (`InMemorySessionStore`) and lost on restart; alerts persist on the user profile.

### Behaviour notes

- **Bars cap:** a selection of more than **100,000** bars is rejected with 422 `validation_invalid_request`, `context.reason = "too_many_rows"` and `context.limit`. Narrow `from`/`to`. There is no pagination.
- **Quotes** are derived from bars: bid = ask = close of the latest bar at or before `as_of`, sizes `0` ("last price, no book"). A request is all-or-nothing: one instrument without a quote makes the whole request 404.
- **Depth** over a bar dataset is 404 `market_data_unavailable`; no levels are fabricated.
- Parquet files carry no reference data: instruments are derived as INR equities, lot 1, tick 0.05, and files are read as 1-minute bars.
- Prices are `f64` observations; timestamps are `{iso, unix_nanos}`.

| Case | Status | Code |
|------|--------|------|
| Unknown instrument | 404 | `instrument_not_found` |
| No bar at or before `as_of`; timeframe not held; bar-only depth | 404 | `market_data_unavailable` |
| Bad `tf`/`from`/`to`, empty range, unknown query key, malformed id, `depth` out of range | 422 | `validation_invalid_request` |
| More than 100,000 bars | 422 | `validation_invalid_request` (`too_many_rows`) |
| Unknown route | 404 | `not_found` (an envelope, not an empty body) |
| Known path, unsupported method | 405 | `unsupported` (an envelope) |
| Route not built | 501 | `not_implemented` |
| Port unavailable / timeout / transport | 503 / 504 / 502 | `market_data_unavailable` / `timeout` / `transport_error` |

## Envelope

Every response, success or failure, is an envelope:

```json
{"api_version": "...", "schema_version": 3, "data": {"status": "ok"}}
```

```json
{
  "api_version": "...",
  "schema_version": 3,
  "error": {
    "code": "instrument_not_found",
    "message": "...",
    "retryable": false,
    "context": {"field": "id"}
  }
}
```

Exactly one of `data` and `error` is present. `category` is not on the wire; it is derived from `code`. Use `retryable`, not the status, to decide on retry. (The `schema_version` value shown is illustrative; read it from a response.)

## CLI

- `vyapaar serve --data-dir DIR [--addr ADDR] [--cors-origin ORIGIN]... [--profiles-dir DIR] [--portfolios-dir DIR] [--journals-dir DIR] [--tokens-file FILE]` — boot the REST server.
- `vyapaar verify MANIFEST.json` — verifies a strategy manifest and prints the compiled IR as JSON (the CLI form of `POST /strategies/verify`).
- `vyapaar schema export [--output DIR] [--typescript DIR] [--client DIR] [--openapi DIR] [--pyi DIR] [--mcp DIR] [--docs DIR]` — writes the generated contract artifacts; `vyapaar codegen` is an alias that emits every one (see [Contracts from Rust](../architecture/contracts_from_rust.md)).
- `vyapaar jobs add/list/run/tick/install/logs/pause/resume/approvals/approve/deny` — job scheduler CLI (see [Runtime Job Scheduler](../architecture/runtime_jobs.md)).

Use the typed [Python client](python-sdk.md) rather than parsing this by hand.
