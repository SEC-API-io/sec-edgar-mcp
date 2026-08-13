# Connect Jan to SEC EDGAR Data with MCP

Jan is an open-source desktop app for local and remote models. It is an MCP
host, and its **Add MCP Server** dialog offers a Streamable HTTP transport.

## Prerequisites

- Jan Desktop with the MCP Servers panel in Settings. Jan does not publish a
  minimum version for remote MCP support.
- A model that supports tool use.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

Use the UI. Open **Settings**, then **MCP Servers**.

Jan stores its data in a folder that you can see and change under **Settings**,
then **General**, then **Jan Data Folder**. The defaults are
`~/Library/Application Support/Jan` on macOS and
`C:\Users\<you>\AppData\Roaming\Jan` on Windows.

## Steps

1. Open **Settings**, then **MCP Servers**.
2. Click **+ Add MCP Server** in the top right corner.
3. Set the server name to `sec-api`.
4. Set **Transport** to **HTTP**. That is Streamable HTTP. Do not pick SSE.
5. Set **URL** to this value:

   ```text
   https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
   ```

6. Leave **Args** and **Env** empty. They apply to STDIO servers only.
7. Click **Save**.

There is nothing to install. The server is hosted.

## Reload

Jan starts the server when you save. Toggle the server off and on in the MCP
Servers list to reconnect after an error.

## Verify

Open **Settings**, then **MCP Servers**. Expand the `sec-api` entry. It lists
the tools that it loaded. Expect **49**.

A wrong key gives HTTP 401 and the server stays inactive.

## First prompt

```text
List the three most recent 10-K filings from Apple with their filing dates and
accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`. Jan asks for permission
before the `filing-search` call unless you turned that off.

## Known quirks

- **Smart MCP tool routing hides tools.** Jan can send only a subset of tools to
  the model each turn. The count then looks lower than 49. Turn routing off in
  **Settings**, then **MCP Servers**, while you check the setup.
- **Jan's tool call timeout defaults to 30 seconds.** Raise Jan's **Tool call
  timeout** to 60 seconds.
- **Permission prompts.** Turn on **Allow All MCP Tool Permissions** to
  auto-approve calls. The setting is global and applies to every server.
- **The key sits in the URL.** It is stored in plain text in the Jan data
  folder. Rotate the key at [sec-api.io](https://sec-api.io/profile) if it
  leaks.
- **Header auth is an option.** If your build offers a headers field, use
  `https://api.sec-api.io/mcp` with `Authorization: Bearer YOUR_API_KEY`. The
  server accepts that form.

## Removal

Open **Settings**, then **MCP Servers**. Toggle `sec-api` off, then delete the
entry from its row menu.

## Source

[Jan Docs: MCP Servers](https://www.jan.ai/docs/desktop/integrations/mcp-servers)
and [Jan Docs: Model Context Protocol](https://www.jan.ai/docs/desktop/mcp)
