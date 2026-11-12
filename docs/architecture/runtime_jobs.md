# Runtime Job Scheduler

> **[STATUS: IMPLEMENTED]** — `vyapaar-runtime-jobs` crate. See [Platform Conformance Matrix](../conformance_matrix.md).
>
> `vyapaar-runtime-jobs` is the **deterministic cron/background job framework** inside `vyapaar-runtime`: typed `JobSpec`s with multi-dimensional budgets, a cron grammar with `next_execution`, a persistent `FileJobRegistry`, fencing leases against split-brain execution, and a bounded worker pool with HTTP-style backpressure. The scheduler itself is a **trait** (`JobScheduler`) — hosts implement the execution loop (the `vyapaar` CLI wires one).

---

## 1. Bounded Context & Responsibility

`vyapaar-runtime-jobs` (ADR 0017 / Jesse budgeting pattern) is scheduling **infrastructure**, not a domain aggregate. It defines *what* runs, *when*, *within what budget*, and *under which approvals* — it does not itself run the trading engine.

| Invariant | How it's enforced |
|-----------|-------------------|
| **Strict cron grammar** | `CronSchedule::new` validates 5 numeric fields at registration; letter suffixes (`L`/`W`) are refused rather than silently never firing |
| **Deterministic next-fire** | `next_execution(after)` implements full Vixie semantics (incl. the both-day-fields-restricted OR rule) over UTC |
| **Durable registry** | `FileJobRegistry` writes `~/.vyapaar/jobs.json` via atomic tmp-rename (`jobs.json.tmp` → rename) |
| **Budgets before work** | `JobBudgets` caps turns, INR/USD friction, tokens, tool calls, data calls; `BudgetUsage` tracks consumption |
| **No split-brain execution** | `FencingLease` carries a monotonic token + expiry + worker id; stale workers get `TokenMismatch`/`WorkerMismatch` |
| **Bounded concurrency** | `BoundedWorkerPool` returns typed HTTP-mapped errors (429/503/500/422), never unbounded spawning |
| **Parallel test directory** | All tests live in `tests/` (`jobs_tests.rs`, `pool_tests.rs`, `registry_tests.rs`, `scheduler_cron_tests.rs`, `unit_tests.rs`); zero inline `#[cfg(test)]` in `src/` |

---

## 2. Core Types

### `JobSpec` (serde, persisted by the registry)

| Field | Type | Meaning |
|-------|------|---------|
| `id` | `Uuid` | Stable job identifier |
| `name` | `String` | Human-readable job name |
| `goal` | `String` | High-level goal or strategy intent |
| `schedule` | `CronSchedule` | 5-field cron expression |
| `description` | `Option<String>` | Optional description |
| `metadata` | `HashMap<String, String>` | Free-form key/value pairs |
| `enabled` / `status` | `bool` / `JobStatus` | Eligibility; `Active`, `Paused`, `PendingApproval`, `Completed`, `Failed` |
| `budgets` / `usage` | `JobBudgets` / `BudgetUsage` | Limits and consumption (§2.2) |
| `missed_policy` | `MissedPolicy` | `Skip` (default) or `CatchUp` for missed scheduled ticks |
| `approval_policy` | `ApprovalPolicy` | `Auto` (within budget) or `Manual` (hold in `PendingApproval` for human signoff) |
| `active_lease` / `fencing_counter` | `Option<FencingLease>` / `u64` | Current fencing lease and its monotonic counter |
| `created_at` / `last_run_at` / `next_run_at` | `DateTime<Utc>` | Timestamps (explicit, never minted mid-run) |
| `read_only` / `once` | `bool` | Strictly read-only execution; run once then `Completed` |

There is **no `Trigger` enum** — a job's schedule *is* its cron expression; manual runs are driven by the host, not by a variant.

### `CronSchedule`
* 5-field Vixie format: minute, hour, day-of-month, month, day-of-week; `*` and `?` both mean any.
* Validation is strict: exactly 5 fields, only digits and `* ? - / ,` — letters are a typed `CronScheduleError::InvalidCharacter`.
* `next_execution` delegates to the `cron` crate over an equivalent 6-field expression (seconds fixed to 0) with the day-of-week field translated from Vixie numbering (0/7 = Sunday) to the crate's 1-based scheme; when **both** day fields are restricted, the Vixie OR rule is evaluated as the earlier of the two one-field schedules.

