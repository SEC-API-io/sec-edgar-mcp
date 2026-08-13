# Connect Warp to SEC EDGAR Data with MCP

Warp is an agentic terminal. Its Agent Mode is an MCP client and supports
URL-based servers over Streamable HTTP and SSE, plus custom headers.

## Prerequisites

- Warp, with Agent Mode enabled.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).

Warp names no minimum version for URL-based MCP servers.

## Config location

Warp holds MCP servers in its own store, not in a file you edit. Reach the page
in any of these ways:

- **Settings > Agents > MCP servers**
- **Warp Drive > Personal > MCP Servers**
- Command palette, search **Open MCP Servers**

## Config

1. Open the MCP servers page.
2. Click **+ Add**.
3. Choose the JSON option and paste this:

```json
{
  "sec-api": {
    "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
  }
}
```

4. Click **Add**. Warp creates one server per top-level key.

To keep the key out of the URL, use a header instead:

```json
{
  "sec-api": {
    "url": "https://api.sec-api.io/mcp",
    "headers": { "Authorization": "Bearer YOUR_API_KEY" }
  }
}
```

## Restart

No restart. Start the server from the MCP servers page. Warp connects at once.
Open a new Agent Mode tab if the tools do not appear.

## Verify

The MCP servers page lists `sec-api` with its running status. Open the entry to
see the tools it exposes. Expect 49.

## First prompt

> Find the five most recent 10-K filings by Apple. Give the filing date and the
> accession number for each.

Expect five rows. Each row carries a `filedAt` date and an `accessionNo` such
as `0000320193-24-000123`. Warp calls `filing-search` once.

## Quirks

- A server entry takes exactly one transport: `url`, or `command` plus `args`,
  or `warp_id`. Do not mix them.
- `env` is only valid with `command`. Warp strips an `env` block from a URL
  server, so put auth in `headers` or in the URL.
- Warp does not support OAuth MCP servers for cloud agents. That does not
  affect this server, which authenticates with a key.
- If a Warp release rejects the URL form, wrap the server in the stdio bridge.
  Add a CLI server that runs `npx mcp-remote <url> --transport http-only`,
  where `<url>` is `https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`.

## Removal

1. Open the MCP servers page.
2. Select `sec-api`.
3. Stop it, then delete it.

For cloud agents, drop the `--mcp` argument that carries this config.

Source: [Warp MCP documentation](https://docs.warp.dev/agent-platform/capabilities/mcp/)
