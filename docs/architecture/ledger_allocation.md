# Ledger Allocation — Portfolio Allocation & Optimization Engine

> **[STATUS: IMPLEMENTED]** — `vyapaar-ledger-allocation` crate, Phase 1. See [Platform Conformance Matrix](../conformance_matrix.md).
>
> `vyapaar-ledger-allocation` (Tier 1B, plan 0001) is the **pure portfolio allocation & optimization engine**. It computes target weights (`TargetWeights`) for a universe of instruments given expected returns, a covariance matrix, and weight bounds — all in exact `rust_decimal::Decimal`, zero-I/O, zero-floats.

---

## 1. Bounded Context & Responsibility

`vyapaar-ledger-allocation` sits inside `vyapaar-ledger` (Tier 1B, Zero-I/O domain heart). It is a **read-only optimizer**: it never mutates portfolio state, never touches the network or disk, and never calls the wall clock. It takes a `PortfolioState` (built from `vyapaar-ledger-portfolio`) plus return/covariance estimates and produces `TargetWeights`.

**Key design invariants:**

| Invariant | How it's enforced |
|-----------|-------------------|
| **Zero-I/O** | Pure synchronous functions; no `tokio`, no filesystem, no database, no wall-clock. |
| **Zero floats** | All weights, bounds, returns, covariance entries are `rust_decimal::Decimal`. Matrix inversion via Gaussian elimination in `Decimal`. |
| **Read-adapter over portfolio** | `PortfolioState::from_portfolio(portfolio, marks, as_of)` builds optimization input from the portfolio aggregate. |
| **Deterministic `optimize()`** | Same inputs → same outputs, byte-for-byte. |

---

## 2. Phase 1 — Implemented Capabilities

### `AllocationObjective`

| Variant | Formula | Meaning |
|---------|---------|---------|
| `MeanVariance` | Maximize `(μᵀw − r_f) / √(wᵀΣw)` | Markowitz tangency portfolio (max Sharpe) |
| `RiskParity` | Equal risk contribution `wᵢ(Σw)ᵢ` across instruments | Hierarchical-style equal risk (Phase 1) |

Phases 2–4 (Hrp/Herc, Black-Litterman, discrete/whole-share allocation) are declared but return `NotImplemented` — explicitly out of scope for Phase 1.

### `Constraint::WeightBounds`

| Field | Type | Meaning |
|-------|------|---------|
| `instrument` | `InstrumentId` | Target instrument |
| `min` | `Decimal` | Minimum weight |
| `max` | `Decimal` | Maximum weight |

Box bounds per instrument, projected via iterative clip-and-renormalize: clip each weight to `[min, max]`, re-scale remaining in-bound weights to re-sum to 1, repeat until stable.

### `PortfolioState` — read-adapter

| Method | Meaning |
|--------|---------|
| `from_portfolio(portfolio, marks, as_of)` | Build from `vyapaar-ledger-portfolio` aggregate; universe = union of held positions |
| `current_weights` | Market-value-normalized weights |
| `with_expected_returns(returns)` | Attach expected return estimates |
| `with_covariance(cov)` | Attach covariance matrix |

### `TargetWeights` — auditable output

| Field | Type | Meaning |
|-------|------|---------|
| `as_of` | `DateTime<Utc>` | Optimization timestamp |
| `objective` | `AllocationObjective` | Objective used |
| `weights` | `BTreeMap<InstrumentId, Decimal>` | Target weights (sum to 1 within decimal tolerance) |

Custom JSON serialization as a sorted array of `{instrument, weight}` entries.

### `optimize()` — pure entry point

Solves the unconstrained optimum exactly, then projects into the box via iterative clip-and-renormalize. Exact bound-constrained QP is a follow-on (Phase 4+).

---

## 3. Usage Example

```rust
use vyapaar_ledger_allocation::{optimize, AllocationObjective, Constraint, PortfolioState};
use vyapaar_ledger_portfolio::{AccountState, Price, Money};
use rust_decimal::Decimal;

// 1. Build PortfolioState from the portfolio aggregate
let state = PortfolioState::from_portfolio(&account_state, &marks, as_of)
    .with_expected_returns(&expected_returns)
    .with_covariance(&covariance);

// 2. Define constraints
let constraints = vec![
    Constraint::WeightBounds { instrument: "RELIANCE.NSE".parse().unwrap(), min: dec!("0.05"), max: dec!("0.25") },
    Constraint::WeightBounds { instrument: "TCS.NSE".parse().unwrap(), min: dec!("0.10"), max: dec!("0.30") },
];

// 3. Optimize
let target_weights = optimize(&state, AllocationObjective::MeanVariance, &constraints)?;

for (instrument, weight) in &target_weights.weights {
    println!("{}: {}", instrument, weight);
}
```

---

## 4. Tests

17 tests in `tests/optimize_tests.rs` cover: mean-variance tangency, risk-parity equal risk contribution, weight-bounds projection, clip-and-renormalize convergence, `PortfolioState` construction, `TargetWeights` JSON serialization, and decimal-exactness invariants.

`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, and `cargo test --workspace` all pass.

---

## 5. Future Phases (Open)

| Phase | Objective | Status |
|-------|-----------|--------|
| Phase 1 | MeanVariance, RiskParity | ✅ Implemented |
| Phase 2 | Hrp (Hierarchical Risk Parity) / Herc | ⬜ Declared, returns `NotImplemented` |
| Phase 3 | BlackLitterman | ⬜ Declared, returns `NotImplemented` |
| Phase 4 | Discrete / whole-share allocation | ⬜ Declared, returns `NotImplemented` |

---

## 6. Related Documents

* **[Architecture Layers & Topology](architecture_layers.md)** — Full `vyapaar-ledger` bounded context: OMS, portfolio, journal, allocation, performance.
* **[Ledger Performance](ledger_performance.md)** — Trade-performance journal (sibling crate).
* **[Platform Conformance Matrix](../conformance_matrix.md)** — Implementation status badge.
