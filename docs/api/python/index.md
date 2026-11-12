# Vyapaar Python API Reference

The Vyapaar Labs Python ecosystem provides high-productivity quantitative research and strategy development libraries integrated with the high-performance Rust actor engine.

---

## 1. Core Python Packages

| Package | Repository | Description |
| :--- | :--- | :--- |
| `vyapaar_strategies` | [`vyapaar-strategies`](https://github.com/vyapaar-labs/vyapaar-strategies) | Quantitative strategy authoring SDK, indicator library (SMA, EMA, RSI, ATR, MACD), signal generation, portfolio sizing, and step chaining. |
| `vyapaar_adapters` | [`vyapaar-adapters`](https://github.com/vyapaar-labs/vyapaar-adapters) | Broker ACL SDK: one package, 7 broker modules (Dhan, Zerodha, Angel One, Upstox, Fyers, Kotak, IIFL), canonical `Decimal` models over injected REST transports, deterministic paper broker + JSONL fixture replay. See [Adapters Package](../adapters-package.md). |
| `vyapaar_data` | [`vyapaar-data`](https://github.com/vyapaar-labs/vyapaar-data) | Historical Parquet lakehouse data catalogs, partition management, point-in-time universe resolvers, and tick storage. |
| `vyapaar_examples` | [`vyapaar-examples`](https://github.com/vyapaar-labs/vyapaar-examples) | Production-ready reference strategies: NIFTY/BankNIFTY options selling, equity momentum, and statistical arbitrage. |
| `vyapaar_ffi` | [`vyapaar-ffi`](https://github.com/vyapaar-labs/vyapaar-ffi) | PyO3 native C-extension providing zero-copy memory bridges, PyArrow record batch transfers, and Tokio actor runtime lifecycle management. |

---

## 2. Strategy Authoring Quickstart

```python
from vyapaar_strategies import Strategy, Timeframe
from vyapaar_strategies.indicators import Sma, Rsi
import rust_decimal

class DualMovingAverageStrategy(Strategy):
    def __init__(self, fast_period: int = 10, slow_period: int = 30):
        super().__init__()
        self.fast_sma = Sma(period=fast_period)
        self.slow_sma = Sma(period=slow_period)

    def on_bar(self, bar):
        fast = self.fast_sma.update(bar.close)
        slow = self.slow_sma.update(bar.close)

        if fast > slow and not self.has_position(bar.instrument_id):
            self.buy(bar.instrument_id, quantity=100)
        elif fast < slow and self.has_position(bar.instrument_id):
            self.close_position(bar.instrument_id)
```

---

## 3. High-Performance Client & In-Process Router

For interacting with in-process datasets or serving backtest workloads:

```python
from vyapaar.client import Client

# Connect to in-process Parquet lakehouse catalog
client = Client.inproc(data_dir="./bars")

# Query screener scan across NIFTY 50 universe
results = client.screener_scan(universe=["TCS.NSE", "INFY.NSE", "RELIANCE.NSE"], tf="1m")
for row in results:
    print(row.instrument_id, row.metrics)
```
