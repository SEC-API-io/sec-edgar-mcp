# Connect MCP Inspector to SEC EDGAR Data with MCP

MCP Inspector is the reference test client for MCP servers. One package ships
three clients: a web UI, a CLI and a TUI. Use it to check this server before you
wire it into anything else.

## Prerequisites

- Node.js 22.19.0 or later.
- No install. `npx` runs it.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

Ad-hoc runs need no file.

The writable catalog lives at `~/.mcp-inspector/mcp.json` on macOS, Linux and
Windows. Override the path with `--catalog <path>` or the `MCP_CATALOG_PATH`
environment variable. `--config <path>` opens a file read-only.

## Steps: ad-hoc

```bash
npx @modelcontextprotocol/inspector \
  --server-url https://api.sec-api.io/mcp \
  --transport http \
  --header "Authorization: Bearer YOUR_API_KEY"
```

The launcher prints a URL that carries a one-time session token. Open that URL.

## Steps: catalog entry

Put this in `~/.mcp-inspector/mcp.json`:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

Then run `npx @modelcontextprotocol/inspector` and connect `sec-api` from the
**Servers** tab.

## Reload

The web client reads the catalog at launch. Restart the process after you edit
the file by hand. Edits you make in the **Servers** tab apply at once.

## Verify

Open the **Tools** tab. The list holds **49** tools. The **Prompts** and
**Resources** tabs stay hidden, because this server declares tools only.

From the CLI:

```bash
npx @modelcontextprotocol/inspector --cli \
  --server-url https://api.sec-api.io/mcp \
  --transport http \
  --header "Authorization: Bearer YOUR_API_KEY" \
  --method tools/list --format json | jq '.tools | length'
```

Expect `49`.

## First call

In the **Tools** tab, select `filing-search`. Set `query` to
`ticker:AAPL AND formType:"10-K"` and `size` to `3`. Call it. Expect one text
block holding `{total, query, filings[]}` with three rows.

## Quirks

- **Use `--transport http`, never `sse`.** This server is stateless. It issues
  no `Mcp-Session-Id` and serves no SSE session.
- **Open the printed URL.** Do not type `localhost:6274` from memory. Every
  `/api/*` route needs the per-launch session token.
- **The Network tab shows the truth.** It appears only for HTTP and SSE servers.
  This server answers with `application/json`, not an SSE stream, because it
  runs with JSON responses enabled. The **Console** tab never appears, because
  that tab reads a stdio process's `stderr`.
- **Prefer `--header` over a key in the URL.** The Protocol and Network views
  mask secrets, but a URL key still lands in your shell history.
- **Never set `DANGEROUSLY_OMIT_AUTH`.** It removes the token check on the
  backend that spawns processes on your machine.

## Removal

Ad-hoc runs leave nothing behind. Stop the process.

For a catalog entry, remove the `sec-api` block from `~/.mcp-inspector/mcp.json`,
or delete the server in the **Servers** tab.

## Source

[MCP Inspector | Model Context Protocol](https://modelcontextprotocol.io/docs/tools/inspector)
