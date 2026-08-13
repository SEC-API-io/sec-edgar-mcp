# Connect CrewAI to SEC EDGAR Data with MCP

CrewAI runs teams of role-playing agents. `MCPServerAdapter` from `crewai-tools`
turns MCP tools into CrewAI tools over the `streamable-http` transport. Read the
quirks first. CrewAI has a history of trouble with stateless servers.

## Prerequisites

- Python 3.10 or later. `crewai-tools` 1.15.15 requires it.
- The `mcp` extra of `crewai-tools`.
- An SEC-API.io key from [sec-api.io](https://sec-api.io/profile).

```bash
pip install "crewai-tools[mcp]"
```

## Config location

There is no config file. You declare the server in the module that builds the
crew, for example `src/<project>/crew.py`. Read the key from the environment. Do
not commit it.

## Configuration

```python
import os
from crewai import Agent, Crew, Process
from crewai_tools import MCPServerAdapter

server_params = {
    "url": f"https://api.sec-api.io/mcp?apiKey={os.environ['SEC_API_KEY']}",
    "transport": "streamable-http",
}

with MCPServerAdapter(server_params, connect_timeout=60) as mcp_tools:
    analyst = Agent(
        role="SEC filings analyst",
        goal="Answer questions from SEC EDGAR filings.",
        backstory="You read EDGAR every day.",
        tools=mcp_tools,
        verbose=True,
    )
    crew = Crew(agents=[analyst], tasks=[...], process=Process.sequential)
    crew.kickoff()
```

The transport string is `"streamable-http"`, with a hyphen. The `with` block
owns the connection. Everything that uses the tools must sit inside it. To send
the key as a header instead, drop the query string and add
`"headers": {"Authorization": f"Bearer {key}"}` to `server_params`.

## Reload

There is no daemon. Run `crewai run`, or your own script, again.

## Verify

```python
with MCPServerAdapter(server_params, connect_timeout=60) as mcp_tools:
    print(len(mcp_tools))  # 49
```

Expect **49**. HTTP 401 means the key is wrong.

## First prompt

Give the crew this task description:

> Find the 3 most recent 10-K filings from Apple. Report the filing date and the
> accession number of each.

The agent calls `filing-search` once with `ticker:AAPL AND formType:"10-K"`. The
crew output lists three rows. Each row carries a `filedAt` date and an accession
number such as `0000320193-25-000073`.

## Quirks

- **Stateless servers have been a problem.** CrewAI issue #3230 asked for
  support of Streamable HTTP servers that answer POST only, with no SSE stream
  and no session. It was closed as not planned. This server has that exact
  shape, and `GET` and `DELETE` return 404. If the adapter hangs or reports a
  connection failure, bridge to stdio. Replace `server_params` with
  `{"command": "npx", "args": ["mcp-remote", "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"]}`.
- **Raise `connect_timeout`.** The 30 second default has produced spurious
  connection errors. 60 is a safer start.
- Results arrive as one text block of stringified JSON. Errors return the text
  `sec-api error: <message>`.

## Removal

Delete the `MCPServerAdapter` block and remove `tools=mcp_tools` from the agent.
Then run `pip uninstall crewai-tools` if nothing else needs it.

## Source

[Streamable HTTP Transport, CrewAI docs](https://docs.crewai.com/en/mcp/streamable-http)
