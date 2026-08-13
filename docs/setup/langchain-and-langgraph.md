# Connect LangChain and LangGraph to SEC EDGAR Data with MCP

LangChain builds LLM applications and LangGraph runs them as stateful graphs.
Both load MCP tools through `langchain-mcp-adapters`, which speaks Streamable
HTTP and connects to this server directly.

## Prerequisites

- Python 3.10 or later. `langchain-mcp-adapters` 0.3.2 requires it.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install langchain-mcp-adapters langchain
```

TypeScript users install `@langchain/mcp-adapters`. The field names match.

## Config location

There is no config file. You declare the server in the module that builds the
agent, for example `agent.py` in your project root. Read the key from the
environment. Do not commit it.

## Configuration

```python
import os
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient(
    {
        "sec-api": {
            "transport": "http",
            "url": f"https://api.sec-api.io/mcp?apiKey={os.environ['SEC_API_KEY']}",
        }
    }
)

tools = await client.get_tools()
agent = create_agent("openai:gpt-4.1", tools)
```

`transport` must be `"http"`. `"streamable_http"` is an alias. The hyphenated
`"streamable-http"` is rejected. To send the key as a header instead, drop the
query string and add `"headers": {"Authorization": f"Bearer {key}"}`.

## Reload

There is no daemon. Run the script again. `get_tools()` calls the server on
every run, so a change takes effect at once.

## Verify

```python
print(len(await client.get_tools()))  # 49
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
  `DELETE` on the endpoint return 404. Leave the SSE fallback off.
- **Results are one text block.** JSON arrives as a string in `content[0].text`,
  never as `structuredContent`. Errors arrive the same way, as the text
  `sec-api error: <message>` with `isError` set.

## Removal

Delete the `sec-api` entry from the `MultiServerMCPClient` dictionary. Then run
`pip uninstall langchain-mcp-adapters` if nothing else needs it.

## Source

[Model Context Protocol, LangChain docs](https://docs.langchain.com/oss/python/langchain/mcp)
