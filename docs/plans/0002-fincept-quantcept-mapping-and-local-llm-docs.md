# 0002 — Documenting the Fincept/Quantcept Mapping and the Free-Tier LLM Backend

> **Status**: Proposed — not yet implemented.
> **Target**: `vyapaar-docs`.
> **Depends on reading**:
> - `vyapaar-docs/docs/plans/0001-mcp-gateway-and-quantitative-toolset-reference.md`
> - `docs/future-work-bottoms-up-build-order.md` (relative to the `vyapaar-labs` workspace root —
>   the index into every per-repo `docs/plans/*.md` this round of planning produced; the original
>   scratch research — a mapping doc and a deep-research report — was migrated into those per-repo
>   docs and removed from the repo)
> - `vyapaar-agent/docs/plans/0005-local-slm-model-selection-and-reliability-reference.md` (the
>   local-SLM licensing/fine-tuning/quantization/determinism detail, successor to the removed
>   report)

---

## 0. Executive Summary & Objective

This is a short, deferred-work note for whoever maintains `vyapaar-docs` next, recording two
things worth publishing once the underlying work lands: a terminology clarification between two
external products this workspace has been compared against, and a setup guide that will be needed
once the free-tier LLM backend plan ships.

---

## 1. Context

**Terminology note (to resolve in published docs).** Fincept (`docs.fincept.in`) and Quantcept
(`quantcept.io`) are two separate products. `0001` in this repo's own plans was scoped against
Fincept's MCP toolset specifically. `vyapaar-ui`'s chat-UX research
(`vyapaar-ui/docs/plans/0002-quantcept-web-ux-reference.md`) was scoped against Quantcept's live
chat product. These should not be conflated in published docs — a reader of `vyapaar-docs` should
be able to tell at a glance which product grounded which design decision.

**Free-tier LLM setup guide (not written yet).** Once
`vyapaar-agent/docs/plans/0004-free-tier-llm-backend-and-deferred-local-slm.md`'s OmniRoute-backed
free-tier LLM setup actually ships, `vyapaar-docs` will need a setup guide: how to run OmniRoute
locally, configure it for free-tier-only models, and point `vyapaar-agent-llm` at it. That guide
is flagged here as needed, not written.

---

## 2. Component Specifications

None — this doc records two deferred documentation items and links their source material; it does
not itself specify new doc structure beyond what `0001` already laid out.

---

## 3. Implementation Phases

### Phase 1 (deferred)
Write the OmniRoute setup guide once `vyapaar-agent/docs/plans/0004`'s Phase 2 actually ships.
Until then, link the two source documents above —
`docs/future-work-bottoms-up-build-order.md` (workspace root) and
`vyapaar-agent/docs/plans/0005-local-slm-model-selection-and-reliability-reference.md` — from this
repo's `docs/ai/` index so a reader can find the primary research.
