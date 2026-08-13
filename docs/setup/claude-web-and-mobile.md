# Connect Claude web and mobile to SEC EDGAR Data with MCP

Claude on the web is [claude.ai](https://claude.ai) in a browser. Claude mobile
is the iOS and Android app. Both run remote MCP servers as custom connectors.
You add a connector on the web. Mobile then uses it.

## Prerequisites

- A Claude account. Free allows one custom connector. Pro, Max, Team and
  Enterprise allow more.
- A sec-api key from [sec-api.io](https://sec-api.io/profile).
- On Team and Enterprise, an Owner must add the connector first. Members then
  connect to it.

## Configure: Free, Pro and Max

1. Open [claude.ai](https://claude.ai) in a browser.
2. In the sidebar, click **Customize**, then **Connectors**.
3. Click **Add custom connector**.
4. In **Name**, enter `sec-api`.
5. In the URL field, enter
   `https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`.
6. Leave **Advanced settings** empty. This server does not use OAuth, so it
   needs no Client ID and no Client Secret.
7. Click **Add**.

## Configure: Team and Enterprise

An Owner does this once:

1. Open **Organization settings**, then **Connectors**.
2. Click **Add**, then **Custom**. If Claude asks for a type, choose **Web**.
3. Enter the URL from step 5 above.
4. Click **Add**.

Every member then opens **Customize > Connectors**, finds the entry with the
**Custom** label, and clicks **Connect**.

## Restart

There is nothing to restart. The web app picks the connector up at once. On
mobile, close the app and open it again if the connector does not appear.

## Verify

Open **Customize**, then **Connectors**, and click `sec-api`. The **Tool
permissions** list names every tool. Expect 49.

In a chat, click **+**, then **Connectors**, and check that `sec-api` is on.

## First prompt

> Use sec-api to show Berkshire Hathaway's five largest 13F holdings from the
> most recent quarter.

Claude calls `form-13f-holdings`. The answer is a short table. Each row gives
the issuer, the ticker, the share count, and the market value.

## Quirks

- Anthropic connects to the server from its cloud, not from your phone or
  browser. The server is public, so this works. Your API key is stored by
  Anthropic. Rotate it when you remove the connector.
- You cannot add a custom connector from the mobile app. Add it on the web, then
  open the mobile app. The connector follows your account.
- The **Request headers** field in the Add custom connector dialog accepts a
  fixed credential instead of OAuth. It is in beta and rolls out slowly. Enter
  the full value, including the scheme, for example `Bearer YOUR_API_KEY`. The
  URL query key works today without the beta.
- The server exposes 49 tools and no resources or prompts. Claude picks a tool
  from its name and description. Block the tools you never need under
  **Customize > Connectors**, so the choice is easier.
- Chat answers stream one text block per tool call. Long results, such as a full
  XBRL statement set, may be cut short in the chat window.

## Remove

1. Open **Customize**, then **Connectors**.
2. Find `sec-api`.
3. Click **Remove**, or open the three-dot menu and choose **Remove**.
4. Confirm.

On Team and Enterprise, an Owner removes it from **Organization settings >
Connectors**. Rotate the API key afterwards.

## Source

[Third party connectors with remote MCP](https://claude.com/docs/connectors/custom/remote-mcp)
and [Use connectors to extend Claude's capabilities](https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities)
