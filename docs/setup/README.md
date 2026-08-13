# Client setup guides

This server is remote. It speaks Streamable HTTP at one URL. There is nothing to
install and no stdio server to run.

```text
https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

The server is stateless. It sends no `Mcp-Session-Id`, and it answers `POST`
only. `GET` and `DELETE` return HTTP 404. A client that demands an SSE stream
needs the [mcp-remote bridge](./mcp-remote-bridge.md). The server exposes
**49 tools**. [`compensation`](../tools/compensation.md) instead.

This directory holds one page per client. **55 pages.** Each page gives the
config location, the config itself, a reload step, a verify step, a first prompt
and the client's quirks. Find your client in the
[support matrix](#support-matrix) below.

## Three config shapes

Almost every client takes one of three shapes. Read the shape here, then open
your client's page for the exact key names.

### 1. Remote HTTP JSON

The common form. The client connects straight to the URL.

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

To keep the key out of the URL, send a header:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

The key names drift between clients. The root key is `mcpServers` in most
clients, `servers` in VS Code and `context_servers` in Zed. The URL key is
`url`, `httpUrl` in Gemini CLI and Qwen Code, `serverUrl` in Windsurf and `uri`
in Goose. The transport value is `http`, `streamableHttp` in Cline,
`streamable-http` in Roo Code and LibreChat, `streamable_http` in Goose and
`streamable` in AnythingLLM. The Note column names the difference for each
client.

### 2. The mcp-remote bridge

Use this shape when the config takes only `command` and `args`, or when the
client hangs waiting for an SSE stream.

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "--transport",
        "http-only"
      ]
    }
  }
}
```

`--transport http-only` matters. The default tries HTTP first and then falls
back to SSE, which this server never serves. See
[mcp-remote bridge](./mcp-remote-bridge.md).

### 3. A CLI one-liner

Several clients write the config for you. Quote the URL, because it carries a
query string.

```bash
claude mcp add --transport http sec-api "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
```

The same shape works for `codex mcp add sec-api --url ...`, `gemini mcp add`,
`qwen mcp add --transport http`, `openhands mcp add --transport http`,
`openclaw mcp add` and `smithery mcp add`. Flags differ. Read the client page.

## Support matrix

Status **Vendor docs** means the page was written from current vendor
documentation on 2026-08-13. No page was tested against a running client. The
count of 49 tools comes from the server registry, not from a reading inside a
client.

### Anthropic Claude clients

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [Claude Code](./claude-code.md) | Terminal agent | Yes | CLI command, or JSON file | Vendor docs | `claude mcp add --transport http`. Three scopes. `/mcp` shows the tool count. |
| [Claude Desktop](./claude-desktop.md) | Desktop app | Yes, through a custom connector | UI form, JSON for the bridge | Vendor docs | `claude_desktop_config.json` is documented for stdio only. Use a connector or `mcp-remote`. |
| [Claude web and mobile](./claude-web-and-mobile.md) | Web and mobile app | Yes | UI form | Vendor docs | Add it on the web under Customize > Connectors. Mobile inherits it. |
| [Claude Agent SDK](./claude-agent-sdk.md) | SDK, TypeScript and Python | Yes | Code, or JSON file | Vendor docs | `type: "http"` plus an Authorization header. `allowedTools` needs `mcp__sec-api__*`. |
| [Anthropic Messages API](./anthropic-messages-api.md) | API | Yes | Code, per request | Vendor docs | Beta header `mcp-client-2025-11-20`. `mcp_servers` plus an `mcp_toolset` entry in `tools`. |

### OpenAI clients

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [ChatGPT developer mode](./chatgpt-developer-mode.md) | Web app | Yes | UI form | Vendor docs | Settings > Security and login > Developer mode. Add the URL at chatgpt.com/plugins. |
| [OpenAI Codex CLI](./codex-cli.md) | Terminal agent | Yes | CLI command or TOML file | Vendor docs | `codex mcp add sec-api --url ...` writes `[mcp_servers.sec-api]`. `bearer_token_env_var` hides the key. |
| [OpenAI Agents SDK](./openai-agents-sdk.md) | SDK, Python and TypeScript | Yes | Code | Vendor docs | `MCPServerStreamableHttp`, or `HostedMCPTool`. Raise both Python 5 s timeouts to 60. |
| [OpenAI Responses API](./openai-responses-api.md) | API | Yes | Code, per request | Vendor docs | A `tools[]` entry of type `mcp`. Count the tools in the `mcp_list_tools` output item. |

