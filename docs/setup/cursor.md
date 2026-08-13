# Connect Cursor to SEC EDGAR Data with MCP

Cursor is an AI code editor. It is a full MCP client. It speaks Streamable
HTTP, so it connects to this server directly. No bridge is needed.

## Prerequisites

- Cursor desktop, current release. Cursor states no minimum version for remote
  MCP servers.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Cursor keeps MCP servers in `mcp.json`.

| Scope   | Path                                                          |
| ------- | ------------------------------------------------------------- |
| Project | `.cursor/mcp.json` in the project root                        |
| Global  | `~/.cursor/mcp.json`, on Windows `%USERPROFILE%\.cursor\mcp.json` |

## Config

Add this block. Replace `YOUR_API_KEY` with your key.

```json
{
  "mcpServers": {
    "sec-api": {
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

Cursor picks the HTTP transport from the `url` field. A remote server needs no
`type` field.

To keep the key out of the file, use an environment variable. Cursor
interpolates `${env:NAME}` in `url` and in `headers`.

```json
{
  "mcpServers": {
    "sec-api": {
      "url": "https://api.sec-api.io/mcp?apiKey=${env:SEC_API_KEY}"
    }
  }
}
```

Set `SEC_API_KEY` in your shell profile. Start Cursor from that shell, so it
inherits the variable.

## Reload

Save the file. Cursor reloads the server list. If the server does not appear,
restart Cursor.

## Verify

Open **Customize** in the sidebar, then **MCP**. Select `sec-api`. The entry
lists the tools it exposes. Expect **49 tools**.

## First prompt

> What is Apple's CIK and CUSIP?

Cursor calls `mapping` and asks you to approve the call. The answer gives CIK
`320193` and CUSIP `037833100`. The raw result is one text block that holds a
bare JSON array, not a `data` envelope.

## Quirks

- Cursor warns when too many tools are enabled across all servers. Users report
  a cap near 40 tools. Cursor documents no number. This server alone exposes 49
  tools. Turn off your other MCP servers first if tools go missing.
- `${env:...}` expands when Cursor loads the config. Set the variable in the
  environment that starts Cursor, not in a terminal inside Cursor.
- A project `.cursor/mcp.json` is easy to commit. Never commit a live key. Use
  the `${env:...}` form in a shared repository.
- Read the arguments before you approve a call.
- For logs, open the Output panel with `Cmd+Shift+U` or `Ctrl+Shift+U` and
  select **MCP Logs**. A bad key returns HTTP 401.

## Removal

Delete the `sec-api` block from `.cursor/mcp.json` and save. To keep the entry
but stop the connection, toggle the server off under **Customize** > **MCP**.

Source: [Model Context Protocol (MCP), Cursor Docs](https://cursor.com/docs/mcp)
