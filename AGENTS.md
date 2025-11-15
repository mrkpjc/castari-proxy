Project Overview:
The Claude Agent SDK is a powerful harness for building generally capable agents. It is built around the Claude Code CLi harness. We are building a solution for developers to use Claude Agent SDK with other providers/models. The Claude Agent SDK uses environment variables ANTHROPIC_BASE_URL and ANTHROPIC_API_KEY only. It uses primarily the query() function to run the agent which has a `options` configuration object with various possible properties. One of those properties is `env` which we can use to override the values of ANTHROPIC_BASE_URL and ANTHROPIC_API_KEY for the given request. 

So our solution will be a wrapper around query that looks at the model name and then determine what the provider is. It then will look for that provider's API key in the .env file. It then sets that value for ANTHROPIC_API_KEY. and then we set ANTHROPIC_BASE_URL to a cloudflare worker that acts as a proxy. This worker will convert the request to the correct format, receive the responses, translate it back to Anthroipc format and send to the agent.

For now, we will only focus on adding OpenRouter, which mostly follows the chat completions format. So in the query function, the developer can have the model name "or:gpt-5-mini"

The query wrapper sees that and pulls the value for OPENROUTER_API_KEY and sets that value to the ANTHROPIC_API_KEY in the env override option. The Cloudflare worker needs to handle all translation for anything that might come from the Claude Agent, tool calls, streaming, MCP, etc.. convert to OpenRouter format, run the call. And then translate the response back to Anthropic format for the agent.

One last detail is we will have the worker point to a Cloudflare AI Gateway. This will be so we can track usage and other meta data.

If the model is an Anthropic model, we wil pass that directly to Anthropic's messages API (via the Cloudflare AI Gateway) with no translation and receive the request. This should mimic exactly how the Claude Agent SDK runs by default, just capturing the requests in the Cloudflare AI Gateway.

## Task Tool Subagents

- The SDK's Task tool spawns nested agents that historically defaulted to Anthropic Haiku, even when the main loop used a different provider or tier. This behavior created confusing mismatches and required Anthropic credentials for every project.
- Castari now determines the Task model in the `queryCastari` wrapper before handing control to the SDK. By default, Task inherits the same `options.model` that the developer supplied for the primary agent, so OpenRouter-only projects work out of the box.
- The inheritance behavior can be overridden globally by setting `CASTARI_SUBAGENT_MODEL` (or by passing `subagentModel` in code). Accepted values include:
  - `inherit`/`auto` – the default; reuse the main agent model verbatim (works for Anthropic and OpenRouter slugs alike).
  - Shorthand tiers (`haiku`, `sonnet`, `opus`) – Castari expands these to the latest Claude 4.5/4.1 release IDs before the SDK sees them.
  - Fully-qualified model IDs such as `claude-sonnet-4-5-20250929` or `or:gpt-5-mini`.
- Whatever value we resolve is injected into `CLAUDE_CODE_SUBAGENT_MODEL`, which the Claude Agent SDK already prioritizes when launching the Task tool. No Worker changes are required; routing follows the model string automatically.

# DEVELOPEMENT GUIDELINES

## Rules
- We are building a production application.
- Use TS best practices. Write clean, maintainable, and efficient code. Really do your best. Also be prepared to do new things and create novel solutions.
- Never use simplified or placeholder code.
- Be maximally thorough to ensure proper and complete solutions.
- When we come across bugs, don't witch hunt for the source. This can lead to goose chasing. Instead, take your time to really investigate the source code to gain a granular and complete understanding.
- We are never in a rush. Take your time. Prioritize quality.
- Think Hard

## Testing
- Tests are important. We want good test coverage for what you would expect from a production application. A common issue that can arise is creating tests that aren't useful, or ruining source code to pass an arbitrary test. Tests are a tool for us, we should use them correctly to help us verify our code. But the source code should always be optimized to our designs and not to the tests. Again we never want to use simplified or placeholder code to pass tests. And we should never hard code tests to pass.

IMPORTANT: Please constantly re-reference documentation for the information you need for each step. If you are missing anything, notify the user and request information, or attempt to web search to find yourself.

DOCS:
CASTARI_PROXY/
├── build_docs/
│   └── requirements.md
├── reference_docs/
│   ├── Anthropic/
│   │   ├── Claude_Agent_SDK.md
│   │   ├── Messages.md
│   │   └── Skills.md
│   ├── OpenRouter/
│   │   ├── API_Overview.md
│   │   ├── MCP.md
│   │   ├── Reasoning_Tokens.md
│   │   ├── Tool_Calling.md
│   │   └── Web_Search.md
│   ├── overview.md
│   └── query_wrapper.md
├── src/
├── AGENTS.md
└── CLAUDE.md
