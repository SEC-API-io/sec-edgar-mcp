# aaers

Search Accounting and Auditing Enforcement Releases, the SEC actions about
accounting fraud, audit failures and financial reporting misconduct.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Enforcement                                  |
| Required input  | `query`                                      |
| Returns         | `{total: {value, relation}, data[]}`         |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST https://api.sec-api.io/aaers`          |

## What it does

`aaers` searches the AAER series. An AAER is an enforcement document the SEC
also publishes under an `AAER-xxxx` number because it concerns accounting or
auditing. One row is one AAER document. Each row carries the `respondents`, a
`summary`, the `tags`, the `violatedSections` and any `penaltyAmounts`.
Coverage starts in 1997 and runs to the present. New AAERs arrive as the SEC
publishes them. A request for `aaerNo:*` returned `total.value: 3326`, so the
index held 3,326 AAERs on 2026-08-13.

Field shapes differ from the sibling tools. The date field is `dateTime`, not
`releasedAt`. Documents are in `urls[]`, not `resources[]`. There is no `title`
field. Use `respondentsText` and `summary` instead.

## When to use it

- Which auditors did the SEC sanction for a failed audit?
- Has this company ever been charged with accounting fraud?
- Which AAERs cite Rule 102(e) of the Commission's Rules of Practice?
- What penalty did an audit firm agree to pay?
- Which AAER covers a given respondent, and where is the PDF?

## When to use a different tool

| Situation                                                 | Better tool                                                             | Why                                                                                       |
| --------------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| You want all in-house SEC cases, not only accounting ones | [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | Most AAERs are also administrative proceedings, with a file number and an `orders` field. |
| The SEC sued in federal court                             | [`sec-litigation-releases`](./sec-litigation-releases.md)               | Court cases carry a citation and a docket number.                                         |
| You want the announced, newsworthy actions                | [`sec-enforcement-actions`](./sec-enforcement-actions.md)               | That tool searches press releases.                                                        |
| You want to know who audits a company and for what fee    | [`audit-fees`](./audit-fees.md)                                         | Audit fees come from the proxy statement, not from enforcement.                           |
| You need the AAER document itself                         | [`aaer-file`](./aaer-file.md)                                           | It fetches the file.                                                                      |

## Input

| Parameter | Type    | Required | Constraints                                             | Notes                                                |
| --------- | ------- | -------- | ------------------------------------------------------- | ---------------------------------------------------- |
| `query`   | string  | yes      | Lucene syntax, max 1,000 characters, must contain a `:` | Example: `aaerNo:*`.                                 |
| `from`    | integer | no       | 0 to 10,000                                             | Offset of the first row. Default 0.                  |
| `size`    | integer | no       | 1 to 50                                                 | Rows per call. Default 50. Over 50 returns an error. |
| `sort`    | array   | no       | Elasticsearch sort array                                | Default `[{"dateTime": {"order": "desc"}}]`.         |


Query fields: `aaerNo`, `dateTime`, `releaseNo`, `respondents.name`,
`respondents.type`, `entities.name`, `entities.cik`, `entities.ticker`,
`entities.role`, `tags`, `complaints`, `violatedSections`,
`hasAgreedToSettlement`, `hasAgreedToPayPenalty`,
`penaltyAmounts.penaltyAmount`, `penaltyAmounts.imposedOn`,
`parallelActionsTakenBy`, `otherAgenciesInvolved.country` and `urls.type`.
Other names in the Output table below are response fields. A date range looks
like `dateTime:[2020-01-01 TO 2024-12-31]`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

The tables below list every field the response can hold.

### Envelope

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of AAERs that match the query. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[]` | object[] | One object per AAER. |

### Release

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].id` | string | Unique system-internal identifier of the AAER. |
| `data[].aaerNo` | string | AAER release number, for example `AAER-4597`. |
| `data[].dateTime` | string | Release date and time of the AAER, format `yyyy-MM-ddTHH:mm:ssXXX`. The date field for this tool. |
| `data[].releaseNo[]` | string[] | Other release numbers tied to the AAER, such as litigation release numbers. For example `33-11432` and `34-105966`. |
| `data[].summary` | string | Brief summary of the AAER. |
| `data[].tags[]` | string[] | Tags that describe the AAER, for example `settlement` or `officer and director bar`. |

### Respondents and parties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].respondents[]` | object[] | The parties the SEC named in the AAER. |
| `data[].respondents[].name` | string | Name of the respondent. |
| `data[].respondents[].type` | string | Type of the respondent, such as `individual` or `company`. |
| `data[].respondentsText` | string | The respondent line as the SEC prints it. It can carry a note such as `(Order Granting Extension of Time)`. |
| `data[].entities[]` | object[] | Every person and organisation the action names, not only the respondents. |
| `data[].entities[].name` | string | Name of the entity. |
| `data[].entities[].type` | string | Type of the entity, such as `company`, `individual` or `other`. |
| `data[].entities[].role` | string | Role of the entity in the AAER, such as `respondent`, `defendant`, `plaintiff`, `involved party`, `employer` or `entity audited`. |
| `data[].entities[].cik` | string | Central Index Key of the entity. Set only when the name matches a known public company. |
| `data[].entities[].ticker` | string | Ticker symbol of the entity. Set only when the name matches a known public company. |

### Charges and penalties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].complaints[]` | string[] | The SEC complaints tied to the AAER. One sentence each. Empty for procedural orders. |
| `data[].violatedSections[]` | string[] | The securities laws and rules the parties violated. Empty for procedural orders. |
| `data[].requestedRelief[]` | string[] | The relief requested, such as `disgorgement`, `cease and desist order` or `permanent officer and director bar`. |
| `data[].hasAgreedToSettlement` | boolean | True when the respondent has agreed to a settlement. |
| `data[].hasAgreedToPayPenalty` | boolean | True when the respondent has agreed to pay a penalty. |
| `data[].penaltyAmounts[]` | object[] | One object per penalty imposed on a respondent. |
| `data[].penaltyAmounts[].penaltyAmount` | string | Amount of the penalty. Digits only, for example `75000`. |
| `data[].penaltyAmounts[].penaltyAmountText` | string | The penalty amount as the document states it, for example `$75,000`. |
| `data[].penaltyAmounts[].imposedOn` | string | The party that must pay the penalty. |

### Documents

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].urls[]` | object[] | Links to the documents of the AAER. |
| `data[].urls[].type` | string | Class of the document. The main proceeding has the type `primary` and is always present. Other classes are the administrative summary, the SEC complaint, a judgement, an order, a press release, an administrative proceedings release and a litigation release. |
| `data[].urls[].url` | string | Direct link to the document on sec.gov. |

### Who else acted

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].parallelActionsTakenBy[]` | string[] | Other regulators or organisations that took a parallel action, for example `Tokyo District Court`. |
| `data[].otherAgenciesInvolved[]` | object[] | Other regulators or organisations that took part in the enforcement action. Filled only when another agency took part. |
| `data[].otherAgenciesInvolved[].name` | string | Name of the agency, for example `Brazilian Ministerio Publico Federal`. |
| `data[].otherAgenciesInvolved[].country` | string | Country of the agency. |

