# Connect the Anthropic Messages API to SEC EDGAR Data with MCP

The MCP connector lets the Messages API call a remote MCP server for you. You
write no MCP client. Claude runs the tool calls inside the request. This server
exposes tools only, which is all the connector supports.

## Prerequisites

- An Anthropic API key in `ANTHROPIC_API_KEY`.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).
- The beta header `anthropic-beta: mcp-client-2025-11-20`. The older
  `mcp-client-2025-04-04` header is deprecated.
- The connector runs on the Claude API, Claude Platform on AWS, and Microsoft
  Foundry with a Hosted on Anthropic deployment. It does not run on Amazon
  Bedrock or Google Cloud.

## Config location

There is no config file. You declare the server in each request. Two fields work
together:

- `mcp_servers` holds the connection.
- `tools` holds one `mcp_toolset` object per server.

## Configure

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: mcp-client-2025-11-20" \
  -d '{
    "model": "claude-opus-5",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "List the 5 most recent 10-K filings from Apple."}],
    "mcp_servers": [
      {
        "type": "url",
        "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "name": "sec-api"
      }
    ],
    "tools": [
      {"type": "mcp_toolset", "mcp_server_name": "sec-api"}
    ]
  }'
```

Python:

```python
import anthropic

client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "List the 5 most recent 10-K filings from Apple."}],
    mcp_servers=[
        {
            "type": "url",
            "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
            "name": "sec-api",
        }
    ],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "sec-api"}],
    betas=["mcp-client-2025-11-20"],
)
print(response)
```

## Restart

There is nothing to restart. Every request carries its own server list.

## Verify

Send this prompt and count the names in the answer. Expect 49.

```json
{"role": "user", "content": "List every tool you have from sec-api."}
```

A wrong key makes the connection fail with HTTP 401 from this server. The API
does not report a tool count field, so counting the names is the check.

## First prompt

> List the 5 most recent 10-K filings from Apple.

The response holds an `mcp_tool_use` block that names `filing-search` and
`server_name: "sec-api"`, then an `mcp_tool_result` block with one text item,
then Claude's text answer with five rows.

## Quirks

- 49 tool definitions enter the prompt on every request. Set
  `default_config.defer_loading` to `true` and add the tool search tool, or
  disable the tools you do not need. Both cut input tokens.
- `authorization_token` is documented for OAuth bearer tokens. This server also
  accepts `Authorization: Bearer <key>`, so that field should carry the API key.
  The key in the URL is the documented path for this server. Use it first.
- The connector calls `tools/list` and `tools/call` only. This server offers
  nothing else, so nothing is lost.
- Every server in `mcp_servers` must be named by exactly one `mcp_toolset`.
- Each tool result is one text block with stringified JSON. No tool declares an
  `outputSchema`, so `mcp_tool_result` never carries structured content.
- A failed call sets `is_error: true` and the text reads
  `sec-api error: <message>`.
- The MCP connector is not covered by zero data retention. Tool definitions and
  results follow Anthropic's standard retention policy.

## Remove

Delete the `mcp_servers` entry and its `mcp_toolset` from the request body. No
state stays behind.

## Source

[MCP connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)
