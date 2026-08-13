# Connect Semantic Kernel to SEC EDGAR Data with MCP

Semantic Kernel is Microsoft's SDK for AI orchestration. It turns an MCP server
into a kernel plugin. `MCPStreamableHttpPlugin` speaks Streamable HTTP, so it
connects to this server directly. Python has the fullest MCP support.

## Prerequisites

- Python 3.10 or later, and Semantic Kernel 1.28.1 or later. MCP support starts
  at that version. The current release is 1.44.1.
- The `mcp` extra.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install "semantic-kernel[mcp]"
```

## Config location

There is no config file. You declare the plugin in the module that builds the
kernel, for example `main.py` in your project root. Read the key from the
environment. Do not commit it.

## Configuration

```python
import os, asyncio
from semantic_kernel import Kernel
from semantic_kernel.connectors.mcp import MCPStreamableHttpPlugin

key = os.environ["SEC_API_KEY"]


async def main():
    async with MCPStreamableHttpPlugin(
        name="sec_api",
        description="SEC EDGAR filings and financial data",
        url=f"https://api.sec-api.io/mcp?apiKey={key}",
        load_prompts=False,
        request_timeout=35,
    ) as sec_api:
        kernel = Kernel()
        kernel.add_plugin(sec_api)
        # add a chat service and invoke the kernel here


asyncio.run(main())
```

The plugin name must be a valid identifier. Use `sec_api`, not `sec-api`. The
context manager opens and closes the connection. You can also call
`await plugin.connect()` and `await plugin.close()` yourself. To send the key as
a header, drop the query string and pass
`headers={"Authorization": f"Bearer {key}"}`.

## Reload

There is no daemon. Run the script again. The plugin lists the tools each time
it connects.

## Verify

```python
async with MCPStreamableHttpPlugin(...) as sec_api:
    print(len(sec_api.functions))  # 49
```

Expect **49** kernel functions. HTTP 401 means the key is wrong.

## First prompt

> Find the 3 most recent 10-K filings from Apple. Give the filing date and the
> accession number of each.

The kernel calls `sec_api-filing_search` once with
`ticker:AAPL AND formType:"10-K"`. It answers with three rows. Each row carries
a `filedAt` date and an accession number such as `0000320193-25-000073`.

## Quirks

- **Set `load_prompts=False`.** This server advertises tools only. Microsoft
  warns that the prompt load can hang against a server that has none.
- **Names get rewritten.** A kernel function name cannot hold a hyphen, so
  `filing-search` appears as `filing_search` under the plugin name.
- **The server is stateless.** `GET` and `DELETE` on the endpoint return 404.
- **.NET and Java have no MCP documentation yet.** Microsoft Learn still says
  "coming soon" for both. Use Python, or the newer Microsoft Agent Framework.
- Results arrive as one text block of stringified JSON.

## Removal

Delete the `MCPStreamableHttpPlugin` block and the matching `kernel.add_plugin`
call. Then run `pip install semantic-kernel` without the extra if nothing else
uses MCP.

## Source

[Give agents access to MCP Servers, Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/adding-mcp-plugins)
