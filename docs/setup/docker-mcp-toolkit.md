# Connect Docker MCP Toolkit to SEC EDGAR Data with MCP

Docker MCP Toolkit runs MCP servers in containers and hands them to your clients
through the MCP Gateway. It also accepts remote servers that you reach by URL.

## Prerequisites

- Docker Desktop 4.62 or later. Open **Settings > Beta features**, turn on
  **Enable Docker MCP Toolkit**, then click **Apply**.
- The `docker mcp` CLI plugin. Run `docker mcp version` to check it.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

The Docker Desktop **Catalog** tab lists only servers from Docker's own catalog.
It has no field for a remote URL. Add a remote server with a server entry file
and the CLI.

Put the file anywhere. This page uses `~/.docker/mcp/sec-api.yaml`.

## Steps

1. Write the server entry file.

```yaml
name: sec-api
type: remote
title: SEC API
description: 49 tools over SEC EDGAR filings, XBRL financials, insider trades and enforcement data.
remote:
  url: https://api.sec-api.io/mcp
  transport_type: http
  headers:
    Authorization: Bearer YOUR_API_KEY
```

2. Restrict the file, create a profile, and add the server. The file holds your
   key in clear text. Use an absolute path in the `file://` reference.

```bash
chmod 600 ~/.docker/mcp/sec-api.yaml
docker mcp profile create --name sec
docker mcp profile server add sec --server file:///Users/you/.docker/mcp/sec-api.yaml
```

3. Connect a client to the profile.

```bash
docker mcp client connect claude-desktop --profile sec
```

Supported clients include `claude-code`, `claude-desktop`, `codex`, `continue`,
`cursor`, `gemini`, `goose`, `gordon`, `lmstudio`, `opencode`, `sema4`, `vscode`
and `zed`.

## Reload

Restart the client after `docker mcp client connect`. The gateway itself picks
up profile changes on its next start.

## Verify

```bash
docker mcp tools count
docker mcp tools list
```

Expect **49** tools from `sec-api`. `docker mcp tools` reads the active gateway
configuration. Pass gateway options through `--gateway-arg`, for example
`--gateway-arg=--profile=sec`.

## First call

```bash
docker mcp tools call filing-search '{"query":"ticker:AAPL AND formType:\"10-K\"","size":3}'
```

Expect one text block holding `{total, query, filings[]}` with three rows.

## Quirks

- **The UI cannot do this.** Docker Desktop offers no remote URL field. The CLI
  and a server entry file are the only path.
- **`type: remote` needs the `remote` block.** The older `sseEndpoint` field is
  deprecated. Use `remote.url`. `transport_type` takes `http` or `sse`. Use
  `http`. This server is stateless and serves no SSE session.
- **Legacy catalogs use a different schema.** Files under
  `~/.docker/mcp/catalogs/` nest servers under a `registry:` key. That schema
  will drift from the profile server entry spec. Use profiles.
- **The key sits in the YAML.** The `secrets` field injects environment
  variables into containers. It does not fill headers on a remote server.
- **Results arrive as one text block.** Each tool returns stringified JSON. No
  tool declares an output schema, so there is no `structuredContent`.

## Removal

```bash
docker mcp profile server remove sec --name sec-api
docker mcp profile remove sec
rm ~/.docker/mcp/sec-api.yaml
```

Run `docker mcp client --help` to find the disconnect verb for your version,
then restart the client.

## Source

[Docker MCP Toolkit | Docker Docs](https://docs.docker.com/ai/mcp-catalog-and-toolkit/toolkit/)
and the
[MCP server entry specification](https://github.com/docker/mcp-gateway/blob/main/docs/server-entry-spec.md).
