# Fincept vs. Quantcept: Which Product Grounded Which Design

Two external products that Vyapaar has been compared against are often conflated
in passing conversation. They are **separate products**, and each grounded a
*different* part of Vyapaar's design:

| Product | What it is | Where Vyapaar referenced it |
| :--- | :--- | :--- |
| **Fincept** | A research platform exposing a quantitative **MCP toolset** (analytics, screener, valuation, filing data). | `vyapaar-docs/docs/plans/0001` and its sibling per-repo plans (`vyapaar-market`, `vyapaar-ledger`, `vyapaar-agent`, …) — the Fincept/Quantcept **gap-closing initiative** (see the build order below). |
| **Quantcept** | A live, chat-first **AI terminal** product. | `vyapaar-ui/docs/plans/0002-quantcept-web-ux-reference.md` — the chat-UX research behind the Vyapaar terminal's chat experience. |

## Why the distinction matters

- `vyapaar-docs/docs/plans/0001` (MCP gateway & quantitative toolset reference)
  is scoped against **Fincept's MCP toolset specifically**.
- `vyapaar-ui/docs/plans/0002-quantcept-web-ux-reference.md` is scoped against
  **Quantcept's chat product** (personas, "your instructions", unified chat
  landing pages).
- The gap-closing **build order** lives in `docs/future-work-bottoms-up-build-order.md`
  at the workspace root — it is the index into every per-repo `docs/plans/*.md`
  produced by the initiative, and covers **both** products' feature surfaces.

A reader of this documentation should be able to tell at a glance which
product grounded which design decision. The rule of thumb:

> **Fincept → quantitative tools. Quantcept → chat UX.**

## Mapping

| Vyapaar design decision | Grounded by |
| :--- | :--- |
| 15 quantitative engines + MCP gateway (`vyapaar-agent`) | Fincept toolset |
| Indicators breadth, derivatives pricing (`vyapaar-market`) | Fincept toolset |
| Portfolio allocation & optimization (`vyapaar-ledger`) | Fincept toolset |
| Screener / market-data read ports (`vyapaar-data`) | Fincept toolset |
| Terminal chat experience, personas, chat list/portfolio sync (`vyapaar-ui`) | Quantcept chat product |
