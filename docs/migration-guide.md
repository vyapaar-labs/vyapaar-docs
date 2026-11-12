# `vyapaar-docs` — Subagent Migration & DDD Refactoring Guide

> **Tier**: Tier 6E
> **Source**: `honba-workspace/honba-docs`
> **Target**: `vyapaar-docs`

---

## 1. File & Directory Migration Map

| Source Path (`honba-docs/`) | Target Path (`vyapaar-docs/`) | Migration Action & Notes |
| :--- | :--- | :--- |
| `docs/*` | `docs/*` | Copy all directory trees (`ai/`, `api/`, `architecture/`, `backtesting/`, `examples/`, `getting-started/`, `india/`, `strategies/`, `tutorials/`, `conformance_matrix.md`). Retain target's `design/` and `roadmap/`. |
| `book/src/architecture/crate_hierarchy.md` | `docs/architecture/crate_hierarchy.md` | Port from mdBook into `docs/architecture/`, updating to Vyapaar crate names. |
| `book/src/architecture/message_flow.md` | `docs/architecture/message_flow.md` | Port from mdBook into `docs/architecture/`, updating message flow diagrams to Vyapaar architecture. |
| `book/src/backtesting/avoiding_backtest_biases.md` | `docs/backtesting/avoiding_backtest_biases.md` | Port Tucker Balch's 10 biases guide into `docs/backtesting/`. |
| `book/src/nl-verification.md` | `docs/ai/nl_verification.md` | Replace stub page with full NL-to-strategy documentation from mdBook. |
| `book/src/contributing/*` | `docs/contributing/*` | Port contribution guides (`index.md`, `development_setup.md`, `adding_adapters.md`, `adding_indicators.md`, `dependency_rules.md`). |
| `book/src/cost-model.md` | `docs/india/cost_model.md` | Integrate component breakdowns and statutory Indian transaction tax tables. |
| `book/src/market-calendar.md` | `docs/india/market_calendar.md` | Integrate session windows and auction schedules. |
| `book/` | *(Excluded)* | Eliminate duplicate mdBook tree entirely. |
| `requirements.txt` | `requirements.txt` | Port doc build requirements (`mkdocs`, `mkdocs-material`, `mkdocs-redirects`, `pymdown-extensions`). |
| `requirements-dev.txt` | `requirements-dev.txt` | Port dev doc requirements. |
| `Makefile` | `Makefile` | Update targets for MkDocs single-doc-site build and dev server. |
| `scripts/` | `scripts/` | Port `bootstrap.sh`, `build.sh`, `clean.sh`, `_common.sh`, `serve.sh`, removing mdBook logic. |
| `mkdocs.yml` | `mkdocs.yml` | Update configuration, navigation tree, Material theme settings, and extensions. |

---

## 2. Phased Execution Plan

### Phase 1: Environment & Tooling Setup
1. Create isolated Python 3.13 virtual environment (`.venv`) via `uv venv --python 3.13`.
2. Port `requirements.txt` and `requirements-dev.txt`, removing any pinned dependencies that conflict with modern tooling.
3. Install documentation dependencies (`mkdocs`, `mkdocs-material`, `mkdocs-redirects`, `pymdown-extensions`, `mkdocs-autorefs`, `mkdocstrings`).
4. Port helper scripts in `scripts/` (`_common.sh`, `build.sh`, `serve.sh`, `clean.sh`, `bootstrap.sh`) and `Makefile`, removing all `mdbook` build paths.

### Phase 2: Documentation Asset Migration & mdBook Consolidation
1. Synchronize all subdirectories from `honba-docs/docs/*` to `vyapaar-docs/docs/`, preserving `design/`, `roadmap/`, and `migration-guide.md`.
2. Transfer unique pages from `honba-docs/book/src/`:
   - `book/src/architecture/crate_hierarchy.md` → `docs/architecture/crate_hierarchy.md`
   - `book/src/architecture/message_flow.md` → `docs/architecture/message_flow.md`
   - `book/src/backtesting/avoiding_backtest_biases.md` → `docs/backtesting/avoiding_backtest_biases.md`
   - `book/src/nl-verification.md` → `docs/ai/nl_verification.md` (overwriting placeholder stub)
   - `book/src/contributing/*` → `docs/contributing/*`
