# Python SDK: `vyapaar.client`

Typed access to the [REST read API](rest.md). The same `Client` methods work
over HTTP (against `vyapaar serve`) and in process (no server).

```python
from vyapaar.client import Client, NotFoundApiError

# HttpTransport: against a running `vyapaar serve --data-dir ./bars`
with Client.http("http://127.0.0.1:8080", timeout=5) as api:
    print(api.health().status)
    for bar in api.bars("TCS.NSE", tf="1m", from_="2024-01-01", to="2024-01-02"):
        print(bar.ts_event.to_ns(), bar.close)

# InprocTransport: the same Rust router in process (needs the native extension)
api = Client.inproc("./bars")
quotes = api.quotes(["TCS"], venue="NSE")
try:
    api.instrument("NOPE.NSE")
except NotFoundApiError as err:
    print(err.code, err.category, err.retryable)  # instrument_not_found not_found False
```

## `Client`

| Method | Route | Returns |
|--------|-------|---------|
| `health()` | `GET /health` | `Health` |
| `capabilities()` | `GET /capabilities` | `CapabilityManifest` |
| `schema()` | `GET /schema` | `dict` (the schema info record) |
| `instruments(*, exchange=None, symbol=None)` | `GET /instruments` | `list[InstrumentInfo]` |
| `instrument(instrument_id)` | `GET /instruments/{id}` | `InstrumentInfo` |
| `bars(instrument_id, *, tf=None, from_=None, to=None)` | `GET /bars/{id}` | `list[Bar]` |
| `quotes(symbols, *, venue=None, as_of=None)` | `GET /quotes` | `list[QuoteTick]` |
| `depth(instrument_id, *, levels=None)` | `GET /depth/{id}` | `Depth` |
| `symbol_profile(instrument_id)` | `GET /market-data/symbol-profile/{id}` | `SymbolProfile` (Plan 0003 Phase 3) |
| `market_calendar(*, from_=None, to=None, category=None)` | `GET /market-data/calendar` | `list[CalendarEvent]` (Plan 0003 Phase 3) |
| `screener_scan(universe, filters=None, *, tf=None, as_of=None)` | `GET /screener/scan` | `list[ScreenerResultRow]` |
| `verify_strategy(manifest)` | `POST /strategies/verify` | `dict` (the IR) |
| `strategies()` | `GET /strategies` | `list[CompiledStrategy]` |
| `compile_strategy(manifest)` | `POST /strategies` | `CompiledStrategy` |
| `trade_journal_accounts()` | `GET /trade-journal/accounts` | `list[JournalAccount]` (Plan 0003 Phase 1) |
| `trade_journal_create_account(name, provenance)` | `POST /trade-journal/accounts` | `JournalAccount` (Plan 0003 Phase 1) |
| `trade_journal_overview()` | `GET /trade-journal/overview` | `TradeJournalOverview` (Plan 0003 Phase 1) |
| `trade_journal_trades(*, from_=None, to=None, limit=None, cursor=None)` | `GET /trade-journal/trades` | `list[ClosedTrade]` (Plan 0003 Phase 1) |
| `trade_journal_annotate(trade_id, *, tags=None, rating=None, review=None, planned_risk=None)` | `PATCH /trade-journal/trades/{id}` | `TradeAnnotation` (Plan 0003 Phase 1) |
| `trade_journal_calendar(*, month=None)` | `GET /trade-journal/calendar` | `PnlCalendar` (Plan 0003 Phase 2) |
| `trade_journal_notes(*, account=None, limit=None)` | `GET /trade-journal/notes` | `list[JournalNote]` (Plan 0003 Phase 2) |
| `trade_journal_create_note(account_id, day, note)` | `POST /trade-journal/notes` | `JournalNote` (Plan 0003 Phase 2) |
| `trade_journal_compare(*, account_a=None, account_b=None, account=None, mode=None)` | `GET /trade-journal/compare` | `AccountComparison` (Plan 0003 Phase 2) |
| `backtest_create(manifest)` | `POST /backtests` | `dict` (run_id) |
| `backtest_get(run_id)` | `GET /backtests/{id}` | `dict` |
| `backtest_journal(run_id)` | `GET /backtests/{id}/journal` | `dict` |
| `sweep_create(manifest)` | `POST /sweeps` | `dict` (sweep_id) |
| `sweep_get(sweep_id)` | `GET /sweeps/{id}` | `dict` |
| `orders_place(order_request)` | `POST /orders` | `dict` (risk-gated) |
| `orders_list()` | `GET /orders` | `list[Order]` |
| `orders_cancel(order_id)` | `DELETE /orders/{id}` | `dict` |
| `positions_list()` | `GET /positions` | `list[Position]` |
| `portfolio_get()` | `GET /portfolio` | `Portfolio` |
| `trading_state_get()` | `GET /trading-state` | `TradingState` |
| `trading_state_put(state)` | `PUT /trading-state` | `dict` |

