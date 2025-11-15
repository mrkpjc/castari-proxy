# Castari Query Wrapper Design (queryCastari)

This document specifies the final production design for `queryCastari()`, a drop‑in wrapper around the Claude Agent SDK’s `query()` that enables seamless multi‑provider usage via the Castari Worker (Cloudflare Worker + AI Gateway). It covers API surface, provider/model detection, environment/credential handling, request interception, reasoning controls, web search behavior, streaming, error handling, telemetry, and test strategy.

## Objectives

- Preserve all features and ergonomics of `@anthropic-ai/claude-agent-sdk` `query()`.
- Allow developers to switch providers by changing only the `options.model` string.
- Always route traffic through the Castari Worker by setting `ANTHROPIC_BASE_URL` (observability, governance, analytics via AI Gateway).
- Support OpenRouter as the first non-Anthropic provider with correct credentials and model mapping.
- Provide a provider-agnostic reasoning API (`options.reasoning`) and map it transparently in the Worker.
- Keep web search translation in the Worker; no wrapper changes to tool usage or plugin configuration are required.

## High-Level Architecture

- App calls `queryCastari({ prompt, options })` instead of `query()`.
- Wrapper determines the provider from `options.model`, selects the correct API key, and ensures `ANTHROPIC_BASE_URL` targets the Castari Worker.
- Wrapper installs a fetch interceptor exactly once to attach route hints and reasoning metadata for the Worker. The Claude SDK still performs the actual HTTP call to `POST /v1/messages` with headers and streaming intact.
- Castari Worker translates requests and responses:
  - Anthropic provider: pass-through to Anthropic Messages API via Cloudflare AI Gateway.
  - OpenRouter provider: translate Anthropic Messages shape to OpenRouter Chat Completions, including tools, streaming, web search, and reasoning. Translate responses back to Anthropic shape.

## API Surface

### Function

```ts
function queryCastari({
  prompt,
  options
}: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: CastariOptions;
}): Query
```

`Query` and all streamed message semantics match the SDK’s `query()`.

### Types

```ts
type Provider = 'anthropic' | 'openrouter';

interface ProviderCredentials { apiKey?: string }

interface CastariProviders {
  openrouter?: ProviderCredentials; // expects OPENROUTER_API_KEY if not explicitly provided
  anthropic?: ProviderCredentials;  // expects ANTHROPIC_API_KEY if not explicitly provided
}

type ReasoningEffortPreference = 'auto' | 'low' | 'medium' | 'high' | 'max';
type ReasoningSummaryPreference = 'auto' | 'concise' | 'detailed' | 'none';

// Advanced, explicit reasoning config (provider-agnostic)
interface ReasoningConfig {
  effort?: 'low' | 'medium' | 'high' | 'max';
  maxTokens?: number;               // maps to OpenRouter reasoning.max_tokens or Anthropic budget_tokens logic
  exclude?: boolean;                 // omit reasoning content in responses when supported
  summary?: ReasoningSummaryPreference; // optional: client-side aggregation preference
}

interface CastariMeta {
  app?: string;
  clientId?: string;
  deploymentId?: string;
  version?: string;
  reasoningEffort?: ReasoningEffortPreference;   // convenience preference
  reasoningSummary?: ReasoningSummaryPreference; // convenience preference
}

interface CastariOptions extends Options {
  providers?: CastariProviders;      // explicit BYOK override
  meta?: CastariMeta;                // client metadata for observability
  reasoning?: ReasoningConfig;       // advanced, explicit reasoning config
  reasoningEffort?: ReasoningEffortPreference;    // convenience
  reasoningSummary?: ReasoningSummaryPreference;  // convenience
  subagentModel?: string;            // Task tool default model (inherit when omitted)
}
```

Notes:
- `CastariOptions` extends the SDK `Options` type; all SDK features remain available (tools, MCP, permissions, etc.).
- `options.providers` is purely an override convenience; env vars remain the default source of truth.
- `Options.maxThinkingTokens` is automatically mapped into `metadata.castari.reasoning.max_tokens` when `options.reasoning.maxTokens` is not provided.
- `options.subagentModel` ultimately drives the SDK’s `CLAUDE_CODE_SUBAGENT_MODEL` env (see Task section below).

## Provider and Model Detection

The wrapper resolves a provider from the model string and produces a “wire model” for the downstream provider when applicable.

### Detection Rules

