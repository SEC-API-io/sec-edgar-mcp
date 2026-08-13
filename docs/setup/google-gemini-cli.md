# Connect Google Gemini CLI to SEC EDGAR Data with MCP

Gemini CLI is Google's open-source terminal agent. It is a full MCP client and
speaks stdio, SSE and Streamable HTTP, so it connects to this server directly.

## Prerequisites

- Gemini CLI. Install it with `npm install -g @google/gemini-cli`, or run it
  once with `npx @google/gemini-cli`.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).

Google states no minimum version for the `httpUrl` transport. Use a current
release.

## Config location

Gemini CLI reads the `mcpServers` block from `settings.json`.

| Scope   | Path                      |
| ------- | ------------------------- |
| User    | `~/.gemini/settings.json` |
| Project | `.gemini/settings.json`   |

On Windows the user file is `%USERPROFILE%\.gemini\settings.json`.

## Config

Add the server from the shell:

```bash
gemini mcp add -s user -t http \
  sec-api "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
```

Or edit `settings.json` by hand:

```json
{
  "mcpServers": {
    "sec-api": {
      "httpUrl": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
      "timeout": 60000
    }
  }
}
```

To keep the key out of the URL, send it as a header:

```json
{
  "mcpServers": {
    "sec-api": {
      "httpUrl": "https://api.sec-api.io/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" },
      "timeout": 60000
    }
  }
}
```

## Restart

Quit Gemini CLI and start it again. Inside a running session you can instead
run `/mcp reload`.

## Verify

Run `/mcp list` in a session. The server reports its tool count:

```text
sec-api - Ready (49 tools)
```

`gemini mcp list` shows the entry too, but it does not connect to the server.

## First prompt

> Find the five most recent 10-K filings by Apple. Give the filing date and the
> accession number for each.

Expect five rows. Each row carries a `filedAt` date and an `accessionNo` such
as `0000320193-24-000123`. Gemini calls `filing-search` once.

## Quirks

- Use `httpUrl`. The `url` key selects SSE. This server is stateless and issues
  no session id, so the SSE path fails.
- `timeout` is in milliseconds. Set 60000 or more.
- The key sits in `settings.json` as plain text. Set the file to mode 600.
- Set `"trust": true` only if you accept tool calls without a prompt.

## Removal

```bash
gemini mcp remove sec-api -s user
```

Removal is scope aware. If the entry survives, delete the `sec-api` block from
the `settings.json` file that holds it.

Source: [Gemini CLI MCP server documentation](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/mcp-server.md)
