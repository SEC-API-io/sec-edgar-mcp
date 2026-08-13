# Connect Pydantic AI to SEC EDGAR Data with MCP

Pydantic AI is a typed agent framework from the Pydantic team. It ships a
first-class MCP client. `MCPToolset` speaks Streamable HTTP, so it connects to
this server directly.

## Prerequisites

- Python 3.10 or later. `pydantic-ai-slim` 2.29.0 requires it.
- The `mcp` extra. The MCP client is not in the base install.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install "pydantic-ai-slim[mcp]"
```

## Config location

There is no config file. You declare the server in the module that builds the
agent, for example `agent.py` in your project root. Read the key from the
environment. Do not commit it.

## Configuration

```python
import os
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPToolset

key = os.environ["SEC_API_KEY"]
sec_api = MCPToolset(f"https://api.sec-api.io/mcp?apiKey={key}")

agent = Agent("openai:gpt-5.2", toolsets=[sec_api])


async def main():
    async with agent:
        result = await agent.run("Which 10-K filings did Apple file in 2025?")
        print(result.output)
```

`async with agent:` opens and closes the connection. Calls made outside that
block fail.

A header works instead of the query string. Drop the query string and pass
`headers={"Authorization": f"Bearer {key}"}`. If another server exports the same
tool name, separate them with `MCPToolset(url).prefixed('sec')`.

## Reload

There is no daemon. Run the script again. The tool list is fetched each time the
agent context opens.

## Verify

Pydantic AI prints no tool count. Count the tools on the wire instead:

```bash
curl -s "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' \
  | python3 -c 'import json,sys; print(len(json.load(sys.stdin)["result"]["tools"]))'
```

Expect **49**. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The agent calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **The server is stateless.** There is no `Mcp-Session-Id`, and `GET` and
  `DELETE` on the endpoint return 404.
- **Set no sampling model for this toolset.** The server advertises tools only.
  It never sends sampling, elicitation or logging requests.
- **Results are one text block.** JSON arrives as a string, never as
  `structuredContent`. Errors arrive as the text `sec-api error: <message>`.

## Removal

Remove the toolset from `toolsets=[...]` and delete the `MCPToolset` line. Then
run `pip install pydantic-ai-slim` without the extra if nothing else uses MCP.

## Source

[Client, Pydantic AI MCP docs](https://pydantic.dev/docs/ai/mcp/client/)
