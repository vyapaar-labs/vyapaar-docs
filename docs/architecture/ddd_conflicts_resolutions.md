# DDD Bounded Context Conflicts & Resolutions

> **[STATUS: IMPLEMENTED]** — Migration from `honba-workspace` to `vyapaar-labs`; 7 of 9 conflicts resolved, 1 HIGH priority (Zero-I/O) in progress, 1 partial (Screener/Signal ownership).
>
> This document identifies **DDD bounded context conflicts** that arose during the migration from `honba-workspace` to `vyapaar-labs` (the **new design**). Each conflict is mapped to its resolution, with responsibility assignments and dependency ordering.

---

## Executive Summary

The migration from `honba-workspace` to `vyapaar-labs` introduced a strict DDD architecture with 15 bounded repositories across 6 layers. During this migration, 9 bounded-context conflicts were identified. This document records each conflict, the old (`honba-*`) state, the new (`vyapaar-*`) state, and the resolution.

Overall status: **7/9 Resolved** | **1 HIGH Priority** (Zero-I/O domain purity) | **1 Partial** (Screener/Signal ownership).

---

## 1. Architecture Layer Dependencies

```mermaid
flowchart BT
    T0[Tier 0: vyapaar-core\n(Shared Kernel - No deps)]
    T1A[Tier 1A: vyapaar-risk\n(Zero-I/O)]
    T1B[Tier 1B: vyapaar-ledger\n(Zero-I/O)]
    T2A[Tier 2A: vyapaar-market\n(Compute: Zero-I/O,\nAdapters: Async)]
    T2B[Tier 2B: vyapaar-exec\n(Async I/O)]
    T3A[Tier 3A: vyapaar-agent\n(Async + Pure Core)]
    T3B[Tier 3B: vyapaar-runtime\n(Async)]
    T4[Tier 4: vyapaar-ffi\n(FI Bridge)]
    T5[Tier 5: vyapaar-ui\n(WASM / TUI)]
    T6[Tier 6: Python Ecosystem]

    T0 --> T1A
    T0 --> T1B
    T0 --> T2A
    T0 --> T2B
    T1A --> T2B
    T1A --> T3A
    T1A --> T3B
    T1B --> T3B
    T2A --> T3B
    T2B --> T3B
    T3A --> T3B
    T3B --> T4
    T0 --> T5
    T3B --> T5
    T4 --> T6
```

See [Architectural Layers & Bounded Context Topology](architecture_layers.md) for the full layer table.

---

## 2. Conflicting Responsibilities & Resolutions

### Conflict #1: Shared Kernel Bloat

| Aspect | `honba-core` (Old) | `vyapaar-core` (New) | Resolution |
|:-------|:-------------------|:---------------------|:-----------|
| **Portfolio Snapshot Types** | Included `PortfolioSnapshot`, `PortfolioPosition`, `PortfolioHolding`, `PortfolioTotals`, `PortfolioGreeks` | **MOVED** to `vyapaar-runtime-persistence` as repository read models | ✅ Clean separation: Domain events in core, read models in application layer |
| **Screener Types** | Included `ScreenerScanRequest`, `ScreenerRow`, `FilterOp`, `MetricDefinition`, `Signal` | **MOVED** to `vyapaar-runtime` (strategy) or `vyapaar-ui` (UI) | ⚠️ **PENDING**: Decide between runtime (domain signals) vs UI (screening UI) |
| **Signal Types** | Included `Signal`, `SignalDirection`, `SignalHorizon`, `Timeframe` | **REMAINS** in `vyapaar-core` (unclear boundary) | ⚠️ **CONSIDER**: Move to `vyapaar-runtime-strategy` or `vyapaar-agent` |
| **Size Method Types** | Included `SizingMethod`, `RiskBudget`, `PositionSizingRequest`, `PositionSizingResult` | **REMAINS** in `vyapaar-core` (shared across agent/runtime) | ✅ Correct - these are domain abstractions |

**Status**: ✅ Partially resolved | ⚠️ Pending (Screener/Signal)

**Action Required**: Decide on Screener/Signal ownership:
- **Option A**: `vyapaar-runtime-strategy` owns all strategy-related types (`Signal`, `Screener`)
- **Option B**: `vyapaar-ui` owns `Screener` (UI concern), `vyapaar-runtime-strategy` owns `Signal` (domain concept)

