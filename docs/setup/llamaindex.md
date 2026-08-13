# Connect LlamaIndex to SEC EDGAR Data with MCP

LlamaIndex builds RAG pipelines and agent workflows. The `llama-index-tools-mcp`
integration turns an MCP server into a list of LlamaIndex tools. It supports
Streamable HTTP, so it connects to this server directly.

## Prerequisites

- Python 3.10 or later. `llama-index-tools-mcp` 0.5.0 requires it.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install llama-index-tools-mcp
```

## Config location

There is no config file. You declare the server in the module that builds the
agent, for example `agent.py` in your project root. Read the key from the
environment. Do not commit it.

## Configuration

```python
import os, asyncio
from llama_index.tools.mcp import BasicMCPClient, McpToolSpec
from llama_index.core.agent.workflow import FunctionAgent
from llama_index.llms.openai import OpenAI

key = os.environ["SEC_API_KEY"]
mcp_client = BasicMCPClient(f"https://api.sec-api.io/mcp?apiKey={key}")

tool_spec = McpToolSpec(client=mcp_client)
tools = asyncio.run(tool_spec.to_tool_list_async())

agent = FunctionAgent(
    tools=tools,
    llm=OpenAI(model="gpt-5-mini"),
    system_prompt="You answer questions about SEC EDGAR filings.",
)
```

`BasicMCPClient` picks the transport from the argument shape. A URL that does
not end in `/sse` gets Streamable HTTP, which is what this server needs.

49 tools make a long prompt. Narrow the list with
`McpToolSpec(client=mcp_client, allowed_tools=["filing-search", "extractor"])`.
Leave `include_resources` off. This server has no resources.

## Reload

There is no daemon. Run the script again. The tool list is fetched on every run.

## Verify

```python
print(len(tools))  # 49
```

Expect **49** when you set no filter. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The agent calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **The server is stateless.** There is no `Mcp-Session-Id`, and `GET` and
  `DELETE` on the endpoint return 404. Never point the client at an `/sse` URL.
- **Results are one text block.** JSON arrives as a string, never as
  `structuredContent`. Errors arrive the same way, as the text
  `sec-api error: <message>` with `isError` set.

## Removal

Delete the `BasicMCPClient` and `McpToolSpec` lines and drop the tools from the
agent. Then run `pip uninstall llama-index-tools-mcp` if nothing else needs it.

## Source

[Using MCP Tools with LlamaIndex](https://developers.llamaindex.ai/python/framework/module_guides/mcp/llamaindex_mcp/)
