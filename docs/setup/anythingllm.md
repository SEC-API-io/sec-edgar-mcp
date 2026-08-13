# Connect AnythingLLM to SEC EDGAR Data with MCP

AnythingLLM is a desktop and Docker app for chat over your own documents. It is
an MCP host, and it supports remote servers over Streamable HTTP.

## Prerequisites

- AnythingLLM Desktop or Docker with MCP support. The vendor names no minimum
  version.
- A model that supports tool use.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

AnythingLLM reads one file, `plugins/anythingllm_mcp_servers.json`, inside its
storage directory. Open the **Agent Skills** page once and the app creates the
file for you.

| Target  | Path                                                                                              |
| ------- | ------------------------------------------------------------------------------------------------- |
| macOS   | `~/Library/Application Support/anythingllm-desktop/storage/plugins/anythingllm_mcp_servers.json`  |
| Windows | `C:\Users\<you>\AppData\Roaming\anythingllm-desktop\storage\plugins\anythingllm_mcp_servers.json` |
| Linux   | `~/.config/anythingllm-desktop/storage/plugins/anythingllm_mcp_servers.json`                      |
| Docker  | `$STORAGE_LOCATION/plugins/anythingllm_mcp_servers.json`                                          |

## Config

Add this entry:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "streamable",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

To keep the key out of the URL, use the `headers` field:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "streamable",
      "url": "https://api.sec-api.io/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

`headers` is an AnythingLLM extension. It has no effect in other clients.

## Reload

Open the **Agent Skills** page and click **Refresh**. AnythingLLM rereads the
file and restarts its servers. There is no app restart.

## Verify

Look at the MCP Servers list on the **Agent Skills** page. The `sec-api` card
shows its status and the tools that it loaded. Expect **49**.

An error appears on the card itself. A wrong key gives HTTP 401.

## First prompt

MCP tools run in agent mode only. Start the message with `@agent`.

```text
@agent List the three most recent 10-K filings from Apple with their filing
dates and accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`.

## Known quirks

- **`"type": "streamable"` is required.** AnythingLLM assumes `sse` when `type`
  is missing. This server issues no session ID, so the SSE path fails. The value
  is `streamable`, not `streamable-http` and not `http`.
- **Tools work only under `@agent`.** A plain chat message never calls them.
  This is the most common cause of "the server loaded but nothing happens".
- **Autostart.** Every server starts with the app unless you set
  `"anythingllm.autoStart": false` on it. You can also stop and start a server
  from the gear icon on the **Agent Skills** page.
- **The key sits in a plain JSON file.** Rotate it at
  [sec-api.io](https://sec-api.io/profile) if it leaks.
- **49 tools is a large tool list.** Small local models pick badly with that
  many choices. Name the tool in the prompt if the answer looks wrong.

## Removal

Click the `sec-api` server on the **Agent Skills** page, open the gear icon and
click **Delete**. That removes the entry from the file and stops the connection.
You can also delete the object by hand and click **Refresh**.

## Source

[AnythingLLM Docs: MCP Compatibility Overview](https://docs.anythingllm.com/mcp-compatibility/overview)
and [MCP on AnythingLLM Desktop](https://docs.anythingllm.com/mcp-compatibility/desktop)