---

### Conflict #2: Portfolio Aggregate Responsibility

| Aspect | `honba-kokoro-portfolio` (Old) | `vyapaar-ledger-portfolio` (New) | Resolution |
|:-------|:-------------------------------|:---------------------------------|:-----------|
| **FIFO Tax-Lot Matching** | Embedded in `InMemoryPositionRepository` (application layer) | **ENCAPSULATED** in `Portfolio` aggregate root | ✅ Correct - business logic in domain |
| **Tax Lot Types** | Separate `Fitolot`, `CostBasisLot`, `TaxLot` | **UNIFIED** into single `TaxLot` entity | ✅ Correct - single canonical model |
| **Account State** | `AccountState` with `positions`, `holdings`, `totals` | **REMAINS** as domain state (not read model) | ✅ Correct - this is domain state machine |

**Key Difference**:
- **Old**: `InMemoryPositionRepository.match_lots()` was an application service method
- **New**: `PortfolioAggregate.apply_fill(fill)` encapsulates FIFO matching

**Status**: ✅ RESOLVED

---

### Conflict #3: Exchange / ProductType Unification

| Aspect | `honba-*` (Old) | `vyapaar-*` (New) | Resolution |
|:-------|:----------------|:------------------|:-----------|
| **Exchange Enum** | Redefined in `honba-kokoro-oms`, `honba-kokoro-ledger`, `honba-me-market`, `honba-te-*` | **CANONICAL** in `vyapaar-core::Exchange` with variants `{Nse, Bse, Nfo, Mcx, Cds}` | ✅ Correct - single source of truth |
| **Product Type** | Redefined in multiple places | **ADDED** to `vyapaar-core::ProductType` (if missing) | ⚠️ **VERIFY**: Ensure `ProductType::{Delivery, Intraday, Margin, CoverOrder, BracketOrder}` exists |

**Status**: ✅ RESOLVED (Exchange) | ⚠️ VERIFY (ProductType)

