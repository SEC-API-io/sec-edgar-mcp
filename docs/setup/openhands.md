# Connect OpenHands to SEC EDGAR Data with MCP

OpenHands is an open-source software agent. It runs as a CLI and as a web app.
Both are MCP clients. Both speak Streamable HTTP, which OpenHands calls SHTTP.
No bridge is needed.

## Prerequisites

- OpenHands CLI 1.0.0 or later for the CLI path. Release 1.0.0 moved MCP config
  from TOML to JSON. Older setups must be redone.
- A running OpenHands app for the app path.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

| Product           | File                    |
| ----------------- | ----------------------- |
| OpenHands CLI     | `~/.openhands/mcp.json` |
| OpenHands app     | `config.toml`, section `[mcp]` |

The app also exposes the same settings in the UI, under `Settings > MCP`.

## Config: the CLI

Do not hand-edit the JSON. Run the add command. It writes the entry for you.

```bash
openhands mcp add sec-api --transport http \
  --header "Authorization: Bearer YOUR_API_KEY" \
  https://api.sec-api.io/mcp
```

The key may also travel in the URL. Drop the `--header` line and pass
`"https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"` as the target instead.

## Config: the app

Add this to `config.toml`. TOML needs the inline array form.

```toml
[mcp]
shttp_servers = [
    { url = "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY", timeout = 60 }
]
```

`timeout` is the tool execution timeout in seconds. It accepts 1 to 3600.

## Restart

The CLI mounts servers when a conversation starts. Restart the conversation
after any change. The app reads the MCP config at startup, so restart the app.

## Verify

In the CLI, run `openhands mcp list` to confirm the entry, then start a
conversation and type `/mcp`. It lists the active servers and any pending
change. In the app, open `Settings > MCP`. Expect **49 tools** under `sec-api`.

## First prompt

> Find the newest Apple 10-K. Give me the filing date and the primary document
> URL.

The agent calls `filing-search`. The answer is one text block with stringified
JSON. It holds `total` and a `filings` array. Each filing carries
`accessionNo`, `filedAt` and `linkToFilingDetails`.

## Quirks

- `/mcp` is read-only. Change servers with `openhands mcp` commands, not from
  inside a conversation.
- The app's TOML form accepts `api_key` but no arbitrary headers. Put the key in
  the URL there, or use the CLI, which does take `--header`.
- OpenHands can fail silently when an MCP server rejects the connection. If the
  tools do not appear, test the URL with `curl` first. A bad key returns
  HTTP 401.
- For headless or evaluation runs, set `enable_mcp=True` on the agent config.
  The `[mcp]` block alone is not enough there.
- Do not wrap this server in a stdio proxy. It is already an HTTP server.

## Removal

```bash
openhands mcp remove sec-api
```

To keep the entry but stop the connection, run `openhands mcp disable sec-api`.
In the app, delete the entry from `shttp_servers` in `config.toml`, or remove it
on the `Settings > MCP` page. Restart afterwards.

Source:
[OpenHands CLI MCP servers](https://docs.openhands.dev/openhands/usage/cli/mcp-servers)
and
[Model Context Protocol (MCP) settings](https://docs.openhands.dev/openhands/usage/settings/mcp-settings)