3. Merge detailed reference sections from `book/src/cost-model.md` and `book/src/market-calendar.md` into `docs/india/cost_model.md` and `docs/india/market_calendar.md`.

### Phase 3: Subsystem Terminology & DDD Refactoring
Execute automated and manual refactoring across all markdown files in `docs/`:
- **Subsystem & Crate Names**:
  - `honba-core` → `vyapaar-core` (`honba_core` → `vyapaar_core`)
  - `honba-kokoro` → `vyapaar-ledger` (`honba_kokoro` → `vyapaar_ledger`)
  - `honba-mon` → `vyapaar-risk` (`honba_mon` → `vyapaar_risk`)
  - `honba-me` → `vyapaar-market` (`honba_me` → `vyapaar_market`)
  - `honba-te` → `vyapaar-exec` (`honba_te` → `vyapaar_exec`)
  - `honba-ya` → `vyapaar-runtime` (`honba_ya` → `vyapaar_runtime`)
  - `honba-nou` → `vyapaar-agent` (`honba_nou` → `vyapaar_agent`)
  - `honba-tamashii` → `vyapaar-ffi` (`honba_tamashii` → `vyapaar_ffi`)
  - `honba-kao` → `vyapaar-ui` (`honba_kao` → `vyapaar_ui`)
  - `honba-fe` → `vyapaar-fe` (`honba_fe` → `vyapaar_fe`)
  - `honba-data` → `vyapaar-data` (`honba_data` → `vyapaar_data`)
  - `honba-adapters` → `vyapaar-adapters` (`honba_adapters` → `vyapaar_adapters`)
  - `honba-strategies` → `vyapaar-strategies` (`honba_strategies` → `vyapaar_strategies`)
  - `honba-examples` → `vyapaar-examples` (`honba_examples` → `vyapaar_examples`)
  - `honba-docs` → `vyapaar-docs` (`honba_docs` → `vyapaar_docs`)
- **Python Imports & Namespaces**:
  - `import honba` → `import vyapaar`
  - `from honba.` → `from vyapaar.`
- **Organization & Branding**:
  - `honba-labs` → `vyapaar-labs`
  - "Honba" → "Vyapaar" (preserving contextual historical design origins where appropriate)

### Phase 4: MkDocs Configuration & Navigation Restructuring
1. Rewrite `mkdocs.yml`:
   - Set `site_name: Vyapaar Labs Documentation`
   - Set `site_url: https://docs.vyapaar-labs.io`
   - Configure Material theme palette (slate / indigo), copy buttons, section indexes.
   - Configure markdown extensions (`pymdownx.superfences`, `pymdownx.highlight`, `admonition`, `tables`, `footnotes`, `attr_list`, `toc`).
2. Construct comprehensive navigation tree incorporating:
   - Home (`index.md`)
   - Architecture & Design (including `design/architecture.md`, crate hierarchy, message flow, DDD patterns)
   - Getting Started (installation, first backtest, loading data)
   - Indian Markets (brokers, adapters, options, mutual funds, universes, cost model, calendar)
   - Strategies (writing strategies, indicators, styles, portfolio construction, chaining)
   - Backtesting (three-tier pipeline, cost & latency modeling, statistical validation, avoiding biases)
   - AI & Quant Research (research loop, RL training, MCP gateway, NL verification)
   - Tutorials (NIFTY50, BankNIFTY options, SIP, Alpha30 factor model, autoresearch)
   - Examples & Case Studies
   - API Reference
   - Contributing (setup, adding adapters, adding indicators, dependency rules)
   - Conformance Matrix & Roadmap (`conformance_matrix.md`, `roadmap/roadmap.md`, `migration-guide.md`)

### Phase 5: Link Sanitization & Strict Build Verification
1. Fix relative link anomalies (e.g. `../architecture/` links in root `conformance_matrix.md`, and broken links to `conformance_matrix.md` in `architecture/*.md`).
2. Remove stale dead external links (e.g. dead GitHub Pages mdBook external links).
3. Validate and execute strict build:
   ```bash
   mkdocs build --strict
   ```
   Ensure build finishes with 0 errors and 0 warnings.

---

## 3. Strict Verification Commands
```bash
# Run from the repository root
source .venv/bin/activate
mkdocs build --strict
```