### `JobScheduler` (trait)
`register_job(spec, Arc<dyn Job>)`, `unregister_job{,_by_name}`, `get_job`, `list_jobs`, `start`, `stop`, `state()` (`Stopped`/`Running`/`Paused`). **This repository ships the trait and cron evaluation, not a concrete loop** — process owners implement it (the `vyapaar` CLI in `vyapaar-ui` wires one; jobs can also be driven tick-by-tick from tests).

### Job traits (`job` module)
| Trait | Methods |
|---|---|
| `Job` | `name()`, `execute() -> Result<(), JobError>`, `spec()` |
| `CancellableJob` | adds `cancel()` |
| `ProgressJob` | adds `progress() -> u8` (0–100) |

### `JobBudgets` — six dimensions
`max_turns`, `max_inr` (statutory + commission friction, exact `Decimal`), `max_usd`, `max_tokens`, `max_tool_calls`, `max_data_calls`. Every dimension is optional; `BudgetExceededError` carries the exceeded dimension.

### `FencingLease`
Monotonic fencing token + lease expiration + worker identity. Validation errors: `LeaseExpired`, `TokenMismatch` (older token), `WorkerMismatch` — the split-brain defence.

### `BoundedWorkerPool`
`WorkerPoolConfig` with core budgeting + `max_queued`; submission beyond the queue is `PoolError::QueueSaturated` with **HTTP 429** semantics, `ShuttingDown` → 503, `ExecutionPanic` → 500, `CoreAllocation` → 422. Panics are isolated per job. Sweep execution passes a `SweepContext`.

---

## 3. Persistence & Recovery

* `FileJobRegistry` holds every `JobSpec` in one JSON document at `~/.vyapaar/jobs.json` (custom path supported). Updates serialize to a temporary sibling file and rename over the destination, so a crash mid-write cannot corrupt the registry.
* Run-level crash recovery (normalizing `Pending`/`Running` runs to `Failed` at boot) lives in the **API run store** (`vyapaar-runtime-api-rest`), not in this crate.

## 4. Run Journals vs Job Registry

The jobs crate does **not** write per-run journals. Backtest/sweep run journals (`manifest.json`, `events.ndjson`, run events) are owned by the REST layer:

| Artifact | Owner |
|----------|-------|
| `~/.vyapaar/jobs.json` (job specs, statuses, budgets, leases) | `FileJobRegistry` (this crate) |
| `<journals-dir>/<run_id>/manifest.json`, `events.ndjson` | `vyapaar-runtime-api` contract + `vyapaar-runtime-api-rest` `run_store.rs` |

See [REST API](../api/rest.md) for `/backtests/*`, `/sweeps/*` and `/journals/{id}`.

## 5. Built-in Jobs

Shipped jobs are **simulated stubs** (they sleep and log; the real broker work is pluggable via the `Job` trait):

| Job | Cron | Purpose | Current body |
|-----|------|---------|--------------|
| `PreMarketTOTPJob` | `0 8 * * 1-5` (08:00 weekdays) | Refresh TOTP secrets before market open | simulated 100 ms sleep |
| `EODSettlementJob` | `0 15 * * 1-5` (15:00 weekdays) | EOD settlement / position reconciliation | simulated 200 ms sleep |
| `SnapshotJob` | `0 0 * * *` (midnight daily) | `positions` / `portfolio` / `system` snapshot | simulated 150 ms sleep |

Each accepts `with_schedule(...)` for custom crons.

## 6. CLI (`vyapaar jobs`, in `vyapaar-ui`)

`jobs add` / `list` / `run <id>` / `tick [--all]` / `install` / `logs <id>` / `pause <id>` / `resume <id>` / `approvals` / `approve <id>` / `deny <id>` — the CLI is a scheduler host; pause/resume and approve/deny flip `JobStatus` / honor `ApprovalPolicy::Manual` through the registry.

## 7. Related Documents

* **[REST API](../api/rest.md)** — run journals and the `/backtests`/`/sweeps` endpoints the scheduler can drive.
* **[Actor Runtime & Event Loop](event_loop.md)** — the pipeline jobs may target.
* **[Strategy Tester & Enhancer](strategy_tester_and_enhancer.md)** — sweeps the worker pool can execute.
* **[Platform Conformance Matrix](../conformance_matrix.md)** — implementation status badge.
