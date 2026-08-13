# Query language

Most tools on this server take a Lucene query string. This page explains that
syntax, lists the field names that are confirmed to work, and shows the mistakes
that cost the most time.

Every count on this page was measured live on 2026-08-13. Counts change as EDGAR
grows. The syntax does not.

## Three kinds of input

The 49 tools split into three groups. Check which group your tool is in before
you write anything.

| Group | Tools | How you filter |
| ----- | ----- | -------------- |
| Lucene `query` | 30 search tools | A query string such as `ticker:AAPL AND formType:"10-K"`. |
| Phrase search | [`full-text-search`](./tools/full-text-search.md) only | A phrase plus the `ciks`, `formTypes`, `startDate` and `endDate` parameters. **No field syntax.** |
| Named parameters | 18 fetch tools | Fixed keys such as `ticker`, `cik`, `url`, `accessionNo`, `crd`, `date`. No query at all. |

Tools in the third group include [`float`](./tools/float.md),
[`mapping`](./tools/mapping.md), [`xbrl-to-json`](./tools/xbrl-to-json.md),
[`extractor`](./tools/extractor.md), [`get-edgar-file`](./tools/get-edgar-file.md),
[`form-npx-file`](./tools/form-npx-file.md) and every
`form-adv-schedule-*` tool. Do not send a `query` to them.

## Lucene basics

### field:value

One field, a colon, one value. No space around the colon.

```text
ticker:AAPL
cik:320193
formType:"10-K"
```

Nested fields use a dot path. The path must match the response shape exactly.

```text
issuer.tradingSymbol:AAPL
owners.amountAsPercent:20.3
offeringData.offeringSalesAmounts.totalOfferingAmount:5000000
```

### Quote the value

Quote any value that holds a space, a hyphen or a slash. This is the single most
common cause of wrong results.

| Query | Rows | What comes back |
| ----- | ---- | --------------- |
| `ticker:AAPL AND formType:"10-K"` | 33 | Apple 10-K filings only. |
| `ticker:AAPL AND formType:10-K` | 371 | Wrong. The first rows are 10-Q and 8-K. |

The unquoted form breaks `10-K` into pieces and matches far too much. Always
write `formType:"10-K"`, `formType:"10-K/A"`, `formType:"SC 13D"`.

Quoting is also how you match an exact name.

```text
companyName:"Apple Inc."
name:"Tesla"
auditorName:"Ernst & Young LLP"
```

### Case rules

Two different rules apply, and they trip people up.

- **Field names are case sensitive.** `formtype:"10-K"` returns 0 rows and no
  error. `formType:"10-K"` returns 33.
- **Values are usually not.** `ticker:aapl` and `ticker:AAPL` both return 33.

### AND, OR, NOT

Write the operators in **capital letters**. Lowercase `and` is not an operator.
It becomes a search term, and the query falls back to OR.

| Query | Rows | Meaning |
| ----- | ---- | ------- |
| `ticker:AAPL AND formType:"10-K"` | 33 | Correct. |
| `ticker:AAPL and formType:"10-K"` | 10,000 or more | Wrong. Matches nearly everything. |

Group alternatives with parentheses.

```text
ticker:(AAPL OR MSFT) AND formType:"8-K"
formType:("10-K" OR "10-K/A") AND ticker:TSLA
```

`NOT` removes matches.

```text
formType:"10-K" AND NOT ticker:AAPL
```

### Ranges

Square brackets include the ends. Curly braces exclude them. A `*` leaves one
end open.

```text
filedAt:[2020-01-01 TO 2024-12-31]
filedAt:{2025-10-30 TO 2025-11-01}
filedAt:[2025-01-01 TO *]
owners.amountAsPercent:[10 TO *]
publicOfferingPrice.total:[100000000 TO *]
```

The `>=` shorthand also works. Both of these return the same 130 rows.

```text
ticker:AAPL AND filedAt:>=2025-01-01
ticker:AAPL AND filedAt:[2025-01-01 TO *]
```

