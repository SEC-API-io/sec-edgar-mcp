# Tool reference

The SEC EDGAR MCP server exposes **49 tools**. Every tool is listed
below with a link to its own page.

Every tool below works through MCP.

## All tools

| Tool | Purpose | Required input | Category |
| ---- | ------- | -------------- | -------- |
| [`filing-search`](./filing-search.md) | Search SEC EDGAR filings from 1993 to present to retrieve metadata | `query` | Search and discovery |
| [`full-text-search`](./full-text-search.md) | Full-text search over SEC EDGAR filings | `query` | Search and discovery |
| [`filing-to-pdf`](./filing-to-pdf.md) | Fetch any SEC EDGAR filing and exhibit as PDF | `url` | Filings and documents |
| [`get-edgar-file`](./get-edgar-file.md) | Download any SEC EDGAR filing or exhibit in its original format | `url` | Filings and documents |
| [`extractor`](./extractor.md) | Extract a section from a 10-K/10-Q/8-K filing | `url`, `item` | Filings and documents |
| [`xbrl-to-json`](./xbrl-to-json.md) | Convert XBRL financials to normalized JSON | none | Filings and documents |
| [`form-13f-holdings`](./form-13f-holdings.md) | Search Form 13F institutional holdings | `query` | Ownership and insiders |
| [`form-13f-cover-pages`](./form-13f-cover-pages.md) | Search Form 13F cover pages | `query` | Ownership and insiders |
| [`form-13d-13g`](./form-13d-13g.md) | Search Schedule 13D/13G beneficial-ownership filings | `query` | Ownership and insiders |
| [`insider-trading`](./insider-trading.md) | Search insider trading disclosures (Form 3/4/5) | `query` | Ownership and insiders |
| [`form-144`](./form-144.md) | Search Form 144 intent-to-sell filings | `query` | Ownership and insiders |
| [`form-nport`](./form-nport.md) | Search Form N-PORT fund portfolio holdings | `query` | Funds |
| [`form-npx`](./form-npx.md) | Search Form N-PX proxy voting records | `query` | Funds |
| [`form-npx-file`](./form-npx-file.md) | Get a Form N-PX file by accession number | `accessionNo` | Funds |
| [`form-ncen`](./form-ncen.md) | Search Form N-CEN investment company census | `query` | Funds |
| [`form-adv-firms`](./form-adv-firms.md) | Search Form ADV firm registrations | `query` | Investment advisers |
| [`form-adv-individuals`](./form-adv-individuals.md) | Search Form ADV individual registrations | `query` | Investment advisers |
| [`form-adv-brochures`](./form-adv-brochures.md) | Get Form ADV Part 2 brochures for an adviser | `crd` | Investment advisers |
| [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md) | Get Form ADV Schedule A, direct owners of an adviser | `crd` | Investment advisers |
| [`form-adv-schedule-b-indirect-owners`](./form-adv-schedule-b-indirect-owners.md) | Get Form ADV Schedule B, indirect owners of an adviser | `crd` | Investment advisers |
| [`form-adv-schedule-d-1-b`](./form-adv-schedule-d-1-b.md) | Get Form ADV Schedule D-1-B, other business names | `crd` | Investment advisers |
| [`form-adv-schedule-d-5-k`](./form-adv-schedule-d-5-k.md) | Get Form ADV Schedule D-5-K, separately managed accounts | `crd` | Investment advisers |
| [`form-adv-schedule-d-7-a`](./form-adv-schedule-d-7-a.md) | Get Form ADV Schedule D-7-A, financial industry affiliations | `crd` | Investment advisers |
| [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md) | Get Form ADV Schedule D-7-B-1, private funds advised | `crd` | Investment advisers |
| [`form-s1-424b4`](./form-s1-424b4.md) | Search S-1 / 424B4 registration filings | `query` | Offerings and registrations |
| [`form-d`](./form-d.md) | Search Form D private-placement filings | `query` | Offerings and registrations |
| [`form-c`](./form-c.md) | Search Form C Regulation Crowdfunding offerings | `query` | Offerings and registrations |
| [`form-8k`](./form-8k.md) | Search structured 8-K material-event filings | `query` | Offerings and registrations |
| [`reg-a-search`](./reg-a-search.md) | Search Regulation A / A+ offerings | `query` | Offerings and registrations |
| [`reg-a-form-1a`](./reg-a-form-1a.md) | Search structured Reg A Form 1-A filings | `query` | Offerings and registrations |
| [`reg-a-form-1k`](./reg-a-form-1k.md) | Search structured Reg A Form 1-K filings | `query` | Offerings and registrations |
| [`reg-a-form-1z`](./reg-a-form-1z.md) | Search structured Reg A Form 1-Z filings | `query` | Offerings and registrations |
| [`compensation`](./compensation.md) | Search executive compensation data | `query` | Governance and compensation |
| [`compensation-by-key`](./compensation-by-key.md) | Get executive compensation by CIK or ticker | `cikOrTicker` | Governance and compensation |
| [`audit-fees`](./audit-fees.md) | Search auditor / audit-fee disclosures | `query` | Governance and compensation |
| [`directors-and-board-members`](./directors-and-board-members.md) | Search directors and board members | `query` | Governance and compensation |
| [`float`](./float.md) | Fetch public float and share count | none | Company and entity |
| [`subsidiaries`](./subsidiaries.md) | Search Exhibit 21 subsidiary disclosures | `query` | Company and entity |
| [`edgar-entities`](./edgar-entities.md) | Search EDGAR entity master | `query` | Company and entity |
| [`mapping`](./mapping.md) | Company identifier mapping (CIK/ticker/CUSIP/name) | `param`, `value` | Company and entity |
| [`sec-enforcement-actions`](./sec-enforcement-actions.md) | Search SEC enforcement actions | `query` | Enforcement |
| [`sec-litigation-releases`](./sec-litigation-releases.md) | Search SEC litigation releases | `query` | Enforcement |
| [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | Search SEC administrative proceedings | `query` | Enforcement |
| [`aaers`](./aaers.md) | Search Accounting and Auditing Enforcement Releases (AAER) | `query` | Enforcement |
| [`aaer-file`](./aaer-file.md) | Fetch an AAER document file | `aaerNo`, `fileTypeAndName` | Enforcement |
| [`sro`](./sro.md) | Search SRO (self-regulatory organization) rule filings | `query` | Enforcement |
| [`form-x-17a-5`](./form-x-17a-5.md) | Search Form X-17A-5 (broker-dealer FOCUS reports) | `query` | Broker-dealers |
| [`edgar-ingestion-log`](./edgar-ingestion-log.md) | Get all SEC filings ingested at a date | `date` | Ingestion logs |
| [`api-key-usage`](./api-key-usage.md) | Get sec-api API key monthly bandwidth usage | `date` | Account |

## Pick the right tool

These pairs get confused. The deciding question is in the first column.

| Question | Tools | Rule |
| -------- | ----- | ---- |
| Filter filings by metadata, or find filings that mention a term? | [`filing-search`](./filing-search.md) vs [`full-text-search`](./full-text-search.md) | `filing-search` filters on fields such as ticker and form type, sorted by date. `full-text-search` searches the document text, sorted by relevance. |
| Need one section, the whole file, or a PDF? | [`extractor`](./extractor.md) vs [`get-edgar-file`](./get-edgar-file.md) | Use `extractor` for a single item such as Risk Factors. Use `get-edgar-file` for the raw file as filed. Use `filing-to-pdf` for a rendered PDF. |
| Institutional holdings, or the filings that carry them? | [`form-13f-holdings`](./form-13f-holdings.md) vs [`form-13f-cover-pages`](./form-13f-cover-pages.md) | `form-13f-holdings` returns individual positions. `form-13f-cover-pages` returns one row per filing, useful to enumerate managers first. |
| Search compensation, or look one company up? | [`compensation`](./compensation.md) vs [`compensation-by-key`](./compensation-by-key.md) | `compensation` takes a Lucene query. `compensation-by-key` takes one CIK or ticker and returns up to 200 rows. |
| Search proxy votes, or read one filing in full? | [`form-npx`](./form-npx.md) vs [`form-npx-file`](./form-npx-file.md) | `form-npx` returns filing metadata. `form-npx-file` returns every vote record in one filing, with no paging, and can exceed 1 MB. |

## By category

### Search and discovery (2)

- [`filing-search`](./filing-search.md). Search SEC EDGAR filings from 1993 to present to retrieve metadata
- [`full-text-search`](./full-text-search.md). Full-text search over SEC EDGAR filings

### Filings and documents (4)

- [`filing-to-pdf`](./filing-to-pdf.md). Fetch any SEC EDGAR filing and exhibit as PDF
- [`get-edgar-file`](./get-edgar-file.md). Download any SEC EDGAR filing or exhibit in its original format
- [`extractor`](./extractor.md). Extract a section from a 10-K/10-Q/8-K filing
- [`xbrl-to-json`](./xbrl-to-json.md). Convert XBRL financials to normalized JSON

### Ownership and insiders (5)

- [`form-13f-holdings`](./form-13f-holdings.md). Search Form 13F institutional holdings
- [`form-13f-cover-pages`](./form-13f-cover-pages.md). Search Form 13F cover pages
- [`form-13d-13g`](./form-13d-13g.md). Search Schedule 13D/13G beneficial-ownership filings
- [`insider-trading`](./insider-trading.md). Search insider trading disclosures (Form 3/4/5)
- [`form-144`](./form-144.md). Search Form 144 intent-to-sell filings

### Funds (4)

- [`form-nport`](./form-nport.md). Search Form N-PORT fund portfolio holdings
- [`form-npx`](./form-npx.md). Search Form N-PX proxy voting records
- [`form-npx-file`](./form-npx-file.md). Get a Form N-PX file by accession number
- [`form-ncen`](./form-ncen.md). Search Form N-CEN investment company census

### Investment advisers (9)

- [`form-adv-firms`](./form-adv-firms.md). Search Form ADV firm registrations
- [`form-adv-individuals`](./form-adv-individuals.md). Search Form ADV individual registrations
- [`form-adv-brochures`](./form-adv-brochures.md). Get Form ADV Part 2 brochures for an adviser
- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md). Get Form ADV Schedule A, direct owners of an adviser
- [`form-adv-schedule-b-indirect-owners`](./form-adv-schedule-b-indirect-owners.md). Get Form ADV Schedule B, indirect owners of an adviser
- [`form-adv-schedule-d-1-b`](./form-adv-schedule-d-1-b.md). Get Form ADV Schedule D-1-B, other business names
- [`form-adv-schedule-d-5-k`](./form-adv-schedule-d-5-k.md). Get Form ADV Schedule D-5-K, separately managed accounts
- [`form-adv-schedule-d-7-a`](./form-adv-schedule-d-7-a.md). Get Form ADV Schedule D-7-A, financial industry affiliations
- [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md). Get Form ADV Schedule D-7-B-1, private funds advised

