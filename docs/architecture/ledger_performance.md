# Ledger Performance — Trade-Performance Journal

> **[STATUS: IMPLEMENTED]** — `vyapaar-ledger-performance` crate. See [Platform Conformance Matrix](../conformance_matrix.md).
>
> `vyapaar-ledger-performance` is a pure, zero-I/O, zero-float trade-performance journal — a distinct sibling of `vyapaar-ledger-journal` (double-entry accounting) and `vyapaar-ledger-portfolio` (FIFO tax-lot valuation). It owns the trader's *performance/annotations journal*, not the accounting ledger.

---

## 1. Bounded Context & Responsibility

`vyapaar-ledger-performance` (Tier 1B, plan 0002) is the **pure trade-performance journal**. It consumes the derived closed-trade stream (`MatchedLot`) from `vyapaar-ledger-portfolio::AccountState::match_lots()` and layers on trader annotations, performance metrics, an edge-score composite, a daily P&L calendar, and account-vs-account comparison.

**Key design invariants:**

| Invariant | How it's enforced |
|-----------|-------------------|
| **Zero-I/O** | No `tokio`, no `std::fs`, no database, no wall-clock calls. All timestamps arrive as explicit `DateTime<Utc>`. |
| **Zero floats** | All PnL, R-multiple, ratio, and edge-score figures use `rust_decimal::Decimal`. No `f32`/`f64` crosses money/price/quantity paths. |
| **Thin on portfolio** | Does not re-implement FIFO lot matching; consumes `MatchedLot` from `AccountState`. |
| **Parallel test directory** | All tests live in `tests/`; zero inline `#[cfg(test)]` in `src/`. |

---

## 2. Module Topology

| Phase | Module | File | Purpose |
|-------|--------|------|---------|
| Phase 1 | `model` | `src/model.rs` | Data model: `JournalAccount`, `TradeAnnotation`, `JournalNote`, `ClosedTrade` |
| Phase 1 | `store` | `src/store.rs` | In-memory journal store: `JournalStore` |
| Phase 2 | `metrics` | `src/metrics.rs` | Performance metrics engine + edge-score composite |
| Phase 3 | `calendar` | `src/calendar.rs` | Daily P&L calendar aggregation |
| Phase 3 | `compare` | `src/compare.rs` | Account-vs-account comparison |

---

## 3. Data Model (`model.rs`)

### `JournalAccount`
An analytics grouping wrapper, distinct from the live execution `AccountState`. Fields:

| Field | Type | Meaning |
|-------|------|---------|
| `id` | `JournalAccountId` | Strongly-typed string ID |
| `name` | `String` | Human-readable account name |
| `provenance` | `AccountProvenance` | `Manual`, `Imported`, or `BrokerSynced` |
| `pnl_convention` | `PnlConvention` | `Fifo` (v1 only) |

`AccountProvenance` distinguishes how the account was created; `PnlConvention` fixes the PnL calculation convention (`Fifo` in v1).

### `TradeAnnotation`
Per-closed-trade annotation written by the trader:

| Field | Type | Meaning |
|-------|------|---------|
| `tags` | `Vec<String>` | Free-form tags |
| `rating` | `u8` | 1–5 stars |
| `review` | `Option<String>` | Free-text review |
| `planned_risk` | `Option<Money>` | Planned risk for the trade |

### `JournalNote`
Day-keyed note independent of any specific trade:

| Field | Type | Meaning |
|-------|------|---------|
| `account_id` | `JournalAccountId` | Owning account |
| `day` | `NaiveDate` | Calendar day (local) |
| `note` | `String` | Free-text note |

### `ClosedTrade`
Derived closed trade from a portfolio `MatchedLot` plus user annotation — one row per closed trade:

| Field | Type | Meaning |
|-------|------|---------|
| `lot_id` | `TradeId` | Source matched-lot id |
| `symbol` | `String` | Instrument symbol |
| `entry_ts` / `exit_ts` | `DateTime<Utc>` | Entry / exit timestamps |
| `side` | `PositionSide` | Long / Short |
| `entry_price` / `exit_price` | `Price` | Fill prices |
| `quantity` | `Quantity` | Fill quantity |
| `fees` | `Money` | Total transaction costs |
| `realized_pnl` | `Money` | Realized PnL (after fees) |
| `duration` | `Duration` | Hold duration |

**Methods:** `net_pnl()`, `is_win()`, `is_loss()`, `is_breakeven()`, `gross_profit()`, `gross_loss()`, `realized_r()`.

---

## 4. Store (`store.rs`)

`JournalStore` — in-memory collection backed by `HashMap`s:

| Collection | Key | Value |
|------------|-----|-------|
| accounts | `JournalAccountId` | `JournalAccount` |
| annotations | `(account_id, lot_id)` | `TradeAnnotation` |
| notes | `(account_id, day)` | `JournalNote` |
| closed trades | `TradeId` | `ClosedTrade` |

