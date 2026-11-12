# Actor Runtime & Event Loop Architecture

> **[STATUS: IMPLEMENTED]** — `vyapaar-runtime-actors`. See [Platform Conformance Matrix](../conformance_matrix.md).

> **Concurrency Architecture**: Inspired by the actor architectures of **NautilusTrader** and **Barter-rs**, `vyapaar-runtime-actors` organizes runtime execution as a Tokio actor pipeline. Actors own their state and communicate exclusively via strongly typed, bounded `tokio::sync::mpsc` channels — no shared mutable state, and no coupling to Python's Global Interpreter Lock (GIL).

---

## 1. Concurrency Model: Bounded Message Passing

Actors communicate solely via `tokio::sync::mpsc` channels whose mailbox size is the caller-supplied `channel_capacity`. Request/response pairs carry a `oneshot::Sender` reply channel inside the message itself, so a caller can await a specific answer (e.g. the risk verdict for one order) rather than guess by correlation.

```text
 ┌──────────────────────────────────────────────────────────────────────┐
 │                    VYAPAAR ACTOR PIPELINE RUNTIME                     │
 ├──────────────────────────────────────────────────────────────────────┤
 │                                                                      │
 │   ┌───────────────────────┐                                          │
 │   │   MarketDataActor     │◄── Canonical Tick/Bar ingestion           │
 │   └──────────┬────────────┘    (subscription-filtered, coalesced)     │
 │              │ tick per subscriber (try_send) + broadcast(Tick)      │
 │              ▼                                                       │
 │   ┌───────────────────────┐                                          │
 │   │     StrategyActor     │  compiled Rust strategies (or the        │
 │   └──────────┬────────────┘  Python-isolated actor, §3.5)             │
 │              │ candidate OrderRequest                                │
 │              ▼                                                       │
 │   ┌───────────────────────┐      optional (§3.3)                     │
 │   │ PositionSizingActor   │◄─── wired between strategy and risk      │
 │   └──────────┬────────────┘                                          │
 │              │ sized OrderRequest                                    │
 │              ▼                                                       │
 │   ┌───────────────────────┐                                          │
 │   │       RiskActor       │ synchronous vyapaar-risk gate            │
 │   └──────────┬────────────┘  + injected SqliteAuditLedger             │
 │              │ approved OrderRequest                                 │
 │              ▼                                                       │
 │   ┌───────────────────────┐                                          │
 │   │    ExecutionActor     │ broker dispatch + OMS idempotency        │
 │   └──────────┬────────────┘  + injected SqliteAuditLedger             │
 │              │ OrderAck / BrokerError                                │
 │                                                                      │
 │   ┌───────────────────────┘   standalone, not part of Supervisor      │
 │   │    PortfolioActor     │ FIFO tax-lots, statutory fees, ledger     │
 │   └───────────────────────┘                                          │
 └──────────────────────────────────────────────────────────────────────┘
```

There is **no** separate `AuditPersistenceActor`: audit persistence is a
`vyapaar_runtime_persistence::SqliteAuditLedger` injected directly into the
`RiskActor` and `ExecutionActor` (`.with_audit_ledger(...)`), which write rows
inline — risk refusals, order registrations, acks, rejections, cancels — with
risk-breach events mirrored into the persistence crate's type.

## 2. Actor Responsibilities

### 2.1 `MarketDataActor`
* **Role**: Market-data distributor. Ingests `MarketDataMsg::IngestTick`/`IngestBar`, filters by the active subscription set, forwards ticks to every strategy sender with `try_send` (a full mailbox logs a warning and drops, counted in metrics), and broadcasts canonical `Tick`s over a `tokio::sync::broadcast` channel (`resubscribe_ticks()`).
* **Coalescing**: a builder-tuned buffer — `with_max_buffer_size` (default **100**, not a 65,536-entry buffer) and `with_coalesce_interval_ms` (default **50 ms**). The event loop is `tokio::select! { biased }` between message intake and the coalesce flusher.
* **Backpressure policies**: `BackpressurePolicy::{Passthrough, CoalesceLatest, DropNewest, DropOldest}` — `CoalesceLatest` is the default; buffered modes keep only the latest tick per instrument on flush, and full-buffer behaviour is policy-driven with per-drop metrics.
* **Normalization** happens upstream in `vyapaar-market` adapters: this actor receives canonical `vyapaar_core::Tick`/`Bar` values, never raw broker frames.

