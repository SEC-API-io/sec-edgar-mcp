# Connect Flowise to SEC EDGAR Data with MCP

Flowise is a drag-and-drop builder for LLM agents. Its **Custom MCP** node
connects an Agent to any MCP server, over stdio or streamable HTTP.

## Prerequisites

- A recent Flowise release. Flowise published security advisories for the MCP
  node. Keep the install current. See the
  [Flowise security advisories](https://github.com/FlowiseAI/Flowise/security/advisories).
- An Agentflow with an **Agent** node.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

Flowise stores the server config in the flow, in the **MCP Server Config** field
of the Custom MCP node. There is no file to edit.

Keep the key out of the flow. Put it in a Flowise variable.

## Steps

1. In the left sidebar, open **Variables**. Add a variable named `secApiKey`.
   Set its value to `YOUR_API_KEY`.
2. Open your Agentflow and add an **Agent** node.
3. On the Agent node, open **Tools** and add **Custom MCP**.
4. Put this in **MCP Server Config**:

```json
{
  "url": "https://api.sec-api.io/mcp",
  "headers": {
    "Authorization": "Bearer {{$vars.secApiKey}}"
  }
}
```

5. Click the refresh control next to **Available Actions**. Flowise reads the
   server and fills the list.
6. Select the actions you want, or keep them all.
7. Save the flow.

## Reload

Save the chatflow. Refresh **Available Actions** after every change to **MCP
Server Config**. Flowise caches the list.

## Verify

Open the **Available Actions** dropdown. It holds **49** entries.

## First prompt

> What did Berkshire Hathaway hold at the end of the last reported quarter?

Expect one `form-13f-holdings` call. The answer lists issuers, CUSIPs, share
counts and market values.

## Quirks

- **Use the URL form, not the command form.** The stdio shape,
  `{"command": "npx", "args": [...]}`, starts a local process. This server is
  remote. Give it a `url` and `headers`.
- **Variables keep the key out of the flow.** `{{$vars.secApiKey}}` resolves at
  run time. A flow you export with a literal key carries that key with it.
- **The tool list is cached.** New tools do not appear until you refresh
  **Available Actions**.
- **The server is stateless.** It issues no `Mcp-Session-Id`. Flowise opens a
  streamable HTTP request per call, which is exactly what this server wants.
- **Results arrive as one text block.** Each tool returns stringified JSON. No
  tool declares an output schema, so there is no `structuredContent`.
- **Docker deployments need no extra networking.** The server is public. The
  `host.docker.internal` workaround only applies to local MCP servers.

## Removal

Open the Agent node. Remove the **Custom MCP** tool. Save the flow. Then open
**Variables** and delete `secApiKey`.

## Source

[Tools & MCP | FlowiseAI Docs](https://docs.flowiseai.com/tutorials/tools-and-mcp)