Only these routes have methods. Every other route answers 501 and has no client method.
`instrument_id` is `SYMBOL.EXCHANGE` (a string or an `InstrumentId`).
`tf` is `<n><s|m|h|d|w|mo>`, default `1m` on the server. `levels` is 1..=50.
`Client` is a context manager; `close()` releases the transport.

## Models (`vyapaar.client.models`)

Ignore unknown fields so a newer server still parses:

`Health`, `InstrumentInfo`, `Depth`/`DepthLevel`, `CompiledStrategy`, `ScreenerResultRow`,
`CapabilityManifest`, `SymbolProfile`, `CalendarEvent`, `JournalAccount`,
`TradeJournalOverview`, `ClosedTrade`, `TradeAnnotation`, `PnlCalendar`, `JournalNote`,
`AccountComparison`, `Portfolio`, `Position`, `Order`, `TradingState`.

`Bar` and `QuoteTick` are the existing `vyapaar.wire.wire` types.

`CompiledStrategy` has `id` (string, `sha256:<hex>`) and `ir` (dict, the verified IR).

Constructors: `Client.http(base_url, **options)` (options go to `HttpTransport`),
`Client.inproc(data_dir)`, or `Client(transport)` for any `Transport`.

## Capabilities and schema

`capabilities()` returns a `CapabilityManifest`: `crates`, `market_packs`, `endpoints`
(every route as `METHOD /path`), `not_implemented` (the subset that answers 501),
`toolsets`, `adapters` and `features` (a `dict[str, bool]`). Read it instead of
probing for 501s. `schema()` returns the served schema info as a plain dict
(currently the OpenAPI version and the API version).

```python
api = Client.inproc("./bars")
manifest = api.capabilities()
print("POST /backtests" in manifest.not_implemented)  # True
print(api.schema())  # {"openapi": "3.1.0", "version": "..."}
```

## Screener scan

`screener_scan(universe, filters)` returns the instruments passing `filters` as
`ScreenerResultRow(instrument_id, metrics)`, in id order. `universe` is one id or a list of
`SYMBOL.EXCHANGE`; `filters` is a `ScreenerFilterGroup`, a predicate or the equivalent dict
(none matches every instrument). It is evaluated by the Rust engine over the dataset's bars at
`tf` (default `1d`; a Parquet data directory holds `1m` bars, so pass `tf="1m"` there) on the
bars known at `as_of` (inclusive). Metrics computable from bars: `open`, `high`, `low`, `close`,
`volume`, `price_52_week_high`/`_low`, `SMA<N>` and `RSI`. Any other metric is a
`ValidationApiError` with `context["reason"] == "unsupported_metric"`, never an empty result;
`NotFoundApiError` for an unknown instrument; `too_many_rows` over the limits (1,000 instruments,
500 rows, 2,000,000 bars).

```python
rows = Client.inproc("./bars").screener_scan(["TCS.NSE", "INFY.NSE"], tf="1m")
for row in rows:
    print(row.instrument_id, row.metrics)
```

## Trade-Performance Journal (Plan 0003)

The trade-performance journal is **distinct** from `/portfolio/accounts` and from the run/backtest
journals (`/backtests/{id}/journal`). It is backed by `vyapaar-ledger-performance`.

```python
api = Client.inproc("./bars")

# List journal accounts
accounts = api.trade_journal_accounts()
for acct in accounts:
    print(acct.id, acct.name, acct.provenance)

# Create a journal account
acct = api.trade_journal_create_account("My Journal", "Manual")

# Overview: performance metrics + edge score
overview = api.trade_journal_overview()
print(overview.metrics.win_rate, overview.edge_score.score)

# Closed trades (cursor-paginated)
trades = api.trade_journal_trades(limit=50)
for t in trades:
    print(t.symbol, t.realized_pnl, t.tags)

# Annotate a trade
api.trade_journal_annotate(trade_id, tags=["earnings"], rating=4, review="Good setup")

# Daily P&L calendar
cal = api.trade_journal_calendar(month="2026-11")
for day in cal.weeks:
    for d in day.days:
        print(d.date, d.net_pnl)

# Day-keyed notes
notes = api.trade_journal_notes(account=acct.id)
api.trade_journal_create_note(acct.id, "2026-11-06", "Ran the opening-range breakout")

# Account comparison
comp = api.trade_journal_compare(account_a="acct_a", account_b="acct_b")
print(comp.account_a.net_pnl, comp.account_b.net_pnl)
```

## Strategy compilation

Use `compile_strategy(manifest)` to compile and store a strategy manifest in the server's
in-process catalog. The returned `CompiledStrategy` holds a stable `id` (SHA-256 of the
canonical manifest JSON) and the verified IR. The same manifest always gets the same id
and is stored once. The catalog holds at most 1,000 strategies; a new one past the limit
raises `ValidationApiError` with `context["reason"] == "catalog_full"`. Source code in the
manifest (`code` or `source` keys) is rejected with `reason = "source_unsupported"`. Use
`strategies()` to list all compiled strategies of this process, ordered by id.

