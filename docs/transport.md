# Transport

The SEC EDGAR MCP server is remote and speaks **Streamable HTTP**. You POST one
JSON-RPC message to one URL and you get one JSON reply. There is no stdio server
to install, no SSE stream to hold open, and no session to manage.

MCP server URL:

> `https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`

| Property           | Value                                                          |
| ------------------ | -------------------------------------------------------------- |
| Transport          | Streamable HTTP                                                |
| HTTP method        | `POST` only. `GET` and `DELETE` return 404.                    |
| Response body      | `application/json`. Never an SSE stream.                       |
| Session            | **None.** The server never issues `Mcp-Session-Id`.            |
| Server identity    | name `sec-api`, version `1.0.0`                                |
| Capabilities       | `tools` only. No resources, prompts or sampling.               |
| Protocol version   | The server echoes the version the client sends.                |
| Auth               | `?apiKey=` or `Authorization: Bearer <key>`        |

Everything on this page was verified against the live server on 2026-08-13.

## The stateless per-request model

The server builds its transport with an undefined session id generator. The
transport therefore creates no session, sends no `Mcp-Session-Id` header, and
expects none back. Each POST carries all the state that call needs.

What follows from that:

- **No handshake is required.** You can POST `tools/list` or `tools/call` as your
  first request. `initialize` is optional. Most clients still send it, which is
  correct and harmless.
- **No session can expire.** There is no `Mcp-Session-Id` to lose, so there is no
  "session not found" error and no reconnect logic.
- **A session header you send is ignored.** A POST with
  `Mcp-Session-Id: junk-session` returns normal data.
- **Retries are safe at the transport layer.** Resend the same POST. Read the
  tool page first if the tool has side effects. None of the 49 tools write data.
- **Any instance can answer any call.** A load balancer may route two calls from
  one client to two different servers. Nothing breaks.
- **The server never speaks first.** There are no progress notifications, no
  server-initiated sampling, and no resource subscriptions.

This is why no session header is needed. A session id exists to bind later
requests to earlier server state. This server keeps no state between requests,
so the id would carry no meaning.

### What the transport does and does not answer

| Request                                  | Result                                            |
| ---------------------------------------- | -------------------------------------------------- |
| `POST` with a JSON-RPC request           | HTTP 200, one JSON reply                          |
| `POST` with a JSON-RPC notification      | HTTP 202, empty body                              |
| `POST` that asks for an SSE reply stream | Still `application/json`                          |
| `GET` for a server-to-client SSE stream  | HTTP 404, `Cannot GET /mcp`                       |
| `DELETE` to end a session                | HTTP 404, `Cannot DELETE /mcp`                    |
| `resources/list`, `prompts/list`         | JSON-RPC error `-32601 Method not found`          |

