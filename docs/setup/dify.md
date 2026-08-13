# Connect Dify to SEC EDGAR Data with MCP

Dify is a low-code platform for LLM apps. Since v1.6.0 it calls MCP servers
natively, with no plugin. It supports HTTP transport only.

## Prerequisites

- Dify v1.6.0 or later, cloud or self-hosted. Earlier versions need the
  marketplace MCP plugin.
- An HTTPS server URL. The MCP page rejects `http://`.
- A sec-api key. Get one at [sec-api.io](https://sec-api.io/profile).

## Config location

Dify stores this in the workspace, not in a file.

Open your workspace. Go to **Tools**, then the **MCP** tab. Newer builds put the
same page under **Integrations > Tools > MCP**.

## Steps

1. Click **Add MCP Server (HTTP)**. Do not use **Add HTTP API**. That flow is
   for OpenAPI tools and never calls an MCP endpoint.
2. **Server URL**: `https://api.sec-api.io/mcp`
3. **Name**: `SEC API`
4. **Server Identifier**: `sec-api`. Lowercase letters, numbers, underscores and
   hyphens only, 24 characters maximum.
5. Open the advanced settings. Under **Custom Headers**, add:

```text
Authorization: Bearer YOUR_API_KEY
```

6. Set the request timeout to 60 seconds or more.
7. Save.

To skip the header, put the key in the URL instead:

```text
https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

## Reload

Dify connects when you save. If the tool list stays empty, open the server card
and choose **Edit Settings**, then save again.

## Verify

Open **Tools > MCP** and select the `sec-api` card. It lists the imported tools.
Expect **49**.

## First prompt

In an Agent app, add the SEC API tools, then ask:

> Which 8-K filings from last month mention layoffs?

Expect one `full-text-search` call. The answer lists filings with company names,
form types, filing dates and links.

## Quirks

- **The Server Identifier is permanent.** Change it later and every app that
  used its tools breaks. You must re-add the tools in each app.
- **Exported apps carry the identifier.** Recreate the same server, with the
  same identifier, in every workspace you import into.
- **HTTPS only.** Self-hosted Dify cannot use a plain `http://` MCP URL.
- **A bad key returns HTTP 401.** Dify enables Dynamic Client Registration by
  default and may read a 401 as an OAuth challenge. This server uses no OAuth.
  If Dify offers to authorize, your key is wrong. Fix the key.
- **Results arrive as one text block.** Every tool returns stringified JSON. No
  tool declares an output schema, so there is no `structuredContent`. In a
  Workflow, follow the Tool node with a Code node to parse it.
- **The key sits in the workspace.** Prefer the header form, so the key stays
  out of the URL that Dify shows on the server card.

## Removal

Open **Tools > MCP**. Select the `sec-api` card and choose **Remove**. Dify
disconnects the server. Apps that used its tools lose them, so edit those apps
next.

## Source

[Use MCP Tools | Dify Docs](https://docs.dify.ai/en/cloud/use-dify/build/mcp)
