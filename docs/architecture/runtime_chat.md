# Runtime Chat Agent (`/ws/chat`)

> **[STATUS: IMPLEMENTED]** — `vyapaar-runtime-api-rest` chat runtime (experimental). See [Platform Conformance Matrix](../conformance_matrix.md).

The REST server hosts an **LLM chat agent** that can answer market questions with real platform data behind `/ws/chat`. It is the first operator-facing AI surface inside `vyapaar-runtime` itself (distinct from `vyapaar-agent`'s autonomous research loop and MCP server, which are research-time components — see [AI Engine & Quant Researcher](ai_engine.md)).

## 1. What it is

`chat.rs` builds a `ChatRuntime` from environment and reuses the `vyapaar-agent` chat stack — `ChatAgent`, `SessionStore`, `ToolRegistry` — over a WebSocket. The agent is **read-only by construction**: the registered tools are the read-only catalog plus the alert tools; it cannot place orders, cancel anything, or mutate the trading state.

```text
Browser / client ──WS──▶ /ws/chat ──▶ ChatRuntime ──▶ ChatAgent (vyapaar-agent)
                                          │
                            ┌─────────────┼───────────────────┐
                            ▼             ▼                   ▼
                    FeedMarketReader  ProfileAlertStore   FeedContentReader
                    (quote/depth/bars)(persisted on Profile)(news/filings stubs)
```

## 2. LLM backend selection

`ChatRuntime::from_env` resolves the model backend from environment variables, in priority order:

| Priority | Variable(s) | Backend |
| :--- | :--- | :--- |
| 1 | `VYAPAAR_GGUF_PATH` | Local Candle-GGUF inference (no network) |
| 2 | `VYAPAAR_LLM_BASE_URL`, `VYAPAAR_LLM_MODEL`, `VYAPAAR_LLM_API_KEY` | OpenAI/Ollama-compatible HTTP endpoint |
| — | neither set | The socket answers with a clear `chat not configured` message instead of guessing |

The local GGUF path exists so a deployment can keep prompts and data entirely on-host.

## 3. Tools & data readers

* **Market reader** (`chat_market.rs` → `FeedMarketReader`): quote, depth, bars (capped at 500 bars), instruments, search — served from the same in-process catalog the REST routes read. Seed mutators exist for deterministic tests (`tests/chat_market_reader_tests.rs`).
* **Alerts** (`chat_alerts.rs` → `ProfileAlertStore`): the agent can create and list price/rule alerts; they persist on the user's `Profile.alerts`, so they survive restarts with the profile store.
* **Content readers** (`chat_content.rs` → `FeedContentReader`): news and filings are **stub-backed approximations** and cite their owning gaps in code — news → `vyapaar-data/docs/gaps/0006`, filings → `vyapaar-data/docs/gaps/0007` (block deals → `0005`, options chain → `vyapaar-market/docs/gaps/0001`, corporate actions → `0008`).

## 4. Sessions & limits

* Sessions are held in an `InMemorySessionStore` — **a server restart loses chat history**. Durable session persistence is open gap `vyapaar-runtime/docs/gaps/0006`.
* One user profile per server (single-tenant `default` user); profile store caps apply (2 MiB profile, 100 watchlists).

## 5. Quick start

```bash
# Option A — local GGUF model
export VYAPAAR_GGUF_PATH=/models/qwen2.5-7b-instruct-q4.gguf

# Option B — OpenAI/Ollama-compatible endpoint
export VYAPAAR_LLM_BASE_URL=http://localhost:11434/v1
export VYAPAAR_LLM_MODEL=qwen2.5:7b
export VYAPAAR_LLM_API_KEY=optional-for-local

vyapaar serve --data-dir ./bars
# then connect to ws://127.0.0.1:8000/ws/chat
```

Example client frame:

```json
{"type": "message", "content": "What is the latest quote for RELIANCE?"}
```

The reply arrives as streamed assistant messages on the same socket; tool calls (quote lookups, alert creation) are logged in the run/audit trail.

## 6. Invariants

* **Read-only tool surface** — the registry only exposes read-only catalog tools plus alert management; there is no order-placing tool to misuse.
* **Honest stubs** — unavailable data (news, filings, options chain) reports `unavailable` naming the owning gap doc, never fabricated content.
* **Env-gated** — chat stays inert until an LLM backend is explicitly configured.

## Related documents

* [REST API](../api/rest.md) — the `/ws/chat` row and streaming section.
* [AI Engine & Quant Researcher](ai_engine.md) — the research-time agent stack (`vyapaar-agent`), MCP gateway.
* [Actor Runtime & Event Loop](event_loop.md) — the live pipeline the chat readers observe.
