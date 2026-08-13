# Connect Msty to SEC EDGAR Data with MCP

Msty Studio is a desktop app for local and cloud models. Its **Toolbox** adds
MCP tools, and it supports remote servers over Streamable HTTP.

## Prerequisites

- Msty Studio with Streamable HTTP support in the Toolbox. Msty announced the
  feature in December 2025 and names no minimum version.
- A model that supports tool use.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

Local dependencies such as Node and Python are not needed. Those apply to STDIO
tools only.

## Config location

Use the UI. Open **Toolbox**, then **Tools**, then **Add New Tool**.

## Steps

1. Open **Toolbox**, then **Tools**.
2. Click **Add New Tool**.
3. Set **Tool Name** to `SEC EDGAR`.
4. Set **Tool ID** to `sec-api`.
5. Choose the **HTTP** connection type. That is the option for remote MCP
   servers over streamable HTTP. Do not choose **STDIO / JSON**.
6. Set **MCP Server URL** to this value:

   ```text
   https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
   ```

7. Leave **Authentication** empty. The key rides in the URL. To keep it out of
   the URL, use `https://api.sec-api.io/mcp` and add a header instead:

   ```text
   Authorization: Bearer YOUR_API_KEY
   ```

8. Click Save.

## Reload

Saving connects the tool. There is no app restart.

## Verify

Expand **Tool Console** on the tool page and click **List Features**. The
console prints every tool that the server exposes. Expect **49**.

An error here means the URL or the key is wrong. A wrong key gives HTTP 401.

Test one call before you move on. Open the **Tool Call** tab, select
`filing-search`, set the query to `ticker:AAPL AND formType:"10-K"`, and click
**Execute**. The console shows the returned JSON.

## Use it in a chat

Msty gates tools behind toolsets.

1. Open the **Toolset** tab.
2. Click **Create New Toolset**.
3. Enable the `SEC EDGAR` tool.
4. Name the toolset `SEC` and click **Update**.
5. In a chat, click the **Toolset** icon and pick `SEC`.

## First prompt

```text
List the three most recent 10-K filings from Apple with their filing dates and
accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`.

## Known quirks

- **The HTTP option is the right one.** Older Msty guides bridge remote servers
  with `npx mcp-remote` under a STDIO config. Native HTTP replaces that. Use the
  bridge only if the HTTP option fails in your build.
- **A tool does nothing until it is in a toolset.** This step surprises people
  who come from Claude Desktop.
- **The key sits in the URL.** Msty stores it in its local database. Rotate the
  key at [sec-api.io](https://sec-api.io/profile) if it leaks.
- **Streamable HTTP only.** The server is stateless and issues no session ID.

## Removal

Open **Toolbox**, then **Tools**. Delete the `SEC EDGAR` tool. Also remove it
from the `SEC` toolset, or delete the toolset.

## Source

[Msty Studio Docs: Tools](https://docs.msty.ai/studio/toolbox/tools)
