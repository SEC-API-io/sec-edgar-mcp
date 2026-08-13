# Connect Zapier to SEC EDGAR Data with MCP

Zapier automates workflows between apps. Its **MCP Client** app connects Zaps
and Zapier Agents to a remote MCP server. MCP Client is a beta feature.

## Prerequisites

- A Zapier account.
- The **MCP Client** app. It supports tool calls only, over Streamable HTTP or
  SSE.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

Zapier stores this as an app connection. There is no file.

Go to the **Apps** page at `zapier.com/app/connections`.

## Steps

1. Open the **Apps** page.
2. Click **+ Add connection**.
3. Search for **MCP Client** and select it.
4. Click **Add connection**. The **Connect an Account** page opens in a new tab.
5. **Server URL**: `https://api.sec-api.io/mcp`
6. **Transport**: **Streamable HTTP**
7. **OAuth**: **No**
8. **Bearer Token**: `YOUR_API_KEY`. Zapier sends
   `Authorization: Bearer YOUR_API_KEY`.
9. Click **Yes, Continue to MCP Client**.
10. Rename the connection to `SEC API` so it is easy to find later.

To skip the token field, leave it empty and put the key in the URL:

```text
https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

## Reload

Refresh the Zap editor page after you add the connection. Zapier caches the
connection list.

## Verify

Add a **Run Tool** action and pick the SEC API connection. Open the **Tool**
dropdown. It lists **49** tools.

## First step to try

Build a Zap with a Schedule trigger and a **Run Read-Only Tool** action.

- Tool: `filing-search`
- `query`: `ticker:AAPL AND formType:"10-K"`
- `size`: `3`

Expect one output field holding the JSON result, with a `filings` array of three
rows.

## Quirks

- **Beta.** Field names and behaviour can change without notice.
- **Tool calls only.** MCP Client reads no resources and no prompts. This server
  offers tools only, so nothing is lost.
- **Pick Streamable HTTP.** The server is stateless and serves no SSE session.
  Picking SSE fails at connect time.
- **OAuth Yes ignores the Bearer Token field.** Keep OAuth set to **No**.
- **Turn on Handles errors.** With it off, a failed tool still reports Success
  in a Zap, or Completed in an Agent. This server signals failure with `isError`
  and the text `sec-api error: <message>`.
- **Results arrive as one text block.** Each tool returns stringified JSON. No
  tool declares an output schema. Add a Formatter or Code step to pull out
  fields.
- **Static IP is not needed.** This server uses no IP allowlist.

## Removal

Open the **Apps** page. Find the SEC API connection under MCP Client. Open its
menu and choose **Delete**. Zaps that used it stop and need a new connection.

## Source

[Connect remote MCP servers to Zapier using MCP Client | Zapier Help](https://help.zapier.com/hc/en-us/articles/38777069364109-Connect-remote-MCP-servers-to-Zapier-using-MCP-Client)
