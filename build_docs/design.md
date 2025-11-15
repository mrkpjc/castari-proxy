# Castari Worker Design

This document specifies a complete, production‑grade design for the Castari Cloudflare Worker that translates Anthropic Messages API requests from the Claude Agent SDK into provider‑specific APIs (initially OpenRouter) and streams Anthropic‑compatible responses back to the client. It covers architecture, HTTP behavior, request/response translation, streaming, tool handling and orchestration, reasoning, web search, MCP bridging, security, observability, configuration, error handling, performance, testing, phased delivery, and language‑agnostic compatibility.

## 1. Goals and Non‑Goals

### 1.1 Goals

- Maintain feature parity for Claude Agent SDK clients when using non‑Anthropic models.
- Accept Anthropic Messages requests and route to either Anthropic (pass‑through) or OpenRouter (translated) via Cloudflare AI Gateway.
- Preserve SDK semantics for tools, MCP, multi‑turn, streaming, images, system prompts, usage, and stop reasons.
- Deliver robust SSE streaming without breaking `includePartialMessages` behavior in the SDK.
- Enforce clear policies for Anthropic server tools (error, enforce Anthropic, or emulate via backends).
- Provide configuration that is language‑agnostic and works with Node, Python, and future SDK wrappers.

### 1.2 Non‑Goals (v1)

- Full emulation of Anthropic server tools (CodeExecution/ComputerUse/TextEditor) by default. We provide a policy knob; emulation can be implemented later via hardened backends.
- STDIO MCP connectivity in the Worker (not supported in Cloudflare Workers). Only HTTP/SSE MCP is considered for bridging.

## 2. High‑Level Architecture

```
Claude Agent SDK (any language) ──HTTP──> Castari Worker (/v1/messages)
                                              │
                                              ├─ Provider = anthropic → CF AI Gateway → Anthropic Messages API
                                              │
                                              └─ Provider = openrouter → CF AI Gateway → OpenRouter Chat Completions

                                   (Stream bridge translates deltas <→ Anthropic SSE events)
                                   (Tool loop orchestrator optionally executes tools / re‑queries provider)
```

### 2.1 Major Components

- Router: Identifies provider and dispatches to upstream.
- Translator (Anthropic → Provider): Maps request schema, roles, content blocks, tools, reasoning, and web search into the provider request.
- Translator (Provider → Anthropic): Converts provider responses (non‑stream and stream) into Anthropic‑shaped responses and SSE events.
- Stream Bridge: Reads upstream SSE, emits Anthropic SSE with correct event taxonomy and ordering.
- Tool Orchestrator: Detects tool calls, applies policy, optionally executes via backends (MCP HTTP/SSE), and re‑queries provider with tool results.
- Security Filters: Validate inputs, enforce size limits, secrets handling, and allowlists.
- Observability: Structured logs, metrics, and request tracing.
- Config: Environment‑driven behavior and feature flags; compatible with any client language.

## 3. HTTP Interface (Language‑Agnostic)

### 3.1 Inbound (Client → Worker)

- Method: `POST`
- Path: `/v1/messages`
- Headers (accepted):
  - `content-type: application/json` (required)
  - `accept: application/json` or `text/event-stream` (streaming)
  - `x-api-key: <token>` or `authorization: Bearer <token>` (client SDKs vary; Worker normalizes)
  - Optional routing/metadata headers (set by wrappers when available):
    - `x-castari-provider: anthropic | openrouter`
    - `x-castari-model: <original model string>`
    - `x-castari-wire-model: <provider/vendor slug>`
    - `x-client-meta: { ... }` (JSON string: sdk/app/clientId/deployment/version/model/provider/reasoning prefs)
    - `x-worker-token: <shared-secret>` (optional Worker AuthN)

Notes:
- The Worker must also operate without any `x-castari-*` headers. It should infer provider from the request body (`model`) and default behaviors.
- To ensure language agnosticism, the Worker does not rely on node‑specific behaviors; it only consumes standard HTTP + JSON.

### 3.2 Outbound (Worker → Upstreams)

- Anthropic (via CF AI Gateway): `POST {UPSTREAM_ANTHROPIC_BASE_URL}/v1/messages`
  - Headers:
    - `content-type: application/json`
    - `x-api-key: <anthropic_key>`
    - `anthropic-version: <pinned version>` (optional, configurable)

