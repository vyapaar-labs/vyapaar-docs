# Running OmniRoute Locally (Free-Tier LLM Backend)

Vyapaar's AI workloads — the trader loop, chat, research, and report narration —
speak OpenAI-compatible HTTP through `vyapaar-agent-llm`. The supported free-tier
backend is **OmniRoute**, a local router that fronts several free-model providers
behind one endpoint. OmniRoute is the project owner's own tool and is **not
vendored** into Vyapaar; you run it yourself.

This guide covers Phase 2 of `vyapaar-agent/docs/plans/0004`:
run OmniRoute locally, restrict it to free-tier-only models, and point
`vyapaar-agent-llm` at it. No Vyapaar code changes are needed — environment
variables are the entire wiring.

---

## 1. Start OmniRoute

OmniRoute runs locally via npx:

```bash
npx omniroute
```

Configure it to **prioritize free-tier models** across its provider list and
**reject paid ones by default**. Follow OmniRoute's own configuration for your
provider keys; the important setting for Vyapaar is that the router only ever
routes to models on Vyapaar's free-tier allowlist (see §3).

Once it is up, OmniRoute exposes an OpenAI-compatible endpoint root such as
`http://localhost:8080/v1`. Leave it running for the session.

---

## 2. Point `vyapaar-agent-llm` at it

`vyapaar-agent-llm` reads its provider configuration from three environment
variables:

| Variable | Required | Meaning |
| :--- | :--- | :--- |
| `VYAPAAR_LLM_BASE_URL` | yes | OpenAI-compatible endpoint root, e.g. `http://localhost:8080/v1` |
| `VYAPAAR_LLM_MODEL` | yes | Model name to request (must be on the free-tier allowlist, §3) |
| `VYAPAAR_LLM_API_KEY` | no | Bearer token if the endpoint requires one |

Example:

```bash
export VYAPAAR_LLM_BASE_URL="http://localhost:8080/v1"
export VYAPAAR_LLM_MODEL="gpt-4o-mini"
export VYAPAAR_LLM_API_KEY=""   # omit if OmniRoute needs no key
```

The four workloads pick this up with no code changes:

- **Trader loop** (`vyapaar-agent-trader-loop`) — prompt → generate →
  `DecisionParser` validation → risk gate.
- **Chat** (`vyapaar-agent-chat`) — the turn loop and tool dispatch.
- **Research** (`vyapaar-agent-research`) — the `llm` optional feature.
- **Report** (`vyapaar-agent-report`) — tearsheet narrative text.

## 3. The free-tier guardrail

`OpenAiCompatibleProvider` validates every reported response model against a
**free-tier allowlist** (lowercase-substring match, so router-qualified names
like `openai/gpt-4o-mini` are recognized). The guardrail policy is selectable:

- `Off` — any reported model is accepted (default).
- `Warn` — log a warning for a non-free-tier model, still proceed.
- `Enforce` — reject the response outright (`NonFreeTierModel`).

Set the policy in the provider construction site for each workload. For
production use `Enforce` so a misconfigured router can never silently upgrade
you to a paid model.

> **Phase 3 note:** while routing through OmniRoute, run each workload's
> `#[ignore]`-marked integration test once against the live endpoint. These
> tests are ignored by default (`cargo test -- --ignored`) because they need a
> running router; they confirm end-to-end wiring and accumulate the golden
> (prompt → `Decision`) / (prompt → tool-call) fixtures that a future
> embedded local model will fine-tune against.

---

## 4. Verify

```bash
# Trader-loop workload against the live endpoint:
cargo test -p vyapaar-agent-trader-loop -- --ignored
```

If `VYAPAAR_LLM_BASE_URL` / `VYAPAAR_LLM_MODEL` are missing or empty, the
provider loader fails fast with a transport error naming the missing variable.