### Coding agents and IDEs

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [Cursor](./cursor.md) | IDE | Yes | JSON file | Vendor docs | `.cursor/mcp.json` with a bare `url`. No `type` field. `${env:...}` works in `url`. |
| [VS Code with GitHub Copilot](./vs-code-copilot.md) | IDE extension | Yes | JSON file | Vendor docs | Root key `servers` with `"type": "http"`. Needs VS Code 1.102 or later and Agent mode. |
| [Windsurf](./windsurf.md) | IDE | Yes | JSON file | Vendor docs | `~/.codeium/windsurf/mcp_config.json` with `serverUrl`. Cascade caps at 100 tools. |
| [Zed](./zed.md) | IDE | Yes | JSON file | Vendor docs | `context_servers` in `settings.json`, with `url` and optional `headers`. |
| [JetBrains AI Assistant](./jetbrains-ai-assistant.md) | IDE plugin | Yes | UI form | Vendor docs | Paste `mcpServers` JSON in Settings > Tools > AI Assistant > MCP. The key rides in the URL. |

### Open-source coding agents and IDE extensions

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [Cline](./cline.md) | IDE extension | Yes | JSON file or UI form | Vendor docs | The transport value is camelCase `streamableHttp`. No `type` falls back to SSE. |
| [Roo Code](./roo-code.md) | IDE extension | Yes | JSON file | Vendor docs | `type` is required and hyphenated: `streamable-http`. Added in 3.19.2. |
| [Continue](./continue.md) | IDE extension | Yes | YAML file | Vendor docs | `~/.continue/config.yaml`. Tools load in agent mode only. Saving reloads the config. |
| [Aider](./aider.md) | Terminal agent | No | None | Vendor docs | Aider has no MCP client. The page gives two workarounds without MCP. |
| [OpenHands](./openhands.md) | Agent platform | Yes | CLI command or TOML file | Vendor docs | `openhands mcp add --transport http`. The app TOML takes `api_key` but no headers. |
| [Qwen Code](./qwen-code.md) | Terminal agent | Yes | CLI command or JSON file | Vendor docs | Remote servers use `httpUrl`, not `url`. Discovery times out after 5 seconds. |

### Terminal agents and CLI clients

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [Google Gemini CLI](./google-gemini-cli.md) | Terminal agent | Yes | JSON file or CLI command | Vendor docs | Use the `httpUrl` key. The `url` key means SSE. |
| [Goose](./goose.md) | Desktop and CLI agent | Yes | CLI wizard writing YAML | Vendor docs | The endpoint key is `uri` and the type is `streamable_http`. |
| [Amazon Q Developer CLI](./amazon-q-developer-cli.md) | Terminal agent | Yes | JSON file | Vendor docs | `q mcp add` writes stdio entries only. Put the http block in `mcp.json` by hand. |
| [Warp](./warp.md) | Terminal | Yes | UI form with a JSON paste | Vendor docs | Put auth in `headers`. Warp strips `env` from a URL server. |
| [OpenClaw](./openclaw.md) | Terminal agent | Yes | CLI command or JSON file | Vendor docs | `openclaw mcp probe` prints a live tool count. Set `supportsParallelToolCalls` false. |
| [Hermes Agents](./hermes-agents.md) | Agent runtime | Yes | YAML file | Vendor docs | A `url` with no transport key means Streamable HTTP. Tool names become `mcp__sec_api__*`. |

### Chat apps and local-model desktop UIs

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [Perplexity](./perplexity.md) | Web app | Yes | UI form | Vendor docs | Settings > Connectors > Custom connector > Remote. Needs Pro, Max or Enterprise. |
| [Mistral Le Chat](./mistral-le-chat.md) | Web app | Yes | UI form | Vendor docs | Intelligence > Connectors > Custom MCP Connector. Admin only. The name field rejects hyphens. |
| [LM Studio](./lm-studio.md) | Desktop app | Yes | JSON file | Vendor docs | `~/.lmstudio/mcp.json`, edited from the Program tab. Needs 0.3.17 b10 or later. |
| [Jan](./jan.md) | Desktop app | Yes | UI form | Vendor docs | Settings > MCP Servers, Transport HTTP. Raise Jan's 30 second tool call timeout. |
| [Msty](./msty.md) | Desktop app | Yes | UI form | Vendor docs | Toolbox > Tools > Add New Tool, type HTTP. Add the tool to a Toolset, then pick it in chat. |
| [Cherry Studio](./cherry-studio.md) | Desktop app | Yes | UI form | Vendor docs | Settings > MCP > Add Server > Quick Create, Type Streamable HTTP. Update past 1.5.1. |
| [LibreChat](./librechat.md) | Self-hosted web app | Yes | YAML file | Vendor docs | `librechat.yaml` with type `streamable-http` and `timeout` 60000. Restart to load it. |
| [Open WebUI](./open-webui.md) | Self-hosted web app | Yes | UI form | Vendor docs | Admin Settings > Integrations, Type MCP (Streamable HTTP). Needs v0.6.31 or later. |
| [AnythingLLM](./anythingllm.md) | Desktop and self-hosted app | Yes | JSON file | Vendor docs | `plugins/anythingllm_mcp_servers.json`. Set `"type": "streamable"` or it assumes SSE. |

