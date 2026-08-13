# Authentication

The SEC EDGAR MCP server uses one credential. It is your sec-api API key. There
is no OAuth flow, no login, and no session.

Get a key at [sec-api.io/signup](https://sec-api.io/signup).

## About the key

Copy the key exactly as shown in your profile. Changing its case breaks it, and
no spaces, quotes or line breaks may touch the value.

The MCP server and the REST API share the same key.

The model never sees the key. No tool takes an API key as an argument, so the
key stays out of tool inputs and tool results. Do not paste the key into a chat
prompt.

## Where to put the key

The server reads the key from one of two places. Both work.

| Place                    | Form                                 | Use it for                          |
| ------------------------ | ------------------------------------ | ----------------------------------- |
| `apiKey` query parameter | `?apiKey=YOUR_API_KEY`               | Any client with a single URL field  |
| `Authorization` header   | `Authorization: Bearer YOUR_API_KEY` | Clients that support custom headers |

### The `Authorization` header

Put the key in a header when your client supports one. The URL then carries no
key, which keeps it simple to paste and share.

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp",
      "headers": {
        "Authorization": "YOUR_API_KEY"
      }
    }
  }
}
```

Some clients expand environment variables inside the config file. Where your
client supports it, read the key from the environment and keep it out of the
file. Your client's page under [setup guides](./setup/README.md) says whether it
can do this.

Not every MCP client can send custom headers. Chat apps with a simple URL field
often cannot. Those clients need the query form.

### The `apiKey` query parameter

```text
https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

It works in every client and needs one field.

## Keep the key out of git

This repository shows the pattern. `.mcp-example.json` holds the placeholder and
is committed. The real `.mcp.json` is listed in `.gitignore`.

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

## Per-environment keys

Use one key per environment.

| Environment           | Key | Why                                                   |
| --------------------- | --- | ----------------------------------------------------- |
| Desktop MCP client    | A   | Interactive use. You notice a failure at once.        |
| CI or batch job       | B   | High volume. Revoke it without breaking your desktop. |
| Production agent      | C   | Must keep working while you rotate the others.        |
| Demo or shared laptop | D   | Treat it as public. Revoke it after the demo.         |

Separate keys give you two concrete benefits.

- **Usage attribution.** The [`api-key-usage`](./tools/api-key-usage.md) tool
  reports monthly bandwidth for the key that calls it. Separate keys turn one
  number into a per-environment breakdown.
- **Small blast radius.** You revoke one key and one environment stops. The
  others keep running.

## What a 401 looks like

Authentication fails in two different ways. They look nothing alike.

### Failure 1: missing or malformed key, HTTP 401

The endpoint checks the format before it does anything else. A key that fails
the pattern gets no MCP response at all.

```json
{ "error": "invalid or missing apiKey" }
```

The HTTP status is 401. Confirmed triggers:

| Request                           | Result              |
| --------------------------------- | ------------------- |
| No key in URL and no header       | 401                 |
| `?apiKey=abc`                     | 401, too short      |
| `?apiKey=abc` plus a valid header | 401, the query wins |

This check is a format check only. The server does not look up an account here.
A well-formed key that belongs to nobody passes this gate.

In a client, this failure appears as a connection error. The server shows as
failed, disabled, or red, and it reports **0 tools**. The client cannot list
tools, because `tools/list` gets the 401 too.

### Failure 2: well-formed but invalid key, HTTP 200

A key with the right shape but no valid account reaches the tools. `tools/list`
succeeds and returns all 49 tools. The first tool call then fails:

```json
{
  "result": {
    "content": [
      {
        "type": "text",
        "text": "sec-api error: API token invalid. Please get a valid token from sec-api.io"
      }
    ],
    "isError": true
  },
  "jsonrpc": "2.0",
  "id": 2
}
```

The HTTP status is 200. The error lives inside the MCP result, with `isError`
set to true, like every other tool error. See
[response format](./response-format.md).

In a client, the agent sees the error text and usually reports it in the
conversation. The server itself still looks healthy.

### Diagnosis

| Symptom                                       | Cause                              | Fix                                                   |
| --------------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| 0 tools, connection error                     | Format check failed                | Check length, case, and where you put the key         |
| 49 tools, every call says "API token invalid" | Key is well formed but not valid   | Check for a typo, a revoked key, or the wrong account |
| Header ignored, still 401                     | A stale `?apiKey=` in the URL wins | Delete the query string from the URL                  |

One other error is not an authentication problem. HTTP 503 with
`mcp server not initialized` means the server is starting up. Retry it.

## Verify a key in one command

`tools/list` is not a key test. It passes with any well-formed key. Test
with a real tool call. `api-key-usage` is the cheapest one in the server.

```bash
curl -s https://api.sec-api.io/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"api-key-usage","arguments":{"date":"2026-08"}}}'
```

A valid key answers with data and `isError` false:

```json
{
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"date\":\"2026-08\",\"monthlyBandwidthUsedInMb\":39.12}"
      }
    ],
    "isError": false
  },
  "jsonrpc": "2.0",
  "id": 1
}
```

Read the two failure shapes above for anything else.

## Keys and the stdio bridge

Clients that speak only stdio need the `mcp-remote` bridge. The bridge takes the
server URL on its command line, so a key in the URL lands in your shell history
and in the process list. A client that sends headers avoids this. See
[transport](./transport.md) for the bridge setup.

## Related

- [Getting started](./getting-started.md)
- [Setup guides](./setup/README.md)
- [Transport](./transport.md)
- [Limits and errors](./limits-and-errors.md)
- [`api-key-usage` tool](./tools/api-key-usage.md)
- [Your sec-api profile and keys](https://sec-api.io/profile)
