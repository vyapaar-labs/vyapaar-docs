# Installation

Vyapaar requires three toolchains: **Rust** (for the engine), **Python 3.10+**
(for the control plane and strategy API), and optionally **Node.js 18+** (only
if you want to run the frontend dashboard).

## Prerequisites

### Rust

Install via [rustup](https://rustup.rs):

    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    rustup component add rustfmt clippy

Vyapaar pins `rust-version = "1.75"` in the workspace, so any newer stable
toolchain works.

### Python

Python 3.10 or later. We recommend [`uv`](https://github.com/astral-sh/uv) for
fast, reproducible environments:

    curl -LsSf https://astral.sh/uv/install.sh | sh

Or use your system Python with `venv`:

    python3 -m venv .venv
    source .venv/bin/activate

### System Packages

On Debian/Ubuntu:

    sudo apt install build-essential pkg-config libssl-dev

On macOS with Homebrew:

    brew install pkg-config openssl

## Clone and Build

    git clone https://github.com/vyapaar-labs/vyapaar.git
    cd vyapaar

### Build the Rust workspace

    cargo build --workspace

The first build downloads Arrow, Parquet, Tokio, and their dependencies.
Expect 3–5 minutes on a modern machine.

### Verify the dependency hierarchy

The workspace enforces a strict crate layering. This script fails if any crate
imports from a layer above it:

    python3 scripts/dependency_graph.py
    # → Dependency hierarchy OK.

### Build the Python extension

The Python control plane links to the Rust core via PyO3:

    cd python
    pip install -e ".[dev]"
    cd ..

Verify:

    vyapaar version
    # → vyapaar 0.1.0

## Install Broker Adapters

Adapters live in a separate repository — one package, one module per broker —
so the core engine stays broker-agnostic:

    git clone https://github.com/vyapaar-labs/vyapaar-adapters.git
    cd vyapaar-adapters

    pip install -e .          # installs vyapaar_adapters with all seven broker modules

All seven brokers (Dhan, Zerodha, Angel One, Upstox, Fyers, Kotak, IIFL) ship in
the single package; Dhan and Zerodha are the contract-certified ones. The
deterministic paper broker (`PaperBroker`) needs no credentials and no network —
you don't need live broker credentials to run offline tests. See
[Adapters Package](../api/adapters-package.md) and
[Adapter Configuration](../india/adapter_configuration.md).

## Configuration

Copy the example environment file and fill in what you need:

    cp .env.example .env

Minimal `.env` for backtesting only:

    # Leave blank — no broker credentials required for backtests
    DHAN_CLIENT_ID=
    DHAN_ACCESS_TOKEN=
    DHAN_MODE=paper          # build adapters in paper mode (default is live)

For LLM-driven research (optional):

    OPENAI_API_KEY=sk-...
    ANTHROPIC_API_KEY=sk-ant-...

## Verify the Whole Stack

    # Rust tests
    cargo test --workspace

    # Python tests
    pytest python/tests/

    # Run a minimal backtest end-to-end
    python3 examples/backtesting/01_first_backtest.py

If all three commands succeed, you're ready to write your first strategy.

## Common Issues

**`E0034: multiple applicable items in scope` from `arrow-arith`**

This is a chrono/arrow version conflict. Pin chrono below 0.4.40 in the
workspace `Cargo.toml`:

    chrono = { version = ">=0.4.20, <0.4.40", features = ["serde"] }

Then:

    cargo update -p chrono --precise 0.4.39

**`E0583: file not found for module`**

The crate's `lib.rs` declares a module that has no file on disk. Create a
stub for it:

    touch crates/<crate>/src/<module>.rs

**`maturin` fails to find Python headers**

Install the development headers for your Python version. On Debian/Ubuntu:

    sudo apt install python3-dev
