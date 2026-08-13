# Connect OpenClaw to SEC EDGAR Data with MCP

OpenClaw is an open-source personal AI agent with a gateway, a CLI and a web
UI. It is an MCP client and connects to remote servers over Streamable HTTP.

## Prerequisites

- OpenClaw, with the `openclaw` CLI on your path.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).

OpenClaw names no minimum version for remote MCP servers.

## Config location

`~/.openclaw/openclaw.json`. MCP servers live under the `mcp.servers` key. The
web UI edits the same block at **/settings/mcp**.

## Config

Add the server from the shell:

```bash
openclaw mcp add sec-api \
  --url "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY" \
  --transport streamable-http
```

`openclaw mcp add` probes the server before it saves, so a bad key fails fast.

The equivalent block in `openclaw.json`:

```json
{
  "mcp": {
    "servers": {
      "sec-api": {
        "transport": "streamable-http",
        "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "enabled": true,
        "requestTimeoutMs": 60000,
        "connectionTimeoutMs": 10000
      }
    }
  }
}
```

To keep the key out of the URL, pass a header:

```bash
openclaw mcp add sec-api \
  --url "https://api.sec-api.io/mcp" \
  --transport streamable-http \
  --header Authorization "Bearer YOUR_API_KEY"
```

## Restart

```bash
openclaw mcp reload
```

This drops the cached MCP runtimes. A full gateway restart is not needed.

## Verify

```bash
openclaw mcp probe sec-api --json
```

`probe` opens a live session and prints the tool count. Expect `"tools": 49`.
`openclaw mcp status --verbose` shows the resolved transport and timeouts
without connecting.

## First prompt

> Find the five most recent 10-K filings by Apple. Give the filing date and the
> accession number for each.

Expect five rows. Each row carries a `filedAt` date and an `accessionNo` such
as `0000320193-24-000123`. OpenClaw calls `filing-search` once.

## Quirks

- The canonical key is `transport: "streamable-http"` with a hyphen.
  `"type": "http"` is a CLI alias. `openclaw mcp set` and
  `openclaw doctor --fix` rewrite it into `transport`.
- Timeouts are in milliseconds. Keep `requestTimeoutMs` at 60000 or more.
- MCP tools arrive under the `bundle-mcp` plugin. Sandboxed sessions need a
  second grant in `tools.sandbox.tools`. The glob for this server is
  `sec-api__*`.
- Header values expand environment variables, so `Bearer ${SEC_API_KEY}` keeps
  the key out of the file.
- Leave `auth` unset. This server uses a key, not OAuth, so
  `openclaw mcp login` does not apply.

## Removal

```bash
openclaw mcp unset sec-api
openclaw mcp reload
```

Source: [OpenClaw MCP CLI documentation](https://docs.openclaw.ai/cli/mcp)
