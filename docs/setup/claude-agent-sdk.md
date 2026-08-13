# Connect Claude Agent SDK to SEC EDGAR Data with MCP

The Claude Agent SDK builds agents on the same engine as Claude Code. It ships
for TypeScript and Python. It supports Streamable HTTP servers directly, so it
connects to this server without a bridge.

## Prerequisites

- Node.js 18 or later for TypeScript, or Python 3.10 or later.
- An Anthropic API key in `ANTHROPIC_API_KEY`, for the model.
- A sec-api key from [sec-api.io](https://sec-api.io/profile), for this server.

```bash
npm install @anthropic-ai/claude-agent-sdk   # TypeScript
pip install claude-agent-sdk                 # Python
```

## Config location

You declare the server in code, in the `mcpServers` option. The SDK also reads
`.mcp.json` from the project root when `settingSources` includes `"project"`.
Default `query()` options already include it.

## Configure

Keep the key in an environment variable. Do not paste it into source.

```bash
export SEC_API_KEY=YOUR_API_KEY
```

TypeScript:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "List the 5 most recent 10-K filings from Apple.",
  options: {
    mcpServers: {
      "sec-api": {
        type: "http",
        url: "https://api.sec-api.io/mcp",
        headers: { Authorization: `Bearer ${process.env.SEC_API_KEY}` }
      }
    },
    allowedTools: ["mcp__sec-api__*"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

Python:

```python
import asyncio, os
from claude_agent_sdk import query, ClaudeAgentOptions

options = ClaudeAgentOptions(
    mcp_servers={
        "sec-api": {
            "type": "http",
            "url": "https://api.sec-api.io/mcp",
            "headers": {"Authorization": f"Bearer {os.environ['SEC_API_KEY']}"},
        }
    },
    allowed_tools=["mcp__sec-api__*"],
)


async def main():
    async for message in query(prompt="Recent 10-K filings from Apple.", options=options):
        print(message)


asyncio.run(main())
```

## Restart

There is no restart. The SDK connects each time you call `query()`.

## Verify

Read the `system` message with subtype `init`. Filter its `tools` array for
names that start with `mcp__sec-api__`. Expect 49.

```typescript
if (message.type === "system" && message.subtype === "init") {
  console.log(message.tools.filter((n) => n.startsWith("mcp__sec-api__")).length);
  console.log(message.mcp_servers);
}
```

`mcp_servers` reports the status. `connected` is good. `failed` means the key or
the network is wrong. `pending` is not a failure.

## First prompt

> List the 5 most recent 10-K filings from Apple.

The agent calls `mcp__sec-api__filing-search` once. `message.result` holds five
rows. Each row gives the accession number, the filing date, and the document
URL.

## Quirks

- Tool names carry the server name, for example
  `mcp__sec-api__filing-search`. Tool names hold hyphens, so a wildcard in
  `allowedTools` is easier than a list.
- Without `allowedTools`, the agent sees the tools but cannot call them.
- The SDK never runs an OAuth flow. This server does not need one. Pass the key
  in `headers`, or append `?apiKey=YOUR_API_KEY` to the URL.
- A tool result larger than 25,000 tokens goes to a file, and the agent gets the
  path. Raise the cap with `MAX_MCP_OUTPUT_TOKENS`.
- Every result arrives as one text block with stringified JSON. No tool declares
  an `outputSchema`, so there is no `structuredContent`. Parse the text.
- A failed call returns `isError: true` and the text `sec-api error: <message>`.

## Remove

Delete the `sec-api` entry from `mcpServers`, and delete
`"mcp__sec-api__*"` from `allowedTools`. If you used `.mcp.json`, delete the
block there.

## Source

[Connect to external tools with MCP, Agent SDK](https://code.claude.com/docs/en/agent-sdk/mcp)
