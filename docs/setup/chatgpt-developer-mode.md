# Connect ChatGPT to SEC EDGAR Data with MCP

ChatGPT is OpenAI's chat product. Developer mode lets you add your own remote MCP
server to your account, and ChatGPT then calls its tools inside a chat.

## Prerequisites

- A ChatGPT account on the Pro, Plus, Business, Enterprise or Education plan.
  OpenAI limits developer mode to those plans, on the web.
- A browser. You add the server on the web. You can use it on mobile afterwards.
- A sec-api.io API key. Get one at [sec-api.io](https://sec-api.io/profile).
- On a Business or Enterprise workspace, an admin must allow custom MCP servers
  first.

OpenAI renamed connectors to apps in December 2025, and the current developer
documentation calls them plugins. Your account may show any of the three words.

## Config location

There is no config file. You add the server through the ChatGPT web interface.

## Steps

1. Open [chatgpt.com](https://chatgpt.com).
2. Open **Settings → Security and login**. Turn on **Developer mode**.
3. Go to [chatgpt.com/plugins](https://chatgpt.com/plugins). Select the plus
   button.
4. Enter this MCP server URL:

   ```text
   https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
   ```

5. Give it a name, for example `SEC-API.io`. Write a description. The model reads
   the description when it decides whether to use the server.
6. Choose **No authentication**. The key travels in the URL. OpenAI also offers
   OAuth and mixed authentication. This server needs neither.
7. Complete the connection details and create the plugin.
8. Go to [your personal plugins](https://chatgpt.com/plugins?view=personal). Open
   the new entry. Select the plus button to install it.

## Reload

Start a new chat. Chats that you opened before the install do not see the tools.

## Verify

Open the plugin page. It lists every tool the server exposes. Count **49**.

## First prompt

In a new chat, type `@` and pick the app. Then ask:

> Use SEC EDGAR. What are the three most recent 10-K filings from Apple?

Expect three rows. Each row carries the form type, the filed date and a link to
the filing on sec.gov. ChatGPT reads them from the `filing-search` tool.

## Quirks

- **The key sits in the URL.** Anyone who opens the plugin settings can read it.
  Rotate the key after you share a screen.
- **Every call asks for confirmation.** ChatGPT treats a tool as a write action
  unless the tool declares `readOnlyHint`. No tool on this server declares it, so
  ChatGPT asks each time. You can tell it to remember your answer, but only for
  the current chat.
- **Remote only.** ChatGPT does not read local config files and does not start a
  local process. This server is remote, so it needs no bridge.
- **Not a company knowledge source.** That path needs tools named `search` and
  `fetch` with OpenAI's schema. This server has neither. Use it in normal chats.

## Removal

1. Go to [your personal plugins](https://chatgpt.com/plugins?view=personal).
2. Open the SEC EDGAR entry and remove it.
3. Turn off **Developer mode** under **Settings → Security and login** if you no
   longer need it.

OpenAI's guide does not describe removal, so the button label may differ.

## Source

[ChatGPT developer mode](https://developers.openai.com/api/docs/guides/developer-mode)
and the [plugins quickstart](https://developers.openai.com/plugins/quickstart),
read 2026-08-13.