```python
api = Client.inproc("./bars")

manifest = {...}  # strategy manifest as a dict
compiled = api.compile_strategy(manifest)
print(compiled.id)   # sha256:...
print(compiled.ir)   # {"warmup_bars": 20, "universe": {...}, ...}

all_strategies = api.strategies()
```

## Transports

- `HttpTransport(base_url, *, timeout=10.0, retry=None, headers=None, ...)`:
  `base_url` must start with `http://` or `https://`; `timeout` is a positive
  number of seconds; redirects are not followed.
- `InprocTransport(data_dir)`: runs the Rust router in this process over the
  `SYMBOL.EXCHANGE.parquet` files in `data_dir` (`NotADirectoryError` if it is
  not a directory). The directory is loaded once per process and cached, so
  files added later are not seen. Its calls reuse the runtime started by
  [`vyapaar.event_loop`](../architecture/event_loop.md) if there is one.

Both return every HTTP-style answer, error statuses included; the client turns
non-2xx answers into exceptions.

## Time inputs

`from_`, `to` and `as_of` accept a `TimeLike`:

- `UnixNanos`;
- an aware `datetime` (naive raises `naive_datetime`);
- a `date` or `YYYY-MM-DD` string (midnight UTC);
- an RFC 3339 string such as `2024-01-01T09:15:00+05:30` (validated and sent
  verbatim).

Times before 1970-01-01 (`before_epoch`) or after the largest nanosecond `i64`
instant, **2262-04-11T23:47:16.854775807Z** (`invalid_time`), are rejected. A
range with `from_ >= to` is `empty_range`.

## Errors

Bad arguments raise `RequestValidationError` (a `ValueError` with `field` and
`reason`) **before anything is sent**. A failed call raises an `ApiError`
subclass chosen by the envelope `code`. Attributes: `code`, `message`,
`status` (`None` if no response arrived), `retryable` (the server's flag),
`context`, `category`, `api_version`.

| Class | Codes |
|-------|-------|
| `ValidationApiError` | `validation_invalid_request` (422; includes `too_many_rows`) |
| `NotFoundApiError` | `not_found`, `instrument_not_found`, `order_not_found` |
| `AuthApiError` | `unauthorized`, `forbidden` |
| `RateLimitedApiError` | `rate_limited` |
| `RiskApiError` | `risk_*` (422; includes `risk_trading_halted`, `risk_instrument_unknown`, `order-rate`/quantity/tick/band/notional refusals) |
| `OrderRejectedApiError` | `order_rejected`, `order_execution_unavailable` |
| `MarketDataUnavailableApiError` | `market_data_unavailable` (for example no depth on bars) |
| `TransportApiError` | `timeout`, `transport_error` |
| `InternalApiError` | `internal_error` |
| `UnsupportedApiError` | `unsupported` |
| `NotImplementedApiError` (subclass of `UnsupportedApiError`) | `not_implemented` (501) |
| `InvalidResponseError` | `invalid_response`: not an envelope, a payload that does not parse, or a `schema_version` mismatch |

`NotFoundApiError` also covers an unknown route (`not_found`, an envelope rather than an empty
body); an unsupported method on a known path is `UnsupportedApiError` (405).

An unknown code from a newer server stays a plain `ApiError` with the raw code.

`TransportApiError` is also raised by the transports when no answer arrived:
over HTTP for timeouts and connection failures (`retryable=True`), and in
process when the native call raises `OSError` or `ValueError`, for example a
data directory that vanished (`code="transport_error"`, `retryable=False`).

```python
from vyapaar.client import (
    Client, NotImplementedApiError, ValidationApiError, RequestValidationError,
)

api = Client.inproc("./bars")
try:
    api.bars("TCS.NSE", tf="1m", from_="2020-01-01", to="2024-01-01")
except ValidationApiError as err:
    if err.context and err.context.get("reason") == "too_many_rows":
        print("narrow the range; limit =", err.context.get("limit"))
except RequestValidationError as err:
    print(err.field, err.reason)   # raised locally, nothing was sent
```

## Retry policy

Retries exist only on `HttpTransport` and are **off by default**.
`RetryPolicy(delays=(...))` has one delay (seconds) per retry, at most 10, each
finite and >= 0. Only an envelope with `retryable: true` or a connection
failure is retried; anything else returns at once.

```python
from vyapaar.client import Client, RetryPolicy

api = Client.http("http://127.0.0.1:8080", retry=RetryPolicy.fixed(3, 0.5))
```

`RetryPolicy.none()` is the default; `RetryPolicy.fixed(retries, delay)` repeats
one delay.
