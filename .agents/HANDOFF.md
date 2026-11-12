# `vyapaar-docs` — Inter-Agent Handoff

## Phase / Slice Status
* **Runtime docs pass (COMPLETE, 2026-11-12)**: synced the site with the actual
  `vyapaar-runtime` implementation (all facts re-verified against crate source):
  - New pages: `docs/architecture/runtime_overview.md` (14-crate overview,
    four operational surfaces, parity pipeline, invariants) and
    `docs/architecture/runtime_chat.md` (the previously undocumented `/ws/chat`
    chat agent runtime: env-gated LLM backend — `VYAPAAR_GGUF_PATH` or
    `VYAPAAR_LLM_BASE_URL` — read-only tools + `Profile.alerts` alert store,
    in-memory sessions). Both registered in `mkdocs.yml` nav.
  - Rewrote `docs/architecture/event_loop.md` against `vyapaar-runtime-actors`:
    no `AuditPersistenceActor` (audit is the injected `SqliteAuditLedger` on
    Risk/Execution), coalescing buffer default 100 / 50 ms (not 65,536), no CPU
    pinning, supervision is OneForOne/AllForOne *tracked* (no auto-restart
    monitor), added `PositionSizingActor`, `PythonStrategyActor` (OS thread,
    50 ms timeout, halt-on-timeout), backpressure policies, SLOs (2 ms / 1 ms /
    5 ms / 15 µs), metrics (`actor_*` + legacy `honba_*`), `/metrics`.
  - Rewrote `docs/architecture/runtime_jobs.md` against the jobs crate: real
    `JobSpec` fields (no `Trigger` enum), 5-field Vixie cron (letters rejected,
    OR day semantics), `JobScheduler` is trait-only (loop lives in hosts/CLI),
    `FileJobRegistry` atomic tmp-rename, 6 budget dimensions, fencing leases,
    `BoundedWorkerPool` 429/503, built-in jobs are simulated sleep stubs,
    run journals live in the API layer (not in jobs).
  - Fixed `docs/architecture/strategy_tester_and_enhancer.md`: real
    `BacktestConfig` sample (starting_cash/product_type/l2/latency/slippage/
    adverse_fill/mis_square_off/warmup_bars/iccl_margin_holdback_pct); GA
    defaults pop 50 / gen 20 / crossover 0.8 / mutation 0.2 / elitism; TPE
    10 random evals, gamma 0.25; implemented guardrails DSR ≥ 3.0 robust and
    PBO < 0.5 (replacing aspirational 0.95 / 0.10 prose); MC is
    caller-counted permutation + 1.5/2/3× shocks + gap shock; Odysseus node
    kinds (condition/action/composite); regime clustering attributed to
    `vyapaar-agent`; AI refinement loop marked scaffolding.
  - Fixed `docs/api/rest.md`: the four `/portfolio{,/accounts,/positions,
    /holdings}` routes answer **501 `not_implemented`
    (`valuation_engine_mismatch`)**; added `/trades`, `/ws/chat`, the
    streaming/SSE/chat section (`resume_from_seq`), and `vyapaar-data`
    gap citations for the `/market-data/*` sample data (0002 symbol profile,
    0001/0003 calendar); removed the non-existent generic `/reports/{kind}`.
  - Updated `docs/conformance_matrix.md`: chat runtime noted in the runtime
    row + new doc links; enhancer test coverage corrected to
    `crates/vyapaar-runtime-enhancer/tests/unit_tests.rs` only;
    `vyapaar-ledger-journal` row corrected from stale PLANNED to
    **IMPLEMENTED** (double-entry ledger + statutory tax calculator; tests
    `unit_tests.rs`, `tax_calculator_tests.rs`, `ledger_proptest_invariants.rs`,
    `golden.rs`) — summary counts now 17 IMPLEMENTED / 1 IN-PROGRESS / 0 PLANNED.
* Gate: `mkdocs build --strict` **PASS** (fresh run after all edits; only the
  known informational `docs/plans/*`-outside-nav warning).
* Earlier passes (folded): Tier 6E docs migration; Fincept/Quantcept pages
  (`docs/plans/0002`); DDD design docs (`architecture_layers.md`,
  `ddd_conflicts_resolutions.md`); the 2026-10 major re-sync that added
  `ledger_performance.md`, `ledger_allocation.md`, `runtime_jobs.md`,
  rewrites of `api/rest.md` + `api/python-sdk.md`, and updates to
  strategies/india/examples/ai/roadmap pages + conformance matrix.

## Decisions & Discoveries
* `vyapaar-docs` `.venv` holds mkdocs-material; run
  `.venv/bin/mkdocs build --strict` (system `mkdocs` lacks the material theme).
* Two pages live outside `nav` (`docs/plans/*`) — informational warning only,
  not a build failure.
* Cross-directory links use `../` prefixes; keep that style for new pages.
* REST route conflicts were resolved against `v1.rs`, not prose: `/quotes`
  (no id), `/market-data/symbol-profile/:id`, `/approval/queue|approve|reject`,
  and the 4 × 501 portfolio routes are the wire truth.
* Source of truth for capabilities: per-repo `docs/design/`, `docs/plans/`,
  `docs/gaps/`, and `HANDOFF.md` files; the runtime per-crate design docs
  (`vyapaar-runtime/docs/design/code-*.md`) now cover all 12 non-stub crates.

## Immediate Next Step
* None in this repo. All known runtime design changes are reflected. Monitor
  sub-repo HANDOFF.md files for new capabilities that warrant documentation
  additions (e.g. when `vyapaar-runtime` gap 0006 adds durable chat sessions,
  update `runtime_chat.md`).

## Operating Invariants
* `mkdocs build --strict` must stay green; docs code snippets must match tested
  code in `vyapaar-examples` / `vyapaar-strategies`.
* Keep `docs/conformance_matrix.md` in sync with sub-repo plan/gap indexes as
  new capabilities land.
