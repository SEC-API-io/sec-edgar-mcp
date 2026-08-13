# Connect any stdio-only client to SEC EDGAR Data with MCP

`mcp-remote` is a stdio-to-HTTP bridge. It lets a client that speaks stdio only
reach this remote server. Its author calls it experimental.

## When you need it

Read your client's config format.

- The config takes `url`, or `type: "http"`. Connect straight to the server. You
  do not need this page.
- The config takes only `command` and `args`. Use the bridge.

## Prerequisites

- Node.js with `npx` on the PATH.
- A client that runs stdio MCP servers.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

There is no config file for `mcp-remote` itself. Put the command in your
client's own MCP config. The bridge keeps its state in `~/.mcp-auth`.

## The config snippet

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://api.sec-api.io/mcp",
        "--transport",
        "http-only",
        "--header",
        "Authorization:${SEC_API_AUTH}"
      ],
      "env": {
        "SEC_API_AUTH": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

The header value carries no space around the colon. Cursor and Claude Desktop on
Windows mangle spaces inside `args`. Spaces inside `env` values are safe.

The shortest form puts the key in the URL:

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "--transport",
        "http-only"
      ]
    }
  }
}
```

## Reload

Restart the client. It starts one `npx` process per configured server.

## Verify

Run the bridge's own client from a terminal:

```bash
npx -p mcp-remote@latest mcp-remote-client \
  "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY" --transport http-only
```

It connects and prints the tool list. Expect **49** tools. Your client shows the
same count in its own MCP or tools panel.

## First prompt

> Search EDGAR for Tesla 8-K filings from March 2024.

Expect one `filing-search` call. The answer lists filings with form types,
filing dates and document links.

## Quirks

- **Always pass `--transport http-only`.** The default is `http-first`, which
  falls back to SSE on a 404. This server never serves SSE. `http-only` fails
  fast instead of hanging on a retry.
- **No OAuth runs here.** This server uses a static key. The OAuth flow stays
  idle when you pass `--header` or put the key in the URL.
- **Colon spacing.** Write `Authorization:${SEC_API_AUTH}` inside `args`.
- **Stale state blocks connections.** Delete `~/.mcp-auth` and restart the
  client if a connection fails for no clear reason. Add `--debug` for verbose
  logs in `~/.mcp-auth/{server_hash}_debug.log`.
- **The bridge costs a process.** Each server adds a Node process and a startup
  delay. Use the direct HTTP form whenever the client supports it.

## Removal

Delete the `sec-api` block from the client config. Restart the client. Then
remove the bridge state:

```bash
rm -rf ~/.mcp-auth
```

## Source

[geelen/mcp-remote on GitHub](https://github.com/geelen/mcp-remote)
