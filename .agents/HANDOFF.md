# `vyapaar-docs` — Inter-Agent Handoff

## Phase / Slice Status
* **Tier 6E (COMPLETE)**: Documentation site migrated and consolidated in an
  earlier pass; this handoff closes out Tier 6.
* `docs/` contains the full Vyapaar knowledge base (architecture, crate
  hierarchy, message flow, DDD, Python-first, money/costs, strategies,
  backtesting, AI/quant research, tutorials, API reference, conformance matrix,
  roadmap, migration guide) with `honba-*` terminology fully renamed to
  `vyapaar-*`.
* `mkdocs.yml` configures the Material theme (slate/indigo), admonition,
  `pymdownx.superfences`/`highlight`/`snippets`, and a complete navigation tree.
* Tooling: `Makefile` (`bootstrap/build/serve/clean`), `requirements.txt` +
  `requirements-dev.txt`, and `scripts/` helpers (no mdBook remnants).
* CI: `.github/workflows/ci.yml` added — `mkdocs build --strict` gate.

## Decisions & Discoveries
* `mkdocs build --strict` passes with 0 errors/warnings (verified locally with
  mkdocs 1.6.1 + mkdocs-material 9.5.26).

## Immediate Next Step
* Root superproject: update submodule pointers for all five Tier 6 repos and
  mark Tier 6 complete in `docs/migration-guide.md`.

## Operating Invariants
* `mkdocs build --strict` must stay green; docs code snippets must match tested
  code in `vyapaar-examples` / `vyapaar-strategies`.
