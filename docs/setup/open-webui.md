# Connect Open WebUI to SEC EDGAR Data with MCP

Open WebUI is a self-hosted, multi-user chat interface. Since v0.6.31 it speaks
MCP natively, over Streamable HTTP only.

## Prerequisites

- Open WebUI **v0.6.31** or later. Earlier versions need the `mcpo` proxy.
- Admin rights. Only admins can register an MCP server. Users cannot add their
  own.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

Open WebUI has no config file for this. Use the admin UI.

Open **Admin Settings**, then **Integrations**.

## Steps

1. Open **Admin Settings**, then **Integrations**.
2. Click **+ (Add Server)**.
3. Set **Type** to **MCP (Streamable HTTP)**. Do not pick an OpenAPI type.
4. Set **Server URL** to this value:

   ```text
   https://api.sec-api.io/mcp
   ```

5. Set **Auth** to **Bearer** and paste your key in the **Key** field. Open
   WebUI sends `Authorization: Bearer YOUR_API_KEY`, which the server accepts.
6. Optional. Use **Function Name Filter List** to expose a subset of the 49
   tools. See Known quirks.
7. Click **Save**.

The simpler form also works. Set **Auth** to **None** and use
`https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY` as the server URL.

## Reload

Restart Open WebUI if the UI prompts you. Otherwise the server is live at once.

## Verify

Click **Verify Connection** on the server entry in **Integrations**. A success
message confirms the handshake.

To see the tools, open a chat and click **+**, then **Integrations**, then
**Tools**. The `sec-api` entry lists its tools. Expect **49**. Enable it there.

A wrong key gives HTTP 401 and the connection check fails.

## First prompt

```text
List the three most recent 10-K filings from Apple with their filing dates and
accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`.

## Known quirks

- **Streamable HTTP only.** Open WebUI supports no stdio and no SSE. This server
  speaks Streamable HTTP, so no bridge is needed. `mcpo` is only for stdio
  servers.
- **Do not use the OpenAPI connection type.** Pasting an `mcpServers` JSON block
  into an OpenAPI connection is the most common setup error. Pick **MCP
  (Streamable HTTP)**.
- **49 tools is a large tool list.** Small models pick badly with that many
  choices. Use **Function Name Filter List** to keep the ones that you need, for
  example `filing-search`, `full-text-search`, `extractor`, `xbrl-to-json`,
  `insider-trading`.
- **Set `WEBUI_SECRET_KEY` and keep it stable.** A changing key invalidates
  stored credentials on restart.
- **Multi-user.** Every user shares your key.

## Older versions: the mcpo bridge

On a build before v0.6.31, run the [`mcpo`](https://github.com/open-webui/mcpo)
proxy and register its OpenAPI endpoint instead:

```bash
uvx mcpo --port 8000 --api-key "top-secret" -- \
  npx mcp-remote https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

Register the full subpath, not the root URL. The root does not aggregate tools,
and tool calls fail silently against it. Read the generated docs at
`http://localhost:8000/docs` to find the subpath.

## Removal

Open **Admin Settings**, then **Integrations**. Delete the `sec-api` entry.

## Source

[Open WebUI Docs: Model Context Protocol (MCP)](https://docs.openwebui.com/features/extensibility/mcp/)
