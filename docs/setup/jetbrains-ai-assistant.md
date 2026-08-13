# Connect JetBrains AI Assistant to SEC EDGAR Data with MCP

JetBrains AI Assistant is the AI plugin inside IntelliJ IDEA, PyCharm, WebStorm
and the other JetBrains IDEs. It is an MCP client and speaks Streamable HTTP, so
it connects to this server directly. There is no config file to edit. You paste
JSON into a dialog.

## Prerequisites

- A JetBrains IDE with AI Assistant 2026.1 or later. That version documents the
  STDIO, Streamable HTTP and SSE transports.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

AI Assistant stores MCP servers in IDE settings, not in a project file.

Open **Settings** > **Tools** > **AI Assistant** > **Model Context Protocol
(MCP)**. You can also type `/` in the AI chat and select the add command.

## Steps

1. Click **Add**.
2. In the **New MCP Server** dialog, set the connection type to **HTTP**.
3. Paste this JSON. Replace `YOUR_API_KEY` with your key.

   ```json
   {
     "mcpServers": {
       "sec-api": {
         "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
       }
     }
   }
   ```

4. Set the server level. **Global** makes it available in every project.
   **Project** limits it to the current one.
5. Click **OK**.
6. Select the checkbox beside `sec-api`, then click **Apply**.

The root key is `mcpServers`. A remote server needs only `url`. The **Working
directory** field applies to STDIO servers only. Leave it empty.

## Restart

**Apply** starts the server and opens the connection. No IDE restart is needed.
Open the AI chat again after a change, so the tool list refreshes.

## Verify

Watch the **Status** column of the `sec-api` row. When it reports a connection,
click the icon in that column to list the tools. Expect **49 tools**.

## First prompt

> What is Apple's CIK and CUSIP?

The assistant calls `mapping`. The answer gives CIK `320193` and CUSIP
`037833100`. The raw result is one text block that holds a bare JSON array, not
a `data` envelope.

## Quirks

- Use an agent, not plain chat. Community reports say MCP tools stay hidden in
  the simple chat mode. Switch the AI chat to an agent before you prompt.
- The key travels in the URL. AI Assistant documents no `headers` field for
  remote servers, so the URL is the reliable place for it here.
- The server list lives in IDE settings. It does not follow the project into
  version control, and a teammate has to repeat these steps.
- Older AI Assistant builds spoke STDIO only. Bridge to this server if the HTTP
  type is missing:

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

- An admin can preconfigure MCP servers through JetBrains IDE Services and block
  your own. Check with your administrator if **Add** does nothing.
- A bad key returns HTTP 401. Errors from a good key start with
  `sec-api error:`.

## Removal

Clear the checkbox on the `sec-api` row and click **Apply**. That stops the
server and hides its tools. The vendor documentation describes no delete action,
so the row stays in the list.

Source:
[Configure an MCP server, AI Assistant](https://www.jetbrains.com/help/ai-assistant/configure-an-mcp-server.html)
and
[Model Context Protocol (MCP)](https://www.jetbrains.com/help/ai-assistant/mcp.html)