A client that insists on opening the `GET` SSE stream fails on this server. Use
the [stdio bridge](#the-mcp-remote-stdio-bridge) for those clients.

## Request headers

Two headers are mandatory. A hand-written client that omits either one fails
before any tool runs.

| Header         | Required value                                    |
| -------------- | -------------------------------------------------- |
| `Content-Type` | `application/json`                                |
| `Accept`       | Must list **both** `application/json` and `text/event-stream` |

The `Accept` rule surprises people. The server answers with JSON, but the
Streamable HTTP specification still requires the client to accept both types.
Send only `application/json` and you get HTTP 406:

```json
{"jsonrpc":"2.0","error":{"code":-32000,"message":"Not Acceptable: Client must accept both application/json and text/event-stream"},"id":null}
```

### Authentication at the transport layer

Pass the key in the URL as `?apiKey=`, or send an
`Authorization: Bearer <key>` header.

A missing or malformed key is rejected by the HTTP layer, before JSON-RPC runs.
You get HTTP 401 and a plain body that is not a JSON-RPC message:

```json
{"error":"invalid or missing apiKey"}
```

Full key handling, header form and rotation: [authentication](./authentication.md).

### Response compression

The server gzips large responses even when the request asks for `identity`
encoding. `tools/list` returns about 27 KB of JSON, compressed to about 6 KB.
Pass `--compressed` to curl, or your terminal fills with binary. Every HTTP
client library handles this for you.

## Verify the server with curl

You need no MCP client to test the server. These commands are complete and
copy-paste runnable. Replace `YOUR_API_KEY` with your key.

### tools/list

```bash
curl -s --compressed 'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

The reply holds 49 tool definitions. Trimmed to one tool:

```json
{
  "result": {
    "tools": [
      {
        "name": "mapping",
        "title": "Company identifier mapping (CIK/ticker/CUSIP/name)",
        "description": "Look up cross-mappings between company identifiers. ...",
        "inputSchema": {
          "type": "object",
          "properties": {
            "param": {
              "type": "string",
              "enum": ["cik", "ticker", "cusip", "name", "exchange", "sector", "industry"]
            },
            "value": { "type": "string" }
          },
          "required": ["param", "value"]
        }
      }
    ]
  },
  "jsonrpc": "2.0",
  "id": 1
}
```

Count the tools and print their names with `jq`:

```bash
curl -s --compressed 'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' \
  | jq -r '.result.tools[].name'
```

No tool declares an `outputSchema`. Results are therefore text, never
`structuredContent`. See [response format](./response-format.md).

### tools/call

`mapping` is the cheapest tool to test with. It returns under 600 bytes.

```bash
curl -s --compressed 'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"mapping","arguments":{"param":"ticker","value":"AAPL"}}}'
```

The verbatim reply:

```json
{"result":{"content":[{"type":"text","text":"[{\"name\":\"APPLE INC\",\"ticker\":\"AAPL\",\"cik\":\"320193\",\"cusip\":\"037833100\",\"exchange\":\"NASDAQ\",\"isDelisted\":false,\"category\":\"Domestic Common Stock\",\"sector\":\"Technology\",\"industry\":\"Consumer Electronics\",\"sic\":\"3571\",\"sicSector\":\"Manufacturing\",\"sicIndustry\":\"Electronic Computers\",\"famaIndustry\":\"Computers\",\"currency\":\"USD\",\"location\":\"California; U.S.A\",\"id\":\"442d0fd86e9d78fe79d539ee1b7fc0e4\"}]"}],"isError":false},"jsonrpc":"2.0","id":2}
```

Read it as one text content block that holds stringified JSON. Unwrap it in two
steps:

```bash
curl -s --compressed 'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"mapping","arguments":{"param":"ticker","value":"AAPL"}}}' \
  | jq -r '.result.content[0].text' | jq .
```

### A tool error

A failing tool still returns HTTP 200 and a valid JSON-RPC result. The failure
sits inside the result, as `isError: true` plus one text block:

```json
{"result":{"content":[{"type":"text","text":"sec-api error: Invalid parameter. Valid parameters are: cik, cusip, ticker, name, exchange, sic, sector, industry."}],"isError":true},"jsonrpc":"2.0","id":3}
```

Never test for HTTP status alone. Always read `isError`. The full error
catalogue is in [limits and errors](./limits-and-errors.md).

### initialize, and proof that there is no session

```bash
curl -s --compressed -D - -o /dev/null 'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"1.0"}}}'
```

`-D -` prints the response headers. There is no `Mcp-Session-Id` among them.
The body confirms the capability set:

```json
{"result":{"protocolVersion":"2025-06-18","capabilities":{"tools":{}},"serverInfo":{"name":"sec-api","version":"1.0.0"}},"jsonrpc":"2.0","id":0}
```

`capabilities` holds `tools` and nothing else.

### Batching

An array of JSON-RPC requests works, and returns an array of replies. An
`initialize` request inside the array is rejected. MCP removed batching in
protocol version 2025-06-18, so do not build on it. Send one request per POST.

## The mcp-remote stdio bridge

Some MCP clients only spawn a local command and talk over stdio. Some others
support remote servers but demand an SSE session, which this server does not
offer. Both cases need a bridge. `mcp-remote` runs as a local stdio server and
forwards every message to the HTTP endpoint.

Try the native HTTP configuration first. Use the bridge only when that fails.

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "--transport",
        "http-only"
      ]
    }
  }
}
```

Set `--transport http-only`. The default strategy is `http-first`, which falls
back to SSE when it meets a 404. A `GET` on this endpoint returns 404, so the
default can send a healthy client down the SSE path for no reason.
`http-only` removes that risk.

Verified on 2026-08-13 with `mcp-remote` 0.1.38 and Node 20. The bridge reports
`Connected to remote server using StreamableHTTPClientTransport`, then lists all
49 tools.

### Keep the key out of the URL

Pass the key as a header instead. The `env` block keeps it out of the process
argument list:

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.sec-api.io/mcp",
        "--transport",
        "http-only",
        "--header",
        "Authorization:${AUTH_HEADER}"
      ],
      "env": {
        "AUTH_HEADER": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

Write `Authorization:${AUTH_HEADER}` with no space after the colon. Cursor and
Claude Desktop on Windows do not escape spaces inside `args`, which mangles the
header. A space inside the environment variable is safe.

### Bridge requirements and costs

- Node.js 18 or higher. The client uses your system Node, not a newer one you
  installed elsewhere.
- `-y` lets `npx` install the package without a prompt.
- The bridge adds a local process and one extra hop. Expect a little more
  latency and one more thing that can fail.
- Debug with `--debug`. Logs land in `~/.mcp-auth/`.
- Test the bridge outside your client:

```bash
npx -y -p mcp-remote@latest mcp-remote-client \
  'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' --transport http-only
```

## Corporate proxies and firewalls

The server needs one outbound HTTPS destination:

| Host             | Port | Protocol   |
| ---------------- | ---- | ---------- |
| `api.sec-api.io` | 443  | HTTPS 1.1  |

Ask your network team to allow that host. No other host is involved, and the
bridge adds none.

Why this is easy to allow, and what to watch:

- **No long-lived connections.** Each tool call is one short request and one
  response. Proxies that close idle streams or buffer SSE do not affect this
  server, because it sends no SSE.
- **TLS interception breaks certificate checks.** If your proxy re-signs TLS,
  point Node at your corporate root certificate:
  `NODE_EXTRA_CA_CERTS=/path/to/corp-root.pem`. For curl, use
  `--cacert /path/to/corp-root.pem`. Do not disable verification.
- **Explicit proxy for the bridge.** `mcp-remote` ignores `HTTPS_PROXY` unless
  you add `--enable-proxy`. With that flag it reads `HTTP_PROXY`, `HTTPS_PROXY`
  and `NO_PROXY` from the environment.
- **Do not strip the query string.** Some gateways log or rewrite URLs. If yours
  drops query parameters, the `?apiKey=` form fails. Move the key into the
  `Authorization` header.
- **Browser clients.** The endpoint sends `Access-Control-Allow-Origin: *` and
  answers CORS preflight requests, so a browser-based client can call it
  directly. Never ship your key to a browser. Proxy the call through your own
  backend.

Bridge configuration with a proxy:

```json
{
  "mcpServers": {
    "sec-api": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY",
        "--transport",
        "http-only",
        "--enable-proxy"
      ],
      "env": {
        "HTTPS_PROXY": "http://proxy.corp.example:3128",
        "NO_PROXY": "localhost,127.0.0.1",
        "NODE_EXTRA_CA_CERTS": "/path/to/corp-root.pem"
      }
    }
  }
}
```

## Concurrency

The transport is stateless, so parallel calls are legal.

## Troubleshooting

| Symptom                                          | Cause                                        | Fix                                                    |
| ------------------------------------------------ | -------------------------------------------- | ------------------------------------------------------ |
| HTTP 406, `Client must accept both ...`          | `Accept` header misses one type              | Send `application/json, text/event-stream`             |
| HTTP 401, `invalid or missing apiKey`            | No key                                       | Add `?apiKey=` or the Bearer header                    |
| HTTP 200 with `sec-api error: API token invalid` | Key is well formed but unknown               | Check the key at [sec-api.io](https://sec-api.io/profile) |
| HTTP 404, `Cannot GET /mcp`                      | Client opened the SSE stream                 | POST instead, or bridge with `--transport http-only`   |
| Binary noise in the terminal                     | gzip response                                | Add `--compressed` to curl                             |
| Client asks for a session id                     | Client requires an SSE session               | Use the `mcp-remote` bridge                            |
| HTTP 503, `mcp server not initialized`           | Server is still starting                     | Retry after a few seconds                              |

## Related

- [Authentication](./authentication.md) for key placement and rotation.
- [Response format](./response-format.md) for the text block and the envelopes.
- [Limits and errors](./limits-and-errors.md) for paging and error text.
- [Setup guides](./setup/README.md) for per-client configuration.
- [Tool reference](./tools/README.md) for all 49 tools.
