# Connect Amazon Q Developer CLI to SEC EDGAR Data with MCP

Amazon Q Developer CLI is AWS's terminal agent. It is an MCP client and has
supported remote MCP servers over HTTP since September 2025.

## Prerequisites

- Amazon Q Developer CLI, signed in with `q login`.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).

AWS names no version number for remote MCP support. It shipped on 2025-09-18,
so any release after that date works. Run `q --version` to check.

## Config location

Amazon Q reads two kinds of file. The legacy `mcp.json` is the simpler one and
still loads by default.

| Scope     | Path                                    |
| --------- | --------------------------------------- |
| Global    | `~/.aws/amazonq/mcp.json`               |
| Workspace | `.amazonq/mcp.json`                     |
| Agents    | `~/.aws/amazonq/cli-agents/<name>.json` |

Both files load. On a name conflict the workspace entry wins and Q prints a
warning.

## Config

Create or edit `~/.aws/amazonq/mcp.json`:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

To keep the key out of the URL, send it as a header:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

A custom agent takes the same `mcpServers` block, plus a `tools` list. Use
`"@sec-api"` there to admit every tool from this server.

## Restart

Quit the chat session and run `q chat` again. Q loads MCP servers in the
background, so tools appear a few seconds after the prompt returns.

## Verify

Run `/tools` in the session. The `sec-api` group lists its tools. Expect 49.
Run `/mcp` to see the connection state of the server itself.

## First prompt

> Find the five most recent 10-K filings by Apple. Give the filing date and the
> accession number for each.

Expect five rows. Each row carries a `filedAt` date and an `accessionNo` such
as `0000320193-24-000123`. Q calls `filing-search` once.

## Quirks

- `q mcp add` has no `--type` or `--url` flag in some releases. It only builds
  stdio entries. Edit `mcp.json` by hand for this server.
- Raise the startup timeout if the server loads slowly:
  `q settings mcp.initTimeout 20000`. The value is in milliseconds.
- Legacy `mcp.json` loading is controlled by `useLegacyMcpJson` in
  `~/.aws/amazonq/agents/default.json`. It defaults to true. If your entry is
  ignored, check that flag first.
- The key sits in `mcp.json` as plain text. Set the file to mode 600.

## Removal

```bash
q mcp remove --name sec-api
```

Or delete the `sec-api` block from `mcp.json`. Then start a new session.

Source: [Amazon Q Developer MCP configuration in the CLI](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line-mcp-config-CLI.html)
