# Indian Markets

This section documents what is true of the Indian market and false of
every other market. Everything here lives in the `vyapaar-market` crate
and the Python `vyapaar.markets.india` module — providing clean, self-contained
market rules that can be tested without a full engine.

## The Documents

- **[Market Calendar](market_calendar.md)** — trading hours, holidays,
  Muhurat sessions, and the distinction between trading holidays and
  settlement holidays.
- **[Cost Model](cost_model.md)** — STT, exchange fees, SEBI turnover
  fee, stamp duty, GST, and brokerage. The single most consequential
  set of assumptions in a backtest.
- **[Universes](universes.md)** — NIFTY 50, BANK NIFTY, and the Alpha
  30 family. How constituents are selected, when they rebalance, and
  how to avoid look-ahead bias.
- **[Mutual Funds](mutual_funds.md)** — AMFI NAV data, TER, SIP
  backtesting, and the direct-vs-regular plan distinction.
- **[Options](options.md)** — weekly and monthly expiry cycles, strike
  intervals, Greeks, and the STT asymmetry between premium and
  exercise.
- **[Brokers](brokers.md)** — practical differences between the supported
  broker adapters.

## India Data Lakehouse (`vyapaar-data`)

`vyapaar-data` (Tier 6, Python + DuckDB) is the **financial lakehouse** for
Indian markets. Implemented capabilities (gaps 0001–0003 closed):

| Capability | Module | Status |
|------------|--------|--------|
| Economic calendar ingestion | `vyapaar_data.service.macro_reader` | ✅ Closed (macro series behind `PythonMacroReader`) |
| Filings ingestion | `vyapaar_data.service.filings` | ✅ Closed (`FilingsReader` / `LakehouseFilingsReader`) |
| Symbol profile / company fundamentals | `vyapaar_data.service.profile` | ✅ Closed (`LakehouseProfileReader`) |
| Parquet catalog coverage | `vyapaar_data.store.parquet` | ✅ Closed |
| Calendar data (earnings, dividends, splits, IPOs) | `vyapaar_data.store.calendar` | Stubbed pending vendor decision |

These feed the `/market-data/calendar` and `/market-data/symbol-profile/{id}`
REST endpoints in `vyapaar-runtime-api-rest` (Plan 0003 Phase 3 — currently
serving illustrative sample data pending the `vyapaar-data` → Rust FFI bridge).


## The Two Facts That Change Everything

Most strategies that fail in Indian markets fail because of one of
these:

1. **Costs are large.** A delivery equity round-trip costs roughly
   0.14% per side once STT, exchange fees, stamp duty, GST, and
   brokerage are summed. A strategy trading weekly needs a substantial
   gross edge just to break even.
2. **The calendar is not simple.** Pre-open sessions, Muhurat trading,
   and settlement-vs-trading holiday distinctions all affect when
   orders execute and when funds settle.

The [Cost Model](cost_model.md) and [Market Calendar](market_calendar.md)
documents are the two most important in this section.