- OpenRouter provider:
  - `or:<slug>` (preferred), e.g., `or:gpt-5-mini` or `or:google/gemini-2.0`
    - If `<slug>` contains `/`, treat as vendor‑qualified and pass through as `<vendor>/<model>`.
    - If `<slug>` contains no `/`, prefix a default vendor `${OPENROUTER_DEFAULT_VENDOR}/${slug}` (defaults to `openai`).
  - `openrouter/<vendor>/<slug>`, e.g., `openrouter/openai/gpt-5-mini`
  - `openai/<slug>`, e.g., `openai/gpt-4o-mini` (explicit vendor-qualified slug routed via OpenRouter)
- Anthropic provider:
  - `claude-*`, e.g., `claude-3-7-sonnet`
  - `anthropic/<slug>`, e.g., `anthropic/claude-3-7-sonnet`
- Default fallback: `anthropic` (preserves SDK’s Anthropic-first behavior).

### Wire Model Mapping

- OpenRouter:
  - `or:gpt-5-mini` → `${OPENROUTER_DEFAULT_VENDOR}/gpt-5-mini` (defaults to `openai/gpt-5-mini`)
  - `or:google/gemini-2.0` → `google/gemini-2.0`
  - `openrouter/openai/gpt-5-mini` → `openai/gpt-5-mini`
  - `openai/gpt-4o-mini` → `openai/gpt-4o-mini`
- Anthropic:
  - Unchanged (e.g., `claude-3-7-sonnet`).

Wrapper attaches both original and wire model to headers (see Interceptor) for Worker robustness.

## Environment and Credentials

### Always Use the Worker Base URL

- `ANTHROPIC_BASE_URL` is always set to the Castari Worker endpoint.
- Resolution precedence:
  1) `options.env.ANTHROPIC_BASE_URL`
  2) `process.env.ANTHROPIC_BASE_URL`
  3) `options.baseUrl` (if provided to installer)
  4) `process.env.CASTARI_GATEWAY_URL`
- If no value is available, throw an error instructing how to set the Worker URL.

### API Key Selection

- For `provider='openrouter'`: require `OPENROUTER_API_KEY` (from `options.providers.openrouter.apiKey`, `options.env.OPENROUTER_API_KEY`, or `process.env.OPENROUTER_API_KEY`).
- For `provider='anthropic'`: require `ANTHROPIC_API_KEY` (from `options.providers.anthropic.apiKey`, `options.env.ANTHROPIC_API_KEY`, or `process.env.ANTHROPIC_API_KEY`).
- Wrapper writes the selected key into `options.env.ANTHROPIC_API_KEY` so the SDK emits it as `x-api-key`.

### Additional Env Keys

- `X_WORKER_TOKEN`: forwarded as `x-worker-token` header to the Worker for auth.
- Metadata envs (optional): `CASTARI_APP`, `CASTARI_CLIENT_ID`, `CASTARI_DEPLOYMENT_ID`, `CASTARI_VERSION`, `CASTARI_REASONING_EFFORT`, `CASTARI_REASONING_SUMMARY`.
- `CASTARI_SUBAGENT_MODEL`: optional default for Task subagents (see below). When omitted, the wrapper falls back to the main agent model.

### Task Tool Subagent Model Selection

- Claude Agent SDK consults `process.env.CLAUDE_CODE_SUBAGENT_MODEL` before using the Task tool’s `model` field or subagent defaults. Castari now sets this env for every query so Task stays consistent with the developer’s intent.
- Resolution order:
  1. `options.subagentModel` if supplied in code.
  2. `options.env.CASTARI_SUBAGENT_MODEL` (includes `process.env` merged into the wrapper env).
  3. Default: `'inherit'`, which reuses the primary `options.model`.
- Accepted values:
  - `inherit`, `auto`, or `default` → take the main agent model verbatim.
  - Shorthand tiers `haiku`, `sonnet`, `opus` (case-insensitive) → expanded to `claude-haiku-4-5-20251001`, `claude-sonnet-4-5-20250929`, and `claude-opus-4-1-20240808`.
  - Any fully qualified model slug (e.g., `claude-haiku-4-5-20251001`, `or:gpt-5-mini`, `openrouter/openai/gpt-4o-mini`).
- Unless the caller already provided `CLAUDE_CODE_SUBAGENT_MODEL`, the wrapper writes the resolved value to `options.env` so every Task invocation within the session uses the configured model, regardless of provider.

## Interceptor Design