### Offerings and registrations (8)

- [`form-s1-424b4`](./form-s1-424b4.md). Search S-1 / 424B4 registration filings
- [`form-d`](./form-d.md). Search Form D private-placement filings
- [`form-c`](./form-c.md). Search Form C Regulation Crowdfunding offerings
- [`form-8k`](./form-8k.md). Search structured 8-K material-event filings
- [`reg-a-search`](./reg-a-search.md). Search Regulation A / A+ offerings
- [`reg-a-form-1a`](./reg-a-form-1a.md). Search structured Reg A Form 1-A filings
- [`reg-a-form-1k`](./reg-a-form-1k.md). Search structured Reg A Form 1-K filings
- [`reg-a-form-1z`](./reg-a-form-1z.md). Search structured Reg A Form 1-Z filings

### Governance and compensation (4)

- [`compensation`](./compensation.md). Search executive compensation data
- [`compensation-by-key`](./compensation-by-key.md). Get executive compensation by CIK or ticker
- [`audit-fees`](./audit-fees.md). Search auditor / audit-fee disclosures
- [`directors-and-board-members`](./directors-and-board-members.md). Search directors and board members

### Company and entity (4)

- [`float`](./float.md). Fetch public float and share count
- [`subsidiaries`](./subsidiaries.md). Search Exhibit 21 subsidiary disclosures
- [`edgar-entities`](./edgar-entities.md). Search EDGAR entity master
- [`mapping`](./mapping.md). Company identifier mapping (CIK/ticker/CUSIP/name)

