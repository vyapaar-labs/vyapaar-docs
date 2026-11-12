# Platform Conformance Matrix

> **Single source of truth** mapping every Vyapaar crate/module to its documented
> capability and implementation status. This is the authoritative reference for
> closing the documentation↔implementation lag (Gap 7). Status badges on each
> architecture page mirror this matrix.

## Status taxonomy

| Badge | Meaning |
|:------|:--------|
| `[STATUS: IMPLEMENTED]` | Code is compiled, unit-tested, and passing CI conformance. |
| `[STATUS: IN-PROGRESS]` | Core contracts and basic engines active; edge features under development. |
| `[STATUS: SCAFFOLD]` | Uniform API skeleton and interfaces defined; execution logic stubbed. |
| `[STATUS: PLANNED]` | Architectural blueprint formulated; implementation not yet started. |

## Matrix

| Crate / module | Documented capability | Status | Doc reference | Test coverage |
|:---------------|:----------------------|:-------|:--------------|:--------------|
| `vyapaar-core` | Value objects (`Price`, `Quantity`, `Money`), `InstrumentId`, `Decision`, `Order`, `Portfolio`, `Signal`, `Sizing`, `Market`, `Calendar`, `Events`, `Wire`, `Types`, quant transform primitives (`simple_returns`, `realized_volatility`, `ewma_volatility`, `rolling_mean`, `rolling_stdev`, `sharpe_ratio`, `max_drawdown`, `correlation`, `beta`, `historical_var`, `historical_cvar`, `tracking_error`), statistical validation gates (`ValidationBlock`: DSR, PBO, walk-forward, holdout, Monte Carlo, cost-stress) | `[STATUS: IMPLEMENTED]` | [DDD](architecture/domain_driven_design.md), [Money](architecture/money.md), [Contracts](architecture/contracts_from_rust.md), [Analytics Gates](architecture/domain_driven_design.md), [Quant Primitives](architecture/domain_driven_design.md) | `tests/core_tests.rs`, `tests/proptest_invariants.rs`, `tests/golden_test.rs`, `tests/schema_compatibility_test.rs`, `tests/quant_tests.rs`, `tests/quant_proptest.rs`, `tests/analytics_gates_tests.rs` |
| `vyapaar-ledger` | Double-entry FIFO ledger, OMS state machine, tax-lot matching, portfolio accounting, repository traits | `[STATUS: IMPLEMENTED]` | [Money](architecture/money.md), [Order Reject & Cancel](architecture/order_reject_cancel.md), [Tradebook](architecture/tradebook.md), [Architecture Layers](architecture/architecture_layers.md) | `crates/vyapaar-ledger-book/tests/ledger_proptest_invariants.rs`, `crates/vyapaar-ledger-portfolio/tests/apply_fill_tests.rs`, `crates/vyapaar-ledger/tests/repository_and_sizing_tests.rs`, `crates/vyapaar-ledger-oms/tests/*`, `crates/vyapaar-ledger-journal/tests/*` |
| `vyapaar-ledger-allocation` | Portfolio allocation & optimization: mean-variance, risk-parity target weights, weight-bounds constraints, `PortfolioState` read-adapter over `vyapaar-ledger-portfolio`, pure deterministic `optimize()` | `[STATUS: IMPLEMENTED]` | [Ledger Allocation](architecture/ledger_allocation.md) | `tests/optimize_tests.rs` |
| `vyapaar-ledger-performance` | Trade-performance journal: annotations (tags/rating/review/planned risk), `JournalStore`, performance metrics engine + edge score composite (v1), daily P&L calendar, account-vs-account comparison | `[STATUS: IMPLEMENTED]` | [Ledger Performance](architecture/ledger_performance.md) | `tests/model_tests.rs`, `tests/store_tests.rs`, `tests/metrics_tests.rs`, `tests/calendar_tests.rs`, `tests/compare_tests.rs` |
| `vyapaar-risk` | SEBI algo_id registry, OTR circuit breakers, price bands, freeze limits, pre-trade risk gate | `[STATUS: IMPLEMENTED]` | [Order Reject & Cancel](architecture/order_reject_cancel.md), [Risk Migration Plan](architecture/migration_plan_kokoro_risk_to_mon.md) | `tests/proptest_tests.rs`, `tests/risk_tests.rs`, `tests/phase7_tests.rs` |
| `vyapaar-market` | Multi-broker binary WebSocket feeds, tick coalescing, L2/L3 depth, 100+ StockSharp indicators, Parquet Lakehouse, Bhavcopy loaders | `[STATUS: IMPLEMENTED]` | [Adapter Interface](architecture/adapter_interface.md), [WASM Indicators](architecture/wasm_indicators.md) | `crates/vyapaar-market-indicators/tests/*`, `crates/vyapaar-market-feed/tests/*`, `crates/vyapaar-market-market/tests/*`, `crates/vyapaar-market-loaders/tests/*` |
| `vyapaar-exec` | Token-bucket rate limiter, freeze slicer, headless TOTP auth, Smart Order Router (SOR), SQLite WAL audit journal, broker execution adapters | `[STATUS: IMPLEMENTED]` | [Adapter Interface](architecture/adapter_interface.md), [Order Reject & Cancel](architecture/order_reject_cancel.md) | `crates/vyapaar-exec-core/tests/*`, `crates/vyapaar-exec-router/tests/*`, `crates/vyapaar-exec-slicer/tests/*`, `crates/vyapaar-exec-audit/tests/*`, `crates/vyapaar-exec-auth/tests/*` |
| `vyapaar-runtime` | Tokio actor pipeline, backtest engine, REST API server, persistence, strategy tester/runner, GA/Bayesian/WFO/DSR enhancer, CLI, job scheduler (`vyapaar-runtime-jobs`), trade-journal + market reference REST endpoints (`vyapaar-runtime-api-rest`), chat agent runtime (`/ws/chat` with read-only tools + profile alert store; sessions in-memory pending gap 0006) | `[STATUS: IMPLEMENTED]` | [Actor Runtime & Event Loop](architecture/event_loop.md), [Strategy Tester & Enhancer](architecture/strategy_tester_and_enhancer.md), [Runtime Overview](architecture/runtime_overview.md), [Runtime Chat Agent](architecture/runtime_chat.md), [Using the Simulator](architecture/simulator_usage.md), [Job Scheduler](architecture/runtime_jobs.md), [REST API](api/rest.md) | `crates/vyapaar-runtime-backtest/tests/*`, `crates/vyapaar-runtime-api-rest/tests/*`, `crates/vyapaar-runtime-actors/tests/*`, `crates/vyapaar-runtime-enhancer/tests/*`, `crates/vyapaar-runtime-persistence/tests/*`, `crates/vyapaar-runtime-jobs/tests/*`, `crates/vyapaar-runtime-api-rest/tests/trade_journal.rs`, `crates/vyapaar-runtime-api-rest/tests/market_reference.rs`, `crates/vyapaar-runtime-api-rest/tests/chat_*` |
| `vyapaar-runtime-jobs` | Deterministic cron/background job scheduler (`JobScheduler`, `JobSpec`, `CronSchedule`), bounded worker pool, fencing leases, approval queue, per-run journal (`manifest.json`, `events.ndjson`, `trades.json`, `metrics.json`) | `[STATUS: IMPLEMENTED]` | [Job Scheduler](architecture/runtime_jobs.md), [REST API](api/rest.md) | `crates/vyapaar-runtime-jobs/tests/*` |
| `vyapaar-ffi` | PyO3 FFI bridge, zero-copy Arrow/PyCapsule memory, `vyapaar` Python wheel, GIL management (`py.allow_threads`), codegen | `[STATUS: IMPLEMENTED]` | [Python-First Architecture](architecture/python_first.md), [Contracts from Rust](architecture/contracts_from_rust.md) | `crates/vyapaar-ffi-py/tests/*`, `crates/vyapaar-ffi-codegen/tests/*`, `tests/*.py` |
| `vyapaar-agent` | Autonomous research loop, MCP server, prompt replayer, LLM integration, genetic strategy synthesis, DSR/PBO/walk-forward analytics | `[STATUS: IMPLEMENTED]` | [AI Engine & Quant Researcher](architecture/ai_engine.md) | `crates/vyapaar-agent-analytics/tests/*`, `crates/vyapaar-agent-mcp/tests/mcp_tests.rs`, `crates/vyapaar-agent-agent/tests/*` |
| `vyapaar-ui` | Leptos CSW frontend, Strategy Studio visual canvas, Simulator port, openalgo-charts bridge, WASM charting | `[STATUS: IMPLEMENTED]` | [vyapaar-ui Simulator Port](architecture/kao_simulator_port.md), [vyapaar-ui Strategy Studio](architecture/strategy_studio.md), [WASM Indicators](architecture/wasm_indicators.md) | `crates/openalgo-charts-rs/tests/*`, `crates/vyapaar-ui-app/tests/*`, `crates/vyapaar-ui-api-wasm/tests/*` |
| `vyapaar-strategies` | Production strategy catalog (quoting, arbitrage, TWAP/VWAP slicers, Turtle breakout, risk parity), `step.toml` manifests, `registry.json` | `[STATUS: IMPLEMENTED]` | [Strategies index](strategies/index.md), [Step Manifests & Chaining](strategies/step_manifests_and_chaining.md) | `test_readme_sections.py`, `test_step_manifests.py`, per-strategy `tests/` |
| `vyapaar-adapters` | Community broker connectors (Dhan, Zerodha implemented; Angel One, Fyers, Upstox, Kotak, IIFL scaffolded) | `[STATUS: IN-PROGRESS]` | [Brokers](india/brokers.md), [Adapter Configuration](india/adapter_configuration.md) | `crates/vyapaar-exec-zerodha/tests/*`, `crates/vyapaar-exec-upstox/tests/*`, `crates/vyapaar-exec-angelone/tests/*` |
| `vyapaar-data` | Point-in-time corporate actions, splits, dividends, coverage ledgers, Parquet catalog, DuckDB queries | `[STATUS: IMPLEMENTED]` | [Universes](india/universes.md), [Cost Model](india/cost_model.md) | Python test suite (`pytest`) |
| `vyapaar-examples` | Runnable Python reference scripts (backtesting, screening, SIP, options, AI research, production) | `[STATUS: IMPLEMENTED]` | [Examples](examples/index.md), [Known Issues](examples/known_issues.md) | `pytest` per numbered script `Pitfalls::` sections |
| `vyapaar-fe` | TypeScript web desktop UI: dockview workspace, trading terminal, real-time depth, screener, portfolio management | `[STATUS: IMPLEMENTED]` | [vyapaar-ui Strategy Studio](architecture/strategy_studio.md) | `vitest`, `tsc --noEmit` |
| `vyapaar-ledger-journal` | Double-entry general ledger for the Indian equity trading lifecycle (`AccountCode`-keyed chart of accounts, `Transaction`/`DoubleEntryLedger`) and the Indian statutory tax calculator (STT, GST, Stamp Duty, SEBI fees, DP charges, exchange turnover fee); pure zero-I/O, temporal determinism | `[STATUS: IMPLEMENTED]` | [Tradebook (Journal & Replay)](architecture/tradebook.md), [Money](architecture/money.md) | `crates/vyapaar-ledger-journal/tests/unit_tests.rs`, `tax_calculator_tests.rs`, `ledger_proptest_invariants.rs`, `golden.rs` |
| `vyapaar-runtime-enhancer` | GA, Bayesian optimization, walk-forward analysis, DSR gate (sub-crate of vyapaar-runtime) | `[STATUS: IMPLEMENTED]` | [Strategy Tester & Enhancer](architecture/strategy_tester_and_enhancer.md) | `crates/vyapaar-runtime-enhancer/tests/unit_tests.rs` |

