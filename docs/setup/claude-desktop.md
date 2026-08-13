# Connect Claude Desktop to SEC EDGAR Data with MCP

Claude Desktop is the Claude chat app for macOS, Windows and Linux. It reaches
remote MCP servers through custom connectors. Its JSON config file is documented
for local stdio servers only.

## Prerequisites

- Claude Desktop, current version. Anthropic states no minimum version.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).
- A Claude plan that allows custom connectors. Free allows one. Pro, Max, Team
  and Enterprise allow more. On Team and Enterprise an Owner must add the
  connector first.
- For the bridge method only: Node.js 18 or later.

## Configure: custom connector, recommended

This is the documented path for a remote HTTP server.

1. Open Claude Desktop.
2. Open **Settings**, then **Connectors**.
3. Click **Add custom connector**.
4. In **Name**, enter `sec-api`.
5. In the URL field, enter
   `https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`.
6. Leave **Advanced settings** empty. This server does not use OAuth.
7. Click **Add**.

Anthropic connects to the server from its own cloud, not from your computer.
This server is public, so that works.

## Configure: mcp-remote bridge

Use this only when you must keep the key in a local file. Open **Settings**,
then **Developer**, then **Edit Config**. The file opens at:

| OS | Path |
| -- | ---- |
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

Add this entry:

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"]
    }
  }
}
```

## Restart

Quit Claude Desktop completely, then start it again. Closing the window is not
enough. On macOS use Cmd+Q. On Windows quit it from the tray icon.

## Verify

Open **Customize**, then **Connectors**, and select `sec-api`. The **Tool
permissions** list shows every tool the server exposes. Expect 49.

In a chat, click **+**, then **Connectors**, and check that `sec-api` is on.

## First prompt

> Which 8-K filings from the last month mention quantum computing?

Claude calls `full-text-search`. The answer holds a short list of filings. Each
one gives the company, the filing date, and a link to the document.

## Quirks

- The connector runs from Anthropic's servers. Your API key travels there and
  Anthropic stores it. Rotate the key if you remove the connector.
- Header authentication exists in the **Request headers** section of the Add
  custom connector dialog, but it is in beta and rolls out slowly. The URL query
  key works everywhere today.
- The vendor documentation does not describe a `url` or `type` field for
  `claude_desktop_config.json`. Do not put the raw HTTPS URL there. Use a
  connector, or the `mcp-remote` bridge above.
- Claude Desktop starts config commands with a minimal PATH. If `npx` fails in
  Method B, write the full path, for example `/usr/local/bin/npx`.
- On Windows, users report that the MSIX build reads a different config file from
  the one **Edit Config** opens. Check
  `%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json`
  if Method B loads no tools.
- 49 tools is a large set. Block the tools you do not need under **Customize >
  Connectors**, so Claude picks faster.

## Remove

1. Open **Customize**, then **Connectors**.
2. Find `sec-api`.
3. Open the three-dot menu and click **Remove**.

For Method B, delete the `sec-api` block from `claude_desktop_config.json` and
restart the app.

## Source

[Third party connectors with remote MCP](https://claude.com/docs/connectors/custom/remote-mcp)
and [Getting started with local MCP servers on Claude Desktop](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop)