Ranges work on dates and on numbers. They do not work on text.

### Wildcards

`*` matches many characters. `?` matches one.

```text
companyName:Apple*
name:Tesla*
```

`field:*` is the idiom for "this field exists and holds a value". It is the way
to find filings that carry a structured block.

```text
item4_01:*
aaerNo:*
genInfo.regLei:*
```

Wildcards fail on hyphenated exact values. `formType:10-K*` returns 0 rows. Use
`formType:"10-K"` and `formType:"10-K/A"` as two terms instead.

[`form-adv-individuals`](./tools/form-adv-individuals.md) also accepts
`_exists_`, which reads better on deep paths.

```text
Info.lastNm:Kim AND _exists_:DRPs.DRP
```

## Confirmed field vocabulary

Every field below returned rows on a live call. A field that is not on this list
may still work. Test it before you trust it, because an unknown field returns
`total: 0` and never an error.

| Tool | Confirmed query fields |
| ---- | ---------------------- |
| [`filing-search`](./tools/filing-search.md) | `ticker`, `formType`, `companyName`, `filedAt`, `periodOfReport` |
| [`form-13f-holdings`](./tools/form-13f-holdings.md) | `cik`, `holdings.ticker` |
| [`form-13f-cover-pages`](./tools/form-13f-cover-pages.md) | `cik`, `periodOfReport` |
| [`form-13d-13g`](./tools/form-13d-13g.md) | `owners.name`, `owners.amountAsPercent` |
| [`insider-trading`](./tools/insider-trading.md) | `issuer.tradingSymbol`, `documentType`, `periodOfReport`, `reportingOwner.relationship.isDirector` |
| [`form-144`](./tools/form-144.md) | `issuerInfo.issuerTicker`, `securitiesInformation.aggregateMarketValue` |
| [`form-nport`](./tools/form-nport.md) | `genInfo.regName`, `genInfo.regLei` |
| [`form-npx`](./tools/form-npx.md) | `cik`, `periodOfReport` |
| [`form-ncen`](./tools/form-ncen.md) | `entities.cik`, `registrantInfo.registrantState` |
| [`form-adv-firms`](./tools/form-adv-firms.md) | `Info.FirmCrdNb`, `Info.BusNm`, `Info.SECNb`, `MainAddr.State`, `Rgstn.FirmType`, `FormInfo.Part1A.Item5F.Q5F2C` |
| [`form-adv-individuals`](./tools/form-adv-individuals.md) | `Info.indvlPK`, `Info.lastNm`, `CrntEmps.CrntEmp.orgPK`, `CrntEmps.CrntEmp.CrntRgstns.CrntRgstn.regAuth`, `Exms.Exm.exmCd`, `_exists_` |
| [`form-s1-424b4`](./tools/form-s1-424b4.md) | `ticker`, `cik`, `entityName`, `formType`, `filedAt`, `accessionNo`, `tickers.exchange`, `underwriters.name`, `auditors.name`, `publicOfferingPrice.total` |
| [`form-d`](./tools/form-d.md) | `primaryIssuer.cik`, `primaryIssuer.entityName`, `primaryIssuer.entityType`, `primaryIssuer.issuerAddress.stateOrCountry`, `submissionType`, `filedAt`, `offeringData.offeringSalesAmounts.totalOfferingAmount`, `offeringData.offeringSalesAmounts.totalAmountSold`, `offeringData.industryGroup.industryGroupType`, `relatedPersonsList.relatedPersonInfo.relatedPersonName.lastName` |
| [`form-c`](./tools/form-c.md) | `cik`, `companyName`, `formType`, `filedAt`, `issuerInformation.companyName`, `issuerInformation.issuerInfo.legalStatus.jurisdictionOrganization`, `offeringInformation.maximumOfferingAmount` |
| [`form-8k`](./tools/form-8k.md) | `ticker`, `cik`, `companyName`, `formType`, `filedAt`, `periodOfReport`, `items`, `item4_01`, `item4_02`, `item5_02` |
| [`reg-a-search`](./tools/reg-a-search.md) | `cik`, `companyName`, `ticker`, `fileNo`, `formType`, `filedAt`, `summaryInfo.indicateTier1Tier2Offering` |
| [`reg-a-form-1a`](./tools/reg-a-form-1a.md) | `cik`, `companyName`, `ticker`, `fileNo`, `formType`, `filedAt`, `summaryInfo.indicateTier1Tier2Offering`, `summaryInfo.financialStatementAuditStatus`, `summaryInfo.totalAggregateOffering`, `issuerInfo.stateOrCountry`, `issuerInfo.nameAuditor`, `employeesInfo.jurisdictionOrganization` |
| [`reg-a-form-1k`](./tools/reg-a-form-1k.md) | `cik`, `companyName`, `formType`, `filedAt`, `periodOfReport`, `item1.fiscalYearEnd`, `item1.stateOrCountry`, `item1Info.jurisdictionOrganization`, `item2.regArule257`, `summaryInfo.issuerNetProceeds` |
| [`reg-a-form-1z`](./tools/reg-a-form-1z.md) | `cik`, `companyName`, `formType`, `filedAt`, `item1.stateOrCountry`, `item1.commissionFileNumber`, `summaryInfoOffering.issuerNetProceeds`, `summaryInfoOffering.auditorSpName`, `certificationSuspension.approxRecordHolders` |
| [`compensation`](./tools/compensation.md) | `ticker`, `year` |
| [`audit-fees`](./tools/audit-fees.md) | `entities.ticker`, `filedAt` |
| [`directors-and-board-members`](./tools/directors-and-board-members.md) | `ticker`, `directors.age` |
| [`subsidiaries`](./tools/subsidiaries.md) | `ticker`, `cik`, `companyName`, `accessionNo`, `filedAt`, `subsidiaries.name`, `subsidiaries.jurisdiction` |
| [`edgar-entities`](./tools/edgar-entities.md) | `cik`, `name`, `sic`, `sicLabel`, `stateOfIncorporation`, `businessAddress.state`, `cfOffice`, `auditorName`, `filerCategory`, `shellCompany`, `formTypes.10-K` |
| [`sec-enforcement-actions`](./tools/sec-enforcement-actions.md) | `releaseNo`, `releasedAt` |
| [`sec-litigation-releases`](./tools/sec-litigation-releases.md) | `releaseNo` |
| [`sec-administrative-proceedings`](./tools/sec-administrative-proceedings.md) | `releaseNo` |
| [`aaers`](./tools/aaers.md) | `aaerNo`, `dateTime` |
| [`sro`](./tools/sro.md) | `sro`, `issueDate` |
| [`form-x-17a-5`](./tools/form-x-17a-5.md) | `entities.cik`, `entities.fileNo`, `formType`, `filedAt`, `periodOfReport`, `submissionInformation.materialWeakness` |

