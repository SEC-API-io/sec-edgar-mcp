# Connect Continue to SEC EDGAR Data with MCP

Continue is an open-source coding assistant for VS Code and JetBrains. It is an
MCP client. It speaks Streamable HTTP, so it connects to this server directly.
No bridge is needed.

## Prerequisites

- VS Code or a JetBrains IDE, with the Continue extension installed.
- A chat model with tool calling. Continue needs the `tool_use` capability for
  agent mode. It detects this for most models.
- A sec-api API key. See [authentication](../authentication.md).

## Config location

Continue reads two places. Pick one. The global config suits a personal setup.

| Platform      | Path                                  |
| ------------- | ------------------------------------- |
| macOS, Linux  | `~/.continue/config.yaml`              |
| Windows       | `%USERPROFILE%\.continue\config.yaml` |

To share the server with a team, create the folder `.continue/mcpServers/` at
the top of the workspace and add one YAML file in it.

Open the global file from the IDE. Press `Cmd/Ctrl + L` in VS Code, or
`Cmd/Ctrl + J` in JetBrains. Click the config dropdown above the chat input.
Then click the cog next to **Local Config**.

## Config

Add this to `config.yaml`. Replace `YOUR_API_KEY` with your key.

```yaml
mcpServers:
  - name: sec-api
    type: streamable-http
    url: https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

A workspace block needs the three metadata fields. Save it as
`.continue/mcpServers/sec-api.yaml`:

```yaml
name: SEC API mcpServer
version: 0.0.1
schema: v1
mcpServers:
  - name: sec-api
    type: streamable-http
    url: https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

To keep the key out of the URL, cut the `?apiKey=` part and add a
`requestOptions` block with
`headers: { Authorization: Bearer ${{ secrets.SEC_API_KEY }} }`. Continue
documents `requestOptions` for `sse` and `streamable-http` servers.

## Reload

Save the file. Continue refreshes the config on save. No restart is needed. If
the tools do not appear, close and reopen the IDE.

## Verify

Switch the mode selector to **Agent**. Open the tools list above the chat input.
The `sec-api` group appears there. Expect **49 tools**.

## First prompt

> Find the newest Apple 10-K. Give me the filing date and the primary document
> URL.

Continue calls `filing-search` and asks you to approve the call. The answer is
one text block with stringified JSON. It holds `total` and a `filings` array.
Each filing carries `accessionNo`, `filedAt` and `linkToFilingDetails`.

## Quirks

- MCP works in **agent** mode only. Chat mode and plan mode ignore the server.
- The transport value is `streamable-http`, with a hyphen.
- The `config.yaml` reference page still lists only the stdio properties. The
  MCP deep dive page is the current source for `type` and `url`.
- `${{ secrets.NAME }}` reads a Continue secret, not a shell variable. Set the
  secret in the Continue hub or in a local `.env` next to the config.
- Continue also loads a JSON MCP file from `.continue/mcpServers/`. Use that
  only if you already have one from another client.

## Removal

Delete the `sec-api` entry from `config.yaml`, or delete
`.continue/mcpServers/sec-api.yaml`. Save. Continue drops the tools at once.

Source:
[How to Set Up Model Context Protocol (MCP) in Continue](https://docs.continue.dev/customize/deep-dives/mcp)