## Status summary

| Status | Count | Crates/modules |
|:-------|:------|:---------------|
| IMPLEMENTED | 17 | vyapaar-core, vyapaar-ledger, vyapaar-ledger-allocation, vyapaar-ledger-journal, vyapaar-ledger-performance, vyapaar-risk, vyapaar-market, vyapaar-exec, vyapaar-runtime, vyapaar-runtime-jobs, vyapaar-ffi, vyapaar-agent, vyapaar-ui, vyapaar-strategies, vyapaar-data, vyapaar-examples, vyapaar-fe |
| IN-PROGRESS | 1 | vyapaar-adapters |
| SCAFFOLD | 0 | — |
| PLANNED | 0 | — |

## How this matrix is verified

- Rust crates: `cargo test --workspace` must be green; clippy with `-D warnings`.
- Python packages (`vyapaar-ffi`, `vyapaar-strategies`, `vyapaar-adapters`, `vyapaar-data`, `vyapaar-examples`): `pytest -q` must be green.
- `scripts/verify_conformance.py` parses `Cargo.toml`/`pyproject.toml` members and git history to flag crates referenced in docs but absent from the workspace (and vice versa). Run in CI.

## Change log

- 2026-11-12: Runtime documentation pass — added chat-agent runtime + runtime overview rows/links; corrected `vyapaar-ledger-journal` to IMPLEMENTED (double-entry ledger + statutory tax calculator); corrected enhancer test-coverage reference.
- 2026-10-30: Initial matrix published (M4-1). All statuses verified against crate source and test files in the workspace.
