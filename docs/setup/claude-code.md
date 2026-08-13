# Connect Claude Code to SEC EDGAR Data with MCP

Claude Code is Anthropic's coding agent for the terminal and the desktop app. It
speaks Streamable HTTP natively, so it connects to this server directly. You do
not need a bridge.

## Prerequisites

- Claude Code installed. Print the version with `claude --version`.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).
- Anthropic states no minimum version for the HTTP transport. Version 2.1.221 or
  later also shows a cached tool count in `/mcp`.

## Config location

Claude Code writes the server into one of three places. Pick the scope with
`--scope`.

| Scope             | File                                             | Loads in      | Shared                       |
| ----------------- | ------------------------------------------------ | ------------- | ---------------------------- |
| `local` (default) | `~/.claude.json`, under the current project path | This project  | No                           |
| `project`         | `.mcp.json` in the project root                  | This project  | Yes, through version control |
| `user`            | `~/.claude.json`                                 | Every project | No                           |

Windows uses the same names in your home directory, for example
`C:\Users\<you>\.claude.json`.

## Configure

Run one command. Quote the URL, because it carries a query string.

```bash
claude mcp add --transport http sec-api "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
```

To keep the key out of the URL, send it as a header:

```bash
claude mcp add --transport http sec-api https://api.sec-api.io/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

You can also write the JSON yourself. The `type` field is required. Claude Code
skips an entry that has a `url` but no `type`.

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

## Restart

`claude mcp add` writes the file immediately. Start a new session to load the
server. In a running session, open `/mcp` and reconnect it.

A project-scoped server needs your approval the first time. Claude Code asks in
the session. Reset those answers with `claude mcp reset-project-choices`.

## Verify

Run `/mcp`. The panel lists `sec-api` and prints the tool count next to it.
Expect 49.

`claude mcp list` prints `✔ Connected`. A wrong key gives
`✘ Failed to connect` with HTTP 401 in the detail line.

## First prompt

> Use sec-api to list the 5 most recent 10-K filings from Apple.

Claude calls `filing-search` once. The answer holds five rows. Each row gives
the accession number, the filing date, and the document URL. See
[filing-search](../tools/filing-search.md).

## Quirks

- Claude Code writes a tool result larger than 25,000 tokens to a file and tells
  Claude the path. [`xbrl-to-json`](../tools/xbrl-to-json.md) and
  [`form-npx-file`](../tools/form-npx-file.md) hit this often.
- Project scope commits your key to `.mcp.json` unless .gitignored. Use local or user scope, or the
  `Authorization` header, when the repository is public.

## Remove

```bash
claude mcp remove sec-api
```

For project scope, delete the `sec-api` block from `.mcp.json` and commit the
change.

## Source

[Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
