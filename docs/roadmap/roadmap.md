# `vyapaar-docs` — Documentation Roadmap

## Milestone 1: Architecture, DDD Bounded Contexts & Migration Guides (P0) — COMPLETE
- [x] Port `docs/architecture/`, `docs/backtesting/`, `docs/ai/`, `docs/tutorials/`, and `docs/reference/` from `honba-docs/docs/` into `vyapaar-docs`.
- [x] Consolidate unique mdBook pages (`crate_hierarchy.md`, `message_flow.md`, `avoiding_backtest_biases.md`, `nl_verification.md`, `contributing/`) into unified `docs/` structure.
- [x] Update all subsystem names, crate references, and diagrams to `vyapaar-*`.

## Milestone 2: Strict Build Gate (`mkdocs build --strict`) (P0) — COMPLETE
- [x] Update `mkdocs.yml` navigation tree with 100% doc coverage.
- [x] Fix all relative link anomalies between `conformance_matrix.md` and module subdirectories.
- [x] Verify zero warnings and clean exit under `mkdocs build --strict`.

## Milestone 3: New DDD Design Documentation (P0) — COMPLETE
- [x] Add `docs/architecture/architecture_layers.md` — 6-layer, 15-repo DDD topology (ported from workspace root `docs/design/architecture.md`).
- [x] Add `docs/architecture/ddd_conflicts_resolutions.md` — DDD bounded-context conflict resolutions from the `honba` → `vyapaar` migration (ported from workspace root `docs/ddd-conflict.md`).
- [x] Cross-reference `docs/design/architecture.md` to the layered topology.
- [x] Verify `mkdocs build --strict` green.

## Milestone 4: New Sub-Crate & Capability Documentation (P1) — COMPLETE
- [x] Add `docs/architecture/ledger_performance.md` — `vyapaar-ledger-performance` trade-performance journal (annotations, metrics, edge score, calendar, comparison; plan 0002).
- [x] Add `docs/architecture/ledger_allocation.md` — `vyapaar-ledger-allocation` portfolio allocation & optimization engine (mean-variance, risk-parity; plan 0001 Phase 1).
- [x] Add `docs/architecture/runtime_jobs.md` — `vyapaar-runtime-jobs` deterministic cron/background job scheduler (plan 0001 Stage 1).
- [x] Update `docs/api/rest.md` — full endpoint reference including `/trade-journal/*`, `/market-data/symbol-profile/*`, `/market-data/calendar`, `/approval/*`, `/profile/*`, updated `vyapaar serve` flags.
- [x] Update `docs/api/python-sdk.md` — typed client methods for all new routes.
- [x] Update `docs/conformance_matrix.md` — new crates (`vyapaar-ledger-allocation`, `vyapaar-ledger-performance`, `vyapaar-runtime-jobs`), updated statuses (16 IMPLEMENTED, 1 IN-PROGRESS, 1 PLANNED).
- [x] Update `docs/strategies/index.md` — versioned strategy manifests & baskets (plan 0001: `StrategyManifest`, `BasketManifest`, `PromotionStatus`).
- [x] Update `docs/strategies/step_manifests_and_chaining.md` — manifests & baskets section.
- [x] Update `docs/india/index.md` — India data lakehouse (calendar, symbol profile, filings; gaps 0001–0003 closed).
- [x] Update `docs/india/brokers.md` — adapter status (Dhan, Zerodha implemented; Angel One, Fyers, Upstox, Kotak, IIFL scaffold).
- [x] Update `docs/examples/index.md` — stages 6–8 (StockSharp port, AI research, production).

## Milestone 5: Ongoing (P2)
- [ ] Add per-repo code reference pages for `vyapaar-ledger-*` sub-crates (OMS, portfolio, journal) migrated from `docs/design/code-*.md` in each sub-repo.
- [ ] Add `docs/architecture/ui_workstation.md` — `vyapaar serve`, CLI 40-command TUI, six-gate ValidationBlock UI card, per-trade journal rows.
- [ ] Add `docs/ai/omniroute_details.md` — expanded OmniRoute free-tier LLM backend reference (plan 0004).
- [ ] Add `docs/examples/stages.md` — detailed stage 6–8 walkthroughs.
- [ ] Add `docs/adapters/` — broker adapter certification status and contract test coverage (plan 0002, gaps 0001–0007).
- [ ] Keep `docs/conformance_matrix.md` in sync with sub-repo plan/gap indexes as new capabilities land.
