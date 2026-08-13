# Connect LibreChat to SEC EDGAR Data with MCP

LibreChat is a self-hosted, multi-user chat platform. It is an MCP host, and its
`streamable-http` transport connects to remote servers.

## Prerequisites

- A running LibreChat deployment. LibreChat names no minimum version for the
  `streamable-http` transport.
- Access to `librechat.yaml`, or the MCP permission that lets you add servers
  from the UI.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

`librechat.yaml` sits in the project root. Copy `librechat.example.yaml` to
`librechat.yaml` if the file is missing.

With Docker, mount it into the API container through
`docker-compose.override.yml`:

```yaml
services:
  api:
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

## Config

Add this block to `librechat.yaml`:

```yaml
mcpServers:
  sec-api:
    type: streamable-http
    url: https://api.sec-api.io/mcp
    headers:
      Authorization: 'Bearer ${SEC_API_KEY}'
    timeout: 60000
    initTimeout: 30000
    serverInstructions: true
```

Then set `SEC_API_KEY=YOUR_API_KEY` in your `.env` file. The header form keeps
the key out of the YAML and out of the logs. The query-string form,
`url: https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`, also works.

## Restart

Restart LibreChat after every change to `librechat.yaml`. Connections are
initialized at startup.

```bash
docker compose restart
```

Servers that you add from the UI need no restart.

## Verify

Open the **MCP Settings** panel in the right sidebar. The `sec-api` entry shows
a status icon. Click it to open the server and count its tools. Expect **49**.

The Agent Builder shows the same server as one entry. Expand it for per-tool
control.

A wrong key gives HTTP 401 and the server fails to initialize. Check the API
container logs.

## First prompt

```text
List the three most recent 10-K filings from Apple with their filing dates and
accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`.

## Known quirks

- **A reverse proxy can cut the connection.** Nginx, Traefik and Cloudflare
  often close a stream before LibreChat gives up. Keep connections alive and
  turn off response buffering on the `/api` path.
- **49 tools crowd the model context.** Trim the list in the Agent Builder if
  answers get worse after you add the server.

## Removal

Delete the `sec-api` block from `librechat.yaml` and restart. For a UI-added
server, open the **MCP Settings** panel and delete the entry.

## Source

[LibreChat Docs: MCP](https://www.librechat.ai/docs/features/mcp) and
[LibreChat Docs: MCP Servers Object Structure](https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/mcp_servers)
