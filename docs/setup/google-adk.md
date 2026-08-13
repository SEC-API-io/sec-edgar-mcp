# Connect Google ADK to SEC EDGAR Data with MCP

The Agent Development Kit is Google's open-source framework for building agents.
`McpToolset` loads the tools of an MCP server into an agent, and
`StreamableHTTPConnectionParams` speaks Streamable HTTP. It connects to this
server directly.

## Prerequisites

- Python 3.10 or later. `google-adk` 2.6.3 requires it.
- A model key, for example `GOOGLE_API_KEY` for Gemini.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install google-adk
```

## Config location

ADK loads an agent from a package directory. Put the server in the agent module,
`./sec_agent/agent.py`, and both keys in `./sec_agent/.env`. `adk web` reads
both. Keep `.env` out of git.

## Configuration

```python
# sec_agent/agent.py
import os
from google.adk.agents.llm_agent import Agent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import (
    StreamableHTTPConnectionParams,
)

root_agent = Agent(
    model="gemini-flash-latest",
    name="sec_filings_agent",
    description="Answers questions about SEC EDGAR filings.",
    instruction="Use the SEC-API tools.",
    tools=[
        McpToolset(
            connection_params=StreamableHTTPConnectionParams(
                url="https://api.sec-api.io/mcp",
                headers={
                    "Authorization": f"Bearer {os.environ['SEC_API_KEY']}",
                    "Accept": "application/json, text/event-stream",
                },
            )
        )
    ],
)
```

The key can also ride in the URL, as
`https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`. Then drop the `Authorization`
header. Java uses `StreamableHttpServerParameters.builder()`.

## Reload

Stop and restart `adk web` or `adk run`. The toolset lists the tools on each
agent start.

## Verify

Run `adk web` from the parent directory of `sec_agent`, open
`http://localhost:8000`, and pick the agent in the top-left selector. The trace
panel lists the loaded tools. Expect **49**. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The agent calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **Check the class names against your version.** Python has shipped both
  `McpToolset` and `MCPToolset`, and both `StreamableHTTPConnectionParams` and
  `StreamableHTTPServerParams`. The import path
  `google.adk.tools.mcp_tool.mcp_session_manager` is the stable one.
- **Narrow the tool list.** 49 tools make a long prompt. Use the `tool_filter`
  argument of `McpToolset` to keep only what the agent needs.
- **The server is stateless.** `GET` and `DELETE` on the endpoint return 404.
- Results arrive as one text block of stringified JSON.

## Removal

Delete the `McpToolset` entry from the `tools` list and remove the import lines.
Remove `SEC_API_KEY` from `.env`.

## Source

[MCP tools, Agent Development Kit docs](https://adk.dev/tools-custom/mcp-tools/)
