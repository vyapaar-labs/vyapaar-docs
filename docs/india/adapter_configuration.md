# Adapter Configuration

How the broker adapters in `vyapaar-adapters` are configured: constructor
injection, the pydantic `AdapterConfig`/`BrokerCredentials` models, the Zone 3
`configs/*.toml` templates, paper mode, and credential handling. For the package
overview see [Adapters Package](../api/adapters-package.md); for broker-by-broker
differences see [Brokers](brokers.md); for the Rust-side production adapters see
[Adapter Interface](../architecture/adapter_interface.md).

> Synced against `vyapaar-adapters` source on 2026-11-12.

## What is implemented

All seven brokers live in **one package** (`vyapaar-adapters`, import name
`vyapaar_adapters`) — there are no separate per-broker distributions:

| Broker | Module | Implemented | Contract-certified |
|---|---|---|---|
| Dhan | `vyapaar_adapters.dhan` | Yes (REST) | Yes (`tests/test_dhan.py`) |
| Zerodha (Kite) | `vyapaar_adapters.zerodha` | Yes (REST, incl. real candle history) | Yes (`tests/test_zerodha.py`) |
| Angel One | `vyapaar_adapters.angelone` | Yes (REST) | No (gap 0001) |
| Upstox | `vyapaar_adapters.upstox` | Yes (REST) | No (gap 0001) |
| Fyers | `vyapaar_adapters.fyers` | Yes (REST) | No (gap 0001) |
| Kotak | `vyapaar_adapters.kotak` | Yes (REST) | No (gap 0001) |
| IIFL | `vyapaar_adapters.iifl` | Yes (REST) | No (gap 0001) |

"Certified" here means canned-payload contract tests against `MockTransport` exist
for the wire→canonical translation (exact `Decimal` fields, enum mappings, error
paths). The adapters are development/research-plane Python (see
[Brokers](brokers.md#the-static-ip-requirement)); the Rust production path is
`vyapaar-market`/`vyapaar-exec`.

## Constructor injection

Adapters take everything explicitly — no implicit environment reads:

```python
from vyapaar_adapters.common import BrokerCredentials, MockTransport, UrllibTransport
from vyapaar_adapters.dhan import DhanAdapter

dhan = DhanAdapter(
    transport=UrllibTransport(timeout=15.0),   # default; MockTransport in tests
    base_url=None,                             # optional override of the default
    credentials=BrokerCredentials(
        access_token="...",                    # from your secret store
        client_id="...",
    ),
)
```

Each adapter uses the credential fields it needs in its `_headers()`:
Dhan sends `access-token` + `client-id`; Zerodha composes
`Authorization: token <api_key>:<access_token>`; Angel One adds
`X-ClientCode`/`X-ApiKey`; the Bearer-token brokers (Upstox, Kotak, IIFL) use
`access_token` alone; Fyers sends the raw token as `Authorization`.

## The pydantic config models

`vyapaar_adapters.common.config` defines the declarative shape:

```python
class BrokerCredentials(BaseModel):
    api_key: str = ""          # broker API key / app key
    access_token: str = ""     # OAuth access token (Zerodha/Upstox/Fyers/...)
    client_id: str = ""        # broker client / user id
    api_secret: str = ""       # API secret (Angel One / IIFL / Kotak)

class AdapterConfig(BaseModel):
    broker: str                # registry name: "dhan", "zerodha", "paper", ...
    base_url: str | None = None
    paper: bool = False
    credentials: BrokerCredentials = BrokerCredentials()
```

## Zone 3 TOML templates

`configs/` ships templates matching that shape:

| Template | Broker | Paper | Status |
|---|---|---|---|
| `configs/paper.toml` | `paper` | `true` | present |
| `configs/dhan.toml` | `dhan` | `false` | present |
| `configs/zerodha.toml` | `zerodha` | `false` | present |
| `angelone/upstox/fyers/kotak/iifl.toml` | — | — | missing (gap 0007) |

```toml
# configs/zerodha.toml (copy to adapter.toml and fill)
broker = "zerodha"
paper = false
# base_url = "https://api.kite.trade"

[credentials]
api_key = "YOUR_API_KEY"
access_token = "YOUR_ACCESS_TOKEN"
client_id = ""
```

To use one, copy the template to `adapter.toml`, fill credentials, and construct
`AdapterConfig` from it. Two pieces are still pending (gap 0007, plan 0003):
an `AdapterConfig.from_toml(path)` loader, and a test asserting every template in
`configs/` parses — until then the templates are reference documents, and
credential assembly is code-side.

## Paper mode

There are two paper paths:

1. **Deterministic paper broker** — `vyapaar_adapters.paper.PaperBroker` fills
   orders from a quote map with exact `Decimal` arithmetic (market fills at the
   quote, limit fills on crossing). No credentials, no network. Combined with
   `RecordingAdapter`/`ReplayingAdapter` it produces replayable JSONL fixtures, so
   CI is fully deterministic. See
   [Adapters Package](../api/adapters-package.md#paper-broker-fixture-replay-vyapaar_adapterspaper).
2. **Broker REST in paper mode** — the `paper` field of `AdapterConfig` /
   `configs/paper.toml` declares the intent; the wiring that routes a loaded config
   into a `PaperBroker` is part of pending gap 0007.

The fill model is deliberately simple (single price, no depth/slippage); its
realism upgrade is tracked as improvement gap 0006.

## Credentials & secrets

- Credentials come from your environment or a secrets manager and are injected
  into `BrokerCredentials` — the repo never reads them implicitly and the
  committed templates carry `YOUR_*` placeholders only.
- `BrokerCredentials` fields are plain strings held on the adapter instance and
  used only in request headers; do not log adapter `repr()`s.
- Token lifetimes are broker-specific (Zerodha's access token is daily;
  Dhan's can be long-lived) — see [Brokers](brokers.md#authentication-differences).
- There is no TOTP/headless login flow yet (gap 0005): tokens must be obtained
  outside Vyapaar today.

## Registry resolution

`AdapterConfig.broker` names resolve through
`vyapaar_adapters.registry.load_adapters()` — the `vyapaar.adapters` entry-point
group with a built-in `BROKER_ADAPTERS` fallback (kept in sync by
`tests/test_registry.py`). Naming an unknown broker is a `KeyError` at dict access,
not a silent fallback to paper.

## Verification

```bash
cd vyapaar-adapters
make test    # pytest -q — offline, MockTransport only, no live credentials
make lint    # ruff check .
make check   # lint + format-check + tests
```

The suite never contacts a live broker; recorded-fixture replay
(`tests/test_replay.py`) and canned `MockTransport` payloads cover the wire
translations that exist today (gaps 0001–0003 track the rest).
