# Connect Windsurf to SEC EDGAR Data with MCP

Windsurf is an AI code editor from Cognition, now documented as Devin Desktop.
Its Cascade agent is a full MCP client. It speaks Streamable HTTP, so it
connects to this server directly. No bridge is needed.

## Prerequisites

- Windsurf desktop, current release. The vendor states no minimum version for
  remote HTTP servers.
- MCP access enabled. Team and enterprise admins can block it, and enterprise
  users must switch it on in settings.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Cascade keeps MCP servers in `mcp_config.json`.

| Product                   | Path                                  |
| ------------------------- | ------------------------------------- |
| Windsurf editor           | `~/.codeium/windsurf/mcp_config.json` |
| Windsurf JetBrains plugin | `~/.codeium/mcp_config.json`          |

On Windows the same path sits under your user folder, for example
`%USERPROFILE%\.codeium\windsurf\mcp_config.json`.

Reach the file from the UI in two ways. Click the **MCPs** icon in the top right
menu of the Cascade panel, or open **Devin Settings** > **Cascade** > **MCP
Servers**. The JetBrains plugin puts the same list under **Windsurf Settings**,
behind the **View Raw Config** button. The config is global. Cascade reads no
per-project file.

## Config

Add this block. Replace `YOUR_API_KEY` with your key.

```json
{
  "mcpServers": {
    "sec-api": {
      "serverUrl": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

A remote server needs `serverUrl` or `url`. Both work. A remote entry that has
neither is read as a stdio entry and fails.

To keep the key out of the URL, send it as a header instead:

```json
{
  "mcpServers": {
    "sec-api": {
      "serverUrl": "https://api.sec-api.io/mcp",
      "headers": { "Authorization": "Bearer ${env:SEC_API_KEY}" }
    }
  }
}
```

Cascade interpolates `${env:NAME}` in `serverUrl`, `url` and `headers`.

## Reload

Save the file, then press the refresh button in the MCP panel. If the server
stays disconnected, quit Windsurf completely and open it again. Closing the
window is not enough.

## Verify

Click the **MCPs** icon in the Cascade panel and select `sec-api`. The settings
page for the server lists every tool with a toggle. Expect **49 tools**.

## First prompt

> What is Apple's CIK and CUSIP?

Cascade calls `mapping`. The answer gives CIK `320193` and CUSIP `037833100`.
The raw result is one text block that holds a bare JSON array, not a `data`
envelope.

## Quirks

- Cascade uses at most 100 tools at one time, across all servers. This server
  takes 49 of them. Over the cap, tools drop out with no error.
- The vendor page marks this configuration for the **legacy Cascade agent**. New
  tabs open the Devin Local agent, which reads the Devin CLI config instead. Add
  the server there with `devin mcp add sec-api https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`.
- `mcp_config.json` holds the key in plain text. Prefer the header form with an
  environment variable on a shared machine.
- A bad key returns HTTP 401. Errors from a good key start with
  `sec-api error:`.

## Removal

Delete the `sec-api` block from `mcp_config.json`, save, then press refresh. You
can also open the server page from the **MCPs** icon and remove it there.

Source:
[Model Context Protocol (MCP), Cascade](https://docs.devin.ai/desktop/cascade/mcp)
and
[MCP Configuration, Devin CLI](https://docs.devin.ai/cli/extensibility/mcp/configuration)