Size behaviour. `size` defaults to 50 and cannot exceed 50. Page with `from`.
This example set `size: 1` and returned 1,110 bytes for one row, so a `size: 50`
call can reach about 55 KB.

## Example

Prompt: "Show me the newest AAER and link its order."

```json
{ "name": "aaers", "arguments": { "query": "aaerNo:*", "size": 1 } }
```

```json
{
  "total": { "value": 3326, "relation": "eq" },
  "data": [
    {
      "dateTime": "2026-07-22T13:35:49-04:00",
      "aaerNo": "AAER-4597",
      "releaseNo": ["33-11432", "34-105966"],
      "respondents": [
        { "name": "L&L Energy, Inc.", "type": "company" },
        { "name": "Dickson Lee, CPA", "type": "individual" }
      ],
      "respondentsText": "L&L Energy, Inc. and Dickson Lee, CPA (Order Granting Extension of Time)",
      "urls": [
        {
          "type": "primary",
          "url": "https://www.sec.gov/files/litigation/opinions/2026/33-11432.pdf"
        }
      ],
      "summary": "The SEC has granted an extension of time for the Division of Enforcement to respond to Dickson Lee's request to vacate a permanent officer and director bar imposed in a previous settlement.",
      "tags": ["settlement", "officer and director bar"],
      "entities": [
        { "name": "Dickson Lee", "type": "individual", "role": "respondent" }
      ],
      "complaints": [],
      "hasAgreedToSettlement": true,
      "requestedRelief": ["permanent officer and director bar"]
    }
  ]
}
```

## Limits and errors

- A query without a `:` fails with HTTP 400 `Invalid Lucene query string`.
- A query over 1,000 characters fails with
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- Sorting by `releasedAt` returns nothing useful. The date field here is
  `dateTime`.
- Not every AAER is a charging document. The newest row in the example is an
  order granting an extension of time, with empty `complaints` and empty
  `violatedSections`. Filter on those fields when you want charges only.
- To read the document, follow `urls[].url` to sec.gov.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`aaer-file`](./aaer-file.md). Fetch the AAER document itself.
- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). The wider in-house case set.
- [`sec-litigation-releases`](./sec-litigation-releases.md). Civil cases filed in federal court.
- [`audit-fees`](./audit-fees.md). Who audits a company, and for what fee.
- [sec-api.io AAER Database API docs](https://sec-api.io/docs/aaer-database-api)