Each tool page carries a longer list that includes unverified names. Treat those
as candidates, not as facts.

## The ticker trap

There is no single ticker field. The name changes per tool, and the wrong name
returns zero rows with no error message.

| Tool | Ticker field |
| ---- | ------------ |
| [`filing-search`](./tools/filing-search.md), [`form-8k`](./tools/form-8k.md), [`subsidiaries`](./tools/subsidiaries.md), [`compensation`](./tools/compensation.md), [`directors-and-board-members`](./tools/directors-and-board-members.md), [`reg-a-search`](./tools/reg-a-search.md), [`form-s1-424b4`](./tools/form-s1-424b4.md) | `ticker` |
| [`audit-fees`](./tools/audit-fees.md) | `entities.ticker` |
| [`insider-trading`](./tools/insider-trading.md) | `issuer.tradingSymbol` |
| [`form-144`](./tools/form-144.md) | `issuerInfo.issuerTicker` |
| [`form-13f-holdings`](./tools/form-13f-holdings.md) | `holdings.ticker` for the held stock, `cik` for the manager |
| [`edgar-entities`](./tools/edgar-entities.md) | **none.** Use `cik` or `name`. |

Measured proof. Each of these returns `total: 0` and HTTP 200:

```text
audit-fees        ticker:NVDA
insider-trading   ticker:AAPL
subsidiaries      entities.ticker:AAPL
filing-search     entities.ticker:AAPL
edgar-entities    ticker:AAPL
```