**Action Required**: Verify `ProductType` enum exists in `vyapaar-core/src/types.rs`:
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum ProductType {
    Delivery,
    Intraday,
    Margin,
    CoverOrder,
    BracketOrder,
}
```

---

### Conflict #4: Market Finance Ownership

| Aspect | `honba-me-market` (Old) | `vyapaar-market-*` (New) | Resolution |
|:-------|:------------------------|:-----------------------|:-----------|
| **Indian Market Profile** | `honba-me-market` owned `NseCalendar`, `CostModel`, `FnoContract`, statutory costs | **ASSIGNED** to `vyapaar-market-finance` | ✅ Correct |
| **Cost Schedule** | STT, Stamp Duty, GST, SEBI, Exchange turnover | **ASSIGNED** to `vyapaar-market-finance::india::costs` | ✅ Correct |
| **Macro/Financial Series** | Mixed with market data | **ASSIGNED** to `vyapaar-market-finance` | ✅ Correct |

**Sub-Crate Split**: `honba-me-feed` → `vyapaar-market-feed` + `vyapaar-market-store`

| `vyapaar-market-feed` | `vyapaar-market-store` |
|:---------------------:|:---------------------:|
| Live stream traits | Parquet lakehouse storage |
| `CoalescingTickBuffer` | `lakehouse.rs`, `parquet_ticks.rs` |
| `ReconnectionManager` | `recorder.rs`, `export/`, `import/` |
| `live_quotes.rs` | `dataset.rs` |

**Status**: ✅ RESOLVED

---

### Conflict #5: Zero-I/O Domain Purity

| Crate | `honba-*` Dependencies | `vyapaar-*` Dependencies | Status |
|:------|:----------------------|:------------------------|:-------|
| `vyapaar-core` | No async | No `tokio` | ✅ Compliant |
| `vyapaar-risk` | No async | No `tokio` | ✅ Compliant |
| `vyapaar-ledger` | `honba-kokoro` had `tokio` | **MUST REMOVE** `tokio` from workspace | 🚧 **IN PROGRESS** |
| `vyapaar-ledger-oms` | Had `#[tokio::test]` | Synchronous only with `now: DateTime<Utc>` | 🚧 **IN PROGRESS** |
| `vyapaar-ledger-portfolio` | Had `async fn` methods | Synchronous with explicit timestamp | 🚧 **IN PROGRESS** |
| `vyapaar-market-*` | Mixed `tokio`/`async-trait` | Compute crates: no `tokio`; Adapters: `tokio` OK | 🚧 **IN PROGRESS** |
| `vyapaar-exec-*` | Had `tokio` | Async OK (application layer) | 🚧 **IN PROGRESS** |
| `vyapaar-runtime-*` | Had `tokio` | Async OK (application layer) | 🚧 **IN PROGRESS` |

**Key Rule**:
- **Domain Layer (L0-L1)**: `vyapaar-core`, `vyapaar-risk`, `vyapaar-ledger-*` — **ZERO `tokio`**
- **Application Layer (L2+)**: `vyapaar-market-*`, `vyapaar-exec-*`, `vyapaar-runtime-*` — `tokio` OK

**Status**: 🚧 **IN PROGRESS** (vyapaar-ledger workspace tokio removal) — **HIGH Priority**

---

### Conflict #6: Anti-Corruption Layer (ACL) Boundaries

| Component | Old Location | New Location | Responsibility |
|:----------|:-------------|:-------------|:---------------|
| **Zerodha Kite Wire Format** | `honba-te-zerodha` (mixed) | `vyapaar-exec-zerodha` | ✅ ACL: `zerodha_proto` ↔ `vyapaar_core` |
| **Upstox Protobuf Wire** | `honba-te-upstox` (mixed) | `vyapaar-exec-upstox` | ✅ ACL: `upstox_proto` ↔ `vyapaar_core` |
| **Angel One SmartAPI** | `honba-te-angelone` (mixed) | `vyapaar-exec-angelone` | ✅ ACL: `angelone_json` ↔ `vyapaar_core` |
| **Risk Engine Integration** | `honba-te-router` called `honba_mon::RiskEngine` | `vyapaar-exec-router` calls `vyapaar_risk::RiskEngine` | ✅ ACL: Risk check before broker order |

**Status**: ✅ RESOLVED (ACL pattern established, implementation pending)

---

### Conflict #7: Read Model vs Domain State

| Type | Old Location | New Location | Reason |
|:-----|:-------------|:-------------|:-------|
| `PortfolioSnapshot` | `honba-core` (shared kernel) | `vyapaar-runtime-persistence` | Repository read model (application layer) |
| `PortfolioPosition` | `honba-core` (shared kernel) | `vyapaar-runtime-persistence` | Repository read model |
| `PortfolioHolding` | `honba-core` (shared kernel) | `vyapaar-runtime-persistence` | Repository read model |
| `PortfolioTotals` | `honba-core` (shared kernel) | `vyapaar-runtime-persistence` | Repository read model |
| `PortfolioGreeks` | `honba-core` (shared kernel) | `vyapaar-runtime-persistence` | Repository read model |
| `PortfolioSnapshot` (domain) | N/A | `vyapaar-ledger-portfolio::AccountState` | Domain state machine |

**Status**: ✅ RESOLVED (read models moved to application layer)

**Key Insight**:
- **Domain**: `PortfolioAggregate` has internal state `positions: Vec<PortfolioPosition>` (domain entity)
- **Application**: `PortfolioSnapshot` is serialized read model for UI/API (DTO)

---

### Conflict #8: Test Organization

| Crate | Old Pattern | New Pattern | Status |
|:------|:-----------|:-----------|:-------|
| `honba-*` | `#[cfg(test)] mod tests` inline in `src/` | `crates/*/tests/*.rs` | 🚧 Migration in progress |
| `vyapaar-*` | — | Strict parallel `tests/` directory | ✅ Policy established |

**Rule**:
```
❌ VIOLATION: src/lib.rs contains #[cfg(test)] mod tests
✅ CORRECT: tests/unit_tests.rs parallel to src/lib.rs
```

**Status**: 🚧 **IN PROGRESS**

---

### Conflict #9: Agent-Runtime Dependency Cycle

| Aspect | `honba-nou` (Old) | `vyapaar-agent` (New) | Resolution |
|:-------|:-----------------|:---------------------|:-----------|
| **Dependency on Strategy** | `honba-nou-trader-loop` → `honba-ya-strategy` | `vyapaar-agent-trader-loop` → `vyapaar_risk` only | ✅ Cycle broken |
| **Strategy as Dependency** | `vyapaar-runtime` could depend on `vyapaar-agent` | **FORBIDDEN**: `vyapaar-runtime` depends on `vyapaar-core` only | ✅ Dependency graph simplified |