### Enforcement (6)

- [`sec-enforcement-actions`](./sec-enforcement-actions.md). Search SEC enforcement actions
- [`sec-litigation-releases`](./sec-litigation-releases.md). Search SEC litigation releases
- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). Search SEC administrative proceedings
- [`aaers`](./aaers.md). Search Accounting and Auditing Enforcement Releases (AAER)
- [`aaer-file`](./aaer-file.md). Fetch an AAER document file
- [`sro`](./sro.md). Search SRO (self-regulatory organization) rule filings

### Broker-dealers (1)

- [`form-x-17a-5`](./form-x-17a-5.md). Search Form X-17A-5 (broker-dealer FOCUS reports)

### Ingestion logs (1)

- [`edgar-ingestion-log`](./edgar-ingestion-log.md). Get all SEC filings ingested at a date

### Account (1)

- [`api-key-usage`](./api-key-usage.md). Get sec-api API key monthly bandwidth usage

## Notes that apply to every tool

- Results arrive as one text block. JSON is stringified, and no tool declares an
  output schema. See [response format](../response-format.md).
- Response envelopes differ per tool. Do not assume `data[]`.
- Some tools accept no pagination at all and return their full result set.
  `float` and `form-npx-file` are the ones to watch.

<!-- generated by scripts/gen-index.js. do not edit by hand. -->
