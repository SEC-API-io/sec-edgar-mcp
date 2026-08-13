# Connect OpenAI Codex CLI to SEC EDGAR Data with MCP

Codex CLI is OpenAI's terminal coding agent. It connects to remote MCP servers
over Streamable HTTP, and to local servers over stdio.

## Prerequisites

- Codex CLI, installed and signed in. Run `codex --version` to see the build.
- A sec-api.io API key. Get one at [sec-api.io](https://sec-api.io/profile).
- An HTTPS URL. Codex refuses plain HTTP for a remote server. This server is
  HTTPS.

OpenAI states no minimum Codex version for remote MCP servers. Use a current
build.

## Config location

| Scope       | Path                                |
| ----------- | ----------------------------------- |
| User        | `~/.codex/config.toml`              |
| Custom home | `$CODEX_HOME/config.toml`           |
| Project     | `.codex/config.toml` in the project |

Codex CLI, the IDE extension and the ChatGPT desktop app share this file. Project
config loads only for trusted projects.

## Add the server

One command:

```bash
codex mcp add sec-api --url "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
```

Or edit `~/.codex/config.toml` by hand:

```toml
[mcp_servers.sec-api]
url = "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
```

### Keep the key out of the file

The server also reads an `Authorization: Bearer` header. Put the key in an
environment variable instead:

```toml
[mcp_servers.sec-api]
url = "https://api.sec-api.io/mcp"
bearer_token_env_var = "SEC_API_KEY"
```

```bash
export SEC_API_KEY=YOUR_API_KEY
```

Codex reads the variable at startup. Export it in your shell profile.

## Reload

Quit Codex and start it again. `codex mcp add` writes the file at once, but a
running session keeps the server list it loaded.

## Verify

```bash
codex mcp list
```

The `sec-api` row shows the URL. Inside a session, run `/mcp`. It lists the
connected servers and their tools. Expect **49** tools.

## First prompt

> Use the sec-api tools. List the last five 8-K filings from Tesla with their
> item numbers.

Expect five rows. Each row carries a filed date, an accession number and the 8-K
items. Codex prints the tool call and the raw result above the answer.

## Quirks

- **The key is `mcp_servers`, not `mcpServers`.** Codex uses TOML, not JSON.
  `[mcp.servers."name"]` never connects.
- **No `type` field.** A `url` key makes the server Streamable HTTP. A `command`
  key makes it stdio.
- **Server names take letters, numbers, `-` and `_` only.** Codex rejects the
  rest.
- **Project config needs trust.** If `.codex/config.toml` looks ignored, the
  directory is not trusted. Move the server to `~/.codex/config.toml`.
- **Timeouts.** Codex allows 10 seconds to start a server and 60 seconds per tool
  call. Raise `startup_timeout_sec` on a slow link.
- **No OAuth.** Do not run `codex mcp login sec-api`. The key in the URL or in
  the bearer header is the whole authentication story. A bad key returns HTTP
  401.

Trim the list when 49 tools crowd the context:

```toml
[mcp_servers.sec-api]
url = "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
enabled_tools = ["filing-search", "full-text-search", "extractor", "xbrl-to-json"]
```

## Removal

```bash
codex mcp remove sec-api
```

Or delete the `[mcp_servers.sec-api]` block from `config.toml`. Restart Codex.

## Source

[Codex MCP guide](https://developers.openai.com/codex/mcp) and the
[Codex config reference](https://developers.openai.com/codex/config-reference),
read 2026-08-13.
