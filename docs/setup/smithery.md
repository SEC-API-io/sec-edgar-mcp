# Connect Smithery to SEC EDGAR Data with MCP

Smithery is a registry and hosting platform for MCP servers. Its CLI adds any
remote MCP URL as a connection, and can write the same server into a local
client's config.

## Prerequisites

- Node.js 20 or later.
- The CLI: `npm install -g smithery@latest`
- A Smithery account for hosted connections: `smithery auth login`
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

Smithery offers two paths.

- A **Smithery connection** lives in your Smithery namespace in the cloud.
  Headers are stored there.
- A **client install** writes the server into that client's own config file, for
  example `~/Library/Application Support/Claude/claude_desktop_config.json`.

## Steps: a Smithery connection

```bash
smithery auth login
smithery mcp add "https://api.sec-api.io/mcp" \
  --id sec-api \
  --name "SEC API" \
  --headers '{"Authorization": "Bearer YOUR_API_KEY"}'
```

`--headers` takes one JSON object as a string. Smithery stores it securely.

## Steps: install into a client

```bash
smithery mcp add "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY" --client claude
```

Clients include `claude`, `cursor`, `vscode`, `codex`, `windsurf`, `cline`,
`roocode`, `librechat`, `opencode`, `goose`, `zed`, `boltai`, `witsy` and
`enconvo`.

## Reload

A Smithery connection needs no restart. After `--client`, restart that client.

## Verify

```bash
smithery mcp get sec-api
smithery tool list sec-api
```

`smithery tool list` prints the tools. Expect **49**.

## First call

```bash
smithery tool call sec-api filing-search '{"query":"ticker:AAPL AND formType:\"10-K\"","size":3}'
```

Expect one text block holding `{total, query, filings[]}` with three rows.

## Quirks

- **Quote the URL.** The query-string form carries `?` and `&`. An unquoted URL
  breaks in the shell.
- **A bare name means a registry lookup.** `smithery mcp add sec-api` searches
  the Smithery registry. Pass the full `https://` URL to skip that.
- **Without `--id`, Smithery generates one.** Always pass `--id sec-api` so
  later commands are predictable.
- **Smithery dropped stdio in September 2025.** Remote HTTP is the only path,
  which suits this server. The server is stateless, so a Smithery connection
  holds no session state to lose.
- **Results arrive as one text block.** Each tool returns stringified JSON. No
  tool declares an output schema, so there is no `structuredContent`.
- **Do not republish this server.** `smithery mcp publish` puts a server in the
  registry under your namespace. This server is already public.

## Removal

```bash
smithery mcp remove sec-api
```

For a client install, delete the `sec-api` entry from that client's config file
and restart it.

## Source

[Smithery CLI | Smithery Documentation](https://smithery.ai/docs/concepts/cli)