The correct forms return 22, 1,331, 26 and 33 rows. When a query returns zero,
check the field name first.

If you only have a company name, resolve it with
[`mapping`](./tools/mapping.md) first. Then query on the CIK.

## Dates

### filedAt against periodOfReport

Two date fields exist on most filing tools. They answer different questions.

| Field | Meaning | Use it for |
| ----- | ------- | ---------- |
| `filedAt` | When the filer submitted the document. | "What arrived in March?" |
| `periodOfReport` | The period the document covers. | "What covers fiscal 2024?" |

They differ by weeks or months. Apple's fiscal 2025 10-K has
`periodOfReport` `2025-09-27` and `filedAt` `2025-10-31T06:01:26-04:00`. A
`filedAt` range for 2025 finds it. A `periodOfReport` range for 2025 finds it
too, but the two ranges return different sets across a full year.

### Format

Write plain `YYYY-MM-DD` in a query, even though `filedAt` comes back as a full
ISO 8601 timestamp. A one-day range covers the whole day.

```text
ticker:AAPL AND filedAt:[2025-10-31 TO 2025-10-31]
```

That returns the 10-K filed at 06:01 that morning.

### Time zone

Most handlers run date ranges in `America/New_York`. Send a `time_zone` argument
to change that. [`form-144`](./tools/form-144.md) applies no default time zone.

### Other date field names

Not every tool calls its date `filedAt`.

| Tool | Date field |
| ---- | ---------- |
| [`aaers`](./tools/aaers.md) | `dateTime` |
| [`sec-enforcement-actions`](./tools/sec-enforcement-actions.md), [`sec-litigation-releases`](./tools/sec-litigation-releases.md), [`sec-administrative-proceedings`](./tools/sec-administrative-proceedings.md) | `releasedAt` |
| [`sro`](./tools/sro.md) | `issueDate` |
| [`form-13d-13g`](./tools/form-13d-13g.md) | `filedAt`, and `eventDate` for the trigger |
| [`form-144`](./tools/form-144.md) | `filedAt`, and `securitiesInformation.approxSaleDate` for the planned sale |

## Sorting and paging

`sort` takes an Elasticsearch sort array. One object per field.

```json
{
  "name": "filing-search",
  "arguments": {
    "query": "ticker:AAPL AND formType:\"10-K\"",
    "size": 5,
    "sort": [{ "periodOfReport": { "order": "asc" } }]
  }
}
```

Sort on date and numeric fields. Confirmed sort fields include `filedAt`,
`periodOfReport` and `FormInfo.Part1A.Item5F.Q5F2C`.

### Default sort

Every search tool sorts newest first by default. The field differs.

| Default sort | Tools |
| ------------ | ----- |
| `filedAt` descending | `filing-search`, `form-8k`, `form-c`, `form-d`, `form-ncen`, `form-nport`, `form-npx`, `form-s1-424b4`, `form-x-17a-5`, `reg-a-search`, `reg-a-form-1a`, `reg-a-form-1k`, `reg-a-form-1z`, `subsidiaries`, `form-13d-13g`, `form-13f-holdings`, `form-13f-cover-pages`, `insider-trading`, `form-144` |
| `releasedAt` descending | `sec-enforcement-actions`, `sec-litigation-releases`, `sec-administrative-proceedings` |
| `dateTime` descending | `aaers` |
| `issueDate` descending | `sro` |
| `cikUpdatedAt` descending | `edgar-entities` |
| `Info.FirmCrdNb` descending | `form-adv-firms` |
| `Info.indvlPK` descending | `form-adv-individuals` |
| relevance, not sortable | `full-text-search` |

