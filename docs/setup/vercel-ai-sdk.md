# Connect Vercel AI SDK to SEC EDGAR Data with MCP

The Vercel AI SDK is a TypeScript toolkit for LLM applications. Its MCP client
lives in the `@ai-sdk/mcp` package. It speaks Streamable HTTP, so it connects to
this server directly.

## Prerequisites

- Node.js 18 or later.
- `@ai-sdk/mcp` 2.0.31 or later, plus the MCP TypeScript SDK for the transport.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
npm install ai @ai-sdk/mcp @ai-sdk/openai @modelcontextprotocol/sdk
```

## Config location

There is no config file. You create the client in the module that calls the
model, for example `app/api/chat/route.ts` in a Next.js project or `index.ts` in
a plain Node script. Read the key from `process.env`.

## Configuration

```ts
import { createMCPClient } from '@ai-sdk/mcp';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';
import { openai } from '@ai-sdk/openai';
import { generateText, stepCountIs } from 'ai';

const url = new URL(
  `https://api.sec-api.io/mcp?apiKey=${process.env.SEC_API_KEY}`,
);

const mcpClient = await createMCPClient({
  transport: new StreamableHTTPClientTransport(url),
});

try {
  const tools = await mcpClient.tools();
  const { text } = await generateText({
    model: openai('gpt-4.1'),
    tools,
    stopWhen: stepCountIs(10),
    prompt: 'Which 10-K filings did Apple file in 2025?',
  });
  console.log(text);
} finally {
  await mcpClient.close();
}
```

Do not set `sessionId` on the transport. This server issues no session. To send
the key as a header instead, drop the query string and pass
`{ requestInit: { headers: { Authorization: 'Bearer ' + key } } }` as the second
argument to the transport.

## Reload

There is no daemon. Run the script again, or let the dev server hot reload. The
tool list is fetched on every `mcpClient.tools()` call.

## Verify

```ts
console.log(Object.keys(await mcpClient.tools()).length); // 49
```

Expect **49**. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The model calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **Close the client.** Call `mcpClient.close()` in a `finally` block, or in the
  `onFinish` callback of `streamText`. An open client keeps the process alive.
- **The server is stateless.** `GET` and `DELETE` on the endpoint return 404.
  The transport may log an SSE stream error. Tool calls still work, because they
  are plain POSTs.
- **Results are one text block.** JSON arrives as a string. No tool declares an
  output schema, so `structuredContent` is never populated.

## Removal

Delete the `createMCPClient` block and stop passing `tools` to `generateText`.
Then run `npm uninstall @ai-sdk/mcp @modelcontextprotocol/sdk` if nothing else
uses them.

## Source

[MCP Tools, AI SDK Core docs](https://ai-sdk.dev/docs/ai-sdk-core/mcp-tools)
