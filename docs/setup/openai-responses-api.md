# Connect the OpenAI Responses API to SEC EDGAR Data with MCP

The Responses API calls a remote MCP server for you. You pass one tool object of
`type: "mcp"`. OpenAI lists the tools, runs the calls, and feeds the results back
to the model.

## Prerequisites

- An OpenAI API key in `OPENAI_API_KEY`.
- A model that supports the MCP tool. OpenAI states it works on most recent
  models.
- A sec-api.io API key. Get one at [sec-api.io](https://sec-api.io/profile).
- A publicly reachable server. This one is public and speaks Streamable HTTP,
  which the API supports.

## Config location

There is no config file. The server goes in the `tools` array of every request.

## Request

```bash
curl https://api.openai.com/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-5.6",
    "tools": [{
      "type": "mcp",
      "server_label": "sec_api",
      "server_description": "SEC EDGAR filings, XBRL financials, insider trades and SEC enforcement data.",
      "server_url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
      "require_approval": "never"
    }],
    "input": "What are the three most recent 10-K filings from Apple?"
  }'
```

```python
from openai import OpenAI

client = OpenAI()
resp = client.responses.create(
    model="gpt-5.6",
    tools=[{
        "type": "mcp",
        "server_label": "sec_api",
        "server_url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "require_approval": "never",
        "allowed_tools": ["filing-search", "full-text-search", "extractor"],
    }],
    input="What are the three most recent 10-K filings from Apple?",
)
print(resp.output_text)
```

Send the key in a header instead of the URL if you prefer:

```json
{
  "type": "mcp",
  "server_label": "sec_api",
  "server_url": "https://api.sec-api.io/mcp",
  "headers": { "Authorization": "Bearer YOUR_API_KEY" },
  "require_approval": "never"
}
```

## Reload

Not applicable. Every request carries its own tool definition.

## Verify

The first response holds an `mcp_list_tools` output item. Count its `tools`
array:

```bash
jq '[.output[] | select(.type == "mcp_list_tools") | .tools[]] | length'
```

Expect **49**.

## First prompt

> Summarize the risk factors in Boeing's most recent 10-K.

Expect a prose summary. The model calls `filing-search` to find the filing, then
`extractor` to pull section 1A. Each result is one text block of stringified
JSON, or raw text for `extractor`.

## Quirks

- **Resend the URL and headers every time.** OpenAI keeps only the scheme, domain
  and subdomains of `server_url` after a request. It discards header values, and
  they never appear in the response object. A follow-up with
  `previous_response_id` still needs the full tool definition.
- **The tool list is cached in the context.** While the `mcp_list_tools` item
  stays in the request, OpenAI does not fetch the list again.
- **Use `allowed_tools`.** 49 tool definitions ride along on every turn and cost
  input tokens.
- **Approval is on by default.** Set `require_approval` to `"never"`, or answer
  each `mcp_approval_request` with an `mcp_approval_response` item.
- **No structured content.** No tool declares an `outputSchema`. Every result is
  one text block. Parse the JSON yourself if you need fields.
- **Errors come back as text.** A failed call returns `isError` and the text
  `sec-api error: <message>`. A bad key returns HTTP 401.
- **Azure OpenAI blocks unknown MCP domains.** Add `api.sec-api.io` to the
  allowed domains, or the request fails before it reaches the server.

## Removal

Delete the `type: "mcp"` object from the `tools` array. Rotate the key at
sec-api.io if it reached a log.

## Source

[MCP and connectors guide](https://developers.openai.com/api/docs/guides/tools-connectors-mcp),
read 2026-08-13.