### Agent frameworks and SDKs

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [LangChain and LangGraph](./langchain-and-langgraph.md) | SDK, Python and TypeScript | Yes | Code | Vendor docs | `MultiServerMCPClient`. Transport is `http` or `streamable_http`. The hyphen form is rejected. |
| [LlamaIndex](./llamaindex.md) | SDK, Python | Yes | Code | Vendor docs | `BasicMCPClient` plus `McpToolSpec`. The transport follows the URL shape. |
| [Pydantic AI](./pydantic-ai.md) | SDK, Python | Yes | Code | Vendor docs | `MCPToolset` passed in `toolsets=[]`. Install the `[mcp]` extra. |
| [Vercel AI SDK](./vercel-ai-sdk.md) | SDK, TypeScript | Yes | Code | Vendor docs | `createMCPClient` from `@ai-sdk/mcp`. `experimental_createMCPClient` is stale. |
| [Mastra](./mastra.md) | SDK, TypeScript | Yes | Code | Vendor docs | `MCPClient` in `src/mastra/mcp.ts`. `url` must be a `URL` object. |
| [CrewAI](./crewai.md) | SDK, Python | Yes, bridge may be needed | Code | Vendor docs | `MCPServerAdapter` as a context manager. Fall back to `mcp-remote` if it waits for SSE. |
| [Microsoft AutoGen](./microsoft-autogen.md) | SDK, Python | Yes | Code | Vendor docs | `StreamableHttpServerParams` plus `mcp_server_tools`. Raise the 30.0 s default to 35.0. |
| [Semantic Kernel](./semantic-kernel.md) | SDK, Python | Yes | Code | Vendor docs | `MCPStreamableHttpPlugin` on a Kernel. Set `load_prompts=False`. The server has no prompts. |
| [Spring AI](./spring-ai.md) | SDK, Java | Yes | `application.yml` plus a bean | Vendor docs | Issue #6505 strips the URL query, so send the key in an Authorization header. |
| [Google ADK](./google-adk.md) | SDK, Python | Yes | Code | Vendor docs | `McpToolset` with `StreamableHTTPConnectionParams` in `agent.py`. Class names drift by version. |
| [Strands Agents](./strands-agents.md) | SDK, Python | Yes | Code | Vendor docs | `MCPClient` wraps `streamablehttp_client`. Keep every call inside the `with` block. |

### Automation platforms, packaging and infrastructure

| Client | Type | Remote HTTP MCP | Config style | Status | Note |
| ------ | ---- | --------------- | ------------ | ------ | ---- |
| [n8n](./n8n.md) | Automation platform | Yes | UI form, MCP Client Tool node | Vendor docs, node source | Field names come from `McpClientTool.node.ts`. The doc page still shows the old SSE field. |
| [Dify](./dify.md) | LLM app platform | Yes | UI form | Vendor docs | Tools > MCP > Add MCP Server (HTTP). HTTPS only. The Server Identifier is permanent. |
| [Flowise](./flowise.md) | Automation platform | Yes | JSON in a node field | Vendor docs | Custom MCP node, `url` plus `headers` and a `$vars` variable. Never the stdio form. |
| [Zapier](./zapier.md) | Automation platform | Yes | UI form | Vendor docs | Apps > Add connection > MCP Client. Beta. Tool calls only. It has a Bearer Token field. |
| [Docker MCP Toolkit](./docker-mcp-toolkit.md) | Gateway | Yes | YAML entry plus CLI | Vendor docs, entry spec | The Desktop UI has no remote URL field. Use the CLI and a `type: remote` entry. |
| [Smithery](./smithery.md) | Registry CLI | Yes | CLI command | Vendor docs, CLI source | `smithery mcp add` with `--id`, `--name`, `--headers` and `--client`. |
| [MCP Inspector](./mcp-inspector.md) | Debug tool | Yes | CLI flags or JSON file | Vendor docs | `--server-url` with `--transport http`. The catalog sits at `~/.mcp-inspector/mcp.json`. |
| [mcp-remote bridge](./mcp-remote-bridge.md) | stdio bridge | Bridge | JSON file, in the host client | Vendor docs, README | Pass `--transport http-only`. The default falls back to SSE, which this server never serves. |
| [AWS Bedrock AgentCore Gateway](./aws-bedrock-agentcore-gateway.md) | Managed gateway | Yes | CLI command | Vendor docs | `create-gateway-target`. Gateway renames every tool to `targetName___toolName`. |

