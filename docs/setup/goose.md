# Connect Goose to SEC EDGAR Data with MCP

Goose is Block's open-source AI agent. It ships as a CLI and a desktop app. It
calls MCP servers "extensions" and supports remote Streamable HTTP directly.

## Prerequisites

- Goose CLI or Goose Desktop.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).

Block states no minimum version. The "Remote Extension (Streamable HTTP)"
option arrived after July 2025, so use a current release.

## Config location

| OS          | Path                                       |
| ----------- | ------------------------------------------ |
| macOS/Linux | `~/.config/goose/config.yaml`              |
| Windows     | `%APPDATA%\Block\goose\config\config.yaml` |

## Config

Use the wizard. It writes the correct field names.

1. Run `goose configure`.
2. Choose **Add Extension**.
3. Choose **Remote Extension (Streamable HTTP)**.
4. Name it `sec-api`.
5. Paste the endpoint URI
   `https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`.
6. Set the timeout to 60 seconds.
7. Give it a description, for example `SEC EDGAR filings and data`.

The wizard writes this into `config.yaml`:

```yaml
extensions:
  sec-api:
    enabled: true
    type: streamable_http
    name: sec-api
    description: SEC EDGAR filings and data
    uri: https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
    timeout: 60
```

To send the key as a header, drop the query string and add a `headers` map:

```yaml
    uri: https://api.sec-api.io/mcp
    headers:
      Authorization: Bearer ${SEC_API_KEY}
```

## Restart

Start a new session with `goose session`. Goose Desktop reloads extensions when
you save the extension settings page.

## Verify

Goose prints no tool count. Run `goose info -v` and confirm `sec-api` appears in
the enabled extensions. Then start a session and ask the agent to list its
`sec-api` tools. Expect 49.

## First prompt

> Find the five most recent 10-K filings by Apple. Give the filing date and the
> accession number for each.

Expect five rows. Each row carries a `filedAt` date and an `accessionNo` such
as `0000320193-24-000123`. Goose calls `filing-search` once.

## Quirks

- The endpoint key is `uri`, not `url`. Most other clients use `url`. A config
  copied from another client will not connect.
- The type is `streamable_http` with an underscore. Not `streamable-http`, not
  `http`.
- `timeout` is in seconds here, not milliseconds. The default is 300.
- Header values expand environment variables, so `Bearer ${SEC_API_KEY}` keeps
  the key out of the file.
- Goose has an open report of "Transport channel closed" against some
  Streamable HTTP servers. If the extension will not start, add a stdio
  extension that runs
  `npx mcp-remote https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY` instead.

## Removal

1. Run `goose configure` and choose **Toggle Extensions**. Disable `sec-api`.
   Goose removes only disabled extensions.
2. Run `goose configure` again, choose **Remove**, select `sec-api` with the
   spacebar, then press Enter.

You can also delete the `sec-api` block from `config.yaml`.

Source: [Goose extension documentation](https://goose-docs.ai/docs/getting-started/using-extensions/)
