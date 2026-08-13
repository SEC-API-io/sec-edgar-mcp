# Connect Roo Code to SEC EDGAR Data with MCP

Roo Code is an agentic coding extension for VS Code. It is a full MCP client. It
speaks Streamable HTTP, so it connects to this server directly. No bridge is
needed.

## Prerequisites

- VS Code, or a fork.
- Roo Code **3.19.2** or later. That release added the `streamable-http`
  transport. Earlier versions handle stdio and SSE only.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Roo Code reads two files. The project file wins when both name the same server.

**Global.** The file is `mcp_settings.json`.

| Platform | Path                                                                                                       |
| -------- | ---------------------------------------------------------------------------------------------------------- |
| macOS    | `~/Library/Application Support/Code/User/globalStorage/rooveterinaryinc.roo-cline/settings/mcp_settings.json` |
| Linux    | `~/.config/Code/User/globalStorage/rooveterinaryinc.roo-cline/settings/mcp_settings.json`                    |
| Windows  | `%APPDATA%\Code\User\globalStorage\rooveterinaryinc.roo-cline\settings\mcp_settings.json`                    |

**Project.** The file is `.roo/mcp.json` in the workspace root.

Open either file from the UI. Click the MCP icon in the top navigation of the
Roo Code pane, then scroll to **Edit Global MCP** or **Edit Project MCP**.

## Config

Add this block. Replace `YOUR_API_KEY` with your key.

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "streamable-http",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
      "disabled": false,
      "alwaysAllow": []
    }
  }
}
```

For a project file that you commit, keep the key out of the URL. Drop the
`?apiKey=` part and add
`"headers": { "Authorization": "Bearer ${env:SEC_API_KEY}" }` to the same block.

## Reload

Save the file. Roo Code reconnects. To force a retry, click the restart button
next to the server in the MCP settings view.

## Verify

Click the MCP icon in the top navigation. Confirm **Enable MCP Servers** is on.
The `sec-api` entry appears in the list. Expand it to see the tools. Expect
**49 tools**.

## First prompt

> Use the SEC tools. Find the newest Apple 10-K and give me the filing date and
> the primary document URL.

Roo Code calls `filing-search` and asks you to approve the call. The answer is
one text block with stringified JSON. It holds `total` and a `filings` array.
Each filing carries `accessionNo`, `filedAt` and `linkToFilingDetails`.

## Quirks

- The transport value is `streamable-http`, with a hyphen. `streamableHttp` from
  the Cline docs fails validation here.
- `type` is mandatory on any URL server. Roo Code cannot infer it from `url`.
- The schema forbids `command`, `args` and `env` on a URL server. Use `headers`.
- Installs upgraded from Roo Cline may still hold `cline_mcp_settings.json` in
  the same folder. Check which file exists before you edit.
- `${env:VAR}` expands when Roo Code loads the config. Set the variable in the
  environment that starts VS Code, not in a terminal inside VS Code.
- Add a tool name to `alwaysAllow` only after you trust it.

## Removal

Delete the `sec-api` block from `mcp_settings.json` or `.roo/mcp.json` and save.
To keep the entry but stop the connection, set `"disabled": true`.

Source:
[Using MCP in Roo Code](https://docs.roocode.com/features/mcp/using-mcp-in-roo/)
and
[MCP server transports](https://docs.roocode.com/features/mcp/server-transports)