- OpenRouter (via CF AI Gateway): `POST {UPSTREAM_OPENROUTER_BASE_URL}/v1/chat/completions`
  - Headers:
    - `content-type: application/json`
    - `authorization: Bearer <openrouter_key>`
    - Optional OpenRouter headers (e.g., `HTTP-Referer`, `X-Title`) if configured.

Header Normalization:
- If the client sent `x-api-key`, the Worker re‑maps it as needed per upstream.
- If the client sent `authorization: Bearer`, the Worker passes it through or converts to `x-api-key` for Anthropic.

## 4. Provider and Model Detection

### 4.1 Determination Order
1) Header `x-castari-provider` if present.
2) Otherwise, infer from `body.model` using prefixes:
   - Anthropic: `claude-*` or `anthropic/<slug>`
   - OpenRouter: `or:<slug>` | `openrouter/<vendor>/<slug>` | `openai/<slug>`
3) Fallback: `anthropic` (for strict backward compatibility).

### 4.2 Wire Model Mapping (OpenRouter)

- `or:<slug>`
  - If `<slug>` contains `/` → treat as `<vendor>/<model>` (e.g., `or:google/gemini-2.0`).
  - Else → prefix `${OPENROUTER_DEFAULT_VENDOR}/${slug}` (default vendor: `openai`).
- `openrouter/<vendor>/<slug>` → `<vendor>/<slug>`
- `openai/<slug>` → unchanged

## 5. Request Translation (Anthropic → Provider)

### 5.1 Common Fields

- `model` → provider model (wire model for OpenRouter)
- `system` → prepend a system message for providers that use message role `system`
- `messages` → map each turn:
  - Roles: `user` ↔ `user`, `assistant` ↔ `assistant`
  - Assistant prefill (assistant last): pass through to provider as partial assistant message
  - Tool results (Anthropic `tool_result` blocks are nested under a `user` message) → emit a separate provider message `{ role: 'tool', tool_call_id, content }`
- Sampling: `max_tokens`, `temperature`, `top_p` pass through; `stop_sequences` → provider `stop`
- `stream` → provider streaming flag
- `metadata` → preserved; the Worker reads `metadata.castari.*` (reasoning, web configs) but otherwise forwards opaque keys when possible

### 5.2 Content Blocks

- Text: `{ type: 'text', text }` → message `content: string`
- Images:
  - URL source: `{ type: 'image', source: { type: 'url', url } }` → `{ type: 'image_url', image_url: { url } }`
  - Base64 source: require data URI form `data:<mime>;base64,<data>`; enforce size caps

### 5.3 Tools (Client/Function Tools)

- Anthropic client tool definition `{ name, description, input_schema }` → OpenAI function tool `{ type: 'function', function: { name, description, parameters: input_schema } }`
- `tool_choice` mapping:
  - `auto` → `auto`
  - `none` → `none`
  - `tool{name}` → `{ type: 'function', function: { name } }`
  - `any` (Anthropic) → `auto`

### 5.4 Web Search

- If request declares Anthropic server WebSearchTool:
  - Apply policy: `SERVER_TOOLS_MODE = error | enforceAnthropic | emulate`
  - For `emulate`: add OpenRouter plugin `[{ id: 'web', ...options }]` or use `:online` suffix. Map tunables when present (engine, max_results, search_context_size).

### 5.5 Reasoning

- Read `metadata.castari.reasoning` `{ effort?, max_tokens?, exclude? }` injected by wrappers.
- Map to OpenRouter `reasoning` object.
- If wrappers don’t inject but model supports reasoning, keep defaults.

### 5.6 MCP (Server‑Run)

- If `mcp_servers` present in request (Anthropic server MCP):
  - Policy:
    - `enforceAnthropic`: route entire request to Anthropic
    - `error`: respond 400 with guidance
    - `bridge-http-sse` (opt‑in): for HTTP/SSE MCP only, enumerate tools, convert to function tools for this turn, and run a tool loop when invoked

## 6. Response Translation (Provider → Anthropic)

### 6.1 Non‑Streaming

- Provider `message.content` → Anthropic assistant content blocks
- Provider `message.tool_calls[]` → Anthropic `tool_use` content blocks with `id`, `name`, `input`
- Usage: map `prompt_tokens`, `completion_tokens`; optionally enrich later
- Stop reasons: map provider finish to Anthropic `stop_reason` (`end_turn`, `stop_sequence`, `tool_use`, `content_filter`, `max_tokens`)

### 6.2 Streaming (SSE Bridge)

