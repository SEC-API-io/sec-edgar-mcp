<div align="center">

# SEC EDGAR Data MCP

Give any AI agent access to SEC EDGAR data. No install, no local process, no API wrapper to maintain. Access 20 million filings over one hosted MCP server.

<!-- **Works with** -->

[![Claude](https://img.shields.io/badge/Claude-D97757?logo=claude&logoColor=fff)](./docs/setup/claude-desktop.md)
[![Claude Code](https://img.shields.io/badge/Claude_Code-D97757?logo=claude&logoColor=fff)](./docs/setup/claude-code.md)
[![ChatGPT](https://img.shields.io/badge/ChatGPT-000000)](./docs/setup/chatgpt-developer-mode.md)
[![Codex](https://img.shields.io/badge/Codex-000000)](./docs/setup/codex-cli.md)
[![Gemini](https://img.shields.io/badge/Gemini-4285F4?logo=googlegemini&logoColor=fff)](./docs/setup/google-gemini-cli.md)
[![Cursor](https://img.shields.io/badge/Cursor-000?logo=cursor&logoColor=fff)](./docs/setup/cursor.md)
[![VS Code](https://img.shields.io/badge/VS_Code-0098FF)](./docs/setup/vs-code-copilot.md)
[![Windsurf](https://img.shields.io/badge/Windsurf-09B6A2?logo=windsurf&logoColor=fff)](./docs/setup/windsurf.md)
[![Zed](https://img.shields.io/badge/Zed-084CCF?logo=zedindustries&logoColor=fff)](./docs/setup/zed.md)
[![JetBrains](https://img.shields.io/badge/JetBrains-000?logo=jetbrains&logoColor=fff)](./docs/setup/jetbrains-ai-assistant.md)
[![Perplexity](https://img.shields.io/badge/Perplexity-1FB8CD?logo=perplexity&logoColor=fff)](./docs/setup/perplexity.md)
[![Le Chat](https://img.shields.io/badge/Le_Chat-FA520F?logo=mistralai&logoColor=fff)](./docs/setup/mistral-le-chat.md)
[![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?logo=langchain&logoColor=fff)](./docs/setup/langchain-and-langgraph.md)
[![n8n](https://img.shields.io/badge/n8n-EA4B71?logo=n8n&logoColor=fff)](./docs/setup/n8n.md)

[**+ 41 more clients**](./docs/setup/README.md), from Cline and Goose to
LlamaIndex, CrewAI and Zapier.

[![Documentation](https://img.shields.io/badge/Documentation-sec--api.io-blue)](https://sec-api.io/docs/mcp-server)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Tools](https://img.shields.io/badge/Tools-49-orange)](./docs/tools/README.md)
[![Clients](https://img.shields.io/badge/Clients-55-blueviolet)](./docs/setup/README.md)

</div>

MCP server URL:

> `https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY`

Get an API key at [sec-api.io](https://sec-api.io/profile).

![./assets/intro.gif](./assets/intro.gif)

---

## Quick start

Add this to your client's MCP configuration:

```json
{
  "mcpServers": {
    "sec-api": {
      "type": "http",
      "url": "https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY"
    }
  }
}
```

Restart the client. It should report 49 tools.

Your client does not use JSON config? See the
[setup guides](./docs/setup/README.md) for per-client instructions.

## What you get

| Coverage                                   | Detail                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **20M+ EDGAR filings, 100M+ exhibits**     | From license agreements, investor presentations and other Reg FD disclosures, over insider trading, credit and bond agreements, bylaws, IPOs, secondaries and shelf offerings, M&A terms, government contracts, audit reports, SEC enforcement actions, AAERs and executive employment agreements, to board composition, subsidiaries, public float and cybersecurity incidents.                                |
| **1.1M+ entities, survivorship-bias free** | Every SEC-regulated filer that ever reported, including delisted companies, dissolved funds, terminated advisers and entities no longer reporting. Insiders, public and private companies, financial advisers, ETFs, mutual funds, hedge funds, money-market funds, institutional investors, foreign private issuers, BDCs, REITs, shell companies, brokers, dealers, asset-backed securities issuers and SROs. |
| **All 500+ EDGAR form types**              | Annual and quarterly reports (10-K, 10-Q, 20-F, 40-F), proxy statements (DEF 14A, PRE 14A) and proxy voting records, registration statements and prospectuses, and everything in between, including form types no longer in use.                                                                                                                                                                                |
| **Full historical range, 1993 to present** | Updated in real time as EDGAR ingests new filings.                                                                                                                                                                                                                                                                                                                                                              |

All of it reaches your agent through the [49 tools](./docs/tools/README.md)
below. Documents come back as raw source, PDF or a single extracted section.
Structured data comes back as JSON.

## Examples

Questions and AI-generated answers. Click to expand.

<details open>
<summary><b>Find the link to NVIDIA's latest 8-K.</b></summary>
<br>

NVIDIA's most recent 8-K was filed on **2 July 2026**, reporting under Item 5.02,
departure or appointment of directors and officers.

- Document: [nvda-20250628.htm](https://www.sec.gov/Archives/edgar/data/1045810/000104581026000060/nvda-20250628.htm)
- Filing index: [0001045810-26-000060-index.htm](https://www.sec.gov/Archives/edgar/data/1045810/000104581026000060/0001045810-26-000060-index.htm)
- Accession number: `0001045810-26-000060`

Want the full text of Item 5.02?

</details>

<details>
<summary><b>Summarize the differences between the risk factors from Lam Research's two most recent 10-K filings.</b></summary>
<br>

Comparing the [FY2026 10-K](https://www.sec.gov/Archives/edgar/data/707549/000070754926000037/lrcx-20260628.htm) and
the [FY2025 10-K](https://www.sec.gov/Archives/edgar/data/707549/000070754925000075/lrcx-20250629.htm), risk
disclosure grew by about 20 percent. Three changes stand out.

**New legal exposure.** Lam now warns that law enforcement may investigate or
charge the company or its employees over improper receipt or misuse of
designated technology or third-party confidential information under national and
economic-security measures, with civil or criminal liability.

**Cybersecurity risk doubled in length.** The 2026 filing names specific threats
for the first time: prompt injection, ransomware, and state-sponsored intrusion.

**A compliance admission.** The regulatory factor now concedes "there have been,
and may continue to be, instances of our compliance policies and procedures not
being effective."

...

</details>

<details>
<summary><b>Summarize the debt covenants in Boeing's most recent credit agreement.</b></summary>
<br>

Boeing's most recent facility is a **$3.0 billion 364-day credit agreement**
signed in August 2025, with Citibank and JPMorgan Chase as syndication agents.

| Covenant          | Term                                             |
| ----------------- | ------------------------------------------------ |
| Leverage ratio    | Maximum consolidated debt to total capital       |
| Minimum liquidity | Floor on unrestricted cash and equivalents       |
| Negative pledge   | Restricts liens on principal properties          |
| Cross-default     | Triggers on other indebtedness above a threshold |
| ERISA liability   | Capped at a stated amount                        |
| ...               | ...                                              |

</details>

<details>
<summary><b>What are the terms of Plug Power's latest shelf offering?</b></summary>
<br>

Plug Power filed a [**Form S-3ASR**](https://www.sec.gov/Archives/edgar/data/1093691/000110465925052737/tm2515625-1_s3asr.htm)
in May 2025 covering a **$1 billion at-the-market common stock offering**, later
updated by a [424B5](https://www.sec.gov/Archives/edgar/data/1093691/000110465925079028/tm2523585-1_424b5.htm).

- Initial sales agent: B. Riley Securities
- Later sales agent: Yorkville Securities
- Commission: 3.0 to 5.0 percent of gross proceeds
- ...

</details>

<details>
<summary><b>Summarize the latest earnings from the technology sector.</b></summary>
<br>

**212** Item 2.02 earnings filings from technology companies between 1 and 12
August 2026.

- **AMD**: Q2 revenue of $11.536 billion, up 50 percent year over year.
- **Arista Networks**: first $3 billion quarter.
- **Palantir**: the outlier, with 93 percent revenue growth.
- ...

</details>

<details>
<summary><b>Which companies disclosed a cybersecurity incident this quarter?</b></summary>
<br>

Seven Form 8-K Item 1.05 filings between July and August 2026. Three are new
disclosures.

| Company     | Incident                      | Filing                                                                                                          |
| ----------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Amgen       | Cloud data exfiltration       | [0000318154-26-000119](https://www.sec.gov/Archives/edgar/data/318154/000031815426000119/amgn-20260729.htm)     |
| Navient     | Ransomware attack             | [0001140361-26-027441](https://www.sec.gov/Archives/edgar/data/1593538/000114036126027441/ef20077249_8k.htm)    |
| AdaptHealth | Social engineering compromise | [0001104659-26-080297](https://www.sec.gov/Archives/edgar/data/1725255/000110465926080297/ahco-20260627x8k.htm) |
| ...         | ...                           | ...                                                                                                             |

</details>

More worked examples, with the tool calls behind them, in
[use cases](./docs/use-cases.md).

## Tools

Full reference with input schemas, response shapes and examples:
**[docs/tools/README.md](./docs/tools/README.md)**

<!-- tools:start -->
| Tool | Purpose | Category |
| ---- | ------- | -------- |
| [`filing-search`](./docs/tools/filing-search.md) | Search SEC EDGAR filings from 1993 to present to retrieve metadata | Search and discovery |
| [`full-text-search`](./docs/tools/full-text-search.md) | Full-text search over SEC EDGAR filings | Search and discovery |
| [`filing-to-pdf`](./docs/tools/filing-to-pdf.md) | Fetch any SEC EDGAR filing and exhibit as PDF | Filings and documents |
| [`get-edgar-file`](./docs/tools/get-edgar-file.md) | Download any SEC EDGAR filing or exhibit in its original format | Filings and documents |
| [`extractor`](./docs/tools/extractor.md) | Extract a section from a 10-K/10-Q/8-K filing | Filings and documents |
| [`xbrl-to-json`](./docs/tools/xbrl-to-json.md) | Convert XBRL financials to normalized JSON | Filings and documents |
| [`form-13f-holdings`](./docs/tools/form-13f-holdings.md) | Search Form 13F institutional holdings | Ownership and insiders |
| [`form-13f-cover-pages`](./docs/tools/form-13f-cover-pages.md) | Search Form 13F cover pages | Ownership and insiders |
| [`form-13d-13g`](./docs/tools/form-13d-13g.md) | Search Schedule 13D/13G beneficial-ownership filings | Ownership and insiders |
| [`insider-trading`](./docs/tools/insider-trading.md) | Search insider trading disclosures (Form 3/4/5) | Ownership and insiders |
| [`form-144`](./docs/tools/form-144.md) | Search Form 144 intent-to-sell filings | Ownership and insiders |
| [`form-nport`](./docs/tools/form-nport.md) | Search Form N-PORT fund portfolio holdings | Funds |
| [`form-npx`](./docs/tools/form-npx.md) | Search Form N-PX proxy voting records | Funds |
| [`form-npx-file`](./docs/tools/form-npx-file.md) | Get a Form N-PX file by accession number | Funds |
| [`form-ncen`](./docs/tools/form-ncen.md) | Search Form N-CEN investment company census | Funds |
| [`form-adv-firms`](./docs/tools/form-adv-firms.md) | Search Form ADV firm registrations | Investment advisers |
| [`form-adv-individuals`](./docs/tools/form-adv-individuals.md) | Search Form ADV individual registrations | Investment advisers |
| [`form-adv-brochures`](./docs/tools/form-adv-brochures.md) | Get Form ADV Part 2 brochures for an adviser | Investment advisers |
| [`form-adv-schedule-a-direct-owners`](./docs/tools/form-adv-schedule-a-direct-owners.md) | Get Form ADV Schedule A, direct owners of an adviser | Investment advisers |
| [`form-adv-schedule-b-indirect-owners`](./docs/tools/form-adv-schedule-b-indirect-owners.md) | Get Form ADV Schedule B, indirect owners of an adviser | Investment advisers |
| [`form-adv-schedule-d-1-b`](./docs/tools/form-adv-schedule-d-1-b.md) | Get Form ADV Schedule D-1-B, other business names | Investment advisers |
| [`form-adv-schedule-d-5-k`](./docs/tools/form-adv-schedule-d-5-k.md) | Get Form ADV Schedule D-5-K, separately managed accounts | Investment advisers |
| [`form-adv-schedule-d-7-a`](./docs/tools/form-adv-schedule-d-7-a.md) | Get Form ADV Schedule D-7-A, financial industry affiliations | Investment advisers |
| [`form-adv-schedule-d-7-b-1`](./docs/tools/form-adv-schedule-d-7-b-1.md) | Get Form ADV Schedule D-7-B-1, private funds advised | Investment advisers |
| [`form-s1-424b4`](./docs/tools/form-s1-424b4.md) | Search S-1 / 424B4 registration filings | Offerings and registrations |
| [`form-d`](./docs/tools/form-d.md) | Search Form D private-placement filings | Offerings and registrations |
| [`form-c`](./docs/tools/form-c.md) | Search Form C Regulation Crowdfunding offerings | Offerings and registrations |
| [`form-8k`](./docs/tools/form-8k.md) | Search structured 8-K material-event filings | Offerings and registrations |
| [`reg-a-search`](./docs/tools/reg-a-search.md) | Search Regulation A / A+ offerings | Offerings and registrations |
| [`reg-a-form-1a`](./docs/tools/reg-a-form-1a.md) | Search structured Reg A Form 1-A filings | Offerings and registrations |
| [`reg-a-form-1k`](./docs/tools/reg-a-form-1k.md) | Search structured Reg A Form 1-K filings | Offerings and registrations |
| [`reg-a-form-1z`](./docs/tools/reg-a-form-1z.md) | Search structured Reg A Form 1-Z filings | Offerings and registrations |
| [`compensation`](./docs/tools/compensation.md) | Search executive compensation data | Governance and compensation |
| [`compensation-by-key`](./docs/tools/compensation-by-key.md) | Get executive compensation by CIK or ticker | Governance and compensation |
| [`audit-fees`](./docs/tools/audit-fees.md) | Search auditor / audit-fee disclosures | Governance and compensation |
| [`directors-and-board-members`](./docs/tools/directors-and-board-members.md) | Search directors and board members | Governance and compensation |
| [`float`](./docs/tools/float.md) | Fetch public float and share count | Company and entity |
| [`subsidiaries`](./docs/tools/subsidiaries.md) | Search Exhibit 21 subsidiary disclosures | Company and entity |
| [`edgar-entities`](./docs/tools/edgar-entities.md) | Search EDGAR entity master | Company and entity |
| [`mapping`](./docs/tools/mapping.md) | Company identifier mapping (CIK/ticker/CUSIP/name) | Company and entity |
| [`sec-enforcement-actions`](./docs/tools/sec-enforcement-actions.md) | Search SEC enforcement actions | Enforcement |
| [`sec-litigation-releases`](./docs/tools/sec-litigation-releases.md) | Search SEC litigation releases | Enforcement |
| [`sec-administrative-proceedings`](./docs/tools/sec-administrative-proceedings.md) | Search SEC administrative proceedings | Enforcement |
| [`aaers`](./docs/tools/aaers.md) | Search Accounting and Auditing Enforcement Releases (AAER) | Enforcement |
| [`aaer-file`](./docs/tools/aaer-file.md) | Fetch an AAER document file | Enforcement |
| [`sro`](./docs/tools/sro.md) | Search SRO (self-regulatory organization) rule filings | Enforcement |
| [`form-x-17a-5`](./docs/tools/form-x-17a-5.md) | Search Form X-17A-5 (broker-dealer FOCUS reports) | Broker-dealers |
| [`edgar-ingestion-log`](./docs/tools/edgar-ingestion-log.md) | Get all SEC filings ingested at a date | Ingestion logs |
| [`api-key-usage`](./docs/tools/api-key-usage.md) | Get sec-api API key monthly bandwidth usage | Account |

<!-- tools:end -->

## Documentation

| Guide                                            | What it covers                                |
| ------------------------------------------------ | --------------------------------------------- |
| [Getting started](./docs/getting-started.md)     | Key, config, first query, in five minutes     |
| [Setup guides](./docs/setup/README.md)           | Per-client configuration for every MCP client |
| [Tool reference](./docs/tools/README.md)         | All 49 tools, one page each                   |
| [Query language](./docs/query-language.md)       | Lucene syntax and the field vocabulary        |
| [Authentication](./docs/authentication.md)       | Key handling and rotation                     |
| [Transport](./docs/transport.md)                 | Streamable HTTP, and the stdio bridge         |
| [Response format](./docs/response-format.md)     | Envelopes, text blocks, raw documents         |
| [Limits and errors](./docs/limits-and-errors.md) | Paging and error text                         |

## Transport

The server is remote and speaks Streamable HTTP. There is nothing to install and
no stdio server to run.

Clients that only support stdio can bridge to it:

```bash
npx mcp-remote https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY
```

Verify the server from a terminal without any client:

```bash
curl -s https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

## Limits

- Some tools return large payloads. See
  [limits and errors](./docs/limits-and-errors.md) before wiring them into an
  agent loop.

## Citation

```bibtex
@software{secapi_sec_edgar_mcp,
  author  = {{SEC-API.io}},
  title   = {{SEC EDGAR MCP: Model Context Protocol server for SEC EDGAR filings}},
  year    = {2026},
  version = {1.0.0},
  url     = {https://github.com/sec-api/sec-edgar-mcp}
}
```

See [CITATION.cff](./CITATION.cff).

## License

[MIT](./LICENSE). Use of the MCP server is governed by the
[sec-api.io terms](https://sec-api.io/).

## Need help?

Contact us at [support@sec-api.io](mailto://support@sec-api.io).
