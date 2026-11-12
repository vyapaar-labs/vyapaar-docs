# Broker Adapters

Vyapaar connects to Indian brokers through the adapters in the `vyapaar-adapters`
package. Each adapter is a module of that single package implementing the same
contract — the `Adapter` protocol (`connect`, `disconnect`, `get_quote`,
`get_bars`, `place_order`, `cancel_order`, `get_positions`, `get_margin`) — but the
underlying broker APIs differ in authentication, symbol format,
data granularity, rate limits, and regulatory constraints.

This document summarises the practical differences. The package itself is
documented in [Adapters Package](../api/adapters-package.md); configuration in
[Adapter Configuration](adapter_configuration.md).

*Last verified: 2026-11-12 (adapter status synced to source; broker API facts
re-check against the brokers' own documentation before relying on any detail
below).*

## Adapter Status

All seven brokers are **implemented** as modules of the single `vyapaar-adapters`
package, with wire→canonical translation and entry-point registration. Contract
certification (canned-payload tests against `MockTransport`) currently covers
**Dhan** and **Zerodha** only; the other five are implemented but untested
(gap 0001 in the adapter repo). All adapters are REST-only today: `get_bars`
returns real OHLCV history for Zerodha but a single-quote stub for the other six
(gap 0002), `cancel_order` does not yet parse broker responses (gap 0003), and
there is no WebSocket streaming or headless login (gaps 0004/0005) — tokens must
be pre-provisioned. Track the parity work in the repo's plan 0003.

| Broker | Module | Implemented | Contract-certified |
|--------|-----------------|-------------|--------------------|
| Zerodha | `vyapaar_adapters.zerodha` | Yes (REST) | Yes |
| Dhan | `vyapaar_adapters.dhan` | Yes (REST) | Yes |
| Angel One | `vyapaar_adapters.angelone` | Yes (REST) | No (gap 0001) |
| Upstox | `vyapaar_adapters.upstox` | Yes (REST) | No (gap 0001) |
| Fyers | `vyapaar_adapters.fyers` | Yes (REST) | No (gap 0001) |
| Kotak | `vyapaar_adapters.kotak` | Yes (REST) | No (gap 0001) |
| IIFL | `vyapaar_adapters.iifl` | Yes (REST) | No (gap 0001) |

## The Primary Brokers

The brokers most commonly used by algorithmic traders in India:

| Broker | API | Auth | Sandbox | Market data |
|--------|-----|------|---------|-------------|
| Zerodha | Kite Connect | OAuth (daily) | No | Paid (₹500/mo) |
| Dhan | DhanHQ | Token | **Yes** | Paid |
| Angel One | SmartAPI | TOTP + API key | No | Included |
| Fyers | API v3 | OAuth | No | Included |
| Upstox | Uplink | OAuth (daily) | **Yes** | Included |
| Kotak | Kotak Neo | OAuth | No | Included |

## Dhan Adapter

Dhan (`vyapaar_adapters.dhan`, `DhanAdapter`) is a REST adapter over the DhanHQ
v2 API with headers `access-token` and `client-id`. It maps the canonical
`NSE`/`BSE`/`NFO`/`MCX` exchanges to Dhan segments (`NSE_EQ`, `BSE_EQ`,
`NSE_FNO`, `MCX_COMM`), order types to `LIMIT`/`MARKET`/`SL`/`SL_M`, and the
delivery/intraday products to `CNC`/`INTRADAY`. Positions come back as net
quantity `buyQty - sellQty` (long positive), and margin from
`GET /v2/funds`. Its wire translation is contract-certified
(`tests/test_dhan.py` against `MockTransport` payloads).

Current caveats (tracked in the adapter repo): `get_bars` surfaces a quote
instead of candle history — the Dhan candles endpoint is keyed by security id,
which needs an instrument-master mapping (gap 0002); `cancel_order` does not yet
parse the cancel response (gap 0003).

## Zerodha Adapter

Zerodha (`vyapaar_adapters.zerodha`, `ZerodhaAdapter`) is a REST adapter over
Kite Connect with `Authorization: token <api_key>:<access_token>` headers. It is
the most complete module today:

- **Order vocabulary.** Products map to `CNC` (delivery) and `MIS` (intraday);
  order types to `LIMIT`/`MARKET`/`SL`/`SL-M`; `DAY`/`IOC` validity.
  Quantities and prices are serialized as exact decimal strings.
- **Real candle history.** `get_bars` calls
  `GET /instruments/historical/{symbol}/{interval}`, parses `data.candles` into
  canonical `Bar`s (exact `Decimal` OHLC, integer volume) and honors `days`.
  This is the only broker with real OHLCV today (gap 0002 tracks the others).
- **Order submission.** `POST /orders/regular` with `variety=regular`; a missing
  `data.order_id` in the response raises `ValueError` instead of fabricating
  an order.
- **Contract-certified.** `tests/test_zerodha.py` covers the wire translation
  against canned payloads.

Kite's WebSocket ticker is *not* implemented in this package — streaming is
tracked as gap 0004, and the Rust-side binary decoder lives in
`vyapaar-market` (see
[Adapter Interface](../architecture/adapter_interface.md)).

## The Static IP Requirement

SEBI requires a registered static IP for API-based order placement. This
is the single biggest practical constraint on live algorithmic trading
in India.

- **Upstox and Dhan** offer sandbox environments that sidestep the
  static IP requirement for testing.
- **Angel One and Zerodha** do not, which means their write paths can
  be tested only after registering a static IP with the broker.[reference:24]

For a research-first platform like Vyapaar, this means: **do backtesting
and paper trading with any adapter, but only go live after registering a
static IP with your broker**. The Sandbox adapter, which simulates
execution locally over real market data, does not require a static IP.

## Authentication Differences

Each broker handles auth differently, and this is the most common source
of adapter bugs:

- **Zerodha Kite Connect (Python adapter):** OAuth flow with a daily access token
  generated from a login redirect. The token expires every morning,
  which is inconvenient for headless systems but is what Kite
  requires.
- **Dhan:** Token-based. The access token can be long-lived, which
  makes Dhan the simplest adapter to run unattended.
- **Angel One SmartAPI:** TOTP-based two-factor authentication. The
  adapter generates TOTP codes from a shared secret.
- **Fyers:** OAuth v3 with token refresh.
- **Upstox:** OAuth with a daily browser login and no headless
  refresh — this is Upstox's main limitation for automated systems.

In every case, credentials are supplied to the adapter constructor through
`BrokerCredentials` (`api_key`, `access_token`, `client_id`, `api_secret`) —
sourced from your environment or secret store; see
[Adapter Configuration](adapter_configuration.md).

