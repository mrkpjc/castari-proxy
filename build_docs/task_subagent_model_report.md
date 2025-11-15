# Task Subagent Model Handling — Findings & Design Report

## 1. Context and Goals
- Castari Proxy wraps the Claude Agent SDK so any model (Anthropic or OpenRouter) can power the primary agent via the Cloudflare Worker. Users often rely on the built-in **Task** tool to spawn autonomous sub-agents.
- Today the Task tool defaults to Anthropic Haiku regardless of the user’s selected model, which produces inconsistent behavior, requires an Anthropic key, and makes cost/latency hard to predict.
- We need a production-ready design that keeps the developer experience simple: by default the Task tool should inherit the model the user selected for the primary agent, but there must also be a clear, ergonomic way to override that choice (globally or per deployment) without editing SDK internals.
- The solution must keep working for users who only have an OpenRouter key, while still letting Anthropic users pin Task to Haiku/Sonnet/Opus when desired.

## 2. Current Behavior (Detailed)

### 2.1 queryCastari wrapper
- `src/queryCastari.ts` fully controls the SDK environment per call. It already sets `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, `CASTARI_GATEWAY_URL`, and optional worker token before invoking `query()`.
- Provider resolution happens purely from `options.model`. OpenRouter slugs (e.g., `or:gpt-5-mini`) get mapped to vendor-specific wire models, and the wrapper picks the right credential.
- The wrapper installs a global fetch interceptor so every SDK HTTP request is routed through the Castari Worker.
- **Important:** the wrapper currently does **not** set `CLAUDE_CODE_SUBAGENT_MODEL`, so the SDK internally falls back to whatever the built-in agent definitions specify.

### 2.2 SDK Task tool (single source of truth)
- The Task tool definition lives in `@anthropic-ai/claude-agent-sdk/cli.js` (demo dependency). Relevant snippets:
  - `function VbA(agentModel, mainLoopModel, toolInputModel)` (line ~699) returns:
    1. `process.env.CLAUDE_CODE_SUBAGENT_MODEL` if it exists.
    2. Otherwise the explicit `model` field from the Task tool input (`"sonnet" | "opus" | "haiku"` per `sdk-tools.d.ts`).
    3. Otherwise the `model` defined on the subagent itself (from user settings or built-in defaults such as Haiku).
    4. Otherwise the main loop model (only useful when the main loop is already Anthropic).
  - The resolved value is normalized via `function LU`, which maps shorthand tiers to the latest Claude SKUs. Non-Anthropic strings simply pass through untouched.
- When Task fires, the SDK creates a new `query()` call using the same options (same env overrides, same worker URL) but swaps the model to whatever `VbA` returns.

### 2.3 Worker routing implications
- The Worker (`worker/src/index.ts`) decides which upstream to call by inspecting the `model` string in the request body or the optional Castari headers.
- If a Task subagent sets `model: "claude-sonnet-4-5-20250929"`, the Worker automatically routes the request to Anthropic using the API key supplied by the wrapper. If the subagent model were `or:gpt-5-mini`, it would route to OpenRouter with the OpenRouter key (assuming the client supplied one).
- Therefore, the only real lever we need to pull is **which model string Task inserts**—routing and credential selection follow automatically.

## 3. Pain Points Observed
- Built-in subagents are hard-coded to Haiku, so Task acts differently from the main session even when the user explicitly chose Sonnet/Opus or an OpenRouter model.
- Users without Anthropic credentials cannot successfully run Task: the SDK keeps trying to hit Anthropic Haiku and fails authentication, even though the rest of the session is on OpenRouter.
- Changing Task behavior today requires editing `.claude` settings or manually exporting `CLAUDE_CODE_SUBAGENT_MODEL`, which is undocumented and easy to forget.
- There is no way to configure Task at the Castari layer, so every project must reinvent its own workaround.

## 4. Requirements (Functional & DX)
1. **Default inheritance:** Task should automatically inherit the model the user passed to `queryCastari` (Anthropic or OpenRouter) unless the user explicitly overrides it.
2. **Global override:** A single config knob (env or option) should let teams pin Task to a specific model or tier regardless of the main session.
3. **Provider-agnostic:** Users with only OpenRouter creds must be able to run Task; the system can no longer unconditionally force Anthropic.
4. **Ergonomic mapping:** Accept friendly aliases like `haiku`/`sonnet`/`opus` and translate them to the current versioned Claude SKUs.
5. **Extensible:** Leave room for future per-subagent overrides without breaking the simple path.
6. **Documentation:** Clearly explain the new behavior and configuration so the DX stays “obvious”.

## 5. Proposed Solution (Complete Design)

### 5.1 New configuration surface
- Extend `CastariOptions` with an optional `subagentModel` field (string | `'inherit'`). Expose the same control via env vars:
  - `CASTARI_SUBAGENT_MODEL` — default for all subagents.
  - `CASTARI_SUBAGENT_MODEL_<AGENTTYPE>` (optional future enhancement) — targeted overrides for specific `subagent_type` values.
- Accept values in any of these forms:
  - `'inherit'` / `'auto'`: use the main session’s `options.model` verbatim.
  - Shorthand tiers: `'haiku'`, `'sonnet'`, `'opus'` (case-insensitive). We map these through `LU()` semantics to the latest Claude SKU.
  - Fully qualified model IDs: e.g., `claude-sonnet-4-5-20250929`, `or:gpt-5-mini`, `openrouter/openai/gpt-4o-mini-2024-08-06`.

### 5.2 queryCastari implementation
1. During option preparation (before installing the interceptor), resolve the desired subagent model:
   ```ts
   const resolvedSubagentModel = resolveSubagentModel({
     explicit: options.subagentModel,
     envDefault: effectiveEnv.CASTARI_SUBAGENT_MODEL || process.env.CASTARI_SUBAGENT_MODEL,
     mainModel: model,
   });
   ```
2. `resolveSubagentModel` logic:
   - If `explicit` is provided, use it; else fall back to env; else default to `'inherit'`.
   - If the outcome is `'inherit'`, return the exact `model` string supplied by the caller (Anthropic or OpenRouter).
   - If it matches a shorthand tier, map via the same normalization table `LU()` uses (we can inline the mapping in TypeScript to avoid importing CLI internals).
   - Otherwise, treat it as a literal slug and return as-is.
3. Unless `effectiveEnv.CLAUDE_CODE_SUBAGENT_MODEL` was already set by the caller, set it to `resolvedSubagentModel`. This ensures we do not override advanced users who explicitly injected a different value.
4. Document the new option/env so users understand how to configure it.

### 5.3 OpenRouter-only users
- Because the resolved subagent model becomes the literal string inside the HTTP body, the Worker will automatically reuse the same provider/credential decision it uses for the main session. Example:
  - User sets `options.model = "or:gpt-5-mini"` and leaves `subagentModel` undefined.
  - The wrapper resolves `'inherit'` → `"or:gpt-5-mini"` and sets `CLAUDE_CODE_SUBAGENT_MODEL` to that value.
  - The SDK launches Task with `model: "or:gpt-5-mini"` and the Worker routes it to OpenRouter using the OPENROUTER_API_KEY.
  - No Anthropic key is required anywhere.

### 5.4 Anthropic-centric defaults & overrides
- Users who prefer Haiku for delegated tasks can set `CASTARI_SUBAGENT_MODEL=haiku` (or supply `subagentModel: 'haiku'` in code). The wrapper maps that to `claude-haiku-4-5-20251001` (exact version pinned via helper) and injects it into the env.
- Because we resolve once per query, Task stays deterministic for the entire session.

### 5.5 Optional future enhancements
- Per-subagent overrides: allow a JSON env (e.g., `CASTARI_SUBAGENT_MODEL_MAP=Explore:sonnet,Implement:haiku`). We can inject a lightweight hook or wrapper around Task tool calls to adjust `CLAUDE_CODE_SUBAGENT_MODEL` on-the-fly. Not required for v1 but the current design leaves room for it.
- UI/CLI surface: expose the new config in `claude-agent-demo`’s `.env.example` and docs so developers see it immediately.

## 6. Implementation Checklist
1. **Wrapper code:** implement `resolveSubagentModel` and env injection in `src/queryCastari.ts`. Add tests/unit coverage if we have a harness; otherwise include thorough manual verification steps.
2. **Docs:** update `AGENTS.md`, `reference_docs/query_wrapper.md`, and the new `task_subagent_model_report.md` to describe inheritance behavior and configuration knobs.
3. **Demo app:** set `CASTARI_SUBAGENT_MODEL=inherit` (explicit default) in `claude-agent-demo/.env.example`, and mention the flag in README/setup instructions.
4. **Validation:**
   - Run a primary session on `or:gpt-5-mini`, invoke Task, confirm Worker routes both parent and subagent via OpenRouter without needing an Anthropic key.
   - Repeat with `subagentModel: 'haiku'` to ensure the Worker switches to Anthropic only for the Task tool.
   - Verify streaming/tool results flow unchanged.

## 7. Summary
- By leveraging the SDK’s built-in `CLAUDE_CODE_SUBAGENT_MODEL` escape hatch inside `queryCastari`, we can give every Castari user consistent, intuitive Task behavior without touching the Worker or requiring custom agent definitions.
- Default inheritance plus an easy override meets all the requirements: it keeps DX clean, supports OpenRouter-only deployments, and still lets Anthropic-heavy teams pin Task to a cheaper tier.
- The implementation is localized and low-risk: a small addition to the wrapper, documentation, and a clear migration path for existing users.