### Paging

`size` runs from 1 to 50 and defaults to 50. Above 50 the server returns an
error:

```text
sec-api error: Maximum 'size' limit of 50 exceeded. Please adjust 'size' to 50 or less.
```

`from` is the offset. The search window stops near 10,000 rows. To go deeper,
split the query by date range instead of raising `from`.

### Read total.value with care

`total` is an object, not a number. A value of exactly `10000` with
`relation: "gte"` is the search-window ceiling. Read it as "10,000 or more" and
narrow the query.

```json
{ "total": { "value": 10000, "relation": "gte" } }
```

## full-text-search is different

[`full-text-search`](./tools/full-text-search.md) does not accept Lucene fields.
`ticker:AAPL` there returns 0 rows.

| Feature | Lucene tools | `full-text-search` |
| ------- | ------------ | ------------------ |
| Field syntax | yes | no |
| Company filter | `ticker:` or `cik:` in the query | the `ciks` parameter |
| Form filter | `formType:` in the query | the `formTypes` parameter |
| Date filter | `filedAt:[a TO b]` | `startDate` plus `endDate` |
| Sort | `sort` array, date by default | relevance only, no `sort` |
| Page size | `size`, 1 to 50 | fixed at 100 |
| Paging | `from` | `page`, 1-based |

Quote a phrase. `OR` between phrases works.

```json
{
  "name": "full-text-search",
  "arguments": {
    "query": "\"quantum computing\" OR \"quantum annealing\"",
    "formTypes": ["8-K"]
  }
}
```

Two more rules. Set `startDate` and `endDate` together, or the server silently
searches the last 12 months only. Rows are documents, not filings, so one filing
with four exhibits can produce five rows.

## Ten worked examples

Counts come from live calls on 2026-08-13.

### 1. One company, one form type

```json
{ "name": "filing-search", "arguments": { "query": "ticker:AAPL AND formType:\"10-K\"", "size": 1 } }
```

33 filings.

### 2. Add a date range

```json
{ "name": "filing-search", "arguments": { "query": "ticker:AAPL AND formType:\"10-K\" AND filedAt:[2020-01-01 TO 2024-12-31]", "size": 5 } }
```

5 filings.

### 3. Two companies with OR

```json
{ "name": "filing-search", "arguments": { "query": "ticker:(AAPL OR MSFT) AND formType:\"8-K\"", "size": 10 } }
```

520 filings.

### 4. Exclude one company

```json
{ "name": "filing-search", "arguments": { "query": "formType:\"10-K\" AND NOT ticker:AAPL", "size": 10 } }
```

10,000 or more. The ceiling means the query is too broad. Add a date range.

### 5. Two attribute filters

```json
{ "name": "edgar-entities", "arguments": { "query": "sic:3571 AND stateOfIncorporation:DE", "size": 10 } }
```

43 companies. Computer makers incorporated in Delaware.

### 6. A different ticker field

```json
{ "name": "audit-fees", "arguments": { "query": "entities.ticker:NVDA AND filedAt:[2020-01-01 TO 2026-12-31]", "size": 10 } }
```

7 filings. `ticker:NVDA` here returns 0.

### 7. A nested field plus a form code

```json
{ "name": "insider-trading", "arguments": { "query": "issuer.tradingSymbol:AAPL AND documentType:4", "size": 10 } }
```

1,331 Form 4 filings.

### 8. Field exists, inside a date range

```json
{ "name": "form-8k", "arguments": { "query": "item4_01:* AND filedAt:[2024-01-01 TO 2024-12-31]", "size": 10 } }
```

1,003 auditor-change 8-K filings from 2024. `item4_01:*` means "this filing
carries a parsed Item 4.01 block".

### 9. Text match plus a numeric range

```json
{ "name": "form-13d-13g", "arguments": { "query": "owners.name:Point72 AND owners.amountAsPercent:[10 TO *]", "size": 10 } }
```