The `vyapaar-adapters` package has no headless login flow yet (gap 0005): there
is no TOTP generation, OAuth handshake, or token refresh in the package, so
today you obtain the access token outside Vyapaar and inject it. Automating
that daily handshake is the Rust-side
[headless auth](../architecture/adapter_interface.md#32-headless-authentication-vyapaar-exec-auth)
design; the Python side is tracked as gap 0005.

Because every adapter is synchronous REST over an injected transport, there is
no async `connect()` coroutine returning a session object — `connect()` /
`disconnect()` are plain methods, no-ops for stateless token auth.

## Data Granularity

| Broker | Live depth | Historical (intraday) | Historical (daily) |
|---|---|---|---|
| Zerodha | 5 levels | 1-min (paid) | From inception |
| Dhan | 5, 20, 200 levels | 1-min, 5-year window | From inception |
| Angel One | 5 levels | 1-min | From inception |
| Fyers | 5, 50 on separate feed | 1-min from mid-2017 | From inception |
| Upstox | 5, 30 on paid tier | 1-min | From 2000 |

[reference:25]

For options strategies, none of these brokers provide sufficient
historical option data for serious backtesting. You must supplement
with NSE bhavcopy archives or a paid vendor.

## Order Update Notifications

Brokers deliver order status updates through two mechanisms: **postback
webhooks** and **WebSocket streams**. Postback is more reliable but
requires a publicly reachable endpoint; WebSocket is faster but can drop
messages.

| Broker | Postback | WebSocket |
|---|---|---|
| AliceBlue | Yes | Yes |
| Upstox | Yes | Yes |
| Fyers | Yes | Yes |
| Dhan | Yes | Yes |
| Zerodha | — | Yes |
| Angel One | — | Yes |

[reference:26]

The `vyapaar-adapters` package is REST-only today (gap 0004): it does not
consume postbacks or WebSocket order streams. Order-state updates come from
pull-style reads of the order book endpoints; wiring push notifications is part
of the streaming work tracked in the adapter repo's plan 0003.

## Brokerage Defaults

Vyapaar's cost model ships with per-broker brokerage defaults, current as
of the last verification date:

| Broker | Equity delivery | Equity intraday | F&O |
|---|---|---|---|
| Zerodha | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Dhan | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Angel One | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Fyers | ₹0 | ₹20 or 0.03% | ₹20 per order |
| Upstox | ₹0 | ₹20 or 0.05% | ₹20 per order |

These defaults are configurable. Verify against the broker's current
pricing page before relying on them — brokerage schedules change.

## Adapter Configuration

Adapters are constructor-injected (transport + base URL + `BrokerCredentials`)
and have an optional declarative TOML shape in `configs/`:

    # configs/dhan.toml (copy to adapter.toml and fill)
    broker = "dhan"
    paper = false

    [credentials]
    api_key = ""
    access_token = "${DHAN_ACCESS_TOKEN}"   # resolve from your secret store
    client_id = "${DHAN_CLIENT_ID}"

Credentials belong in the environment or a secret manager, never committed.
The full config model (`AdapterConfig`/`BrokerCredentials`), the template
status (3 of 7 present, loader pending — gap 0007), and the registry name
resolution are described in
[Adapter Configuration](adapter_configuration.md).

## Paper vs Live

The offline path is the deterministic **paper broker**
(`vyapaar_adapters.paper.PaperBroker`): market orders fill at the simulated
quote, limit orders on crossing, positions and bars are maintained with exact
`Decimal` arithmetic, and sessions can be recorded to JSONL fixtures and
replayed in CI. It needs no credentials and no network. Its fill model is
simple (single price, no depth/slippage) — realistic fills are improvement
gap 0006.

Live mode is the same module classes with a real `UrllibTransport` and real
credentials. Because the adapters are development/research-plane Python, live
order placement additionally requires the broker-side prerequisites
(static IP registration — see above). Engine-level paper simulation
(backtesting against the simulator) is a `vyapaar-runtime` concern, not an
adapter one.

## Rate Limits

All Indian broker APIs impose rate limits. The limits differ:

- Zerodha: 3 requests/second for order placement, 10/second for market
  data (varies by plan).
- Dhan: 25 requests/second.
- Angel One: 3 requests/second for historical data.

Rate limiting is not yet implemented in the `vyapaar-adapters` package — the
Python adapters are synchronous REST over an injected transport with no
throttling layer. Keep per-broker budgets in mind when polling; the
token-bucket rate limiter exists on the Rust production path
(`vyapaar-exec`, see
[Adapter Interface](../architecture/adapter_interface.md#33-token-bucket-rate-limiter-vyapaar-exec-ratelimit)),
and an adapter-side throttle is a candidate improvement for the parity plan.

## Adding a New Broker

New brokers are added as **modules of the single package** (not separate
distributions): subclass `BaseRestAdapter` in
`vyapaar_adapters/<broker>.py`, translate wire→canonical with exact
`Decimal`s, add a `tests/test_<broker>.py` contract test against
`MockTransport`, register under the `vyapaar.adapters` entry point **and**
`BROKER_ADAPTERS`, and add `configs/<broker>.toml`.

The full walkthrough with the checklist is in
[Adding Adapters](../contributing/adding_adapters.md); the shipped modules
(`dhan.py`, `zerodha.py`) are the templates.

## Source of Truth

- Broker API documentation (linked from each broker's developer portal).
- SEBI circular on static IP requirements for API trading.
- Broker pricing pages for brokerage schedules.
