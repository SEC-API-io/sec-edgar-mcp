# Connect VS Code to SEC EDGAR Data with MCP

VS Code is Microsoft's code editor. Its chat agent is a full MCP client. It
speaks Streamable HTTP, so it connects to this server directly.

## Prerequisites

- VS Code 1.102 or later. MCP support became generally available in that
  release, and the configuration moved into `mcp.json`.
- The GitHub Copilot Chat extension, signed in.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

VS Code keeps MCP servers in `mcp.json`. There are two locations.

| Scope        | Path                                   | Command to open it                           |
| ------------ | -------------------------------------- | -------------------------------------------- |
| Workspace    | `.vscode/mcp.json` in the project root | **MCP: Open Workspace Folder Configuration** |
| User profile | `mcp.json` in your user profile folder | **MCP: Open User Configuration**             |

Run the commands from the Command Palette, `Cmd+Shift+P` or `Ctrl+Shift+P`. Use
one location only. Two entries for one server conflict.

## Config

Add this block. Replace `YOUR_API_KEY` with your key.

```json
{
  "servers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

The root key is `servers`. It is not `mcpServers`.

To keep the key out of the file, let VS Code ask for it once:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "sec-api-key",
      "description": "sec-api API key",
      "password": true
    }
  ],
  "servers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp?apiKey=${input:sec-api-key}"
    }
  }
}
```

## Restart

Save the file. VS Code starts the server and asks you to trust it. Accept the
prompt. To restart later, run **MCP: List Servers**, select `sec-api`, then
select **Restart**.

## Verify

Open the Chat view with `Ctrl+Alt+I` or `Cmd+Ctrl+I`. Set the mode to
**Agent**. Click **Configure Tools** in the chat input and expand `sec-api`.
Expect **49 tools**.

## First prompt

> What is Apple's CIK and CUSIP?

The agent calls `mapping` and asks you to confirm the call. The answer gives CIK
`320193` and CUSIP `037833100`. The raw result is one text block that holds a
bare JSON array, not a `data` envelope.

## Quirks

- MCP tools work in **Agent** mode only. In Ask mode the tools stay invisible.
- VS Code does not forward servers that use `${input:...}` to the Agent Host.
  Put the key in the URL if you use the Agent Host, or use a workspace
  `.mcp.json` file, which the Agent Host reads natively.
- Run **MCP: Reset Cached Tools** if the tool count looks wrong after a change.
- Settings Sync can copy the user profile `mcp.json`, and with it your key. Use
  the `${input:...}` form to avoid that.
- For logs, run **MCP: List Servers**, select `sec-api`, then **Show Output**. A
  bad key returns HTTP 401.

## Removal

Delete the `sec-api` block from `mcp.json` and save. To keep the entry but stop
the connection, run **MCP: List Servers**, select `sec-api`, then **Disable**.

Source:
[Add and manage MCP servers in VS Code](https://code.visualstudio.com/docs/agent-customization/mcp-servers)
and the
[MCP configuration reference](https://code.visualstudio.com/docs/agents/reference/mcp-configuration)
