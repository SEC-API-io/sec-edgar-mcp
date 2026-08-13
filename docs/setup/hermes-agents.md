# Connect Hermes Agents to SEC EDGAR Data with MCP

Hermes Agent is Nous Research's open-source agent. MCP ships with the standard
install. It runs local stdio servers and remote HTTP servers from one config.

## Prerequisites

- Hermes Agent, with the `hermes` CLI on your path.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).

Nous Research names no minimum version for remote HTTP MCP servers.

## Config location

| Scope       | Path                                    |
| ----------- | --------------------------------------- |
| Default     | `~/.hermes/config.yaml`                 |
| Per profile | `~/.hermes/profiles/<name>/config.yaml` |

Secrets belong in `~/.hermes/.env`.

## Config

Add this to `config.yaml`. `mcp_servers` is a map keyed by server name, not a
list.

```yaml
mcp_servers:
  sec-api:
    url: "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    enabled: true
    timeout: 60
    connect_timeout: 30
```

A `url` with no `transport` key means Streamable HTTP. That is what this server
speaks. Do not set `transport: sse`.

To keep the key out of the URL, send it as a header:

```yaml
mcp_servers:
  sec-api:
    url: "https://api.sec-api.io/mcp"
    headers:
      Authorization: "Bearer YOUR_API_KEY"
    enabled: true
    timeout: 60
```

## Restart

Run `/reload-mcp` in a session. It rereads the config and refreshes the tool
list. A new session or a gateway restart also picks up the change.

## Verify

Confirm the entry parsed:

```bash
hermes config show | grep -A 12 mcp_servers
```

Then start a session and ask the agent to list its `sec-api` tools. Expect 49.
Each one registers as `mcp__sec_api__<tool>`.

## First prompt

> Find the five most recent 10-K filings by Apple. Give the filing date and the
> accession number for each.

Expect five rows. Each row carries a `filedAt` date and an `accessionNo` such
as `0000320193-24-000123`. Hermes calls `filing-search` once.

## Quirks

- `timeout` and `connect_timeout` are in seconds, not milliseconds. The
  defaults are 300 and 60.
- Tool names get sanitized. Every character that is not a letter, a digit or an
  underscore becomes an underscore, so `filing-search` shows as
  `mcp__sec_api__filing_search`.
- Write `tools.include` and `tools.exclude` filters with the original names,
  hyphens and all. Use `filing-search`, not `filing_search`.
- Leave `ssl_verify` alone. Turning it off exposes your key.
- A crashed connection is reported as a timeout, not as an error. Check the
  URL and the key before you blame the network.
- `hermes import-agent claude-code` converts an existing `mcpServers` block
  from `~/.claude.json` into `mcp_servers` for you.

## Removal

Delete the `sec-api` block from `config.yaml`, then run `/reload-mcp`. The
`hermes mcp` picker also uninstalls entries it installed.

Source: [Hermes Agent MCP config reference](https://hermes-agent.nousresearch.com/docs/reference/mcp-config-reference/)
