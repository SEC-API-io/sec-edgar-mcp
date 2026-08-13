# Connect LM Studio to SEC EDGAR Data with MCP

LM Studio is a desktop app that runs local models. Since version 0.3.17 it also
acts as an MCP host, and it connects to local and remote MCP servers.

## Prerequisites

- LM Studio **0.3.17 (b10)** or later. Earlier builds have no MCP support, and
  builds before b10 have no remote server support.
- A loaded model that supports tool use.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

LM Studio keeps one file, `mcp.json`. It uses Cursor's notation.

| OS            | Path                            |
| ------------- | ------------------------------- |
| macOS         | `~/.lmstudio/mcp.json`          |
| Linux         | `~/.lmstudio/mcp.json`          |
| Windows       | `%USERPROFILE%/.lmstudio/mcp.json` |

Edit it in the app. Open the right sidebar, click the **Program** tab, the one
with the terminal icon, then click **Install**, then **Edit mcp.json**.

## Config

Add this entry:

```json
{
  "mcpServers": {
    "sec-api": {
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

To keep the key out of the URL, send it as a header instead:

```json
{
  "mcpServers": {
    "sec-api": {
      "url": "https://api.sec-api.io/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

If you already have servers in the file, copy only the block after
`"mcpServers": {` and before its closing brace. Do not paste a second
`mcpServers` object.

## Reload

Save the file. LM Studio reloads `mcp.json` on save and loads the server.
Restart the app if the entry does not appear.

## Verify

Open the **Program** tab in the right sidebar. The `sec-api` entry lists the
tools that it loaded. Expect **49**. The vendor doc does not describe this
panel, so the layout may differ in your build.

A wrong key gives HTTP 401 and the server fails to load.

## First prompt

```text
List the three most recent 10-K filings from Apple with their filing dates and
accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`. LM Studio shows a
confirmation dialog before the first `filing-search` call. Review the arguments,
then allow the tool.

## Known quirks

- **Streamable HTTP works. SSE does not.** LM Studio does not negotiate SSE, and
  this server is stateless and issues no session ID. The pair matches.
- **Two open bugs to know.**
  [#1453](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/1453)
  reports `SSE error: Non-200 status code (404)` for URL-only entries.
  [#1371](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/1371)
  reports that `~/.lmstudio` can be missing on macOS, with a copy at
  `~/.cache/lm-studio/mcp.json`. Update the app, and edit the file in the app.
- **Confirmation dialogs slow you down.** Choose "always allow" for the tools
  that you use often. Manage the choices in App Settings.
- **The API needs a separate switch.** To call these tools through the local
  server API, turn on **Allow calling servers from mcp.json** in Server
  Settings.
- **49 tools is a lot for a small model.** A 7B model often picks the wrong one.
  Name the tool in the prompt if the answer looks wrong.

## Removal

Open **Edit mcp.json** and delete the `sec-api` block. Save the file. LM Studio
drops the server on reload.

## Source

[LM Studio Docs: Use MCP Servers](https://lmstudio.ai/docs/app/mcp)
