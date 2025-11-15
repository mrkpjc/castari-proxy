# Session Report — Task Tool Subagent Model Routing

## Overview
- **Objective:** Ensure the Claude Agent SDK Task tool (subagents) inherits the primary agent’s model/provider when running through the Castari proxy, while allowing explicit overrides.
- **Key challenges:**
  - Task defaulted to Anthropic Haiku regardless of the main model, forcing Anthropic credentials and inconsistent behavior.
  - The SDK resolves Task models via `process.env.CLAUDE_CODE_SUBAGENT_MODEL`, so we needed to control that before the SDK spawns subagents.
  - The demo app cached an older copy of `castari-proxy`, so changes to `src/queryCastari.ts` were not visible until we re-synced it.

## Chronology and Findings
1. **Repository familiarization:** Reviewed design docs, worker implementation, and demo app integration. Confirmed `queryCastari` is the single ingress point for SDK requests and that Task resolution occurs inside the SDK (`@anthropic-ai/claude-agent-sdk/cli.js`, function `VbA`).
2. **Design articulation:** Authored `build_docs/task_subagent_model_report.md`, describing pain points, requirements, and the proposed solution (inherit by default, override via env, map friendly aliases).
3. **Implementation (library):**
   - Extended `CastariOptions` with `subagentModel`.
   - Added `resolveSubagentModel` in `src/queryCastari.ts` to map `inherit|auto` and the `haiku|sonnet|opus` aliases to canonical Claude model IDs.
   - When building options, resolved `subagentModel` from `options.subagentModel`, `CASTARI_SUBAGENT_MODEL`, or the primary `options.model` (default `inherit`).
   - Injected the resolved value into both `options.env.CLAUDE_CODE_SUBAGENT_MODEL` and `process.env.CLAUDE_CODE_SUBAGENT_MODEL` (so the SDK sees it before spawning Task).
4. **Documentation updates:**
   - `AGENTS.md` gained a Task-focused section explaining default inheritance, config knobs, and routing implications.
   - `reference_docs/query_wrapper.md` documents the new option/env behavior and how it feeds into `CLAUDE_CODE_SUBAGENT_MODEL`.
   - The Task design report captures the end-to-end reasoning.
5. **Demo app wiring:**
   - Updated `claude-agent-demo/lib/env.ts` to load `CASTARI_SUBAGENT_MODEL` from `.env`.
   - Ensured `buildOptions` forwards that env into the SDK overrides.
   - Added `CASTARI_SUBAGENT_MODEL=inherit` to the demo `.env` for visibility.
6. **Runtime sync fix:**
   - The demo loads `castari-proxy` from `node_modules`. Added `predev`, `prebuild`, and `prestart` scripts that re-run `scripts/sync-castari-proxy.mjs` so every `npm run dev` pulls in the latest `src/` changes. Without this, tests kept hitting the stale Haiku-only build.

## Testing Instructions
1. **Setup:** Inside `claude-agent-demo/.env`, set your desired main model (e.g., `MODEL_NAME=or:gpt-5-mini`) and leave `CASTARI_SUBAGENT_MODEL=inherit`.
2. **Dev server:** Run `npm run dev` (pre-script will sync `castari-proxy`). Start a session and invoke the Task tool. The Worker logs should show both parent and subagent requests using the OpenRouter slug and credentials.
3. **Override path:** Change `.env` to `CASTARI_SUBAGENT_MODEL=haiku`, restart `npm run dev`, and invoke Task again. The primary loop remains on OpenRouter, but Task requests should now show `model: claude-haiku-4-5-20251001` and hit Anthropic via the configured key.

## Outstanding Considerations
- **Per-agent overrides:** Current implementation is global; future work can interpret `CASTARI_SUBAGENT_MODEL_<AGENTTYPE>` if we need per-subagent granularity.
- **Worker deployment:** No Worker changes were required, but deployments must always use the synced `castari-proxy` package to pick up wrapper fixes.
- **Further automation:** CI should test Task behavior by asserting the Worker receives the expected model string during a simulated Task invocation.

## Files Touched
- `src/queryCastari.ts`: Subagent resolution logic and env injection.
- `reference_docs/query_wrapper.md`: Updated API and env documentation.
- `AGENTS.md`: Task behavior section.
- `build_docs/task_subagent_model_report.md`: Full design write-up.
- `claude-agent-demo/lib/env.ts`, `lib/agent/session.ts`: Env plumbing in the demo.
- `.env` (demo): Example configuration.
- `claude-agent-demo/package.json`: `predev|prebuild|prestart` scripts to sync the local package.
- `build_docs/session_report.md`: This report.

## Conclusion
The Castari wrapper now controls the Task tool’s model selection, defaulting to the user’s chosen provider/model while allowing simple overrides. Syncing the local package before each demo run ensures the new behavior is actually exercised, preventing stale builds from hitting Anthropic unintentionally.

CLAUDE_CODE_SUBAGENT_MODEL wins unconditionally. In the SDK (claude-agent-demo/node_modules/@anthropic-ai/claude-agent-sdk/cli.js, function VbA around line 699) the resolution order is literally:

  function VbA(agentModel, mainLoopModel, toolInputModel) {
    if (process.env.CLAUDE_CODE_SUBAGENT_MODEL) return process.env.CLAUDE_CODE_SUBAGENT_MODEL;
    if (toolInputModel) return LU(toolInputModel);          // Task input override
    if (!agentModel) return LU(DU1);                        // built-in default
    return agentModel === 'inherit' ? mainLoopModel : LU(agentModel);
  }