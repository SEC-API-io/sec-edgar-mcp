# Getting started

One minute from nothing to your first EDGAR answer. Four steps.

## 1. Get an API key

Sign up at [sec-api.io/signup](https://sec-api.io/signup) and copy your key. It
is the only credential the server needs.

## 2. Add the server to your client

Most MCP clients read a JSON config file. Add this block and replace
`YOUR_API_KEY` with your key:

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

The server is remote, so there is nothing to install. A client that asks only for
a URL takes the same URL. Per-client steps are in the
[setup guides](./setup/README.md).

## 3. Restart the client

The client connects on start. After the restart it must report **49 tools** under
the name `sec-api`. If it reports none, see [Troubleshooting](#troubleshooting).

## 4. Ask one verification question

> Get the newest Apple 10-K. Give me the filing date and the primary document
> URL.

The agent calls `filing-search`:

```json
{
  "name": "filing-search",
  "arguments": { "query": "ticker:AAPL AND formType:\"10-K\"", "size": 1 }
}
```

The server answers with one text block that holds stringified JSON:

```json
{
  "total": { "value": 33, "relation": "eq" },
  "filings": [
    {
      "accessionNo": "0000320193-25-000079",
      "filedAt": "2025-10-31T06:01:26-04:00",
      "linkToFilingDetails": "https://www.sec.gov/Archives/edgar/data/320193/000032019325000079/aapl-20250927.htm"
    }
  ]
}
```

Your setup works. The numbers change as Apple files. The shape does not. Every
tool answers the same way: one text content block, JSON as a string. No tool
declares an output schema, so there is no `structuredContent`. An error arrives
with `isError` true and the text `sec-api error: <message>`.

## Troubleshooting

| Symptom               | Cause                         | Fix                                                     |
| --------------------- | ----------------------------- | ------------------------------------------------------- |
| HTTP 401              | Bad or missing key            | Recheck the key in your config, then reload the client. |
| Client shows no tools | Client demands an SSE session | Bridge over stdio. See below.                           |

The server is stateless and sends no `Mcp-Session-Id`. A client that needs a
session can bridge to stdio:

```bash
npx mcp-remote https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

## Next steps

- [Setup guides](./setup/README.md). Configure Claude, Cursor and other clients.
- [Tool reference](./tools/README.md). All 49 tools, one page each.
- [Query language](./query-language.md). Lucene syntax and the field names.
