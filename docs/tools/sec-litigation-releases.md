# sec-litigation-releases

Search SEC litigation releases, the public notices of civil lawsuits the SEC
files in federal court.

|                 |                                                         |
| --------------- | ------------------------------------------------------- |
| Category        | Enforcement                                             |
| Required input  | `query`                                                 |
| Returns         | `{total: {value, relation}, data[]}`                    |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`            |
| REST equivalent | `POST https://api.sec-api.io/sec-litigation-releases`   |

## What it does

`sec-litigation-releases` searches the SEC litigation release archive. One row
is one release, identified by a number such as `LR-26609`. Each row carries the
court `caseCitations`, the parties in `entities`, the allegations in
`complaints`, the statutes in `violatedSections` and any `penaltyAmounts`. A
request for `releaseNo:*` returns `total.value: 10000` with `relation: "gte"`,
so the index holds 10,000 releases or more. That number is a floor, not a
count.

A litigation release is the SEC's own notice of a court filing or a court
outcome. It is not the complaint itself. `resources[]` links the complaint PDF.

## When to use it

- Which civil lawsuits did the SEC file in federal court last month?
- Has the SEC sued this person, and in which district?
- Which insider trading cases settled, and for how much?
- What is the case citation and docket number for an SEC action?

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| The case was decided inside the SEC, not in court | [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | Administrative proceedings carry a `3-xxxxx` file number and an order, not a citation. |
| You want only the actions the SEC announced publicly | [`sec-enforcement-actions`](./sec-enforcement-actions.md) | That tool searches press releases, a smaller and more newsworthy set. |
| The subject is accounting fraud or an audit failure | [`aaers`](./aaers.md) | AAERs are the accounting and auditing subset, numbered separately. |
| You need the trades behind an insider trading case | [`insider-trading`](./insider-trading.md) | Form 4 data shows the transactions. The release only describes them. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `query` | string | yes | Lucene syntax, max 1,000 characters, must contain a `:` | Example: `releaseNo:*`. |
| `from` | integer | no | minimum 0 | Offset of the first row. Default 0. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"releasedAt": {"order": "desc"}}]`. |


Query fields: `releaseNo`, `releasedAt`, `tags`, `caseCitations`,
`entities.name`, `entities.ticker`, `violatedSections` and
`hasAgreedToSettlement`. Every other name in the Output table below is a
response field. The sec-api REST docs treat those as searchable too. A date
range looks like `releasedAt:[2024-01-01 TO 2024-12-31]`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of matching releases. `10000` means 10,000 or more. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[].id` | string | Internal record hash. |
| `data[].releaseNo` | string | Release number, for example `LR-26609`. |
| `data[].releasedAt` | string | Publication timestamp, ISO 8601. |
| `data[].url` | string | URL of the release on sec.gov. |
| `data[].title` | string | Case name, often just the lead defendant. |
| `data[].subTitle` | string | The descriptive headline. `title` is often only a party name. |
| `data[].caseCitations[]` | array | Full court citation with court, docket number and filing date. |
| `data[].summary` | string | One-paragraph summary of the action. |
| `data[].tags[]` | array | Short subject labels, for example `insider trading`. |
| `data[].entities[]` | array | Parties. Each has `name`, `type`, `role`, and `cik` and `ticker` when known. |
| `data[].complaints[]` | array | One sentence per allegation. |
| `data[].violatedSections[]` | array | Statutes and rules the SEC says were violated. |
| `data[].requestedRelief[]` | array | Relief the SEC asks for, such as `permanent injunctions`. |
| `data[].hasAgreedToSettlement`, `data[].hasAgreedToPayPenalty` | boolean | True when the release states a settlement, or a penalty payment. |
| `data[].penaltyAmounts[]` | array | Each has `penaltyAmount` (a numeric string), `penaltyAmountText` and `imposedOn`. |
| `data[].resources[]` | array | Attached documents. Each has `label` and `url`. |
| `data[].investigationConductedBy[]` | array | Staff who ran the investigation. |
| `data[].otherAgenciesInvolved[]` | array | Each has `name` and `country`, for example FINRA. |

Two more arrays name who else acted: `parallelActionsTakenBy` and
`litigationLedBy`.

Size behaviour. `size` defaults to 50 and cannot exceed 50. `from` moves the
window to later rows. This example set `size: 1` and returned 2,010 bytes for
one row, so a `size: 50` call can reach about 100 KB.

## Example

Prompt: "What is the newest SEC litigation release, and did the defendant
settle?"

```json
{ "name": "sec-litigation-releases", "arguments": { "query": "releaseNo:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "releaseNo": "LR-26609",
      "releasedAt": "2026-08-11T21:11:14Z",
      "url": "https://www.sec.gov/enforcement-litigation/litigation-releases/lr-26609",
      "title": "Benjamin Tesfaye",
      "subTitle": "SEC Files Settled Action as to Texas Resident Charged with Insider Trading",
      "caseCitations": ["Securities and Exchange Commission v. Benjamin Tesfaye, No. 3:26-cv-02660-K (N.D. Tex. filed Aug. 11, 2026)"],
      "tags": ["insider trading"],
      "entities": [
        { "name": "Benjamin Tesfaye", "type": "individual", "role": "defendant" },
        { "name": "Calliditas Therapeutics AB", "type": "company", "role": "target of insider trading", "cik": "1795579", "ticker": "CALT" }
      ],
      "hasAgreedToSettlement": true,
      "hasAgreedToPayPenalty": true,
      "penaltyAmounts": [
        { "penaltyAmount": "18668", "penaltyAmountText": "$18,668", "imposedOn": "Benjamin Tesfaye" }
      ],
      "violatedSections": ["Sections 10(b) and 14(e) of the Securities Exchange Act of 1934", "Rules 10b-5 and 14e-3"]
    }
  ]
}
```

## Limits and errors

- `total.value: 10000` with `relation: "gte"` is the search-window ceiling. It
  means 10,000 or more. A date range narrows the query enough for an exact
  count.
- A query without a `:` fails with HTTP 400 `Invalid Lucene query string`.
- A query over 1,000 characters fails with
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- `title` is often only a party name. `subTitle` holds the headline.
- `entities[].role` is free text set per case. Values include `target of insider
  trading` and `acquirer` next to `defendant`. The set of roles is not fixed.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`sec-enforcement-actions`](./sec-enforcement-actions.md). Announced actions, with press release text.
- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). Cases decided inside the SEC.
- [`aaers`](./aaers.md). The accounting and auditing subset.
- [Query language](../query-language.md). Lucene syntax and field names.
- [sec-api.io Litigation Releases API docs](https://sec-api.io/docs/sec-litigation-releases-database-api)