### 2.2 `StrategyActor<S: Strategy>`
* **Role**: Executes strategy signal generation against ticks, bars, and fill reports.
* **Isolation**: Strategies are compiled Rust structs implementing the actor-side async `Strategy` trait. Python strategies do **not** run here — they get a dedicated OS-thread actor (§3.5).
* **Synchronous reply discipline**: the actor emits `RiskMsg::EvaluateOrder` (or `PositionSizingMsg::SizeOrder` when sizing is wired) and **awaits the oneshot reply**, so one order's fate is known before the next decision. It stamps W3C trace context (`derive_or_create_trace_context`) and emits the `OrderGenerated` span.

### 2.3 `PositionSizingActor` (optional, wired by `spawn_with_position_sizing`)
* **Role**: Order sizing between strategy and risk, behind the hexagonal `PositionSizer` port. Default sizer `SimplePositionSizer` delegates to the pure `vyapaar_core::sizing::calculate_position_size`, defaulting to `SizingMethod::FixedFractional { risk_pct: 0.01 }`.
* **Determinism**: entry price is the limit price, else a ₹100.00 placeholder; portfolio state arrives only via explicit `UpdatePortfolio` messages (no hidden fill sync); sizer errors forward the unadjusted order.

### 2.4 `RiskActor` (Subsystem: `vyapaar-risk`)
* **Role**: Pre-trade compliance gatekeeper. Calls the synchronous `risk_engine.check_order(&request, now)`:
  1. Validates the registered SEBI `algo_id`.
  2. Evaluates Order-to-Trade Ratio (OTR) windows (`RecordCancellation`/`RecordFill` feed them).
  3. Verifies limit prices against scrip-wise circuit bands.
  4. Enforces account-level maximum notional, position caps, and daily loss limits; honours `HaltAlgo`/`HaltGlobal` kill switches.
* **Refusal path**: warn + `RiskBreachEvent` (rule code carried) + metrics + an audit row via the injected ledger + `Err(BrokerError::RiskRefusal(...))` on the oneshot + an `ExecutionMsg::RiskBreachEvent` emission.
* **Approval path**: forwards `ExecuteOrder` with a fresh oneshot and relays the broker result to the strategy.

### 2.5 `ExecutionActor` (Subsystem: `vyapaar-exec`)
* **Role**: Outbound dispatcher. Optional `.with_oms(OrderStateMachine)` gives **idempotency**: a duplicate `client_order_id` is rejected as `InvalidRequest` without re-sending; the order is registered at `PendingNew` before the timed `broker.place_order` call.
* **Broker integration**: any `BrokerExecutor` port (injected, `vyapaar-exec-core`); rate limiting, slicing, and SOR live in the `vyapaar-exec` repo behind that port.
* **Audit**: acks, rejections, and cancels write rows via the injected `SqliteAuditLedger`; cancels emit the `Cancelled` span, an audit row, and `RiskMsg::RecordCancellation` (OTR credit).
* **Drain on stop**: the run loop drains remaining queued messages (`try_recv`) before returning `ActorStatus::Stopped`.

### 2.6 `PortfolioActor` (standalone)
Not wired by the `Supervisor` — a host wires it itself. It owns a single default account and processes fills with FIFO tax-lot matching (`apply_fill_to_account`), exact-`Decimal` statutory fees (STT 0.1% delivery / 0.025% intraday sell-side, NSE turnover ₹2.97/crore, SEBI ₹10/crore, stamp duty 0.015% buy-side, 18% GST, DP ₹13.50 + GST on delivery sell), and a `vyapaar-ledger-journal` double-entry posting. `GetSnapshot` computes MTM unrealized PnL from marks; `SettleTrades` is a **stub**.

## 3. Pipeline Variants & Python Isolation

### 3.1 Supervisor wiring (`Supervisor`)
| Constructor | Tree |
|---|---|
| `spawn_with_strategy(...)` | 4 actors: MarketData → Strategy → Risk → Execution |
| `spawn_with_position_sizing(...)` | 5 actors: sizing sits between strategy and risk |
| `spawn_with_observability(...)` | 4 actors + metrics on every actor + audit ledger on Risk/Execution |

All channels share the caller's `channel_capacity`. Accessors: `market_data_sender()`, `strategy_sender()`, `sizing_sender() -> Option<_>`, `risk_sender()`, `execution_sender()`, `resubscribe_ticks()`.

### 3.2 Supervision & Fault Tolerance
* **Restart strategies**: `RestartStrategy::{OneForOne, AllForOne}` is tracked and swap-able (`restart_strategy()`, `restart_count(...)`). There is **no automatic restart-on-failure monitor**: crash detection happens via `JoinHandle` errors, and recovery is caller-driven (e.g. re-spawn the tree after broker reconnects).
* **Graceful shutdown**: `shutdown_gracefully(timeout)` sends `Shutdown` down the chain (MarketData → Strategy → Sizing → Risk → Execution), flushes the market-data buffer first, lets `ExecutionActor` drain its queue, and joins all handles under the timeout.

