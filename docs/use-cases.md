# Use cases

Ten worked jobs. Each one chains several tools. For each job you get the prompt
a user types, the tool calls with their real arguments, what comes back, and the
cost in tool calls.

Every count and payload size on this page is from 2026-08-13. The numbers move
as companies file. The shapes do not.

Read [Getting started](./getting-started.md) first if the server is not
connected yet. Every tool has its own page in the
[tool reference](./tools/README.md).

## How to read the cost

Plan the chain before you run it.

| # | Use case | Calls | Heaviest payload |
| - | -------- | ----- | ---------------- |
| 1 | [Compare risk factors](#1-compare-risk-factors-across-two-10-k-filings) | 3 | 70 KB per section |
| 2 | [Income statement from XBRL](#2-pull-a-full-income-statement-from-xbrl) | 2 | 1.31 MB |
| 3 | [Insider buying over 90 days](#3-track-insider-buying-at-one-ticker-over-90-days) | 2 to 3 | 70 KB |
| 4 | [Rebuild a 13F portfolio](#4-rebuild-a-13f-portfolio-for-one-quarter) | 2 | 266 KB |
| 5 | [Every 8-K that mentions a term](#5-find-every-8-k-that-mentions-a-term) | 1 to 11 | 32 KB per page |
| 6 | [Subsidiary tree from Exhibit 21](#6-map-a-companys-subsidiary-tree-from-exhibit-21) | 2 to 3 | 185 KB |
| 7 | [IPO pipeline](#7-screen-an-ipo-pipeline-from-s-1-and-424b4-filings) | 2 to 8 | 80 KB per page |
| 8 | [Adviser disciplinary history](#8-check-an-investment-adviser-for-disciplinary-history) | 6 to 7 | 90 KB |
| 9 | [Peer compensation table](#9-build-a-peer-executive-compensation-table) | 1 to 4 | 6 KB |
| 10 | [Watch one firm for enforcement](#10-watch-enforcement-actions-against-one-firm) | 4 per sweep | 125 KB |

Three chains are expensive. Use case 2 returns 1.31 MB in one text block. Use
case 4 returns 266 KB. Use case 7 needs 8 calls for a full sweep. Read the
warnings in those sections before you run them.

## 1. Compare risk factors across two 10-K filings

Tools: [`filing-search`](./tools/filing-search.md),
[`extractor`](./tools/extractor.md).

> Compare the risk factors in Apple's two most recent 10-K filings. What is new
> this year?

**Step 1. Find both filings.** The default sort is `filedAt` descending, so
`size: 2` gives you this year and last year.

```json
{
  "name": "filing-search",
  "arguments": { "query": "ticker:AAPL AND formType:\"10-K\"", "size": 2 }
}
```

Returns `{total, query, filings[]}` with `total.value: 33`. Read
`linkToFilingDetails` from each row. The two newest are
`.../000032019325000079/aapl-20250927.htm` and
`.../000032019324000123/aapl-20240928.htm`.

**Step 2. Pull Item 1A from the newer filing.**

```json
{
  "name": "extractor",
  "arguments": {
    "url": "https://www.sec.gov/Archives/edgar/data/320193/000032019325000079/aapl-20250927.htm",
    "item": "1A",
    "type": "text"
  }
}
```

**Step 3. Pull Item 1A from the older filing.** Same call with the 2024 URL.

Each call returns one text block that starts with the heading
` Item 1A. Risk Factors `. The 2025 section measured 69,877 bytes. There is no
JSON envelope and no length field. The agent then diffs the two sections itself.

**Cost: 3 tool calls.**

Watch out.

- The two sections together put about 140 KB of text in the context. Ask for one
  risk topic instead of a full diff when the context is tight.
- `extractor` does not decode HTML entities. The text holds `&#8217;` for an
  apostrophe and `&#8220;` for a quotation mark. Decode before you compare
  strings.
- `extractor` accepts only 10-K, 10-Q and 8-K. The `url` must be the primary
  document, not an exhibit.
- The same chain works across two companies. Two companies cost 4 calls, one
  `filing-search` and one `extractor` each.

## 2. Pull a full income statement from XBRL

Tools: [`filing-search`](./tools/filing-search.md),
[`xbrl-to-json`](./tools/xbrl-to-json.md).

> Give me Apple's fiscal 2025 income statement from the 10-K, as numbers.

**Step 1. Get the accession number.**

```json
{
  "name": "filing-search",
  "arguments": { "query": "ticker:AAPL AND formType:\"10-K\"", "size": 1 }
}
```

Read `filings[0].accessionNo`, which is `0000320193-25-000079`.

**Step 2. Convert the XBRL.**

```json
{ "name": "xbrl-to-json", "arguments": { "accession-no": "0000320193-25-000079" } }
```

Returns a bare JSON object with 84 top-level keys and **1,313,231 bytes**. The
income statement sits under `StatementsOfIncome`, which holds 15 line items.
Each line item is an array of fact objects:

```json
{
  "StatementsOfIncome": {
    "NetIncomeLoss": [
      {
        "decimals": "-6",
        "unitRef": "usd",
        "period": { "startDate": "2024-09-29", "endDate": "2025-09-27" },
        "value": "112010000000"
      }
    ]
  }
}
```

**Cost: 2 tool calls. 1 call when you already hold the accession number.**

Watch out.

- **The response is large.** One 10-K is 1.31 MB in a single text block. A loop
  over 100 filings moves about 130 MB.
- There is no way to ask for one statement. You get all 84 keys or nothing.
- The top-level keys are the filer's own statement names. They differ between
  companies and between years. `StatementsOfIncome` is not a fixed key.
- `value` is always a string. Parse it.
- A fact with no `segment` is the consolidated total. A fact with a `segment` is
  a product or region breakdown. Filter on the absence of `segment` for headline
  figures.
- Send exactly one of `htm-url`, `xbrl-url` or `accession-no`. Sending none
  returns HTTP 400.

## 3. Track insider buying at one ticker over 90 days

Tools: [`insider-trading`](./tools/insider-trading.md).

> Which NVIDIA insiders bought stock on the open market in the last 90 days?

**Step 1. Count the Form 4 activity in the window.**

```json
{
  "name": "insider-trading",
  "arguments": {
    "query": "issuer.tradingSymbol:NVDA AND documentType:4 AND periodOfReport:[2026-05-15 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `{total, transactions[]}` with `total.value: 23`. Twenty-three Form 4
filings, at about 3 KB each.

**Step 2. Keep only open-market purchases.** Transaction code `P` means the
insider bought on the open market.

```json
{
  "name": "insider-trading",
  "arguments": {
    "query": "issuer.tradingSymbol:NVDA AND nonDerivativeTable.transactions.coding.code:P AND periodOfReport:[2026-05-15 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `{"total":{"value":0,"relation":"eq"},"transactions":[]}`. That is the
real answer. No NVIDIA insider bought stock on the open market in that window.
All 23 filings are grants, option settlements, tax withholding and sales.

**Step 3, optional. Prove the field works.** A zero result is easy to
misdiagnose. Run the same code filter with no ticker:

```json
{
  "name": "insider-trading",
  "arguments": {
    "query": "nonDerivativeTable.transactions.coding.code:P AND periodOfReport:[2026-08-01 TO 2026-08-13]",
    "size": 1
  }
}
```

Returns `total.value: 389`. The field works. The ticker had no buys.

**Cost: 2 tool calls. 3 with the control query.**

Watch out.

- The array is named `transactions` but it holds **filings**. One filing can
  carry several trades, in `nonDerivativeTable.transactions[]` and
  `derivativeTable.transactions[]`. Count trades inside each item.
- The ticker field here is `issuer.tradingSymbol`. Other tools use bare `ticker`
  or `entities.ticker`. A user who learns one guesses wrong on the others.
- Transaction codes seen in the data: `P` open-market purchase, `S` open-market
  sale, `A` grant, `M` derivative settled, `F` shares withheld for tax. An `F`
  disposal looks like a sale and is not one.
- `pricePerShare` is optional. When a footnote sets the price, the field is
  absent and `pricePerShareFootnoteId` appears. Read the footnote before you
  report a price.
- `size` defaults to 50. An active issuer with no `size` can return over 100 KB.

## 4. Rebuild a 13F portfolio for one quarter

Tools: [`form-13f-cover-pages`](./tools/form-13f-cover-pages.md),
[`form-13f-holdings`](./tools/form-13f-holdings.md).

> Rebuild Bridgewater's reported portfolio as of 31 March 2026.

**Step 1. Find the manager and the filing.** You know the name, not the CIK.

```json
{
  "name": "form-13f-cover-pages",
  "arguments": { "query": "filingManager.name:\"Bridgewater Associates\"", "size": 2 }
}
```

Returns `total.value: 54` and one row per quarterly report. The row for
2026-03-31 gives you everything you need to size the next call:

```json
{
  "accessionNo": "0001350694-26-000002",
  "cik": "1350694",
  "crdNumber": "105129",
  "periodOfReport": "2026-03-31",
  "filingManager": { "name": "Bridgewater Associates, LP" },
  "tableEntryTotal": 993,
  "tableValueTotal": 22404547213
}
```

993 positions and $22.4 billion of reported value.

**Step 2. Pull the positions.**

```json
{
  "name": "form-13f-holdings",
  "arguments": { "query": "cik:1350694 AND periodOfReport:\"2026-03-31\"", "size": 1 }
}
```

Returns one filing that holds all 993 positions in **265,822 bytes**. One
position looks like this:

```json
{
  "nameOfIssuer": "10X GENOMICS INC",
  "cusip": "88025U109",
  "titleOfClass": "CL A COM",
  "value": 745194,
  "shrsOrPrnAmt": { "sshPrnamt": 35101, "sshPrnamtType": "SH" },
  "investmentDiscretion": "SOLE",
  "votingAuthority": { "Sole": 35101, "Shared": 0, "None": 0 },
  "ticker": "TXG",
  "cik": "1770787"
}
```

`ticker` and `cik` on each holding are added by sec-api, so you do not need a
CUSIP lookup.

**Cost: 2 tool calls.**

Watch out.

- **`size` counts filings, not positions, and it defaults to 50.** A call
  without `size` on this tool asks for 50 filings, each with hundreds of
  positions. Always set `size: 1` first.
- 266 KB arrives in one text block. Sum the positions in code, not by reading.
- The same issuer can appear several times in `holdings[]`, once per
  `otherManager` group. Sum the lines before you report a position. Resolve the
  `otherManager` sequence numbers through
  `otherIncludedManagers[].sequenceNumber` on the cover page.
- The two 13F indices disagree on counts. `cik:1067983` returns 60 cover pages
  and 210 holdings rows. Neither total is a proxy for the other.
- `form-13f-holdings` rewrites your query. If the query does not contain the
  text `13F`, the server prepends `formType:"13F-HR" AND `. Write `13F` yourself
  when you want amendments.
- Holdings are reported quarterly. A quarter ending 31 March is filed up to 45
  days later.

## 5. Find every 8-K that mentions a term

Tools: [`full-text-search`](./tools/full-text-search.md),
[`filing-search`](./tools/filing-search.md),
[`get-edgar-file`](./tools/get-edgar-file.md).

> Which 8-K filings mentioned "tariff surcharge" this year?

**Step 1. Search the document text.**

```json
{
  "name": "full-text-search",
  "arguments": {
    "query": "\"tariff surcharge\"",
    "formTypes": ["8-K"],
    "startDate": "2026-01-01",
    "endDate": "2026-08-13"
  }
}
```

Returns `{total, filings[]}` with `total.value: 5`. Each row is one document,
with `accessionNo`, `cik`, `ticker`, `formType`, `type` (the exhibit type),
`filingUrl` and `filedAt`. The hits are TransAct Technologies, Keysight and
DIRTT, all in `EX-99.1` exhibits.

**Step 2. Get the parent filing.** The search returns the exhibit URL. Use
`accessionNo` to reach the primary document and the item list.

```json
{
  "name": "filing-search",
  "arguments": { "query": "accessionNo:\"0001214659-26-009902\"", "size": 1 }
}
```

Returns `description: "Form 8-K - Current report - Item 2.02 Item 9.01"` and
`linkToFilingDetails` pointing at `x8102628k.htm`.

**Step 3. Read the matched text.** Two options, one call each.

```json
{
  "name": "get-edgar-file",
  "arguments": { "url": "https://www.sec.gov/Archives/edgar/data/1017303/000121465926009902/ex99_1.htm" }
}
```

That returns the exhibit exactly as filed. To read the 8-K body instead, call
`extractor` with the primary document URL and `item: "2-2"`, which is the code
for SEC Item 2.02.

**Cost: 1 tool call for the search.** Add 1 call per document you fetch
straight from `filingUrl`, or 2 calls per filing when you want the primary
document and one item. Reading all five hits costs 6 to 11 calls.

Watch out.

- One row is one **document**, not one filing. A filing with four matching
  exhibits produces up to five rows that share one `accessionNo`. Count unique
  `accessionNo` before you report a filing count. DIRTT shows the other case: it
  appears twice under two different accession numbers, so that is one company
  and two filings.
- Give both `startDate` and `endDate` or neither. If one is missing, the server
  silently replaces both with the last 12 months.
- Coverage starts in 2001. Paging is `page` only, 1-based, 100 rows per page.
  There is no `size` and no `sort`. Results arrive by relevance, so dates are out
  of order.
- **`form-8k` will not find this filing.** That index holds only filings with an
  Item 4.01, 4.02 or 5.02 block. The query
  `accessionNo:"0001214659-26-009902"` returns zero rows there, while
  `ticker:TACT AND items:"Item 2.02"` returns 4 rows, because those filings
  also carry an Item 5.02. Use `full-text-search` or `filing-search` for every
  other 8-K item.
- The response carries no snippet and no highlight. You never see the matched
  sentence until you fetch the document.

## 6. Map a company's subsidiary tree from Exhibit 21

Tools: [`subsidiaries`](./tools/subsidiaries.md),
[`get-edgar-file`](./tools/get-edgar-file.md).

> Map Apple's legal entities and tell me what changed since 2020.

**Step 1. Get the newest Exhibit 21.**

```json
{ "name": "subsidiaries", "arguments": { "query": "ticker:AAPL", "size": 1 } }
```

Returns `total.value: 26`, one row per annual report. The newest row is
accession `0000320193-25-000079` and lists 19 subsidiaries with their
jurisdictions.

**Step 2. Get the 2020 exhibit for the comparison.**

```json
{
  "name": "subsidiaries",
  "arguments": { "query": "ticker:AAPL AND filedAt:[2020-01-01 TO 2020-12-31]", "size": 1 }
}
```

Returns accession `0000320193-20-000096` with a longer list, including entities
that no longer appear: `Apple Computer Trading (Shanghai) Co., Ltd.`,
`Apple France`, `Apple GmbH`, `Apple Operations Europe Limited`. The agent diffs
the two `subsidiaries[]` arrays by name.

**Step 3, optional. Find peers with the same footprint.**

```json
{
  "name": "subsidiaries",
  "arguments": { "query": "subsidiaries.jurisdiction:Ireland", "size": 5 }
}
```

**Cost: 2 tool calls for the diff. 3 with the peer scan.**

Watch out.

- Exhibit 21 is a **flat list, not a tree**. It links the filer to each
  subsidiary. It never links a subsidiary to another subsidiary. You cannot
  build a real hierarchy from this data alone.
- A match on `subsidiaries.name` or `subsidiaries.jurisdiction` returns the
  whole parent row, with every sibling in it. The server does not filter the
  inner array.
- Broad jurisdiction queries are heavy. Ten rows of one jurisdiction measured
  184,676 bytes. All 26 Apple rows together measure 18,046 bytes.
- Coverage is Exhibit 21 only. Companies may omit subsidiaries they call
  immaterial. A missing name is not proof of no ownership.
- For the exhibit as filed, take the Ex-21 URL from `documentFormatFiles[]` in
  `filing-search` and call `get-edgar-file`.

## 7. Screen an IPO pipeline from S-1 and 424B4 filings

Tools: [`form-s1-424b4`](./tools/form-s1-424b4.md),
[`get-edgar-file`](./tools/get-edgar-file.md).

> Which IPOs registered and which priced between 1 July and 13 August 2026?

**Step 1. The pipeline. S-1 filings are registrations, filed before pricing.**

```json
{
  "name": "form-s1-424b4",
  "arguments": {
    "query": "formType:\"S-1\" AND filedAt:[2026-07-01 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `total.value: 287`. A pre-pricing row carries the issuer, the tickers,
the securities, the law firms and the auditors, and **empty** deal objects:

```json
{
  "formType": "S-1",
  "cik": "2068385",
  "ticker": "SHAZ",
  "entityName": "SharonAI Holdings Inc.",
  "tickers": [{ "ticker": "SHAZ", "type": "Class A Ordinary Common Stock", "exchange": "Nasdaq Capital Market" }],
  "securities": [{ "name": "8,021,282 Shares of Class A Ordinary Common Stock" }],
  "publicOfferingPrice": {},
  "underwritingDiscount": {},
  "proceedsBeforeExpenses": {},
  "underwriters": []
}
```

**Step 2. The priced deals. A 424B4 is the final prospectus.**

```json
{
  "name": "form-s1-424b4",
  "arguments": {
    "query": "formType:\"424B4\" AND filedAt:[2026-07-01 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `total.value: 65`. These rows carry `publicOfferingPrice`,
`underwritingDiscount`, `proceedsBeforeExpenses` and the `underwriters[]` list,
each with a numeric field and a `*Text` twin that holds the currency symbol.

**Step 3, optional. Keep only the large deals.** Range syntax works on the
numeric fields.

```json
{
  "name": "form-s1-424b4",
  "arguments": {
    "query": "formType:\"424B4\" AND filedAt:[2026-07-01 TO 2026-08-13] AND publicOfferingPrice.total:[100000000 TO *]",
    "size": 50
  }
}
```

**Cost: 2 tool calls for the two counts. 8 calls to page through both lists
in full**, because 287 S-1 rows need 6 pages and 65 424B4 rows need 2.

Watch out.

- Page with `from` in steps of 50. `from` plus `size` must stay at or below
  10,000. Above that the server returns `{"total":{"value":0},"data":[]}` with no
  error, which looks like the end of the data and is not.
- **424B3 and 424B5 are not in this index.** Both return zero rows. The index
  holds `S-1`, `S-1/A`, `424B4`, `F-1`, `F-1/A` and `S-11`.
- Fields are omitted, not nulled, when the prospectus does not state them. Test
  for an empty `publicOfferingPrice` object before you read `total`.
- The tool description promises use-of-proceeds and a risk-factors summary. No
  such field exists. For prospectus prose, call `get-edgar-file` on `filingUrl`.
  **`extractor` accepts only 10-K, 10-Q and 8-K.** It rejects a 424B4.
- `entityName`, `underwriters.name` and `auditors.name` are analysed text
  fields. A quoted phrase matches loosely. Check the rows you get back.

## 8. Check an investment adviser for disciplinary history

Tools: [`form-adv-firms`](./tools/form-adv-firms.md),
[`form-adv-brochures`](./tools/form-adv-brochures.md),
[`sec-administrative-proceedings`](./tools/sec-administrative-proceedings.md),
[`sec-enforcement-actions`](./tools/sec-enforcement-actions.md),
[`sec-litigation-releases`](./tools/sec-litigation-releases.md),
[`aaers`](./tools/aaers.md).

> Does Morgan Stanley Smith Barney have a disciplinary history? Start from
> Form ADV and then check the SEC's own records.

**Step 1. Find the firm and its CRD number.**

```json
{ "name": "form-adv-firms", "arguments": { "query": "Info.BusNm:\"MORGAN STANLEY\"", "size": 3 } }
```

Returns `total.value: 9`. The phrase match is loose, so the first row is
`MORGAN STANLEY | EATON VANCE CLO MANAGER LLC`, CRD 309263. Read the rows and
pick the right firm. Morgan Stanley Smith Barney LLC is CRD **149777**.

**Step 2. Read the Part 1 disciplinary answers.** One call returns the whole
record, so there is no separate disclosure call.

```json
{ "name": "form-adv-firms", "arguments": { "query": "Info.FirmCrdNb:149777", "size": 1 } }
```

Returns one row of 6,477 bytes. The Item 11 blocks are the disciplinary
questions:

```json
{
  "Item11": { "Q11": "Y" },
  "Item11A": { "Q11A1": "Y", "Q11A2": "Y" },
  "Item11B": { "Q11B1": "Y", "Q11B2": "Y" },
  "Item11C": { "Q11C1": "Y", "Q11C2": "Y", "Q11C3": "N", "Q11C4": "Y", "Q11C5": "Y" },
  "Item11D": { "Q11D1": "N", "Q11D2": "Y", "Q11D3": "N", "Q11D4": "Y", "Q11D5": "Y" },
  "Item11E": { "Q11E1": "Y", "Q11E2": "Y", "Q11E3": "N", "Q11E4": "Y" },
  "Item11F": { "Q11F": "Y" },
  "Item11G": { "Q11G": "Y" },
  "Item11H": { "Q11H1A": "Y", "Q11H1B": "Y", "Q11H1C": "Y", "Q11H2": "Y" }
}
```

The same row holds `FormInfo.Part1A.Item5F.Q5F2C: 1961887313229`, the total
regulatory assets under management, and `Item5A.TtlEmp: 29481`.

**Step 3. Get the brochures.** Part 2 describes the disciplinary record in
plain English.

```json
{ "name": "form-adv-brochures", "arguments": { "crd": "149777" } }
```

Returns `{brochures[]}` with 12 entries, each with `name`, `dateSubmitted` and a
PDF `url` on `files.adviserinfo.sec.gov`.

**Step 4 to 7. Search the four SEC enforcement indices by name.**

```json
{ "name": "sec-administrative-proceedings", "arguments": { "query": "respondents.name:\"Morgan Stanley\"", "size": 50 } }
```

```json
{ "name": "sec-enforcement-actions", "arguments": { "query": "entities.name:\"Morgan Stanley\"", "size": 50 } }
```

```json
{ "name": "sec-litigation-releases", "arguments": { "query": "entities.name:\"Morgan Stanley\"", "size": 50 } }
```

```json
{ "name": "aaers", "arguments": { "query": "respondents.name:\"Morgan Stanley\"", "size": 50 } }
```

The counts: 72 administrative proceedings, 27 announced enforcement actions, 26
litigation releases, 1 AAER. The single AAER is `AAER-2132` from 2004, about
valuation methods in an aircraft leasing portfolio. The newest administrative
proceeding is release `34-105802`, file numbers `3-21825` and `3-21826`, a Fair
Fund disbursement of $119,825,301.06.

**Cost: 6 tool calls. 7 when you must resolve the CRD by name first.**

Watch out.

- **Item 11 gives you `Y` or `N` and nothing else.** There is no event detail,
  no date and no amount in Form ADV Part 1 through this API. The four
  enforcement tools supply the detail.
- There is no key that joins a CRD number to the enforcement indices. The join
  is by name and it is fuzzy. `"Morgan Stanley"` matches Morgan Stanley & Co.
  LLC, Morgan Stanley Smith Barney LLC and Morgan Stanley Investment Management.
  Read `respondents[]` or `entities[]` on each row before you attribute a case.
- The four indices overlap. One event can appear as a press release, an
  administrative proceeding and an AAER at the same time. Deduplicate on the
  document URL, not on the summary text.
- `form-adv-brochures` returns links, never the brochure text. An empty array
  means the adviser files no brochure.
- To check a named person instead of the firm, use `form-adv-individuals` with
  `CrntEmps.CrntEmp.orgPK:149777` plus `Info.lastNm`. That tool also returns
  flags only, in `DRPs.DRP[]`.
- Add a date range to any of the four searches to watch a window. Use
  `releasedAt:[2024-01-01 TO 2026-08-13]`, or `dateTime:[...]` on `aaers`.

## 9. Build a peer executive-compensation table

Tools: [`compensation`](./tools/compensation.md),
[`directors-and-board-members`](./tools/directors-and-board-members.md).

> Compare 2025 executive pay at Apple, Microsoft and Salesforce.

**Step 1. One call covers all three companies.**

```json
{
  "name": "compensation",
  "arguments": {
    "query": "(ticker:AAPL OR ticker:MSFT OR ticker:CRM) AND year:2025",
    "size": 50
  }
}
```

Returns a **bare JSON array** of 16 rows in 5,769 bytes. No `total`, no
envelope. Read `response[0]`, not `response.data[0]`. One row is one named
executive for one year:

```json
{
  "cik": "789019",
  "ticker": "MSFT",
  "name": "Satya Nadella",
  "position": "Chairman and Chief Executive Officer",
  "year": 2025,
  "salary": 2500000,
  "bonus": 0,
  "stockAwards": 84245496,
  "optionAwards": 0,
  "nonEquityIncentiveCompensation": 9555000,
  "changeInPensionValueAndDeferredEarnings": 0,
  "otherCompensation": 196294,
  "total": 96496790
}
```

The three chief executives in that answer: Satya Nadella $96,496,790, Tim Cook
$74,294,811, Marc Benioff $55,074,656.

**Step 2, optional. Add board and committee context.** One call per company.

```json
{ "name": "directors-and-board-members", "arguments": { "query": "ticker:MSFT", "size": 1 } }
```

Returns `{total, data[]}` with `total.value: 16`, one row per proxy statement.
Each row holds a `directors[]` array with `name`, `position`, `age`,
`directorClass`, `dateFirstElected`, `isIndependent`, `committeeMemberships[]`
and `qualificationsAndExperience[]`. Filter `committeeMemberships` for
`Compensation` to find who sets the pay.

**Cost: 1 tool call for the pay table. 4 calls with board context for three
companies.**

Watch out.

- **`compensation` returns a bare array.** There is no `total`, so you cannot
  see how many rows exist beyond your page. `size` caps at 50. Page with `from`
  until a short page comes back.
- `position` is an analysed text field, so a query filter on it is unreliable.
  `position:*Chief Executive*` matched a row whose title is
  `Senior Vice President, Chief Financial Officer`. The wildcard matches per
  word. Filter titles in your own code after the call.
- Row order is not by `total`. Sort yourself.
- The figures are the summary compensation table as the company reported it.
  `stockAwards` is a grant-date fair value, not cash received.
- `directors-and-board-members` mixes officers into `directors[]`. Apple's row
  lists the CFO and the General Counsel next to the outside directors. Check
  `isIndependent` and `dateFirstElected`, and note that `isIndependent` is often
  `null`.

## 10. Watch enforcement actions against one firm

Tools: [`sec-enforcement-actions`](./tools/sec-enforcement-actions.md),
[`sec-administrative-proceedings`](./tools/sec-administrative-proceedings.md),
[`sec-litigation-releases`](./tools/sec-litigation-releases.md),
[`aaers`](./tools/aaers.md).

> Watch for new SEC actions against Morgan Stanley since the start of 2024.

One sweep is four calls, one per index. Run the same four on a schedule and diff
the results on the identifier field.

**Call 1. Announced actions, from press releases.**

```json
{
  "name": "sec-enforcement-actions",
  "arguments": {
    "query": "entities.name:\"Morgan Stanley\" AND releasedAt:[2024-01-01 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `total.value: 2`. The newest is release `2024-193`, a $15 million
settlement over failures to detect financial advisers stealing client funds.
Diff on `releaseNo`.

**Call 2. In-house cases.**

```json
{
  "name": "sec-administrative-proceedings",
  "arguments": {
    "query": "respondents.name:\"Morgan Stanley\" AND releasedAt:[2026-01-01 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `total.value: 1` for 2026 so far, and 72 for all time. Diff on
`fileNumbers[]`, which is the case key, or on `releaseNo[]`, which is an
**array** on this tool and a string on the other two.

**Call 3. Federal court cases.**

```json
{
  "name": "sec-litigation-releases",
  "arguments": {
    "query": "entities.name:\"Morgan Stanley\" AND releasedAt:[2024-01-01 TO 2026-08-13]",
    "size": 50
  }
}
```

Returns `total.value: 1` for that window, and 26 for all time. Diff on
`releaseNo`, for example `LR-26434`. Read `subTitle`, not `title`. `title` is
often only a party name.

**Call 4. Accounting and auditing cases.**

```json
{
  "name": "aaers",
  "arguments": {
    "query": "respondents.name:\"Morgan Stanley\" AND dateTime:[2024-01-01 TO 2026-08-13]",
    "size": 50
  }
}
```

The date field here is `dateTime`, not `releasedAt`. Sorting by `releasedAt`
returns nothing useful. This query returns zero rows, because the only Morgan
Stanley AAER is from 2004. Widen the range to `[2000-01-01 TO 2010-12-31]` and
it returns 1.

**Cost: 4 tool calls per sweep.** At 50 rows per call the worst case is about
125 KB.

Watch out.

- The four indices overlap and none is complete on its own.
  `sec-enforcement-actions` reads the SEC newsroom, and the SEC does not
  announce every action. The server also prepends `tags:* AND ` to your query
  there, so a release with no extracted tags cannot match.
- Field names differ across the four tools. `releaseNo` is a string on
  `sec-enforcement-actions` and `sec-litigation-releases`, and an array on
  `sec-administrative-proceedings`. `aaers` uses `dateTime` and `urls[]` where
  the others use `releasedAt` and `resources[]`.
- `total.value: 10000` with `relation: "gte"` is the search window ceiling, not
  a count. Narrow with a date range for an exact number.
- `entities[].ticker` is not always tradable. Private LLCs get synthetic symbols
  such as `INVES6` and `UPFRON`. Check one with `mapping` before you use it.
- `releasedAt` is the date of the document, not the date of the misconduct. One
  row released in 2026 links a PDF stored under `/admin/2023/`.
- Not every row is a charging document. Orders granting an extension of time
  arrive with empty `complaints` and empty `violatedSections`. Filter on those
  fields when you want charges only.

## Rules that apply to every chain

- **One text block, JSON as a string.** No tool declares an output schema, so
  there is no `structuredContent`.
- **Envelopes differ.** `filings[]`, `data[]`, `transactions[]`, `brochures[]`,
  a bare array, or raw text. `data[]` is one shape among many, not the default.
  See [response format](./response-format.md).
- **Errors arrive as text.** `isError` is true and the text reads
  `sec-api error: <message>`.

## Next steps

- [Tool reference](./tools/README.md). All 49 tools, one page each.
- [Getting started](./getting-started.md). Connect the server in four steps.
- [Query language](./query-language.md). Lucene syntax and the field names.
- [Response format](./response-format.md). The ten envelopes.
- [Limits and errors](./limits-and-errors.md). Sizes and error text.