- Read upstream SSE (`text/event-stream`); OpenRouter emits `delta` chunks + occasional comments (ignore `: keep-alive`).
- Emit Anthropic SSE events with correct lifecycle:
  1) `message_start` once
  2) For each content block:
     - Text: `content_block_start` → repeated `text_delta` → `content_block_stop`
     - Tool use: `content_block_start` with `{ type: 'tool_use', id, name }`
       - Accumulate function.arguments characters and emit `input_json_delta` events progressively
       - On valid JSON closure, finalize `input` and emit `content_block_stop`
  3) `message_delta` as needed (usage or minor metadata)
  4) `message_stop` with normalized `stop_reason`

Edge Cases:
- Multiple parallel tool calls: multiplex tool use blocks by `id`; maintain deterministic order.
- Partial JSON arguments: stream `input_json_delta` raw slices; finalize when parseable; protect against unbounded growth and malformed JSON.

## 7. Tool Loop Orchestration

Purpose: when the provider returns `tool_calls`, decide how to proceed based on policy; optionally run tools and continue the turn until final text is produced.

### 7.1 Policy

- `SERVER_TOOLS_MODE`:
  - `error`: return 400 with a machine‑readable explanation listing the server tools detected
  - `enforceAnthropic`: reroute the request to Anthropic directly
  - `emulate`: execute tool calls via configured backends and then re‑query the provider with `role: tool` messages

### 7.2 Execution (Emulation)

- Backends (pluggable):
  - MCP HTTP/SSE servers (filesystem, browser, code exec) when available
  - Worker internal capabilities (restricted `web_fetch`, allowlisted HTTP GET)
  - Proxies to trusted microservices providing deterministic operations

Flow:
1) Receive provider stream → detect `tool_calls`
2) Pause outward stream for that tool call’s result, but keep emitting heartbeats
3) Execute tool → capture output
4) Send follow‑up provider call with prior assistant message (including `tool_calls`) + new `role: tool` message(s)
5) Resume stream of final assistant content

State:
- Track per‑request tool calls, arguments, results, and re‑query state in memory; optionally persist across turns in Durable Object/KV via a session key.

## 8. Security and Compliance

- Authentication: optional `x-worker-token` secret; reject unauthorized requests when enabled.
- Input validation:
  - Body size cap (e.g., 1–2 MB), image size cap, disallow excessively nested tool schemas
  - Strict JSON parsing with fail‑closed behavior
- Header handling: never forward `x-worker-token` to upstream providers; only set provider auth headers as required.
- Outbound allowlists: for emulated tools (web fetch, browser), enforce domain allowlists and rate limits.
- Sandboxing: do not execute arbitrary code in Worker; only call hardened backends.
- Logging redaction: redact `authorization`, `x-api-key`, and content bodies by default; enable debug sampling via `LOG_LEVEL`.

## 9. Observability and Metrics

- Structured logs: trace_id, provider, wire_model, latency_ms, streamed_bytes, tool_call_count, policy decisions.
- Metrics counters: requests, errors, reroutes, emulations, rate limit hits.
- Percentiles: p50/p90/p99 latency by provider and model.
- Optional export to Cloudflare Analytics Engine or Logpush.

## 10. Configuration

Environment bindings (Workers):

- `UPSTREAM_ANTHROPIC_BASE_URL` (default CF AI Gateway Anthropic path)
- `UPSTREAM_OPENROUTER_BASE_URL` (default CF AI Gateway OpenRouter path)
- `SERVER_TOOLS_MODE` = `error | enforceAnthropic | emulate` (default: `error`)
- `MCP_BRIDGE_MODE` = `off | http-sse` (default: `off`)
- `OPENROUTER_DEFAULT_VENDOR` = `openai` (default), used for `or:<slug>` without vendor
- `WORKER_TOKEN` (optional)
- `LOG_LEVEL` = `debug | info | warn | error` (default: `info`)
- `ALLOW_ORIGINS` (optional, comma-separated)
- `RUNTIME_STORE` binding (Durable Object or KV) for session/tool buffering (optional)

Client‑agnostic behavior:
- All config is carried in Worker env; clients only need to send valid Anthropic Messages requests. `x-castari-*` headers improve routing but are not required.

## 11. Error Handling and Mapping

- Normalize to Anthropic‑style errors with fields `{ type, message, provider_details?, retryable? }`.
- Map HTTP status:
  - 400 → `invalid_request_error`
  - 401 → `authentication_error`
  - 403 → `permission_error`
  - 429 → `rate_limit_error`
  - 5xx → `api_error`
