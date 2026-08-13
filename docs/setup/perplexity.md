# Connect Perplexity to SEC EDGAR Data with MCP

Perplexity is a hosted AI answer engine. It connects to remote MCP servers
through **custom connectors**, and it does not run local stdio servers.

## Prerequisites

- A Perplexity **Pro**, **Max** or **Enterprise** plan. Free accounts cannot add
  custom connectors. If you do not see an **Advanced** section in the connector
  dialog, your plan does not include the feature.
- In an organization, an admin may have to allow custom connectors first.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

Perplexity has no config file. Use the web UI.

Click your profile in the lower left corner, then **All settings**, then
**Connectors** in the left menu.

## Steps

1. Click **+ Custom connector** in the top right corner.
2. Select **Remote** in the dialog.
3. Set **Name** to `SEC EDGAR`.
4. Set **MCP Server URL** to this value:

   ```text
   https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
   ```

5. Expand **Advanced**.
6. Set **Transport** to **Streamable HTTP**. Do not select SSE.
7. Set **Authentication** to **None**. The key already travels in the URL.
8. Tick the box that acknowledges the risk of custom connectors.
9. Click **Add**.

To keep the key out of the URL, use `https://api.sec-api.io/mcp` as the URL and
set **Authentication** to **API Key** instead. Perplexity sends the key as an
`Authorization: Bearer` header. This server accepts that header.

## Reload

There is no restart. The connector goes live at once. Reload the browser tab if
the new card does not appear.

## Verify

Open **All settings**, then **Connectors**, then click the `SEC EDGAR` card. The
card lists the tools that the server exposes. Expect **49**.

A red error tag on the card means validation failed. A wrong key returns HTTP
401.

## First prompt

Open a thread. Click the **+** icon and tick `SEC EDGAR` under connectors and
sources. Then ask:

```text
Use the SEC EDGAR connector. List the three most recent 10-K filings from
Apple with their filing dates and accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`. Perplexity shows the
`filing-search` tool call above the answer.

## Known quirks

- **Streamable HTTP only.** Pick that transport. The server is stateless and
  issues no `Mcp-Session-Id`, so the SSE option can fail on the first request.
- **HTTPS is required.** The URL above already meets that rule.
- **The key sits in the URL.** Anyone who can open your connector settings can
  read it. Use the API Key auth option if that matters to you. Rotate the key at
  [sec-api.io](https://sec-api.io/profile) if it leaks.
- **No OAuth.** This server has no OAuth endpoints. Do not select OAuth 2.0.

## Removal

Open **All settings**, then **Connectors**. Click the `SEC EDGAR` card and
delete it. You can delete only the connectors that you created. An admin manages
organization-wide connectors.

## Source

[Perplexity Help Center: Adding Custom Remote Connectors](https://www.perplexity.ai/help-center/en/articles/13915507-adding-custom-remote-connectors)
