Castari Cloudflare Worker — Requirements and Build Plan

Overview

The Castari Worker is an Anthropic-compatible proxy that receives Claude Agent SDK Messages API requests and routes them to either Anthropic (pass-through) or alternate providers (initially OpenRouter) with full translation of requests, streaming responses, tools, and reasoning. The goal is feature parity for SDK clients regardless of chosen model. All traffic flows through Cloudflare AI Gateway for observability and governance.

Goals

- Preserve the Claude Agent SDK experience: same request shape, same streamed events, same tool semantics, same multi-turn behavior.
- Support Anthropic and OpenRouter providers, selectable by model string; add more providers later via the same translation layer.
- Translate requests and responses losslessly, including content blocks (text, images), tools, web search, reasoning, and usage metadata.
- Maintain robust streaming (SSE) without breaking includePartialMessages flows in the SDK.
- Enforce security, privacy, and controlled execution for any server-side tool emulation.
- Surface clear, actionable errors for unsupported or misconfigured features.

Non‑Goals (v1)

- Full emulation of every Anthropic server tool variant (CodeExecution, ComputerUse, TextEditor) is out-of-scope by default. We provide a policy to either error or enforce Anthropic routing; an emulation path is designed and may be enabled behind a flag for specific backends.
- STDIO-based MCP servers cannot be connected to from a Cloudflare Worker runtime.

Ingress and Egress

- Ingress (from SDK wrapper)
  - Method: POST
  - Path: /v1/messages
  - Auth: Authorization: Bearer <token> (token is provider-specific, but provided in ANTHROPIC_API_KEY env at the client; Worker uses it for the chosen provider)
  - Headers (added by wrapper):
    - x-castari-provider: anthropic | openrouter
    - x-castari-model: original model string (e.g., or:gpt-5-mini)
    - x-castari-wire-model: normalized downstream model (e.g., openai/gpt-5-mini)
    - x-client-meta: JSON string of app/client/version/session metadata
    - x-worker-token: optional shared secret for Worker authorization

- Egress (to upstreams, always via Cloudflare AI Gateway if configured)
  - Anthropic upstream: https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/anthropic/v1/messages
  - OpenRouter upstream: https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/openrouter/v1/chat/completions
  - For local/dev, allow direct upstreams: https://api.anthropic.com/v1/messages, https://openrouter.ai/api/v1/chat/completions

Provider Selection and Model Mapping