An idempotent fetch interceptor wraps only the Claude SDK’s `POST /v1/messages` calls that target the Worker origin.

### Scope and Safety

- Intercept only `POST` requests to path `/v1/messages`.
- Intercept only when the request origin matches the configured Worker base origin(s).
- Do not alter auth headers; the SDK controls `x-api-key`.
- Respect non-JSON bodies (no parsing/injection attempt).

### Headers Added

- `x-castari-provider: 'anthropic' | 'openrouter'`
- `x-castari-model: <original model provided by the app>`
- `x-castari-wire-model: <provider-native model used downstream>`
- `x-worker-token: <X_WORKER_TOKEN>` (if configured)
- `x-client-meta: JSON.stringify({ sdk, app, clientId, deploymentId, version, model, provider, reasoningEffort?, reasoningSummary? })`

These route hints and metadata enable robust translation, observability, and A/B analysis in the Worker/Gateway.

### Reasoning Injection (Body)

When `Content-Type: application/json` and method is `POST`:

- Wrapper injects a neutral, provider-agnostic field into the Anthropic request body:

```jsonc
{
  // ... original Anthropic Messages payload fields ...
  "metadata": {
    // preserved original metadata
    "castari": {
      "reasoning": {
        "effort": "low|medium|high|max",   // optional
        "max_tokens": number,               // optional
        "exclude": boolean,                 // optional
        "summary": "auto|concise|detailed|none" // optional
      }
    }
  }
}
```

The Worker consumes `metadata.castari.reasoning` to construct the provider-native reasoning configuration.

## Reasoning Semantics

Two ways to configure reasoning from the app:

- Simple preferences:
  - `options.reasoningEffort?: 'auto'|'low'|'medium'|'high'|'max'`
  - `options.reasoningSummary?: 'auto'|'concise'|'detailed'|'none'`
- Advanced config:
  - `options.reasoning?: { effort?, maxTokens?, exclude?, summary? }`

Precedence: `options.reasoning` overrides the simple preferences; env/meta can provide defaults.

### Worker Mapping Rules

- For OpenRouter (see reference_docs/OpenRouter/Reasoning_Tokens.md):
  - Map `effort ∈ {low, medium, high}` → `reasoning.effort`.
  - Map `effort = max` → best-effort `reasoning.effort: 'high'`.
  - Map `max_tokens` → `reasoning.max_tokens`.
  - Map `exclude = true` → `reasoning.exclude: true`.
  - Preserve provider behavior when unsupported (ignore unsupported params without failing the request).
- For Anthropic:
  - Either pass through native “extended thinking”/budget semantics when present, or transform `effort/max_tokens` to Anthropic-compatible reasoning settings per up-to-date model docs.

## Web Search Semantics

Wrapper makes no changes to tool configuration. Web search translation is a Worker concern.

### Detection (Worker)

- Detect Anthropic web search intent via:
  - Presence of the Anthropic `WebSearchTool` in `tools` (e.g., `web_search_20250305` in reference_docs/Anthropic/Messages.md).
  - Assistant `tool_use` blocks invoking web search.

### Mapping to OpenRouter (reference_docs/OpenRouter/Web_Search.md)

- Enable OpenRouter web grounding by:
  - Adding `plugins: [{ id: 'web', ... }]`, or
  - Appending `:online` to the wire model.
- Optional param mapping:
  - If the Anthropic tool exposes tunables, map to:
    - `plugins[0].engine` (`native` | `exa` | undefined)
    - `plugins[0].max_results`
    - `web_search_options.search_context_size` (`low|medium|high`)
- Default behavior when not specified:
  - Let OpenRouter select engine: native when available, else `exa`.
- Response mapping:
  - OpenRouter’s `message.annotations[].type = 'url_citation'` are folded into the Anthropic-shaped response content/metadata to preserve usability and traceability.

## Tool Calling and MCP

- SDK tool definitions and MCP configs from the app remain unchanged.
- Worker translates Anthropic tool calls to OpenAI-compatible `tools` for OpenRouter (see reference_docs/OpenRouter/Tool_Calling.md and MCP.md) and back.
- Unsupported tool shapes are ignored or gracefully downgraded with clear error messages in the stream (without breaking the session).

## Streaming Behavior

- The wrapper does not alter SSE semantics. It only injects headers/body prior to dispatch.
- The Worker streams OpenRouter chunks and translates them to Anthropic-like streamed events (`type`, `delta`, tool calls) for the SDK consumer.
- Final usage block at stream end is preserved when provider supports it.