### 3.3 `PythonStrategyActor` (ADR 0032 Phase 8.4)
* **Role**: Runs a sync `PythonStrategyEngine` for Python strategies **on a dedicated named OS thread** (`"honba-py-{algo_id}"`), bridged into the pipeline by an mpsc task; every engine call is wrapped in `std::panic::catch_unwind`.
* **Timeout & halt**: each on-tick/on-bar gets a fresh `CancellationToken` and a bounded wait (default **50 ms**). A timeout or worker crash ⇒ status `Degraded`, an error metric, and a protective `RiskMsg::HaltAlgo { algo_id, reason }` — a wedged Python callback cannot block the Rust pipeline. `PythonExecutionError::{Timeout, WorkerCrashed, GilContention, ChannelDisconnected, StrategyError}`.

## 4. Observability

### 4.1 Metrics (`MetricsCollector`, Prometheus; Axum `/metrics` router)
Phase-8 families: `actor_mailbox_queue_depth`, `actor_processing_latency_seconds`, `actor_backpressure_drops_total`, `actor_errors_total`. Legacy-prefixed families are retained alongside them: `honba_otr_ratio`, `honba_risk_evaluation_duration_seconds`, `honba_order_dispatch_duration_seconds`, `honba_notional_turnover_inr_total`, `honba_daily_pnl_inr`, `honba_risk_refusals_total`, `honba_order_operations_total`. All internal values are `Decimal`; f64 conversion happens only at the Prometheus boundary.

### 4.2 Tracing
Lifecycle spans as `tracing::info_span!` with W3C `traceparent` fields carried in `TraceContext` (no `opentelemetry` dependency): `actor.process_message`, `market_data.coalesce`, `risk.evaluate_order`, `execution.dispatch_wire`, and the order lifecycle `OrderGenerated → RiskEvaluated → BrokerSubmitted → BrokerAcked → Filled/Cancelled`. Helpers: `inject_trace_context` / `extract_trace_context` / `derive_or_create_trace_context`.

### 4.3 SLOs (formal verify functions in `slo.rs`)
| Constant | Bound |
|---|---|
| `SLO_TICK_TO_QUOTE_WS_BROADCAST_MAX` | p99 < 2 ms |
| `SLO_STRATEGY_SIGNAL_TO_ORDER_MAX` | p99 < 1 ms |
| `SLO_END_TO_END_TICK_TO_WIRE_MAX` | p99 < 5 ms |
| `SLO_PRE_TRADE_RISK_GATE_MAX` | p99 < 15 µs |

`verify_ws_broadcast_slo`, `verify_strategy_signal_slo`, `verify_wire_dispatch_slo`, `verify_risk_gate_slo` each return a `SloViolation` with actual-vs-threshold when the bound is exceeded. The stress suite (`tests/e2e_integration_stress.rs`) exercises the full pipeline at 50,000 ticks/sec with zero data loss.

---

## 5. Python Runtime Integration (`vyapaar.event_loop`)

When running inside Python or interactive Jupyter environments, `vyapaar-ffi` embeds and manages the Tokio actor runtime:

```python
from vyapaar import event_loop

# Context manager: Starts Tokio worker threads, drains on exit
with event_loop.running(worker_threads=4):
    # Strategy execution runs in multi-threaded Rust runtime
    ...

# Explicit control
info = event_loop.start(worker_threads=8)
print(f"Runtime active: {info.flavor}, Threads: {info.worker_threads}")
event_loop.stop()
```

### Concurrency guarantees:
* **Zero GIL blocking**: the Tokio runtime runs on dedicated native OS threads; Python strategy callbacks run on their own OS thread (§3.3) with a 50 ms timeout and a protective halt, never inside the hot path.
* **Latency budgets**: see the SLO table in §4.3 — the pre-trade risk gate alone is bounded at 15 µs p99, end-to-end tick-to-wire at 5 ms p99.

---

## Related documents

* [`vyapaar-runtime` per-crate design docs](https://github.com/vyapaar-labs/vyapaar-runtime/tree/main/docs/design) — `code-actors.md` is the source-level reference for this page.
* [Platform Conformance Matrix](../conformance_matrix.md) — implementation status.
* [REST API](../api/rest.md) — the HTTP surface that feeds the pipeline.
