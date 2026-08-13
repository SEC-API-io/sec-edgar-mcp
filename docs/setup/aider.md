# Connect Aider to SEC EDGAR Data with MCP

Aider is a terminal pair programmer that edits files in a git repo.

Aider has no MCP client, so it cannot connect to this server. Call the sec-api
REST API from inside an Aider session instead.

## Prerequisites

- Aider and `curl` on your `PATH`.
- A sec-api API key. See [authentication](../authentication.md).

## Workaround: call the REST API with `/run`

This server wraps the sec-api REST API. Every tool maps to a REST route, and the
same key works on both. Aider runs a shell command with `/run` and offers the
output to the chat.

Export the key before you start Aider:

```bash
export SEC_API_KEY=YOUR_API_KEY
```

A `GET` route needs one line. This is the `mapping` tool:

```text
/run curl -s "https://api.sec-api.io/mapping/ticker/AAPL?token=$SEC_API_KEY"
```

A `POST` route needs a body. Write the JSON to a file first, then pass the file.
This is the `filing-search` tool:

```text
/run curl -s -X POST "https://api.sec-api.io?token=$SEC_API_KEY" -H "Content-Type: application/json" -d @query.json
```

Aider asks whether to add the output to the chat. Answer yes.

## Verify

Run the same `curl` command in a plain terminal first. A valid key returns JSON.
A bad key returns HTTP 401. Inside Aider, `/run` prints the same JSON.

## First prompt

> Summarise the Apple 10-K filings above. Give the filing date and the primary
> document URL for each.

The model reads the JSON from the chat context. It cannot choose a call, and it
cannot follow up with a second call on its own. You issue every request by hand.

## Quirks

- Aider sends the whole response to the model. Narrow each request with `size`
  or a tighter query before you add the output.
- The command text stays in the chat history. Use `$SEC_API_KEY` rather than the
  literal key.
- Third-party projects named `aider-mcp-server` point the other way. They wrap
  Aider so that an MCP client can drive it. They do not let Aider call this
  server.
- AiderDesk is a separate GUI project with its own agent mode. It is not the
  Aider CLI.

## Removal

Nothing to remove. No config file changed. Drop the key from the environment
with `unset SEC_API_KEY`.

Source: [Aider in-chat commands](https://aider.chat/docs/usage/commands.html)
