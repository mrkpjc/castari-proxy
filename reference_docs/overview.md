The Claude Agent SDK is a powerful harness for building generally capable agents. It is built around the Claude Code CLi harness. We are building a solution for developers to use Claude Agent SDK with other providers/models. The Claude Agent SDK uses environment variables ANTHROPIC_BASE_URL and ANTHROPIC_API_KEY only. It uses primarily the query() function to run the agent which has a `options` configuration object with various possible properties. One of those properties is `env` which we can use to override the values of ANTHROPIC_BASE_URL and ANTHROPIC_API_KEY for the given request. 

So our solution will be a wrapper around query that looks at the model name and then determine what the provider is. It then will look for that provider's API key in the .env file. It then sets that value for ANTHROPIC_API_KEY. and then we set ANTHROPIC_BASE_URL to a cloudflare worker that acts as a proxy. This worker will convert the request to the correct format, receive the responses, translate it back to Anthroipc format and send to the agent.

For now, we will only focus on adding OpenRouter, which mostly follows the chat completions format. So in the query function, the developer can have the model name "or:gpt-5-mini"

The query wrapper sees that and pulls the value for OPENROUTER_API_KEY and sets that value to the ANTHROPIC_API_KEY in the env override option. The Cloudflare worker needs to handle all translation for anything that might come from the Claude Agent, tool calls, streaming, MCP, etc.. convert to OpenRouter format, run the call. And then translate the response back to Anthropic format for the agent.

One last detail is we will have the worker point to a Cloudflare AI Gateway. This will be so we can track usage and other meta data.

If the model is an Anthropic model, we wil pass that directly to Anthropic's messages API (via the Cloudflare AI Gateway) with no translation and receive the request. This should mimic exactly how the Claude Agent SDK runs by default, just capturing the requests in the Cloudflare AI Gateway.

