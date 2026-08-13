# Connect Strands Agents to SEC EDGAR Data with MCP

Strands Agents is AWS's open-source agent SDK. Its `MCPClient` wraps any MCP
transport and hands the tools to an agent. It uses the official MCP Python SDK,
so Streamable HTTP works against this server directly.

## Prerequisites

- Python 3.10 or later. `strands-agents` 1.52.0 requires it.
- Model credentials. Strands defaults to Amazon Bedrock.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install strands-agents
```

## Config location

There is no config file. You declare the server in the module that builds the
agent, for example `agent.py` in your project root. Read the key from the
environment. Do not commit it.

## Configuration

```python
import os
from mcp.client.streamable_http import streamablehttp_client
from strands import Agent
from strands.tools.mcp import MCPClient

key = os.environ["SEC_API_KEY"]

sec_api = MCPClient(
    lambda: streamablehttp_client(url=f"https://api.sec-api.io/mcp?apiKey={key}")
)

with sec_api:
    tools = sec_api.list_tools_sync()
    agent = Agent(tools=tools)
    print(agent("Which 10-K filings did Apple file in 2025?"))
```

`MCPClient` takes a callable that builds the transport, not the transport
itself. Every call must sit inside the `with` block. To send the key as a header
instead, drop the query string and pass
`headers={"Authorization": f"Bearer {key}"}` to `streamablehttp_client`.

## Reload

There is no daemon. Run the script again. `list_tools_sync()` calls the server
each time.

## Verify

```python
with sec_api:
    print(len(sec_api.list_tools_sync()))  # 49
```

Expect **49**. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The agent calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **Stay inside the context manager.** The connection lives in a background
  thread that the `with` block owns. A tool call made after the block exits
  fails.
- **Filter the tool list.** 49 tools make a long prompt. `MCPClient` accepts
  `ToolFilters` with an `allowed` pattern list. Load only what the agent needs.
- **The server is stateless.** There is no `Mcp-Session-Id`, and `GET` and
  `DELETE` on the endpoint return 404. Never use `sse_client` against this URL.
- **Set no `elicitation_callback` and no `progress_callback`.** The server
  advertises tools only. It sends neither.
- Results arrive as one text block of stringified JSON.

## Removal

Delete the `MCPClient` block and stop passing its tools to the agent. Nothing
else needs removing. The MCP client is part of the base `strands-agents`
install.

## Source

[Model Context Protocol (MCP) Tools, Strands Agents docs](https://strandsagents.com/docs/user-guide/concepts/tools/mcp-tools/)
