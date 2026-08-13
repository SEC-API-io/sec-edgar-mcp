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
`complaints`, the statutes in `violatedSections` and any `penaltyAmounts`.
Coverage starts in 1995 and runs to the present, with more than 10,000
releases. New releases arrive as the SEC publishes them. A request for
`releaseNo:*` returns `total.value: 10000` with `relation: "gte"`, so the index
holds 10,000 releases or more. That number is a floor, not a count.

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
| `from` | integer | no | 0 to 10,000 | Offset of the first row. Default 0. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"releasedAt": {"order": "desc"}}]`. |


Query fields: `releaseNo`, `releasedAt`, `entities.cik`, `entities.ticker`,
`entities.role`, `entities.type`, `tags`, `violatedSections`,
`requestedRelief`, `hasAgreedToSettlement`, `hasAgreedToPayPenalty`,
`penaltyAmounts.penaltyAmount`, `penaltyAmounts.imposedOn`,
`investigationConductedBy`, `litigationLedBy`, `parallelActionsTakenBy` and
`otherAgenciesInvolved.country`. Other names in the Output table below are
response fields. A date range looks like
`releasedAt:[2024-01-01 TO 2024-12-31]`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

The tables below list every field the response can hold.

### Envelope

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of releases that match the query. `10000` means 10,000 or more. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[]` | array | One object per litigation release. |

### Release

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].id` | string | Internal unique identifier of the enforcement action. |
| `data[].releaseNo` | string | SEC release number of the litigation, for example `LR-26219`. |
| `data[].releasedAt` | string | Publication date and time of the litigation release, format `yyyy-MM-ddTHH:mm:ssXXX`. |
| `data[].url` | string | URL of the original SEC litigation release. |
| `data[].title` | string | Title of the litigation release. Often only the lead defendant. |
| `data[].subTitle` | string | Sub title of the litigation release. This holds the descriptive headline. |
| `data[].caseCitations[]` | string[] | The case citations of the litigation release. Each holds the court caption, the docket number and the court abbreviation. |
| `data[].summary` | string | Brief summary of the litigation. |
| `data[].tags[]` | string[] | Tags for the litigation, such as `bribery` or `insider trading`. |

### Parties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].entities[]` | object[] | The parties involved in the litigation. |
| `data[].entities[].name` | string | Name of the party involved. |
| `data[].entities[].type` | string | Type of the party, such as `individual`, `company` or `other`. |
| `data[].entities[].role` | string | Role of the party, such as `respondent`, `defendant`, `affected entity` or `other`. |
| `data[].entities[].cik` | string | Central Index Key of the party, if available. |
| `data[].entities[].ticker` | string | Ticker symbol of the party, if available. |

### Charges, relief and penalties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].complaints[]` | string[] | The complaints or charges in the case. One sentence each. |
| `data[].violatedSections[]` | string[] | The securities laws the defendants violated. |
| `data[].requestedRelief[]` | string[] | The relief the SEC requests, such as `disgorgement of profits`, `injunction` or `civil penalty`. |
| `data[].hasAgreedToSettlement` | boolean | True when the defendant has agreed to a settlement. |
| `data[].hasAgreedToPayPenalty` | boolean | True when the defendant has agreed to pay a penalty. |
| `data[].penaltyAmounts[]` | object[] | One object per penalty in the release. |
| `data[].penaltyAmounts[].penaltyAmount` | string | The cleaned penalty amount in USD. Digits only, no currency sign and no separators. |
| `data[].penaltyAmounts[].penaltyAmountText` | string | The penalty amount as stated in the release, for example `$18,668`. |
| `data[].penaltyAmounts[].imposedOn` | string | The party on which the penalty was imposed. |

### Documents

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].resources[]` | object[] | Links to related documents, such as the complaint or the judgment. |
| `data[].resources[].label` | string | Name of the related document, for example `SEC Complaint` or `Judgment`. |
| `data[].resources[].url` | string | Direct link to the document. |

### Who acted

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].investigationConductedBy[]` | string[] | The persons or entities that conducted the investigation. Holds staff names, or an office. |
| `data[].litigationLedBy[]` | string[] | The persons or entities that litigated the case. |
| `data[].parallelActionsTakenBy[]` | string[] | Other agencies that took a parallel action, such as `U.S. Department of Justice` for criminal charges. |
| `data[].otherAgenciesInvolved[]` | object[] | Other agencies involved in the investigation or the litigation, for example FINRA. |
| `data[].otherAgenciesInvolved[].name` | string | Name of the agency. |
| `data[].otherAgenciesInvolved[].country` | string | Country of the agency. |

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
