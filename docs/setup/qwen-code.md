# Connect Qwen Code to SEC EDGAR Data with MCP

Qwen Code is a terminal coding agent from the Qwen team. It is a full MCP
client. It handles tools, prompts and resources, and it speaks Streamable HTTP.
No bridge is needed.

## Prerequisites

- Node.js and the `qwen` CLI on your `PATH`.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Qwen Code reads `mcpServers` from `settings.json`. Two scopes exist. The project
scope wins in that project.

| Scope            | Path                                    |
| ---------------- | --------------------------------------- |
| User, default    | `~/.qwen/settings.json`                 |
| Project          | `.qwen/settings.json` in the repo root  |

On Windows the user file is `%USERPROFILE%\.qwen\settings.json`.

## Config

The fastest path is the CLI. Run this once. It writes the entry for you.

```bash
qwen mcp add --transport http --scope user sec-api \
  https://api.sec-api.io/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

The file form does the same job. Note the field name is `httpUrl`, not `url`.

```json
{
  "mcpServers": {
    "sec-api": {
      "httpUrl": "https://api.sec-api.io/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" },
      "timeout": 35000
    }
  }
}
```

The key can also ride in the URL. Drop the `headers` line and set `httpUrl` to
`https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY` instead.

## Restart

Restart `qwen` in the same project after you add the server. A running session
does not pick up a new entry.

## Verify

Start `qwen` and type `/mcp`. The management dialog opens. Select `sec-api` to
see its status and tools. Expect **49 tools**. Prompts and resources show 0,
because this server offers tools only. `qwen mcp list` gives the same status
from outside a session.

## First prompt

> Find the newest Apple 10-K. Give me the filing date and the primary document
> URL.

Qwen Code calls `filing-search` and asks you to confirm the call. The answer is
one text block with stringified JSON. It holds `total` and a `filings` array.
Each filing carries `accessionNo`, `filedAt` and `linkToFilingDetails`.

## Quirks

- Remote servers use `httpUrl`. The `url` field means SSE, and this server has
  no SSE endpoint.
- Discovery for a remote server times out after 5 seconds by default. On a slow
  link, raise it with `"discoveryTimeoutMs": 10000` on the server entry.
- `timeout` is the tool-call timeout in milliseconds. The default is 600000.
- Headers accept `${VAR}` from the environment. Use that to keep the key out of
  a committed `.qwen/settings.json`.
- Leave `trust` at `false` and confirm each call.
- Cut the tool list with `includeTools` if 49 tools crowd the model. Example:
  `"includeTools": ["filing-search", "extractor", "xbrl-to-json"]`.

## Removal

```bash
qwen mcp remove sec-api
```

Or delete the `sec-api` entry from `settings.json`. Restart `qwen` afterwards.

Source:
[Connect Qwen Code to tools via MCP](https://qwenlm.github.io/qwen-code-docs/en/users/features/mcp/)