8 filings. Point72 positions of 10% or more.

### 10. Deep paths, a range, and a sort

```json
{
  "name": "form-adv-firms",
  "arguments": {
    "query": "MainAddr.State:NY AND FormInfo.Part1A.Item5F.Q5F2C:[100000000000 TO *]",
    "size": 10,
    "sort": [{ "FormInfo.Part1A.Item5F.Q5F2C": { "order": "desc" } }]
  }
}
```

63 New York advisers with $100 billion or more in regulatory assets, largest
first. Item 5 values are self-reported. Check them before you rank on them.

## Common mistakes

| Mistake | What happens | Fix |
| ------- | ------------ | --- |
| Lowercase `and`, `or`, `not` | The word becomes a search term. `ticker:AAPL and formType:"10-K"` returns 10,000 or more. | Write `AND`, `OR`, `NOT` in capitals. |
| Unquoted form type | `formType:10-K` returns 371 rows, mostly 10-Q and 8-K. | Write `formType:"10-K"`. |
| Wrong ticker field | `total: 0`, HTTP 200, no error. | Check the ticker table above. |
| Wrong field name case | `formtype:"10-K"` returns 0. | Field names are case sensitive. Copy them exactly. |
| Invented field name | `bogusField:xyz` returns 0, never an error. | An empty result is a spelling signal. Check the field list. |
| No colon in the query | Some tools reject it with HTTP 400 `Invalid Lucene query string`. `filing-search` accepts it and returns junk. A bare `AAPL` returns 13F filings. | Always write `field:value`. |
| `size` above 50 | HTTP 400. | Keep `size` at 50 or less. Page with `from`. |
| Reading `total.value` 10000 as a count | You under-report by an unknown amount. | Check `total.relation`. `gte` means "or more". |
| Lucene syntax in `full-text-search` | `ticker:AAPL` returns 0 rows. | Use `ciks`, `formTypes`, `startDate` and `endDate`. |
| Wildcard on a hyphenated value | `formType:10-K*` returns 0. | List the values: `formType:("10-K" OR "10-K/A")`. |
| Mixing up `filedAt` and `periodOfReport` | You get the right filings from the wrong year. | `filedAt` is the submission date. `periodOfReport` is the covered period. |
| Expecting a nested match to filter the array | [`subsidiaries`](./tools/subsidiaries.md) returns the whole parent row, with every sibling in it. | Filter the inner array yourself after the call. |
| Bare `cik` on [`form-d`](./tools/form-d.md) | `cik:*` returns 0. | Use `primaryIssuer.cik`, zero-padded to 10 digits. |

### Query length and the colon rule

| Tool | Maximum query length | Colon required |
| ---- | -------------------- | -------------- |
| `filing-search` | 3,500 | no |
| `insider-trading`, `form-nport` | 2,000 | yes |
| `full-text-search` | 2,000 | not applicable |
| `form-13f-holdings`, `form-13f-cover-pages`, `form-13d-13g`, `form-8k`, `form-s1-424b4`, `form-npx`, `edgar-entities`, `subsidiaries`, `aaers`, `sro`, `sec-enforcement-actions`, `sec-litigation-releases`, `sec-administrative-proceedings` | 1,000 | yes, except `form-13d-13g` |
| `form-144`, `form-c`, `form-d`, `form-ncen` | not enforced | no |

A tool that requires a colon returns HTTP 400 without one. A tool that does not
require one will run your bare term and return noise. Write `field:value`
everywhere and the difference stops mattering.

## Test before you loop

Send `size: 1` first. Read `total.value`. Only then raise `size` or add a
`from`. This costs one cheap call and stops a broad query from filling your
context window.

## Related

- [Tool reference](./tools/README.md). The field list and the response shape for each tool.
- [Response format](./response-format.md). The ten different envelopes.
- [Limits and errors](./limits-and-errors.md). Paging ceilings and error text.
- [sec-api.io Query API docs](https://sec-api.io/docs/query-api)