**New Dependency Graph**:
```
vyapaar-core ← vyapaar-risk ← vyapaar-agent
                              ↑
              vyapaar-runtime depends only on vyapaar-core
```

**Status**: ✅ RESOLVED (dependency cycle removed)

---

## 3. Remaining Ambiguities

### Ambiguity #1: Signal vs Strategy Ownership

| Question | Options |
|:---------|:-------|
| **Where does `Signal` belong?** | A) `vyapaar-core` (domain primitive)<br>B) `vyapaar-runtime-strategy` (application signal)<br>C) `vyapaar-agent` (AI signal generation) |
| **Current State** | `vyapaar-core/src/signal.rs` contains `Signal`, `SignalDirection`, `SignalHorizon`, `Timeframe` |
| **Recommendation** | **A) `vyapaar-core`** - Signals are domain primitives (like `Order`, `Fill`) |

**Decision Required**: None - current placement is correct.

---

### Ambiguity #2: Screener Ownership

| Question | Options |
|:---------|:-------|
| **Where does `Screener` belong?** | A) `vyapaar-runtime` (strategy scanning)<br>B) `vyapaar-ui` (UI screening UI) |
| **Current State** | `vyapaar-core/src/screener.rs` contains `ScreenerScanRequest`, `ScreenerRow`, `FilterOp`, `MetricDefinition` |
| **Recommendation** | **A) `vyapaar-runtime`** - Screener is strategy tool, UI consumes runtime API |

**Decision Required**: Move `screener.rs` to `vyapaar-runtime-strategy` or keep in `vyapaar-core`?

---

### Ambiguity #3: Market Finance Scope

| Question | Options |
|:---------|:-------|
| **What goes in `vyapaar-market-finance`?** | A) Only `NseCalendar`, `CostModel`<br>B) Also `indicators`, `signals`, `normalizer` |
| **Current State** | Crate exists but empty stub |
| **Recommendation** | **A) Only canonical market data** - indicators/signals belong in separate crates |

**Decision Required**: Keep `vyapaar-market-finance` narrow (calendar/costs) or expand?

---

## 4. Migration Checklist

### Tier 0: Shared Kernel (`vyapaar-core`) ✅
- [x] `Exchange::{Nse, Bse, Nfo, Mcx, Cds}` unified
- [x] Strongly-typed IDs: `ClientOrderId`, `AlgoId`, `AccountId`, `TransactionId`
- [x] Self-validating `Price::try_new`, `Quantity::try_new`
- [ ] ⚠️ Decision on `screener.rs` location
- [ ] ⚠️ Decision on `signal.rs` location
- [ ] Verify `ProductType` enum exists

### Tier 1A: Risk Gate (`vyapaar-risk`) ✅
- [x] Zero `tokio` dependency
- [x] Explicit `now: DateTime<Utc>` on all checks
- [ ] Extract inline tests to `tests/`

### Tier 1B: Ledger Domain (`vyapaar-ledger`) 🚧
- [ ] **REMOVE `tokio`** from workspace dependencies
- [ ] Implement `vyapaar-ledger-oms` with `OrderStateMachine`
- [ ] Implement `vyapaar-ledger-portfolio` with `PortfolioAggregate`
- [ ] Implement `vyapaar-ledger-journal` with double-entry ledger
- [ ] Extract all tests to `tests/`

### Tier 2A: Market (`vyapaar-market`) 🚧
- [ ] Implement `vyapaar-market-finance` (NseCalendar, CostModel, India pack)
- [ ] Split `vyapaar-market-feed` / `vyapaar-market-store`
- [ ] Implement `vyapaar-market-indicators`, `vyapaar-market-signals`
- [ ] Extract all tests to `tests/`

### Tier 2B: Execution (`vyapaar-exec`) 🚧
- [ ] Implement `vyapaar-exec-router` with `vyapaar_risk::RiskEngine` gate
- [ ] Implement ACL modules for all brokers (Zerodha, Upstox, Angel One, Fyers)
- [ ] Extract all tests to `tests/`

### Tier 3A: Agent (`vyapaar-agent`) 🚧
- [ ] Implement `vyapaar-agent-tools`, `vyapaar-agent-plugins`, `vyapaar-agent-report`
- [ ] Ensure no upward dependency on `vyapaar-runtime`
- [ ] Extract all tests to `tests/`

