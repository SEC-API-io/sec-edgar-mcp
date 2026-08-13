# Connect n8n to SEC EDGAR Data with MCP

n8n is a workflow automation tool. Its AI Agent node calls remote MCP servers
through the **MCP Client Tool** sub-node, over HTTP Streamable.

## Prerequisites

- An n8n instance with the **MCP Client Tool** node at type version 1.2 or
  later. Version 1.2 added the **Server Transport** field. The current default
  is 1.4.
- An **AI Agent** node in the workflow.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

n8n has no config file for this. You add a node on the canvas.

Open your workflow. Under the **AI Agent** node, click the **+** on the **Tool**
connector. Search for `MCP Client Tool`.

## Steps

1. Add an **AI Agent** node.
2. Add an **MCP Client Tool** sub-node on its **Tool** connector.
3. Set **Endpoint** to `https://api.sec-api.io/mcp`.
4. Set **Server Transport** to **HTTP Streamable**.
5. Set **Authentication** to **Bearer Auth**.
6. Create a new **Bearer Auth** credential. Put `YOUR_API_KEY` in the token
   field. n8n sends `Authorization: Bearer YOUR_API_KEY`.
7. Leave **Tools to Include** at **All**.
8. Open **Options**, add **Timeout**, and set it to `60000` or more.

Without a credential, set **Authentication** to **None** and put the key in the
URL instead:

```text
https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

## Reload

Save the workflow. n8n reads the tool list when you open the node. Click out of
the **Endpoint** field and back into it to force a refresh.

## Verify

Set **Tools to Include** to **Selected**. The **Tools to Include** list loads
from the server. It holds **49** entries.

## First prompt

> List the three most recent 10-K filings from Apple.

Expect one `filing-search` call. The answer names three filings with their
accession numbers, filing dates and document URLs.

## Quirks

- **Node version matters.** Type version 1 shows only an **SSE Endpoint** field
  and cannot speak HTTP Streamable. Delete the node and add a fresh one.
- **The vendor page is stale.** It still documents the old **SSE Endpoint**
  field. The field names above come from the node source,
  `packages/@n8n/nodes-langchain/nodes/mcp/McpClientTool/McpClientTool.node.ts`.
- **A known bug ignores the dropdown.** Some builds still open an SSE stream
  after you pick HTTP Streamable. See
  [n8n issue 24967](https://github.com/n8n-io/n8n/issues/24967). Work around it
  by switching **Server Transport** to expression mode and setting the literal
  value `httpStreamable`.
- **Never pick SSE.** This server is stateless. It issues no `Mcp-Session-Id`
  and serves no SSE session.
- **Results arrive as one text block.** Each tool returns stringified JSON. No
  tool declares an output schema, so there is no `structuredContent`. Add a Code
  node when a later node needs single fields.
- **Keys in URLs leak.** On a shared instance the endpoint URL shows up in
  execution data. Use the Bearer Auth credential instead.

## Removal

Select the **MCP Client Tool** node and press Delete. Save the workflow. Then
open **Credentials** and delete the Bearer Auth credential you created.

## Source

[MCP Client Tool | n8n Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)
