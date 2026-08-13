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
`summary`, the `tags`, the `violatedSections` and any `penaltyAmounts`. The
capture returned `total.value: 3326` for `aaerNo:*`, so the index held 3,326
AAERs on 2026-08-13.

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

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| You want all in-house SEC cases, not only accounting ones | [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | Most AAERs are also administrative proceedings, with a file number and an `orders` field. |
| The SEC sued in federal court | [`sec-litigation-releases`](./sec-litigation-releases.md) | Court cases carry a citation and a docket number. |
| You want the announced, newsworthy actions | [`sec-enforcement-actions`](./sec-enforcement-actions.md) | That tool searches press releases. |
| You want to know who audits a company and for what fee | [`audit-fees`](./audit-fees.md) | Audit fees come from the proxy statement, not from enforcement. |
| You need the AAER document itself | [`aaer-file`](./aaer-file.md) | It fetches the file. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `query` | string | yes | Lucene syntax, max 1,000 characters, must contain a `:` | Example: `aaerNo:*`. |
| `from` | integer | no | minimum 0 | Offset of the first row. Default 0. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"dateTime": {"order": "desc"}}]`. |

The schema sets `additionalProperties: true`. The handler reads one extra key,
`time_zone`, and applies it to date ranges in the query. Unverified.

Query field verified by a live call: `aaerNo`.

Every other name in the Output table below is a response field. The sec-api REST
docs treat those as searchable too, but none was verified through MCP. Treat
them as unverified, including `dateTime`, `releaseNo`, `respondents.name`,
`respondentsText`, `tags` and `violatedSections`. The sec-api Node SDK uses
`dateTime:[2020-01-01 TO 2024-12-31]`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of matching AAERs. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[].id` | string | Internal record hash. |
| `data[].aaerNo` | string | AAER number, for example `AAER-4597`. |
| `data[].dateTime` | string | Release timestamp, ISO 8601 with an offset. The date field for this tool. |
| `data[].releaseNo[]` | array | Related release numbers, for example `33-11432` and `34-105966`. |
| `data[].respondents[]` | array | Respondents. Each has `name` and `type`. The capture had no `role` here. |
| `data[].respondentsText` | string | The respondent line as printed, including notes such as `(Order Granting Extension of Time)`. |
| `data[].urls[]` | array | Documents. Each has `type` and `url`. The main document has the type `primary`. |
| `data[].summary` | string | One-paragraph summary of the release. |
| `data[].tags[]` | array | Short subject labels, for example `settlement`, `officer and director bar`. |
| `data[].entities[]` | array | All parties, with a `role` such as `respondent` or `entity audited`. |
| `data[].complaints[]` | array | One sentence per allegation. Empty for procedural orders. |
| `data[].violatedSections[]` | array | Statutes and rules cited. Empty for procedural orders. |
| `data[].requestedRelief[]` | array | Relief sought or imposed, such as `permanent officer and director bar`. |
| `data[].hasAgreedToSettlement`, `data[].hasAgreedToPayPenalty` | boolean | True when the document states a settlement, or a penalty payment. |
| `data[].penaltyAmounts[]` | array | Each has `penaltyAmount` (a numeric string), `penaltyAmountText` and `imposedOn`. |
| `data[].parallelActionsTakenBy[]` | array | Other bodies that acted in parallel. |
| `data[].otherAgenciesInvolved[]` | array | Each has `name` and `country`. |

Size behaviour. `size` defaults to 50 and cannot exceed 50. Page with `from`.
The default of 50 comes from the server handler. The capture set `size: 1` and
returned 1,110 bytes for one row, so a `size: 50` call can reach about 55 KB.

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
        { "type": "primary", "url": "https://www.sec.gov/files/litigation/opinions/2026/33-11432.pdf" }
      ],
      "summary": "The SEC has granted an extension of time for the Division of Enforcement to respond to Dickson Lee's request to vacate a permanent officer and director bar imposed in a previous settlement.",
      "tags": ["settlement", "officer and director bar"],
      "entities": [{ "name": "Dickson Lee", "type": "individual", "role": "respondent" }],
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
- Not every AAER is a charging document. The capture's newest row is an order
  granting an extension of time, with empty `complaints` and empty
  `violatedSections`. Filter on those fields when you want charges only.
- To read the document, follow `urls[].url` to sec.gov.
- The error texts above come from the server handler. The capture did not
  trigger them.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`aaer-file`](./aaer-file.md). Fetch the AAER document itself.
- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). The wider in-house case set.
- [`sec-litigation-releases`](./sec-litigation-releases.md). Civil cases filed in federal court.
- [`audit-fees`](./audit-fees.md). Who audits a company, and for what fee.
- [sec-api.io AAER Database API docs](https://sec-api.io/docs/aaer-database-api)
