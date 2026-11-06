# `vyapaar-docs` — Inter-Agent Handoff

## Phase / Slice Status
* **Tier 6E (COMPLETE)**: Documentation site migrated and consolidated in an
  earlier pass; this handoff closes out Tier 6.
* **Fincept/Quantcept Layer 6 (COMPLETE)**: `docs/plans/0002` implemented —
  `docs/ai/fincept-vs-quantcept.md` (Fincept = quantitative MCP toolset,
  Quantcept = chat UX; mapping table of which product grounded which Vyapaar
  design) and `docs/ai/omniroute_setup.md` (local OmniRoute, free-tier-only
  models, `VYAPAAR_LLM_BASE_URL`/`VYAPAAR_LLM_MODEL`/`VYAPAAR_LLM_API_KEY`
  wiring, `FreeTierPolicy` Off/Warn/Enforce guardrail, `--ignored` workload
  integration tests). Both pages linked from `docs/ai/index.md` (which also now
  links the two primary-research docs) and registered in the `mkdocs.yml` nav.
  Gate: `mkdocs build --strict` PASS.

## Decisions & Discoveries
* `mkdocs build --strict` passes with 0 errors/warnings (verified locally with
  mkdocs 1.6.1 + mkdocs-material).
* OmniRoute env-variable names and the free-tier allowlist were verified
  against `vyapaar-agent-llm/src/config.rs` and `openai_compat.rs` (the docs
  quote the real constants, not placeholders).

## Immediate Next Step
* None in this repo. The Fincept/Quantcept deferred set is now fully resolved:
  terminology + OmniRoute guide here, examples in `vyapaar-examples`;
  `vyapaar-data/0002` (filings) remains Level 7, gated on a concrete consumer.

## Operating Invariants
* `mkdocs build --strict` must stay green; docs code snippets must match tested
  code in `vyapaar-examples` / `vyapaar-strategies`.