## Error Handling

- Missing Worker base URL: throw describing required configuration (`ANTHROPIC_BASE_URL` or `CASTARI_GATEWAY_URL`).
- Missing API key: throw with the correct env var name for the detected provider.
- Unknown/unsupported model: error indicating supported model formats with examples.
- Non-JSON body: skip reasoning injection; proceed with headers only.
- Interceptor safety: fall back to original `fetch` on any parsing/logic issue; never exhaust the original request’s body stream.

## Telemetry and Metadata

- `x-client-meta` fields:
  - `sdk: 'claude-agent'`
  - `app`, `clientId`, `deploymentId`, `version` (from `options.meta` or env)
  - `model` (original), `provider`
  - `reasoningEffort`, `reasoningSummary` (effective preferences)
- Worker forwards/augments to AI Gateway for analytics and cost attribution.

## Security Considerations

- The wrapper never logs secrets by default; optional debug redacts tokens.
- API key selection is purely local; only the selected provider key is written to `ANTHROPIC_API_KEY` for the request.
- `X_WORKER_TOKEN` is sent only to the Worker origin.

## Testing Strategy

### Unit Tests

- Provider detection across supported prefixes and edge cases.
- Wire model mapping correctness.
- Env precedence resolution for base URL and API keys.
- Reasoning merge logic (preferences vs. explicit config) and JSON injection.
- Header injection: presence and content of `x-castari-*` and `x-client-meta`.

### Integration (mock fetch)

- Interceptor scoping to `/v1/messages` and configured origins.
- Anthropic model path: pass-through env key, correct base URL, headers attached.
- OpenRouter model path: uses `OPENROUTER_API_KEY`, correct base URL, headers attached, reasoning metadata present.
- Streaming: ensure injection does not break SSE (no header/body corruption).

### Negative Cases

- Missing base URL → descriptive error.
- Missing provider key → descriptive error.
- Non-JSON request bodies → no injection, still route with headers.

## Usage Examples

### Anthropic (pass-through via Worker)

```ts
await queryCastari({
  prompt: 'Summarize the diff',
  options: {
    model: 'claude-3-7-sonnet',
    env: {
      ANTHROPIC_BASE_URL: process.env.CASTARI_GATEWAY_URL, // or set globally
      ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY,
    },
  },
});
```

### OpenRouter with Reasoning

```ts
await queryCastari({
  prompt: 'Plan a 3-day Tokyo itinerary',
  options: {
    model: 'or:gpt-5-mini',
    env: {
      ANTHROPIC_BASE_URL: process.env.CASTARI_GATEWAY_URL,
      OPENROUTER_API_KEY: process.env.OPENROUTER_API_KEY,
    },
    reasoning: {
      effort: 'high',
      exclude: false,
    },
  },
});
```

### Preferences-Only Reasoning

```ts
await queryCastari({
  prompt: 'Which is bigger: 9.11 or 9.9? Explain.',
  options: {
    model: 'openrouter/openai/gpt-4o-mini',
    reasoningEffort: 'medium',
    reasoningSummary: 'concise',
  },
});
```

## Implementation Notes

- Use `AsyncLocalStorage` to thread per-call context (provider, models, worker token, resolved client meta, reasoning) through the interceptor and avoid global mutable state.
- Install the interceptor once; it should be idempotent and safe across concurrent queries.
- Never mutate `authorization`/`x-api-key` headers in the interceptor. Let the SDK handle auth based on `env`.
- Keep injection minimal and schema-stable; only `metadata.castari.*` is added.

## Worker Contract Summary

- If `x-castari-provider = 'anthropic'`: forward payload unmodified to Anthropic Messages API via AI Gateway.
- If `x-castari-provider = 'openrouter'`:
  - Convert Anthropic Messages to OpenRouter Chat Completions format.
  - Map `metadata.castari.reasoning` to OpenRouter `reasoning` fields.
  - Detect Anthropic web search tool usage and enable OpenRouter `web` plugin or `:online` model variant; map tunables when present, else defaults.
  - Stream responses and translate OpenRouter deltas/tool calls/annotations back to Anthropic-like streamed messages for the SDK.

This design ensures developers swap models and optionally supply `options.reasoning` with no other code changes, while the Worker handles all protocol translation (tools, web, streaming, MCP) and AI Gateway telemetry.
