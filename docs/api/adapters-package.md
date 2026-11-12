# Adapters Package (`vyapaar-adapters`)

The pluggable Python **Anti-Corruption Layer (ACL)** between Vyapaar's canonical
domain and the wire formats of Indian brokers: a single installable package,
`vyapaar-adapters`, exposing the `vyapaar_adapters` package with one module per
broker — Dhan, Zerodha (Kite), Angel One, Upstox, Fyers, Kotak, and IIFL.

Every broker payload is translated into canonical models (`Quote`, `Bar`,
`OrderRequest`, `Order`, `Position`, `Margin`) as exact `decimal.Decimal` values —
**no `float` ever crosses the boundary**. Broker-specific structures never leak
outward. For the Rust-side production adapters (feeds and execution in
`vyapaar-market` / `vyapaar-exec`), see
[Adapter Interface](../architecture/adapter_interface.md); for broker-by-broker
practical differences, see [Brokers](../india/brokers.md).

> **Verification note.** This page was synced against the package source on
> 2026-11-12. Capabilities currently *missing* (WebSocket streaming, TOTP login,
> real candle history in most brokers) are listed under
> [Current limitations](#current-limitations) with the tracking gap cited.

## Installation & Layout

```bash
pip install -e ./vyapaar-adapters   # requires Python >= 3.11, pydantic >= 2
```

Runtime dependencies are deliberately thin: `pydantic` (config models only) and the
optional `vyapaar` wheel (the package falls back to stdlib `decimal` when absent, so
the whole suite runs offline). The repo follows a 4-zone layout:

```text
vyapaar-adapters/
├── vyapaar_adapters/     # Zone 1: package (common / broker modules / paper / registry)
├── tests/                # Zone 2: offline contract certification (MockTransport)
├── configs/              # Zone 3: paper + live TOML connection templates
├── examples/ scripts/    # Zone 4: fixture recorder & smoke runner
└── docs/                 # design references, gaps, plans, roadmap
```

## Quickstart

```python
from decimal import Decimal

from vyapaar_adapters import OrderRequest, OrderSide, OrderType
from vyapaar_adapters.common import BrokerCredentials, MockTransport
from vyapaar_adapters.dhan import DhanAdapter

transport = MockTransport()               # or UrllibTransport() in production
dhan = DhanAdapter(
    transport=transport,
    credentials=BrokerCredentials(access_token="...", client_id="..."),
)
dhan.connect()

quote = dhan.get_quote("NSE:RELIANCE-EQ")   # -> Quote(last_price=Decimal("2456.70"))
order = dhan.place_order(
    OrderRequest(
        instrument="NSE:RELIANCE-EQ",
        side=OrderSide.BUY,
        quantity=Decimal("10"),
        order_type=OrderType.LIMIT,
        limit_price=Decimal("2455.00"),
    )
)
positions = dhan.get_positions()            # -> list[Position]
margin = dhan.get_margin()                  # -> Margin(available=..., used=..., total=...)
```

Adapters are constructed with an injected transport (`HttpTransport`), a base-URL
override, and pydantic `BrokerCredentials`; nothing reads the environment implicitly
and no credentials are needed for `MockTransport`-backed offline use.

## Canonical Models (`vyapaar_adapters.common.models`)

All models are `@dataclass(frozen=True, slots=True)` and self-validating in
`__post_init__` — negative prices/quantities raise `ValueError` at construction.

| Model | Financial fields | Notes |
|-------|------------------|-------|
| `Quote` | `last_price` (required), optional `change`/`open`/`high`/`low`/`close`, `volume`, `timestamp` | last-traded snapshot |
| `Bar` | `open`, `high`, `low`, `close` (required), `volume`, `interval`, `ts` | canonical OHLCV; the model `get_bars` should return |
| `OrderRequest` | `quantity`, `limit_price`, `trigger_price` | `LIMIT` orders require `limit_price` |
| `Order` | `quantity`, `filled_quantity`, `price` | broker-acknowledged state; `status` enum |
| `Position` | `quantity` (may be negative = short), `average_price`, optional P&L | |
| `Margin` | `available`, `used`, `total` | `currency` defaults to `"INR"` |

Enums follow the Vyapaar ubiquitous language (`StrEnum`): `OrderSide`
(`BUY`/`SELL`), `OrderType` (`LIMIT`/`MARKET`/`STOP_LIMIT`/`STOP_MARKET`),
`TimeInForce` (`DAY`/`IOC`/`GTD`), `ProductType` (`DELIVERY`/`INTRADAY`/`MARGIN`/
`COVER_ORDER`/`BRACKET_ORDER`), `OrderStatus` (`PENDING`, `OPEN`,
`PARTIALLY_FILLED`, `COMPLETE`, `REJECTED`, `CANCELLED`).

## Zero Floats (`vyapaar_adapters.common.parse`)

Three strict helpers guard the boundary (`as_decimal`, `non_negative_decimal`,
`as_iso_ts`):

- `as_decimal` accepts `Decimal`/`int`/`str` and **raises `TypeError` on `float`**;
  junk strings raise `ValueError`.
- `non_negative_decimal(value, field)` adds the sign check with a field-named error.
- `as_iso_ts` parses ISO-8601 to a UTC-aware `datetime` (`Z` normalized, naive
  assumed UTC).

Transport-side, `UrllibTransport` decodes every JSON number with
`json.loads(..., parse_float=Decimal)`, so a bare number in a broker payload arrives
as an exact `Decimal`, never binary floating point. Together these make float
contamination a loud `TypeError` at construction rather than a silent precision loss.

## Protocols & Transports (`common.protocols`, `common.transport`)

Two `@runtime_checkable` Protocols define the contracts:

```python
class HttpTransport(Protocol):
    def request(self, method, url, headers=None, params=None, payload=None) -> tuple[int, Any]: ...

class Adapter(Protocol):
    def connect(self) -> None: ...
    def disconnect(self) -> None: ...
    def get_quote(self, instrument: str) -> Any: ...
    def get_bars(self, instrument: str, interval: str, days: int = 100) -> list[Any]: ...
    def place_order(self, request: Any) -> Any: ...
    def cancel_order(self, order_id: str) -> Any: ...
    def get_positions(self) -> list[Any]: ...
    def get_margin(self) -> Any: ...
```

| Transport | Purpose | Notes |
|-----------|---------|-------|
| `UrllibTransport(timeout=15.0)` | production | stdlib only; JSON out via `json.dumps`, JSON in via `parse_float=Decimal`; `HTTPError` is returned as `(status, body)` so error mapping stays in the adapter |
| `MockTransport(responses=None)` | tests / smoke | serves canned responses keyed by `"METHOD url"`, records every request in `.history`, raises `KeyError` on an unmatched URL |

`BaseRestAdapter` (`common.base`) is the shared skeleton: injected transport +
`default_base_url` + credentials; `_get`/`_post`/`_delete` helpers that raise
`BrokerApiError(status, detail)` on HTTP ≥ 400; no-op `connect`/`disconnect` for
stateless token auth (subclasses override `_headers()` to add broker auth headers).

## Broker Modules (`vyapaar_adapters.<broker>`)

All seven brokers are implemented as modules of the single package — there are no
separate per-broker distributions. Each module maps its wire fields to the canonical
models (quote, order placement, positions, funds/margin) and is registered under the
`vyapaar.adapters` entry-point group **and** the built-in `BROKER_ADAPTERS` fallback
(registry invariant enforced by `tests/test_registry.py`).

| Broker | Module & class | Default base URL | Auth headers | Contract tests |
|--------|----------------|------------------|--------------|----------------|
| Dhan | `dhan.DhanAdapter` | `https://api.dhan.co` | `access-token`, `client-id` | `tests/test_dhan.py` |
| Zerodha | `zerodha.ZerodhaAdapter` | `https://api.kite.trade` | `Authorization: token <api_key>:<access_token>` | `tests/test_zerodha.py` |
| Angel One | `angelone.AngelOneAdapter` | `https://apiconnect.angelbroking.com` | `Authorization: Bearer`, `X-ClientCode`, `X-ApiKey` | — (gap 0001) |
| Upstox | `upstox.UpstoxAdapter` | `https://api.upstox.com/v2` | `Authorization: Bearer` | — (gap 0001) |
| Fyers | `fyers.FyersAdapter` | `https://api-t1.fyers.in/api/v3` | `Authorization: <access_token>` | — (gap 0001) |
| Kotak | `kotak.KotakAdapter` | `https://gw-napi.kotaksecurities.com/Orders/1.0` | `Authorization: Bearer` | — (gap 0001) |
| IIFL | `iifl.IIFLAdapter` | `https://tools.iiflsec.com/a/equities/1.0` | `Authorization: Bearer`, `client_id` | — (gap 0001) |

Selected wire mappings (full per-broker reference:
[`vyapaar-adapters/docs/design/code-brokers.md`](https://github.com/vyapaar-labs/vyapaar-adapters/blob/main/docs/design/code-brokers.md)):

| Broker | Quote | Order placement | Positions | Margin |
|--------|-------|-----------------|-----------|--------|
| Dhan | `GET /v2/marketfeed/ltp/{symbol}` | `POST /v2/orders` (`transactionType`, `NSE_EQ`/`NSE_FNO` segments, `CNC`/`INTRADAY`) | `GET /v2/positions` (`buyQty - sellQty`) | `GET /v2/funds` |
| Zerodha | `GET /quote?i=` | `POST /orders/regular` (`CNC`/`MIS`, `SL`/`SL-M`, `variety=regular`) | `GET /portfolio/positions` (`data.net`) | `GET /user/margins` |
| Angel One | `GET .../market/v1/quote/` | `POST .../order/v1/placeOrder` (`DELIVERY`/`INTRADAY`) | `GET .../order/v1/getPosition` | `GET .../portfolio/v1/getAllHolding` |
| Upstox | `GET /market-quote/quote?instrument_key=` | `POST /order/place` (`D`/`I` products) | `GET /portfolio/short-term-positions` | `GET /user/get-funds-and-margin` |
| Fyers | `POST /quotes` | `POST /orders` (numeric `type` `1..4`, `side` `1`/`-1`) | `GET /positions` (`netPositions`) | `GET /funds` (`fund_limit`) |
| Kotak | `GET /marketdata/v1/quote/ltp` | `POST /orders/place` (`L`/`MKT`/`SL`/`SL-M`, `C`/`M`) | `GET /positions` | `GET /funds/balance` |
| IIFL | `GET /marketdata/instruments/quotes` | `POST /orders/place` (`DELIVERY`/`INTRADAY`) | `GET /portfolio/positions` | `GET /portfolio/funds` |

Quantities and prices are serialized as strings into order payloads
(`str(request.quantity)`) to preserve exact decimals end to end.

## Paper Broker & Fixture Replay (`vyapaar_adapters.paper`)

The offline harness makes CI deterministic without live credentials:

- **`PaperBroker(quotes, fill_fraction=None)`** — a deterministic broker simulation.
  Market orders fill immediately at the simulated quote; limit orders fill when the
  quote crosses the limit; unknown instruments reject with "no quote". It maintains
  real position state (signed fills, average-price resets on side flips), synthesizes
  `days` daily bars seeded at `2024-01-01`, and returns a fixed ₹1,00,000 margin.
- **`RecordingAdapter(inner, sink)`** — wraps any adapter and appends a JSON-safe
  `{"call": ..., "result": ...}` record per call (`to_plain` encoding).
- **`ReplayingAdapter(records)`** — replays a fixture by cursor order; any divergence
  (unknown call, reordered call, exhausted fixture) raises `KeyError`, so fixture gaps
  surface as test failures.

```python
from vyapaar_adapters import PaperBroker, RecordingAdapter
from vyapaar_adapters.common.fixtures import record_fixture, load_fixture

broker = PaperBroker({"NSE:RELIANCE-EQ": "2456.70"})
sink: list[dict] = []
recorder = RecordingAdapter(broker, sink)
recorder.get_quote("NSE:RELIANCE-EQ")

path = record_fixture(tmp_dir, "session", sink)          # tests/fixtures/session.jsonl
replay = ReplayingAdapter(load_fixture(tmp_dir, "session"))
assert replay.get_quote("NSE:RELIANCE-EQ").last_price == Decimal("2456.70")
```

The lossless codec (`common.codec`) encodes `Decimal` as `{"_dec": "..."}`,
`datetime` as `{"_ts": "<iso>"}`, enums as `{"_enum": ..., "value": ...}` and
dataclasses as `{"_model": ..., "fields": {...}}` — scales, timezone-awareness and
enum identity all survive the JSON round trip.

## Registry (`vyapaar_adapters.registry`)

```python
from vyapaar_adapters.registry import load_adapters

adapters = load_adapters()   # {"dhan": DhanAdapter, "zerodha": ZerodhaAdapter, ...}
```

Discovery iterates the `vyapaar.adapters` entry-point group (with a module-string
fallback) and, when the package is not installed as a distribution, falls back to the
built-in `BROKER_ADAPTERS` map. The invariant "entry points stay in sync with
`BROKER_ADAPTERS`" is asserted by `tests/test_registry.py`, which also asserts every
adapter subclasses `BaseRestAdapter` and builds with `MockTransport`.

## Configuration (`common.config` + `configs/*.toml`)

`AdapterConfig` (pydantic) is the declarative connection shape: `broker`,
`base_url` (optional override), `paper: bool`, nested `credentials` (`api_key`,
`access_token`, `client_id`, `api_secret`). Zone 3 ships TOML templates —
`paper.toml`, `dhan.toml`, `zerodha.toml` (five broker templates, a TOML loader, and
template-parse tests are still pending: gap 0007).

```toml
# configs/dhan.toml (copy to adapter.toml, fill from your secret store)
broker = "dhan"
paper = false

[credentials]
api_key = ""
access_token = "YOUR_ACCESS_TOKEN"
client_id = "YOUR_CLIENT_ID"
```

Templates carry `YOUR_*` placeholders only; real credentials belong in the
environment or a secret manager. Practical broker differences (static-IP
requirement, token lifetimes, sandbox availability) are covered in
[Brokers](../india/brokers.md).

## Testing & Verification

The contract-test pattern (currently applied to Dhan and Zerodha):

```python
# tests/test_dhan.py — shape
transport = MockTransport({
    "GET https://api.dhan.co/v2/marketfeed/ltp/RELIANCE-EQ": (
        200, {"ltp": {"RELIANCE-EQ": "2456.70"}},
    ),
    ...
})
adapter = DhanAdapter(transport=transport)
quote = adapter.get_quote("NSE:RELIANCE-EQ")
assert quote.last_price == Decimal("2456.70")
```

Gates (see the repo `Makefile`):

```bash
make test    # pytest -q (offline; MockTransport only, no live credentials)
make lint    # ruff check .
make check   # lint + format-check + tests
```

CI (`.github/workflows/ci.yml`) runs the same gates on push. Tests live only in the
parallel `tests/` directory (workspace rule — no inline tests in `src`).

## Current Limitations

| Limitation | Tracking gap |
|------------|--------------|
| Contract tests exist only for `dhan` + `zerodha`; the other five brokers are untested | [gap 0001](https://github.com/vyapaar-labs/vyapaar-adapters/blob/main/docs/gaps/0001-broker-contract-tests-missing.md) |
| `get_bars` returns a single-quote stub for 6 of 7 brokers (only Zerodha parses real candles) | gap 0002 |
| `cancel_order` discards the broker response and fabricates a zeroed `Order` in all seven adapters | gap 0003 |
| REST-only: no WebSocket/tick streaming, no depth feed | gap 0004 |
| No TOTP / headless login; credentials must be pre-provisioned | gap 0005 |
| `PaperBroker` single-price fill model; hardcoded margin | gap 0006 |
| TOML templates for 5 brokers, loader, and parse tests missing | gap 0007 |

Everything above is tracked by plan 0003 in the repo
(`docs/plans/0003-adapter-parity-streaming-realism.md`); until then, treat `get_bars`
on non-Zerodha brokers and every returned `cancel_order` `Order` as unreliable, and
poll quotes rather than expecting streams.

## Related

- [Adding Adapters](../contributing/adding_adapters.md) — how to add the eighth broker module.
- [Brokers](../india/brokers.md) — broker API facts: auth styles, static IP, rate limits.
- [Adapter Configuration](../india/adapter_configuration.md) — credential sourcing and templates.
- [Adapter Interface & Feeds](../architecture/adapter_interface.md) — the Rust-side feed/execution topology.
