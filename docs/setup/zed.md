# Connect Zed to SEC EDGAR Data with MCP

Zed is a fast native code editor. It calls MCP servers "context servers". It
supports remote servers over HTTP, so it connects to this server directly. No
bridge is needed.

## Prerequisites

- Zed, current release. Older builds had no remote MCP support and needed the
  `mcp-remote` stdio bridge.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Zed keeps context servers in `settings.json`.

| Platform | Path                                                              |
| -------- | ----------------------------------------------------------------- |
| macOS    | `~/.config/zed/settings.json`                                     |
| Linux    | `~/.config/zed/settings.json`, or `$XDG_CONFIG_HOME/zed/settings.json` |
| Windows  | `%APPDATA%\Zed\settings.json`                                     |

A project can override the user file with `.zed/settings.json` in its root. Open
the user file with the `zed: open settings file` action.

## Config

Add this block. Replace `YOUR_API_KEY` with your key.

```json
{
  "context_servers": {
    "sec-api": {
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

The header form works too, and it keeps the key out of the URL:

```json
{
  "context_servers": {
    "sec-api": {
      "url": "https://api.sec-api.io/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

The UI writes the same entry. Open **Settings** > **AI** > **MCP Servers**,
click **Add Server** in the page header, then select **Add Remote Server**.

## Reload

Save the file. Zed restarts the context server. The editor needs no restart.

## Verify

Open **Settings** > **AI** > **MCP Servers**. The dot beside `sec-api` turns
green, and its tooltip reads "Server is active". Zed prints no tool count. To
count the tools, open the agent profile settings, where every tool of the server
has its own row. Expect **49 tools**.

## First prompt

> What is Apple's CIK and CUSIP?

The agent calls `mapping` and asks you to approve the call. The answer gives CIK
`320193` and CUSIP `037833100`. The raw result is one text block that holds a
bare JSON array, not a `data` envelope.

## Quirks

- Zed starts the MCP OAuth flow when a remote server carries no `Authorization`
  header. This server does not use OAuth. Use the header form above to avoid the
  prompt, or dismiss the prompt and keep the key in the URL.
- Zed supports the MCP tools and prompts features. This server offers tools
  only, so nothing is lost.
- Zed asks for approval before each tool call. `agent.tool_permissions.default`
  controls that. Keep it at `"confirm"` until you trust a tool.
- A model that ignores the tools needs a push. Name the server in the prompt, or
  build a profile that turns the built-in tools off.
- For errors, run `zed: open log` from the Command Palette. A bad key returns
  HTTP 401.

## Removal

Open **Settings** > **AI** > **MCP Servers** and click the trash icon on the
`sec-api` row. You can also delete the `sec-api` block from `settings.json` and
save. To keep the entry, use the toggle on the row instead.

Source: [Model Context Protocol, Zed Docs](https://zed.dev/docs/ai/mcp) and
[Agent Settings](https://zed.dev/docs/ai/agent-settings)
