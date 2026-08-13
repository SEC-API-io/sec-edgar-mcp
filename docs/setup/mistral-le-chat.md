# Connect Mistral Le Chat to SEC EDGAR Data with MCP

Le Chat is Mistral's hosted chat app. It reaches remote MCP servers through
**custom MCP connectors**. It does not run local stdio servers.

## Prerequisites

- Admin rights in your Le Chat workspace. Only admins can add a connector. On
  Free, Pro and Student plans the account owner is the admin by default.
- The server must answer over HTTPS with a valid TLS certificate and speak
  Streamable HTTP. This server does both.
- A SEC-API key. Get one at
  [sec-api.io](https://sec-api.io/profile).

## Config location

Le Chat has no config file. Use the web UI.

Open the side panel. Expand the **Intelligence** menu and click **Connectors**.

## Steps

1. Click **+ Add Connector** on the right side of the page.
2. Switch to the **Custom MCP Connector** tab.
3. Set **Connector name** to `secapi`. The field takes a unique identifier with
   no spaces and no special characters.
4. Set **Server URL** to this value:

   ```text
   https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
   ```

5. Set **Description** to `SEC EDGAR filings and financial data`. This field is
   optional.
6. Click **Connect**.

Le Chat probes the server and detects its authentication method. This server
needs none, because the key rides in the URL. If Le Chat asks for credentials,
choose the HTTP Bearer Token option, use `https://api.sec-api.io/mcp` as the
URL, and paste the key as the token.

## Reload

There is no restart. The connector appears in your organization list as soon as
the probe succeeds. Reload the browser tab if it does not show up.

## Verify

Open **Connectors**, then the **My Connectors** tab. Click the `secapi` card and
open the **Functions** tab. The tab lists every tool. Expect **49**.

The **Always allow** toggle on that tab pre-approves a function, so Le Chat
stops asking before each call.

## First prompt

Start a chat. Enable `secapi` in the tools dropdown. Then ask:

```text
Use the secapi connector. List the three most recent 10-K filings from Apple
with their filing dates and accession numbers.
```

Expect three rows. Each row holds a form type of `10-K`, a filing date, and an
accession number in the shape `0000320193-25-000079`. Le Chat names the
`filing-search` tool in the trace above the answer.

## Known quirks

- **The name field rejects hyphens and spaces.** Use `secapi`, not `sec-api`.
  The prompts on this page use the name that you choose.
- **Mistral does not review custom connectors.** The dialog warns you. You own
  the trust decision for any server that you add.
- **The key sits in the URL.** Every admin in the workspace can read it in the
  connector settings. Rotate the key at
  [sec-api.io](https://sec-api.io/profile) if it leaks.
- **Connectors are workspace-wide.** Adding one shares your API key with the
  whole workspace.

## Removal

Open **Connectors**, then the **My Connectors** tab. Open the `secapi` card and
remove it from the card menu. Mistral does not document the removal step, so the
exact label may differ in your build.

## Source

[Mistral Docs: MCP Connectors](https://docs.mistral.ai/le-chat/knowledge-integrations/connectors/mcp-connectors)
and
[Mistral Help Center: Configuring a Custom Connector](https://help.mistral.ai/en/articles/393572-configuring-a-custom-connector)