- Provide actionable messages when rejecting server tools or MCP features under current policy.

## 12. Performance and Resilience

- Streaming relay with backpressure: use TransformStream / `respondWith` streaming, write as chunks arrive.
- Heartbeats: forward provider keep‑alive comments as comments; generate periodic `: keep-alive` if upstream is silent.
- Timeouts: upstream timeouts with retry/backoff on transient 5xx; never duplicate tool side effects.
- Memory caps: bound tool arguments buffers, cap number of simultaneous tool calls.
- Caching: none by default; optional model metadata cache (e.g., provider capabilities) in KV.

## 13. Testing Strategy

### 13.1 Unit Tests

- Provider detection; wire model mapping (including vendor‑qualified `or:`).
- Header normalization (x-api-key ↔ authorization) and AI Gateway URLs.
- Request translation: roles, system, images, `tool_result` mapping.
- Tools: function schema mapping; `tool_choice` mapping.
- Reasoning: map from `metadata.castari.reasoning`; ignore gracefully when absent.
- Web: plugin injection and `:online` mapping.
- Error mapping normalization.

### 13.2 Integration Tests (Miniflare)

- Anthropic pass‑through: non‑stream and stream happy paths.
- OpenRouter translation: non‑stream and stream, including multi‑turn.
- Streaming with tool_calls: assemble JSON args; emit correct Anthropic SSE.
- Tool loop policies: `error`, `enforceAnthropic`, `emulate` (mock backends).
- Reasoning: effort/max_tokens/exclude variations.
- Web search: plugin on/off; engine selection.

### 13.3 Negative Tests

- Missing/invalid auth; wrong `x-worker-token`.
- Oversized images; malformed JSON; invalid tool schema.
- Unsupported MCP stdio; server tool present when policy forbids.

## 14. Phased Build Plan

### Phase 0 — Scaffold and Pass‑Through

- Worker skeleton (modules syntax), config bindings, basic router.
- Anthropic pass‑through (non‑stream and stream) via AI Gateway.
- Error normalization layer and basic logging.

### Phase 1 — OpenRouter Basic Translation

- Translate messages, system, sampling, stop, metadata.
- Non‑stream response mapping; initial SSE bridge for text only.

### Phase 2 — Streaming and Tool Calls

- Full SSE bridge: text blocks, event lifecycle, heartbeats.
- Tool calls (no execution): detect and re‑emit Anthropic `tool_use` blocks; map incoming `tool_result` from client into provider `role: tool` messages.

### Phase 3 — Reasoning and Web Search

- Map `metadata.castari.reasoning` to provider `reasoning`.
- Inject web plugin/`:online` and map basic tunables.

### Phase 4 — Server Tool Policies

- Detect Anthropic server tools in `tools`. Implement `SERVER_TOOLS_MODE`:
  - `error`: reject clearly
  - `enforceAnthropic`: reroute
  - `emulate`: feature‑flagged stub with mock backend interfaces

### Phase 5 — MCP HTTP/SSE Bridge (Opt‑In)

- Define backend interface for MCP over HTTP/SSE; enumerate tools; on tool call, execute and return results.
- Add session store (Durable Object/KV) for continuity.

### Phase 6 — Security and Observability

- Worker token, size caps, allowlists, redaction, rate limiting.
- Structured logs, metrics, p50/p90/p99, error dashboards.

### Phase 7 — Hardening and Docs

- Conformance tests against SDK streaming behaviors.
- End‑to‑end examples (Node and Python wrappers).
- Operational runbooks and configuration documentation.

## 15. Compatibility With Other SDK Languages

- The Worker accepts pure Anthropic Messages requests with no custom headers; provider inference and translation still work.
- Language wrappers (Node, Python, others) may optionally add `x-castari-*` headers for improved routing and analytics but are not required.
- Reasoning preferences can be passed via `metadata.castari.reasoning` from any language; the Worker remains schema‑driven, not runtime‑dependent.

## 16. Open Questions / TBD

- Exact Anthropic SSE event taxonomy for `tool_use` input streaming (`input_json_delta` vs `text_delta`) — implement the most compatible subset and validate with SDK.
- Usage and cost normalization for streaming when upstream omits `usage` until the end; consider a finalizer call if necessary.
- Breadth of server tool emulation to support first (web fetch vs browser vs text editor).

---

This design ensures the Worker is language‑agnostic, robust, and extensible. It delivers parity with the Claude Agent SDK while unlocking cross‑provider usage through a single, consistent Messages API entrypoint.

