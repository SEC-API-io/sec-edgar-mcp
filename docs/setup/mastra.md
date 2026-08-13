# Connect Mastra to SEC EDGAR Data with MCP

Mastra is a TypeScript agent framework. Its `MCPClient` class manages
connections to several MCP servers and hands their tools to an agent. It tries
Streamable HTTP first, so it connects to this server directly.

## Prerequisites

- Node.js 20 or later, and a Mastra project.
- `@mastra/mcp` 1.16.0 or later.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
npm install @mastra/mcp@latest
```

## Config location

There is no JSON config. You declare the server in TypeScript. The usual file is
`src/mastra/mcp.ts`, and the agent that uses it lives in `src/mastra/agents/`.
Read the key from `process.env`. Put it in `.env` and keep `.env` out of git.

## Configuration

```ts
// src/mastra/mcp.ts
import { MCPClient } from '@mastra/mcp';

export const mcp = new MCPClient({
  id: 'sec-api',
  servers: {
    secApi: {
      url: new URL(
        `https://api.sec-api.io/mcp?apiKey=${process.env.SEC_API_KEY}`,
      ),
      timeout: 60000,
    },
  },
});
```

`url` must be a `URL` object, not a string. Attach the tools to an agent:

```ts
// src/mastra/agents/filings-agent.ts
import { Agent } from '@mastra/core/agent';
import { mcp } from '../mcp';

export const filingsAgent = new Agent({
  id: 'filings-agent',
  name: 'SEC Filings Agent',
  instructions: 'You answer questions about SEC EDGAR filings.',
  model: 'openai/gpt-4.1',
  tools: await mcp.listTools(),
});
```

To send the key as a header, drop the query string and add
`requestInit: { headers: { Authorization: 'Bearer ' + key } }`.

## Reload

Run `mastra dev` again, or save the file and let the dev server reload. The tool
list is fetched on each `listTools()` call.

## Verify

Open the Mastra playground on the dev server port, `http://localhost:4111` by
default. Pick the agent and read the tool list in the right panel. Expect **49**
tools, each name prefixed with the server key. In code, print
`Object.keys(await mcp.listTools()).length`. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The agent calls the namespaced `filing-search` tool once with
`ticker:AAPL AND formType:"10-K"`. It answers with three rows. Each row carries
a `filedAt` date and an accession number such as `0000320193-25-000073`.

## Quirks

- **The SSE fallback will not help.** `MCPClient` falls back to legacy SSE when
  the first connection fails. This server has no SSE endpoint, and `GET` returns
  404. Fix the URL or the key rather than wait for the fallback.
- **Tool names are namespaced.** `listTools()` prefixes each name with the
  server key. Use `listToolsets()` to get the tools grouped per server.
- Results arrive as one text block of stringified JSON.

## Removal

Delete the `secApi` entry from `servers` and remove the `tools` line from the
agent. Then run `npm uninstall @mastra/mcp` if nothing else uses it.

## Source

[MCP Overview, Mastra docs](https://mastra.ai/en/docs/tools-mcp/mcp-overview)
