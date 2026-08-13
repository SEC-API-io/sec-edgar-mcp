# Connect Microsoft AutoGen to SEC EDGAR Data with MCP

AutoGen is Microsoft's multi-agent conversation framework. The `autogen-ext`
package adapts MCP tools into AutoGen tools. `StreamableHttpServerParams` speaks
Streamable HTTP, so it connects to this server directly.

## Prerequisites

- Python 3.10 or later. `autogen-ext` 0.7.5 requires it.
- The `mcp` extra of `autogen-ext`.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install -U "autogen-ext[mcp]" autogen-agentchat
```

## Config location

There is no config file. You declare the server in the module that builds the
agent, for example `agent.py` in your project root. Read the key from the
environment. Do not commit it.

## Configuration

```python
import os, asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.tools.mcp import StreamableHttpServerParams, mcp_server_tools

server_params = StreamableHttpServerParams(
    url="https://api.sec-api.io/mcp",
    headers={"Authorization": f"Bearer {os.environ['SEC_API_KEY']}"},
    timeout=35.0,
    terminate_on_close=False,
)


async def main():
    tools = await mcp_server_tools(server_params)
    agent = AssistantAgent(
        name="filings_analyst",
        model_client=OpenAIChatCompletionClient(model="gpt-4.1"),
        tools=tools,
        system_message="You answer questions about SEC EDGAR filings.",
    )
    print(await agent.run(task="Which 10-K filings did Apple file in 2025?"))


asyncio.run(main())
```

The key can also ride in the URL, as
`https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`. Then drop the `headers`
argument.

## Reload

There is no daemon. Run the script again. `mcp_server_tools` fetches the tool
list on every call.

## Verify

```python
tools = await mcp_server_tools(server_params)
print(len(tools))  # 49
```

Expect **49**. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The agent calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. It
answers with three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **Set `terminate_on_close=False`.** The server is stateless and issues no
  session. `DELETE` on the endpoint returns 404.
- **Watch for schema errors.** AutoGen issue #6906 reports a `TypeError` from
  `StreamableHttpMcpToolAdapter` on valid server schemas. If one tool raises it,
  drop that tool rather than the whole server.
- Results arrive as one text block of stringified JSON.

## Removal

Delete the `StreamableHttpServerParams` and `mcp_server_tools` lines and remove
`tools=tools` from the agent. Then run `pip uninstall autogen-ext` if nothing
else needs it.

## Source

[autogen_ext.tools.mcp API reference](https://microsoft.github.io/autogen/stable/reference/python/autogen_ext.tools.mcp.html)
