# Connect Cherry Studio to SEC EDGAR Data with MCP

Cherry Studio is an open-source desktop client for many model providers. It is
an MCP host and it supports remote servers over Streamable HTTP.

## Prerequisites

- Cherry Studio **1.5.2** or later.
- A model that supports tool use.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

Cherry Studio keeps MCP settings in its internal database. There is no
`mcp.json` to edit. Use the UI.

Open **Settings**, then **MCP**, then **MCP Servers**.

## Steps

1. Open **Settings**, then **MCP**, then **MCP Servers**.
2. Click **Add Server**, then **Quick Create**.
3. Set the **Name** to `sec-api`.
4. Set the **Type** to **Streamable HTTP**. Do not select SSE or STDIO.
5. Set the **URL** to this value:

   ```text
   https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
   ```

6. Leave the header fields empty. The key rides in the URL. To keep it out of
   the URL, use `https://api.sec-api.io/mcp` and add this header:

   ```text
   Authorization: Bearer YOUR_API_KEY
   ```

7. Click **Save**.

## Reload

Enable the server with the toggle in the top right corner of its page. Wait for
the status to turn normal. There is no app restart.

## Verify

Open the `sec-api` entry and click **Tools**. Cherry Studio lists every tool
with its parameters, types, required flags and enum values. Expect **49**.

Expand one tool to read its full Markdown documentation.

A wrong key gives HTTP 401 and the server never reaches the normal state.

## First prompt

```text
List the three most recent 10-K filings from Apple with their filing dates and
accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`.

## Known quirks

- **The JSON import format is not documented.** Cherry Studio offers **Import
  from JSON** next to **Quick Create**. Community sources report that it uses
  `"type": "streamableHttp"` and `baseUrl`, not the common `url` key. Neither
  key is confirmed by the vendor doc. Use **Quick Create** and the form fields
  on this page. They are documented.
- **The key sits in the internal database.** There is no plain file to protect
  or to check. Rotate the key at [sec-api.io](https://sec-api.io/profile) if it
  leaks.
- **Agents need a separate binding.** Some builds require you to attach the
  server to an agent under **Work**, then the agent menu, then **Edit**, then
  **MCP**.
- **Streamable HTTP only.** The server is stateless and issues no session ID.
  SSE can fail on the first request.

## Removal

Open **Settings**, then **MCP**, then **MCP Servers**. Toggle `sec-api` off and
delete the entry.

## Source

[Cherry Studio Docs: MCP and External Tools](https://docs.cherryai.com.cn/docs/en-us/advanced-basic/extensions/mcp.md)
