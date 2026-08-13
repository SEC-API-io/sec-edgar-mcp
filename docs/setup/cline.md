# Connect Cline to SEC EDGAR Data with MCP

Cline is an autonomous coding agent for VS Code. It is a full MCP client. It
speaks Streamable HTTP, so it connects to this server directly. No bridge is
needed.

## Prerequisites

- VS Code, or a fork such as Cursor or Windsurf.
- The Cline extension. Use a recent build. Early remote-server builds wrote no
  transport type and fell back to the deprecated SSE transport.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Cline keeps MCP servers in `cline_mcp_settings.json`.

| Platform | Path                                                                                                          |
| -------- | ------------------------------------------------------------------------------------------------------------- |
| macOS    | `~/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json` |
| Linux    | `~/.config/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json`                     |
| Windows  | `%APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings\cline_mcp_settings.json`                     |

For VS Code Insiders replace `Code` with `Code - Insiders`. For VSCodium replace
it with `VSCodium`. The Cline CLI reads `~/.cline/mcp.json` instead.

Open the file from the UI. Click the **MCP Servers** icon in the Cline panel
toolbar, then click **Configure MCP Servers**.

## Config

Add this block. Replace `YOUR_API_KEY` with your key.

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "streamableHttp",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

To keep the key out of the URL, cut the `?apiKey=` part and add
`"headers": { "Authorization": "Bearer YOUR_API_KEY" }` to the same block. Cut
the query string when you do. A stale query value masks the header.

The UI does the same job. Click the **MCP Servers** icon, open the **Remote
Servers** tab, then fill in **Server Name**, **Server URL** and **Transport
Type**. Pick **Streamable HTTP**. Click **Add Server**.

## Reload

Save the file. Cline reconnects on save. If the server stays red, run
**Developer: Reload Window** from the Command Palette.

## Verify

Open the **MCP Servers** panel and select the **Installed** tab. The `sec-api`
entry must show a green dot. Expand it to list the tools. Expect **49 tools**.

## First prompt

> Get the newest Apple 10-K. Give me the filing date and the primary document
> URL.

Cline calls `filing-search` and asks the model to approve the call. The answer
is one text block with stringified JSON. It holds `total` and a `filings` array.
Each filing carries `accessionNo`, `filedAt` and `linkToFilingDetails`.

## Quirks

- The transport value is `streamableHttp`, in camelCase. `streamable-http` from
  the Roo Code docs does not work here.
- Omit `type` and Cline uses SSE. This server has no SSE endpoint, so the
  connection fails.
- Cline reports remote failures as `fetch failed`, with no status code and no
  body. A bad key returns HTTP 401.
- The settings file stores the URL in plain text. Prefer the header form on a
  shared machine.
- Leave `autoApprove` empty until you trust a tool.

## Removal

Delete the `sec-api` block from `cline_mcp_settings.json` and save. You can also
click **Delete** next to the server in the **MCP Servers** panel.

Source: [Cline MCP documentation](https://docs.cline.bot/mcp/mcp-overview)
