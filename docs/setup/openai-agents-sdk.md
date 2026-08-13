# Connect OpenAI Agents SDK to SEC EDGAR Data with MCP

The Agents SDK builds agents in Python or TypeScript. It reaches an MCP server in
two ways. Your process can call the server over Streamable HTTP, or the OpenAI
Responses API can call it for you as a hosted tool.

## Prerequisites

- Python: `pip install openai-agents`. The SDK supports `mcp>=1.19.0,<3`.
- TypeScript: `npm install @openai/agents`.
- An OpenAI API key in `OPENAI_API_KEY`.
- A sec-api.io API key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

There is no config file. You declare the server in code.

## Option A: your process calls the server

Use this to keep the tool traffic in your own network, or to drive a model that
is not an OpenAI Responses model.

```python
import asyncio
import os
from agents import Agent, Runner
from agents.mcp import MCPServerStreamableHttp

async def main() -> None:
    async with MCPServerStreamableHttp(
        name="sec-api",
        params={
            "url": "https://api.sec-api.io/mcp",
            "headers": {"Authorization": f"Bearer {os.environ['SEC_API_KEY']}"},
            "timeout": 60,
        },
        client_session_timeout_seconds=60,
        cache_tools_list=True,
    ) as server:
        print(len(await server.list_tools()))
        agent = Agent(
            name="EDGAR analyst",
            instructions="Use the sec-api tools.",
            mcp_servers=[server],
        )
        result = await Runner.run(agent, "Apple's three most recent 10-K filings?")
        print(result.final_output)

asyncio.run(main())
```

The TypeScript form takes the same values. Pass the server in `mcpServers`, then
call `connect()` before the run and `close()` after it.

```typescript
import { MCPServerStreamableHttp } from '@openai/agents';

const server = new MCPServerStreamableHttp({
  url: 'https://api.sec-api.io/mcp',
  name: 'sec-api',
  requestInit: {
    headers: { Authorization: `Bearer ${process.env.SEC_API_KEY}` },
  },
  cacheToolsList: true,
  clientSessionTimeoutSeconds: 60,
});
```

## Option B: hosted MCP tool

OpenAI lists and calls the tools. Your process never opens a connection.

```python
from agents import Agent, HostedMCPTool

agent = Agent(
    name="EDGAR analyst",
    instructions="Use the sec-api tools.",
    tools=[
        HostedMCPTool(
            tool_config={
                "type": "mcp",
                "server_label": "sec_api",
                "server_url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
                "require_approval": "never",
            }
        )
    ],
)
```

TypeScript uses `hostedMcpTool({ serverLabel: 'sec_api', serverUrl: '...' })`.

## Restart

Not applicable. Each run reads the current code.

## Verify

`list_tools()` in Python and `listTools()` in TypeScript return the tool list.
Expect **49** entries.

## First prompt

> Which 8-K filings from 2024 mention quantum computing?

Expect a short list of filings with company names and filed dates. The agent
calls `full-text-search` and reads one text block of stringified JSON.

## Quirks

- **Raise the Python timeouts.** `params["timeout"]` and
  `client_session_timeout_seconds` both default to 5 seconds. Set both to 60.
- **Set an explicit `name`.** The TypeScript SDK derives a server name from the
  URL and strips credentials and query parameters first.
- **Prefer the header over the query string** in option A. The URL then holds no
  secret to leak into a trace.
- **No structured content.** No tool declares an `outputSchema`, so
  `use_structured_content` and `useStructuredContent` change nothing.
- **Errors arrive as text, not exceptions.** A failed call returns `isError` and
  the text `sec-api error: <message>`. A bad key returns HTTP 401.
- **Filter the tools.** 49 definitions cost input tokens on every turn. Use
  `tool_filter` or `toolFilter`.
- **Option B sends the key to OpenAI.** The hosted route needs the key in the
  tool config. Use option A if that is not acceptable.

## Removal

Delete the server from `mcp_servers`, or delete the `HostedMCPTool` from `tools`.
Rotate the key at sec-api.io if it reached a log.

## Source

[Python MCP guide](https://openai.github.io/openai-agents-python/mcp/) and
[TypeScript MCP guide](https://openai.github.io/openai-agents-js/guides/mcp/),
read 2026-08-13.