## What could not be verified

No client on this list was installed here. Every page comes from vendor
documentation read on 2026-08-13. Treat UI labels, menu paths and version
numbers as current at that date. These pages carry a named gap:

- [Claude Desktop](./claude-desktop.md). The vendor never states whether
  `claude_desktop_config.json` accepts a remote `url`. The Windows MSIX config
  path comes from a GitHub issue, not from the docs.
- [Claude Desktop](./claude-desktop.md) and
  [Claude web and mobile](./claude-web-and-mobile.md). No source confirms that a
  custom connector accepts a server with no OAuth and a key in the URL query.
  The Request headers field is in a slow beta.
- [Anthropic Messages API](./anthropic-messages-api.md). The docs never say
  whether the API prefixes `authorization_token` with `Bearer `.
- [ChatGPT developer mode](./chatgpt-developer-mode.md). The UI name changed
  from connectors to apps to plugins. The help article blocks fetchers. No
  removal step is documented.
- [OpenAI Codex CLI](./codex-cli.md). Third-party guides show a `type` key that
  the vendor config reference does not list. The page uses `url` alone.
- [Cursor](./cursor.md). The tool cap near 40 is a community report, not a
  vendor number. This server alone exposes 49 tools.
- [Windsurf](./windsurf.md). Ownership moved to Cognition and the vendor now
  marks `mcp_config.json` as legacy. The Windows path is derived, not stated.
- [Zed](./zed.md), [Cline](./cline.md) and [LM Studio](./lm-studio.md). None
  documents where the tool list and count appear, so the verify step is
  inferred from the panel layout.
- [JetBrains AI Assistant](./jetbrains-ai-assistant.md). No `headers` field, no
  minimum version and no delete action are documented. The agent-mode
  requirement is a community report.
- [Roo Code](./roo-code.md). An install upgraded from Roo Cline may still use
  `cline_mcp_settings.json`. No cutover version is published.
- [OpenHands](./openhands.md). The JSON shape for an http server is not
  published, so the page uses the CLI instead.
- [Goose](./goose.md). The docs show `url`, but the source deserializes `uri`.
  The page follows the source. One live test would settle it.
- [Amazon Q Developer CLI](./amazon-q-developer-cli.md). The `q mcp remove`
  flags are inferred from the documented add syntax.
- [Warp](./warp.md) and [Mistral Le Chat](./mistral-le-chat.md). The dialog
  labels and the removal control are not documented.
- [Perplexity](./perplexity.md). The help article blocks direct fetch. The
  steps come from the search index summary plus vendor-quoting third parties.
- [Cherry Studio](./cherry-studio.md). The Import from JSON shape is not
  documented and the old docs page is gone. Use Quick Create.
- [Jan](./jan.md), [Msty](./msty.md) and [AnythingLLM](./anythingllm.md). None
  publishes a minimum version for remote MCP.
- [Pydantic AI](./pydantic-ai.md). A higher-level MCP capability now sits above
  `MCPToolset`. The page documents `MCPToolset`, the explicit path.
- [CrewAI](./crewai.md). Whether current `crewai-tools` still needs a GET SSE
  stream is unconfirmed. Issue #3230 was closed as not planned.
- [Semantic Kernel](./semantic-kernel.md) and [Spring AI](./spring-ai.md). The
  plugin class and the request customizer bean come from source and community
  READMEs, not from the vendor reference pages.
- [n8n](./n8n.md). The vendor node page is stale. Open issue 24967 says the
  Server Transport dropdown is ignored.
- [Docker MCP Toolkit](./docker-mcp-toolkit.md). The profile flag is composed
  from two documented flags. No disconnect verb is documented.
- [AWS Bedrock AgentCore Gateway](./aws-bedrock-agentcore-gateway.md). The CLI
  parameters for `synchronize-gateway-targets` are not shown in the docs.
- [Aider](./aider.md). Aider has no MCP client at all. Two pull requests are
  closed unmerged and one is open.

## Next steps

- [Getting started](../getting-started.md). Key, config and first query.
- [Tool reference](../tools/README.md). All 49 tools, one page each.
- [Transport](../transport.md). Streamable HTTP, the stateless design and the
  stdio bridge.
- [Authentication](../authentication.md). Key handling and rotation.
- [Limits and errors](../limits-and-errors.md). Payload sizes, paging and error text.
