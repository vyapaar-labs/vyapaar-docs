# Adding Broker Adapters

Adapter development happens in the `vyapaar-adapters` repository. This document
describes how to add a new broker **module** to the package, test it offline, and
register it.

> Synced against `vyapaar-adapters` source on 2026-11-12. The seven shipped modules
> (`dhan`, `zerodha`, `angelone`, `upstox`, `fyers`, `kotak`, `iifl`) are the living
> examples — `dhan.py` is the cleanest template, `zerodha.py` the reference for real
> candle history.

## Before You Start

1. Read [Adapters Package](../api/adapters-package.md) for the package model and
   [Adapter Interface](../architecture/adapter_interface.md) for how Python adapters
   relate to the Rust production path (`vyapaar-market` / `vyapaar-exec`).
2. Check the parity state in the repo's `docs/gaps/` index — `dhan` and `zerodha`
   are contract-certified; the other five are implemented but untested (gap 0001).
   New broker modules should land **with** their contract test, never without.
3. Get API access from the broker. Most Indian brokers require registration and a
   static IP for order placement — see
   [Brokers](../india/brokers.md#the-static-ip-requirement).

## The Module Pattern

Adapters are single modules of the one package — there is no per-broker
distribution, no scaffolding generator:

```text
vyapaar-adapters/
├── vyapaar_adapters/
│   ├── <broker>.py          # the adapter module you are adding
│   ├── common/              # shared: models, parse, protocols, transport, base, config, codec, fixtures
│   ├── paper.py             # offline harness
│   └── registry.py          # BROKER_ADAPTERS + entry-point discovery
├── tests/
│   └── test_<broker>.py     # contract test (required in the same change)
├── configs/
│   └── <broker>.toml        # connection template
└── pyproject.toml           # entry-point registration
```

## Step 1: Subclass `BaseRestAdapter`

```python
"""MyBroker REST adapter."""

from __future__ import annotations

from decimal import Decimal

from vyapaar_adapters.common.base import BaseRestAdapter
from vyapaar_adapters.common.models import (
    Margin, Order, OrderRequest, OrderSide, OrderStatus, OrderType, Position, Quote,
)
from vyapaar_adapters.common.parse import as_decimal


class MyBrokerAdapter(BaseRestAdapter):
    """REST adapter for the MyBroker API."""

    default_base_url = "https://api.mybroker.com"

    def _headers(self) -> dict[str, str]:
        headers = super()._headers()
        if self.credentials.access_token:
            headers["Authorization"] = f"Bearer {self.credentials.access_token}"
        return headers
```

`BaseRestAdapter` gives you `_get`/`_post`/`_delete` (returning the decoded JSON
body) and `_raise_for_status` (raising `BrokerApiError(status, detail)` on HTTP
≥ 400). `transport`, `base_url`, and `credentials` are constructor-injected —
never read the environment inside an adapter.

## Step 2: Translate wire → canonical

Implement the `Adapter` protocol methods. Two invariants carry the weight:

**Canonical instrument format.** Instruments arrive as
`<EXCHANGE>:<SYMBOL>` (e.g. `NSE:RELIANCE-EQ`, `NFO:NIFTY24DEC24000CE`). Split and
map to broker identifiers inside the module:

```python
@staticmethod
def _split_instrument(instrument: str) -> tuple[str, str]:
    if ":" not in instrument:
        raise ValueError(f"instrument must be <EXCHANGE>:<SYMBOL>, got {instrument!r}")
    exchange, symbol = instrument.split(":", maxsplit=1)
    return exchange.upper(), symbol
```

**Exact decimals everywhere.** Coerce wire values through `as_decimal` — it accepts
`Decimal`/`int`/`str` and raises `TypeError` on `float`. Serialize outbound values
as strings (`str(request.quantity)`) so no precision is lost in JSON. Never parse a
financial value with `float(...)`.

```python
def get_quote(self, instrument: str) -> Quote:
    _, symbol = self._split_instrument(instrument)
    body = self._get("/quotes", params={"symbol": symbol})
    return Quote(
        instrument=instrument,
        last_price=as_decimal(body["ltp"]),
        open=as_decimal(body["open"]) if "open" in body else None,
    )
```

Enum mappings are plain dicts keyed by the canonical enum, so an unmapped member
fails loudly with `KeyError` instead of silently sending the wrong value:

```python
order_type = {
    OrderType.LIMIT: "LMT",
    OrderType.MARKET: "MKT",
    OrderType.STOP_LIMIT: "SL",
    OrderType.STOP_MARKET: "SLM",
}[request.order_type]
```

Per-method expectations (matching the shipped modules):

- `place_order(request) -> Order` — POST the mapped payload; read the broker's
  order id from the response and return a canonical `Order` (typically
  `status=OrderStatus.PENDING`, `filled_quantity=Decimal(0)`). A missing order id
  is `raise ValueError(f"order submission failed: {body!r}")` — never fabricate one.
- `get_positions() -> list[Position]` — map net quantities (Dhan, for example,
  computes `buyQty - sellQty` so long is positive).
- `get_margin() -> Margin` — map funds/balance into `available`/`used`/`total`
  (all `non_negative_decimal`).
- `get_bars(instrument, interval, days) -> list[Bar]` — return **real** OHLCV
  history (see `zerodha.py`'s candle loop). Returning `[self.get_quote(...)]` is
  the degenerate stub tracked by gap 0002 — do not copy it.
- `cancel_order(order_id) -> Order` — parse the cancel response into the canonical
  `Order`. Fabricating a zeroed `Order` is the stub tracked by gap 0003 — do not
  copy it.

## Step 3: Contract test with `MockTransport`

Mirror `tests/test_dhan.py`: build a `MockTransport` with canned `200` responses
shaped like the broker's real wire payload, then assert exact `Decimal` fields,
enum mappings, and error paths.

```python
# tests/test_mybroker.py — shape
from decimal import Decimal

from vyapaar_adapters.common import MockTransport
from vyapaar_adapters.mybroker import MyBrokerAdapter


def _adapter(**responses):
    return MyBrokerAdapter(transport=MockTransport(responses))


def test_quote_translates_to_decimal():
    adapter = _adapter({"GET https://api.mybroker.com/quotes": (200, {"ltp": "2456.70"})})
    quote = adapter.get_quote("NSE:RELIANCE-EQ")
    assert quote.last_price == Decimal("2456.70")   # exact, never float


def test_missing_order_id_raises():
    adapter = _adapter({"POST https://api.mybroker.com/orders": (200, {})})
    # expect ValueError from place_order, BrokerApiError from HTTP >= 400
```

Assert in every test file:

1. Each canonical field equals an exact `decimal.Decimal` (a bare number in the
   canned payload still arrives as `Decimal` — `MockTransport`/`UrllibTransport`
   decode with `parse_float=Decimal`).
2. Enum round trips (`OrderSide`, `OrderType`, `ProductType`, `TimeInForce`).
3. Error paths: missing order id raises `ValueError`; HTTP ≥ 400 raises
   `BrokerApiError`.

## Step 4: Register in two places (invariant)

1. `pyproject.toml`:

```toml
[project.entry-points."vyapaar.adapters"]
mybroker = "vyapaar_adapters.mybroker:MyBrokerAdapter"
```

2. `vyapaar_adapters/registry.py` — append to `BROKER_ADAPTERS`:

```python
"mybroker": MyBrokerAdapter,
```

`tests/test_registry.py` enforces that the entry-point group and
`BROKER_ADAPTERS` stay in sync and that every adapter subclasses
`BaseRestAdapter` and builds with `MockTransport` — the test fails if you update
one side and not the other.

## Step 5: Config template

Add `configs/mybroker.toml` in the shape of `configs/dhan.toml` (`broker`,
`paper`, `[credentials]`, `YOUR_*` placeholders only). Note: the TOML loader and
the directory-driven template-parse test are pending (gap 0007); still add the
template so it is covered when they land.

## Step 6: Documentation

- Add the wire mappings of your module to `docs/design/code-brokers.md` in the
  adapter repo (endpoint paths, enum/product mappings, positions semantics).
- If the broker has notable quirks (daily tokens, no sandbox, static-IP writes),
  note them in [Brokers](../india/brokers.md).
- Update the tables in [Adapters Package](../api/adapters-package.md).

## Step 7: Verification gates

```bash
cd vyapaar-adapters
make test     # pytest -q
make lint     # ruff check .
make check    # lint + format-check + tests
mypy vyapaar_adapters tests   # strict-ish: disallow_untyped_defs
```

Everything runs offline — no live credentials, no network. Live smoke tests are
not part of CI; the `scripts/smoke_adapters.py` runner only exercises
`MockTransport`-backed construction of every registered adapter.

## Common Mistakes

### Parsing with `float`

The zero-floats invariant is enforced at three layers (`parse_float=Decimal`
transport decode, `as_decimal` coercion, frozen self-validating models). A
`float(...)` in your module is a review-blocking violation: prices, quantities,
and balances must be `Decimal` end to end.

### Returning stubs from `get_bars` / `cancel_order`

Copy the real implementations (`zerodha` candles, parsed cancel responses), not
the stubs the shipped modules still carry (gaps 0002/0003). New modules are
expected to land ahead of the parity plan, not reproduce its debt.

### Registering only one side of the registry

Entry point without `BROKER_ADAPTERS` (or vice versa) breaks
`tests/test_registry.py`. Update both.

### Leaking broker structures

The module is the anti-corruption layer: broker-specific dicts stay inside the
module. Anything crossing back into engine code is a canonical model.

### Skipping the error paths

HTTP ≥ 400 (`BrokerApiError`), missing order ids (`ValueError`), unmapped enums
(`KeyError`), and unknown instruments (`ValueError` from the splitter) are part of
the contract — test them explicitly with `MockTransport` canned error payloads.

## Checklist

- [ ] `vyapaar_adapters/<broker>.py` subclassing `BaseRestAdapter`, `default_base_url` set, `_headers()` auth added
- [ ] All protocol methods implemented with canonical models and `as_decimal` coercion
- [ ] Real `get_bars` (OHLCV) and parsed `cancel_order` — no stubs
- [ ] `tests/test_<broker>.py` with `MockTransport`: Decimal assertions, enum mappings, error paths
- [ ] Entry point in `pyproject.toml` **and** `BROKER_ADAPTERS` in `registry.py`
- [ ] `configs/<broker>.toml` template with placeholders
- [ ] Wire mappings documented in the adapter repo's `docs/design/code-brokers.md`
- [ ] `make test`, `make lint`, `make check`, and `mypy` all green
