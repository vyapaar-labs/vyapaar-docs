# `vyapaar-docs` — Inter-Agent Handoff

## Phase / Slice Status
* **Adapters docs sync (COMPLETE, 2026-11-12)**: rewrote the site's adapter
  story against the actual `vyapaar-adapters` source (single package, 7 broker
  modules, REST-only, Decimal-only). The old pages described a **non-existent**
  per-broker-package architecture (`vyapaar-dhan`/`vyapaar-zerodha`
  distributions, `AdapterCapabilities`/`WireMap`/`routes.toml` contract suite,
  `from_env`/`<PREFIX>MODE`/`VYAPAAR_CACHE_DIR`, async facade, float prices) —
  none of it exists; corrected everywhere it appeared:
  - New page `docs/api/adapters-package.md` (detailed `vyapaar-adapters`
    reference: quickstart, canonical models, zero-floats parse, protocols/
    transports, per-broker wire-mapping table, paper/replay harness, registry,
    config/TOML, testing pattern, limitations table with gap links) — nav
    registered under API Reference.
  - Rewrote `docs/india/adapter_configuration.md` (constructor injection,
    `AdapterConfig`/`BrokerCredentials`, 3-of-7 TOML templates + pending loader
    gap 0007, paper harness, credentials guidance).
  - Rewrote `docs/contributing/adding_adapters.md` (module-in-one-package
    pattern, `as_decimal`/no-float rule, `MockTransport` contract tests,
    entry-point + `BROKER_ADAPTERS` sync invariant); the old version even
    taught `float` prices, violating the workspace zero-floats invariant.
  - Fixed `docs/india/brokers.md` (status table: all 7 implemented, 2
    certified; real Dhan/Zerodha module sections; auth/order-update/config/
    paper/rate-limit/adding sections re-grounded; broker API facts kept).
  - `docs/api/adapters.md` now carries a "Not yet implemented in the `vyapaar`
    SDK" banner: the documented `vyapaar.adapters` registry is planned-only
    (verified absent from `vyapaar-ffi/python/vyapaar`); shipped layer = the
    `vyapaar-adapters` package.
  - `docs/conformance_matrix.md` adapter row corrected (single-package
    description + real test paths, replacing the bogus `vyapaar-exec-*` test
    citations); `architecture/adapter_interface.md` §5 table fixed (source dir
    `vyapaar-adapters/vyapaar_adapters/*.py`, 7 brokers, sync REST);
    `architecture/index.md` + `api/python/index.md` rows updated;
    `getting-started/installation.md` + `contributing/development_setup.md`
    adapter install/testing sections rewritten.
* Gate: `mkdocs build --strict` **PASS** (0 anchor warnings; only the known
  informational `docs/plans/*`-outside-nav warning).
* Runtime docs pass (2026-11-12, earlier same day) — see the folded summary
  below.

## Decisions & Discoveries
* `vyapaar-docs` `.venv` holds mkdocs-material; run
  `.venv/bin/mkdocs build --strict` (system `mkdocs` lacks the material theme).
* Two pages live outside `nav` (`docs/plans/*`) — informational warning only,
  not a build failure.
* Cross-directory links use `../` prefixes; keep that style for new pages.
* Heading anchors strip backticks/dots (`### 3.2 Headless Authentication
  (`vyapaar-exec-auth`)` → `#32-headless-authentication-vyapaar-exec-auth`);
  deep-link to the post-slugify form or the strict build flags it.
* `vyapaar-adapters` wire truth (verified 2026-11-12): all 7 broker modules
  implemented + entry-pointed; contract tests only for dhan/zerodha;
  `zerodha.get_bars` is the only real OHLCV path; all 7 `cancel_order`
  fabricate zeroed Orders; no WebSocket/TOTP/TOML-loader; configs templates
  for dhan/zerodha/paper only. Gaps 0001–0007 still open; plan 0002 retired →
  plan 0003 (`0003-adapter-parity-streaming-realism.md`).
* Runtime REST route conflicts were resolved against `v1.rs`, not prose:
  `/quotes` (no id), `/market-data/symbol-profile/:id`,
  `/approval/queue|approve|reject`, and the 4 × 501 portfolio routes are the
  wire truth.
* Source of truth for capabilities: per-repo `docs/design/`, `docs/plans/`,
  `docs/gaps/`, and `HANDOFF.md` files; the runtime per-crate design docs
  (`vyapaar-runtime/docs/design/code-*.md`) now cover all 12 non-stub crates,
  and `vyapaar-adapters/docs/design/code-*.md` now covers every module
  (parse/config/codec/public-api added 2026-11-12).

## Immediate Next Step
* None in this repo. Monitor sub-repo HANDOFF.md files: when
  `vyapaar-adapters` plan 0003 Phase A lands (5 broker contract tests, real
  `get_bars`, parsed `cancel_order`), update the status tables in
  `api/adapters-package.md`, `india/brokers.md`,
  `india/adapter_configuration.md`, and the conformance matrix row; when the
  SDK-side `vyapaar.adapters` registry lands, re-sync `api/adapters.md`.

## Operating Invariants
* `mkdocs build --strict` must stay green; docs code snippets must match tested
  code in `vyapaar-examples` / `vyapaar-strategies` / adapter `tests/`.
* Keep `docs/conformance_matrix.md` in sync with sub-repo plan/gap indexes as
  new capabilities land.

---

## Folded: runtime docs pass (2026-11-12, earlier)

Synced the site with the actual `vyapaar-runtime` implementation (all facts
re-verified against crate source): new `architecture/runtime_overview.md`
(14-crate overview, four operational surfaces, parity pipeline, invariants) and
`architecture/runtime_chat.md` (`/ws/chat` chat agent runtime: env-gated LLM
backend — `VYAPAAR_GGUF_PATH` or `VYAPAAR_LLM_BASE_URL` — read-only tools +
`Profile.alerts` alert store, in-memory sessions). Rewrote `event_loop.md`
against `vyapaar-runtime-actors` (no `AuditPersistenceActor`; coalescing
default 100/50 ms; no CPU pinning; supervision tracked; `PositionSizingActor`,
`PythonStrategyActor` 50 ms timeout; SLOs 2 ms/1 ms/5 ms/15 µs). Rewrote
`runtime_jobs.md` (real `JobSpec` fields, 5-field Vixie cron, trait-only
`JobScheduler`, `FileJobRegistry` atomic tmp-rename, 6 budget dimensions,
fencing leases, 429/503 backpressure, journals live in the API layer). Fixed
`strategy_tester_and_enhancer.md` (real `BacktestConfig` sample; GA defaults;
TPE; DSR ≥ 3.0 robust / PBO < 0.5; caller-counted MC; Odysseus node kinds).
Fixed `api/rest.md` (four `/portfolio*` routes answer 501
`valuation_engine_mismatch`; added `/trades`, `/ws/chat`, streaming/SSE/chat
section; removed non-existent generic `/reports/{kind}`). Updated
`conformance_matrix.md` (chat runtime noted; `vyapaar-ledger-journal` row
PLANNED → IMPLEMENTED; counts 17 IMPLEMENTED / 1 IN-PROGRESS / 0 PLANNED).
Earlier passes (folded): Tier 6E docs migration; Fincept/Quantcept pages
(`docs/plans/0002`); DDD design docs; the 2026-10 major re-sync.