- Source of truth: x-castari-provider header (else infer from model prefix: claude* → anthropic; or:, openrouter/*, openai/* → openrouter)
- Model mapping:
  - Anthropic: passthrough
  - OpenRouter:
    - `or:<slug>` → If `<slug>` contains `/`, treat as `<vendor>/<model>`; otherwise prefix `${OPENROUTER_DEFAULT_VENDOR}/${slug}` (default vendor is `openai`).
    - `openrouter/<vendor>/<slug>` → `<vendor>/<slug>`
    - `openai/<slug>` → `openai/<slug>`
- Preserve both original and wire model in headers/metadata for traceability.

Request Translation (Anthropic → OpenRouter)

- Top-level fields
  - model → model (wire model)
  - system → prepend as a system message
  - messages[] → map role and content:
    - role: user/assistant map 1:1
    - assistant prefill (last message role=assistant) is supported by OpenRouter; keep content as-is
    - tool_result blocks (Anthropic: user message with content type=tool_result) → a separate message with role=tool, tool_call_id, content JSON/string
  - max_tokens → max_tokens
  - temperature, top_p → pass through; top_k ignored for OpenAI models
  - stop_sequences → stop
  - stream → stream
  - metadata → preserved; the Worker reads metadata.castari.* but otherwise forwards provider‑agnostic fields untouched when possible

- Content blocks
  - Text: { type: 'text', text } → string content
  - Images: { type: 'image', source: { type: 'url', url } } → content part { type: 'image_url', image_url: { url } }
  - Base64 images: data URI preferred (data:<mime>;base64,<data>)

- Tools (client-tools / function tools)
  - Anthropic tools[] (JSON Schema at input_schema) → OpenAI functions: { type: 'function', function: { name, description, parameters } }
  - tool_choice mapping:
    - auto → 'auto'
    - none → 'none'
    - tool{name} → { type: 'function', function: { name } }
    - any (Anthropic) → 'auto'

- Web search
  - If Anthropic server WebSearchTool is declared and provider ≠ Anthropic:
    - Policy (see “Policies and Feature Gates”):
      - enforceAnthropic: route to Anthropic
      - error: respond 400 with guidance
      - emulate: use OpenRouter web plugin and map tunables if provided
  - If the user indicates search via metadata.castari.web_search_options, use OpenRouter web plugin or suffix :online per docs

- Reasoning
  - Wrapper injects metadata.castari.reasoning { effort?, max_tokens?, exclude?, summary? }
  - Map to OpenRouter reasoning:
    - effort → reasoning.effort
    - max_tokens → reasoning.max_tokens
    - exclude → reasoning.exclude
  - For models without exposed reasoning: ignore exclude/effort but keep internal; do not synthesize thinking content

- MCP (server-run)
  - If request includes mcp_servers (Anthropic server MCP):
    - Policy:
      - enforceAnthropic: route to Anthropic
      - error: respond 400 with clear message
      - bridge-http/sse (opt-in): connect to HTTP/SSE MCP servers only, convert exposed MCP tools to function tools for the turn, run loop to execute tool_calls via MCP, then continue model turn
    - STDIO MCP is not supported in Workers

Response Translation (OpenRouter → Anthropic)

- Non-streaming
  - choices[0].message.content → assistant message content block(s)
  - choices[0].message.tool_calls → convert to assistant content block(s) of type tool_use with id, name, input JSON
  - If tool loop executed in the Worker, append resultant assistant text in the final turn only
  - usage: map prompt_tokens, completion_tokens when available; optionally enrich later via generation API if needed

- Streaming (SSE)
  - OpenRouter streams chunks with choices[0].delta.{content?, tool_calls?}
  - Emit Anthropic-like stream:
    - message_start once per assistant turn
    - content_block_start for each block (text or tool_use); for tool_use, start when first tool_call arrives
    - text_delta for content text deltas
    - tool_use.delta style assembly: buffer function.arguments text to valid JSON; when closed, emit content_block_stop for that tool
    - message_delta/stop and message_stop at the end with stop_reason mapping (stop, length, content_filter, tool_calls)

Tool Loop Orchestration

- Worker inspects streamed or final tool_calls from the provider.
- For each tool_call:
  - Execute client-side emulation if policy=emulate and backend configured (e.g., MCP HTTP/SSE, internal fetch, restricted code exec)
  - Otherwise, if policy=enforceAnthropic: short-circuit and route entire request to Anthropic
  - On success, send a follow-up call to provider including the original assistant tool_calls and a role=tool message for each result; continue streaming the final assistant text
- Ensure deterministic mapping of tool_call ids to Anthropic tool_use ids and back

Policies and Feature Gates

- serverToolsMode: 'error' | 'enforceAnthropic' | 'emulate' (default: 'error')
- mcpBridgeMode: 'off' | 'http-sse' (default: 'off')
- webSearchMode: 'native' | 'plugin' | 'auto' (default: 'auto')
- rateLimit: on/off with thresholds
- logging: redact content bodies unless DEBUG set

Security and Compliance

- AuthN: optional x-worker-token shared secret; reject if missing when required
- Input validation: size caps (body, images), JSON schema sanity (tools), deny oversized base64 images
- Outbound allowlists: restrict web fetch or browser actions when emulating server tools
- Sandboxing for emulation: no arbitrary code execution in Worker; only call hardened backends (MCP, trusted services). No STDIO, no shell in Worker
- PII and logs: default redact bodies; store per-request trace IDs; configurable sampling for headers and timing data

Configuration (Worker)

- Environment / Bindings
  - UPSTREAM_ANTHROPIC_BASE_URL (default Cloudflare AI Gateway Anthropic path)
  - UPSTREAM_OPENROUTER_BASE_URL (default Cloudflare AI Gateway OpenRouter path)
  - SERVER_TOOLS_MODE (error|enforceAnthropic|emulate)
  - MCP_BRIDGE_MODE (off|http-sse)
  - WORKER_TOKEN (optional)
  - ALLOW_ORIGINS (comma-separated; used to validate incoming origin if needed)
  - LOG_LEVEL (debug|info|warn|error)
  - RUNTIME_STORE (Durable Object or KV namespace id for session/tool buffering)

- Runtime
  - Cloudflare Workers, modules syntax, streaming fetch enabled
  - Durable Object/KV (optional) for session tool buffering (multi-tool, multi-turn continuity)

Implementation Plan

1) Router and provider selection
   - Parse headers; validate Authorization; decide provider; select upstream URL and auth header
2) Request normalization
   - Parse Anthropic Messages body; validate; extract metadata.castari.*
3) Translation layer (Anthropic → Provider)
   - Messages, system, images, tools, tool_choice, stop, sampling params, reasoning, web search
4) Dispatch
   - Send to upstream with stream flag
5) Stream bridge
   - Convert provider deltas into Anthropic SSE events; assemble tool_calls JSON arguments safely
6) Tool loop
   - If tool_calls present: handle per policy; when emulation enabled execute tool and send follow-up provider call(s)
7) Response finalization
   - Map usage; map stop reasons; return Anthropic-shaped JSON for non-streaming
8) Error mapping and retries
   - Translate provider errors; handle 429/5xx with backoff where appropriate; never duplicate side effects

Detailed Mapping Tables (concise)

- Roles
  - user ↔ user
  - assistant ↔ assistant
  - tool_result (Anthropic user block) ↔ role: tool message with tool_call_id
  - system (Anthropic top-level) ↔ role: system message prepended

- Tools
  - input_schema ↔ parameters JSON Schema
  - tool_use(id,name,input) ↔ tool_calls[{id, function:{name, arguments}}]
  - tool_result(tool_use_id, content) ↔ role: tool with matching tool_call_id

- Reasoning
  - metadata.castari.reasoning.effort ↔ reasoning.effort
  - ...max_tokens ↔ reasoning.max_tokens
  - ...exclude ↔ reasoning.exclude

- Web Search
  - Anthropic WebSearchTool ↔ OpenRouter plugins: [{id:'web', ...options}] or model ':online'

- Sampling
  - temperature ↔ temperature; top_p ↔ top_p; stop_sequences ↔ stop; top_k dropped for OpenAI

Usage and Cost

- Prefer upstream usage when provided. For streaming, attach usage from final chunk or a trailing call when available
- Optionally, fetch generation stats from OpenRouter /api/v1/generation by id (future)
- Populate Anthropic-like usage fields (input_tokens, output_tokens, reasoning_tokens when available) and total_cost_usd if upstream supplies it; otherwise omit cost

Error Handling

- Map upstream finish_reason/content filter to Anthropic stop_reason/end states
- Normalize HTTP errors with a stable JSON envelope { type, message, provider_details?, retryable? }
- Common mappings:
  - 400 invalid_request_error
  - 401 authentication_error
  - 403 permission_error
  - 429 rate_limit_error
  - 5xx api_error

Testing Strategy

- Unit tests
  - Model/provider detection; header parsing; env and policy selection
  - Messages/content mapping (text, images)
  - Tools schema translation; tool_choice mapping; tool_result mapping
  - Reasoning injection and mapping
  - Web plugin mapping
  - Error mapping

- Integration tests (Miniflare or Cloudflare Pages Functions)
  - Non-streaming and streaming with simple text
  - Streaming with tool_calls; multi-tool; argument JSON assembly
  - Tool loop: emulate disabled (error), enforceAnthropic (reroute), emulate (mock MCP backend)
  - Web search with plugin injection
  - Reasoning with effort/max_tokens and exclude

- Negative tests
  - Malformed JSON; invalid tool schema; oversized images; missing auth; wrong x-worker-token
  - Unsupported MCP stdio

Operational Concerns

- Observability: structured logs with trace_id, provider, model, latency, streamed-bytes, tool_calls count
- Metrics: request counts, error rates, tool loop invocations, reroute counts, reasoning usage
- Rate limiting: per clientId/app from x-client-meta; respond 429 with Retry-After
- Config rollout: feature flags via environment variables; safe defaults

Future Enhancements

- Full server tool emulation via MCP backends (code exec, browser, text editor) with hardened sandboxes
- MCP session manager for HTTP/SSE servers; tool discovery cache
- Usage reconciliation by querying OpenRouter generation endpoint
- Additional providers (OpenAI native, Google, etc.) behind the same mapping layer