### Tier 3B: Runtime (`vyapaar-runtime`) 🚧
- [ ] Implement `vyapaar-runtime-jobs`
- [ ] Move `src/tests/*` to `tests/` (both `vyapaar-runtime-strategy` and `vyapaar-runtime-testing`)
- [ ] Extract all tests to `tests/`

### Tier 4: FFI Bridge (`vyapaar-ffi`) ✅
- [x] Port PyO3 FFI bridge (`vyapaar-ffi-py`)
- [x] Implement schema codegen (`vyapaar-ffi-codegen`)
- [x] Build `vyapaar` wheel with `vyapaar._vyapaar` native module
- [x] Ensure GIL release (`py.detach` / `py.allow_threads`) during Rust actor execution

### Tier 5: Workstations (`vyapaar-ui`) 🚧
- [ ] Implement `vyapaar-cli`, `vyapaar-tui`, `vyapaar-highlight`
- [ ] Extract all tests to `tests/`
- [ ] Verify WASM portability (`cargo check -p vyapaar-ui-app --target wasm32-unknown-unknown`)

### Tier 6: Python Ecosystem 🚧
- [ ] **vyapaar-data**: Fix `pyproject.toml` (remove `rust_decimal`, add `duckdb`, `polars`, `pyarrow`, `pydantic`)
- [ ] **vyapaar-adapters**: Fix `pyproject.toml`, configure entry points for broker adapters
- [ ] **vyapaar-strategies**: Fix `pyproject.toml`, port strategy catalog
- [ ] **vyapaar-examples**: Port runnable reference scripts
- [ ] All: Extract tests to `tests/` directories

---

## 5. Conflict Summary Table

| Conflict # | Description | Status | Impact |
|:-----------|:-----------|:-------|:-------|
| #1 | Shared kernel bloat (PortfolioSnapshot, Screener, Signal) | ✅ Partially resolved | Low - pending screener/signal decision |
| #2 | Portfolio FIFO matching in repository | ✅ Resolved | None - fixed |
| #3 | Exchange/ProductType redefinition | ✅ Resolved | None - canonical types in core |
| #4 | Market finance ownership | ✅ Resolved | None - clear split |
| #5 | Zero-I/O domain purity | 🚧 In progress | **HIGH** - vyapaar-ledger tokio must be removed |
| #6 | ACL boundaries | ✅ Pattern established | None - implementation pending |
| #7 | Read model vs domain state | ✅ Resolved | None - clear separation |
| #8 | Test organization | 🚧 In progress | Medium - parallel tests required |
| #9 | Agent-runtime dependency cycle | ✅ Resolved | None - cycle broken |

**Overall Status**: **7/9 Resolved** | **1 HIGH Priority** (Zero-I/O) | **1 Partial** (Screener/Signal)

---

## 6. Next Steps

### Immediate (This Week)
1. **Remove `tokio` from `vyapaar-ledger/Cargo.toml` workspace dependencies**
2. **Verify `ProductType` enum exists** in `vyapaar-core/src/types.rs`
3. **Document screener/signal ownership** decision in this file
4. **Fix Python `pyproject.toml` files** (remove invalid `rust_decimal`, add proper dependencies)

### Short Term (This Month)
1. Implement `vyapaar-ledger-oms`, `vyapaar-ledger-portfolio`, `vyapaar-ledger-journal`
2. Implement `vyapaar-market-finance` with India market profile
3. Extract all tests to `tests/` directories (Rust + Python)
4. Implement schema codegen and FFI bridge (`vyapaar-ffi`)

### Medium Term (Next Quarter)
1. Implement ACL modules in all broker executors
2. Complete Tier 3 (Agent + Runtime) migration
3. Complete Tier 5 (UI) migration
4. Validate zero-I/O compliance across all domain crates (L0-L1)

---

## Related Documents

* **[Architectural Layers & Bounded Context Topology](architecture_layers.md)** — The 6-layer, 15-repo DDD topology.
* **[Domain-Driven Design (DDD)](domain_driven_design.md)** — Deep dive into bounded contexts, aggregates, value objects.
* **[Platform Conformance Matrix](../conformance_matrix.md)** — Per-repo implementation status.