**Errors:** `StoreError::{AccountNotFound, AccountAlreadyExists, AnnotationNotFound}`.

**Key methods:**
- `create_account(account)` / `account(id)`
- `set_annotation(account_id, lot_id, annotation)` / `annotation(account_id, lot_id)`
- `set_closed_trade(trade)` / `closed_trades_in_range(from, to)`
- `set_note(account_id, day, note)` / `notes_for_account(account_id, from, to)`
- `set_pnl_convention(account_id, convention)`

---

## 5. Metrics Engine (`metrics.rs`)

`PerformanceMetrics` — mirrors the Vela `/api/journal/overview` response shape:

| Field | Type | Meaning |
|-------|------|---------|
| `total_trades` | `u64` | Total closed trades |
| `wins` / `losses` / `breakevens` | `u64` | Outcome counts |
| `net_pnl` / `gross_pnl` / `fees` | `Money` | PnL aggregates |
| `win_rate` | `Decimal` | Wins / total |
| `profit_factor` | `Decimal` | Gross profit / gross loss |
| `expectancy` | `Money` | Average PnL per trade |
| `max_drawdown` / `max_drawdown_pct` | `Money` / `Decimal` | Peak-to-trough decline |
| `recovery_factor` | `Decimal` | Net PnL / max drawdown |
| `avg_realized_r` | `Decimal` | Average R-multiple per trade |
| `streaks` | `StreakStats` | Longest win/loss streak |

`EdgeScore` — composite 0–100 score:

| Field | Type | Meaning |
|-------|------|---------|
| `version` | `u8` | `EDGE_SCORE_VERSION = 1` |
| `score` | `Option<Decimal>` | `None` if fewer than `MIN_CLOSED_TRADES_FOR_SCORE = 5` trades |
| `components` | `EdgeScoreComponents` | Six component scores |
| `closed_trades` | `u64` | Trades used |

`EdgeScoreComponents` — six independently scored components (each 0–100): `win_rate`, `profit_factor`, `avg_win_loss`, `drawdown`, `recovery`, `consistency`. Equal-weight blend.

**Entry points:** `compute_metrics(trades) -> PerformanceMetrics`, `compute_edge_score(metrics, trades) -> EdgeScore`. Both pure and deterministic.

---

## 6. Daily P&L Calendar (`calendar.rs`)

`DailyPnl` — per-day bucket: `net_pnl`, `gross_pnl`, `fees`, `trades`, `wins`, `losses`, `breakevens`, `volume`.

`PnlCalendar` — monthly calendar with `weeks: Vec<CalendarWeek>` (mirrors Vela's `/api/journal/calendar?month=YYYY-MM` shape). `CalendarWeek` is a row of up to 7 days.

**Entry points:** `build_calendar(trades, year, month) -> PnlCalendar`, `daily_pnl_buckets(trades) -> Vec<DailyPnl>`, `total_fees(trades) -> Money`.

---

## 7. Account Comparison (`compare.rs`)

`AccountMetricsView` — metrics for a single account over a range.

`AccountComparison` — two-sided comparison: `account_a`, `account_b`, `is_prior_period` flag.

**Entry points:** `compare_accounts(view_a, view_b) -> AccountComparison`, `compare_account_vs_prior(account, boundary) -> AccountComparison`.

---

## 8. REST Exposure

The trade-performance journal is exposed over HTTP via `vyapaar-runtime-api-rest` under the `/trade-journal/*` prefix (plan 0003):

| Method | Path | Phase | Purpose |
|--------|------|-------|---------|
| `GET` | `/trade-journal/accounts` | 1 | List journal accounts |
| `POST` | `/trade-journal/accounts` | 1 | Create journal account |
| `GET` | `/trade-journal/overview` | 1 | Performance metrics + edge score |
| `GET` | `/trade-journal/trades` | 1 | Closed trades with annotations, cursor-paginated |
| `PATCH` | `/trade-journal/trades/{id}` | 1 | Set trade annotation |
| `GET` | `/trade-journal/calendar` | 2 | Daily P&L heatmap (`?month=YYYY-MM`) |
| `GET` | `/trade-journal/notes` | 2 | Day-keyed journal notes |
| `POST` | `/trade-journal/notes` | 2 | Create/replace day-keyed note |
| `GET` | `/trade-journal/compare` | 2 | Two-sided account comparison |

See [REST API](../api/rest.md) for full request/response shapes.

---

## 9. Related Documents

* **[Architecture Layers & Topology](architecture_layers.md)** — Full `vyapaar-ledger` bounded context: OMS, portfolio, journal, allocation, performance.
* **[Ledger Allocation](ledger_allocation.md)** — Portfolio allocation & optimization engine (sibling crate).
* **[REST API](../api/rest.md)** — `/trade-journal/*` endpoint reference.
* **[Platform Conformance Matrix](../conformance_matrix.md)** — Implementation status badge.
