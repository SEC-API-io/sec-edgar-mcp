# sec-administrative-proceedings

Search SEC administrative proceedings, the cases the Commission decides in
house rather than in federal court.

|                 |                                                              |
| --------------- | ------------------------------------------------------------ |
| Category        | Enforcement                                                   |
| Required input  | `query`                                                       |
| Returns         | `{total: {value, relation}, data[]}`                          |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                  |
| REST equivalent | `POST https://api.sec-api.io/sec-administrative-proceedings`  |

## What it does

`sec-administrative-proceedings` searches SEC administrative orders and
notices. One row is one released document, not one case. A case with an order
instituting proceedings, a settlement and a later distribution order produces
several rows that share a file number such as `3-19038`. Each row carries the
`respondents`, the `complaints`, the `violatedSections` and the `orders` the
Commission issued. Coverage starts in 1995 and runs to the present, with more
than 18,000 proceedings. New documents arrive as the SEC publishes them. A
request for `releaseNo:*` returns `total.value: 10000` with `relation: "gte"`,
so the index holds 10,000 documents or more.

Field shapes differ from the sibling tools. `releaseNo` is an **array** here,
not a string. There is no `url` field. Get the document from `resources[].url`.

## When to use it

- What did the SEC order against this adviser, and when?
- Which firms were barred or suspended from practising before the Commission?
- Which proceedings belong to file number `3-19038`?
- Which advisers were charged over 12b-1 fees and share class selection?
- What happened to the money in a disgorgement fund?

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| The SEC sued in federal court | [`sec-litigation-releases`](./sec-litigation-releases.md) | Court actions get a litigation release and a case citation, not an order. |
| You want only the actions the SEC announced publicly | [`sec-enforcement-actions`](./sec-enforcement-actions.md) | That tool searches press releases, a smaller and more newsworthy set. |
| The order is about accounting or auditing | [`aaers`](./aaers.md) | The same order is also published as an AAER, with an `AAER-xxxx` number. |
| You need the adviser's registration record | [`form-adv-firms`](./form-adv-firms.md) | Form ADV holds the disclosure and registration data, keyed by CRD. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `query` | string | yes | Lucene syntax, max 1,000 characters, must contain a `:` | Example: `releaseNo:*`. |
| `from` | integer | no | 0 to 10,000 | Offset of the first row. Default 0. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"releasedAt": {"order": "desc"}}]`. |


Query fields: `releaseNo`, `releasedAt`, `fileNumbers`, `respondents.type`,
`respondents.role`, `entities.name`, `entities.cik`, `entities.ticker`, `tags`,
`complaints`, `orders`, `violatedSections`, `requestedRelief`,
`hasAgreedToSettlement`, `hasAgreedToPayPenalty`, `penaltyAmounts`,
`parallelActionsTakenBy` and `otherAgenciesInvolved`. Other names in the Output
table below are response fields. A date range looks like
`releasedAt:[2024-01-01 TO 2024-12-31]`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

The tables below list every field the response can hold.

### Envelope

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of documents that match the query. `10000` means 10,000 or more. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[]` | array | One object per released document. |

### Proceeding

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].id` | string | Internal unique identifier of the administrative proceeding. |
| `data[].releaseNo[]` | string[] | The SEC release numbers of the proceeding, for example `34-106124`. An AAER release number is listed here too, if the SEC issued one. |
| `data[].fileNumbers[]` | string[] | The file numbers of the proceeding, for example `3-19038`. This is the case key. |
| `data[].releasedAt` | string | Publication date and time of the proceeding, format `yyyy-MM-ddTHH:mm:ssXXX`. |
| `data[].title` | string | Title of the proceeding as stated in the official release. The SEC prints it in capitals. |
| `data[].summary` | string | Brief summary of the proceeding. |
| `data[].tags[]` | string[] | Tags for the proceeding, such as `accounting fraud` or `audit failure`. |

### Respondents and parties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].respondents[]` | object[] | The respondents in the proceeding. |
| `data[].respondents[].name` | string | Name of the respondent. |
| `data[].respondents[].type` | string | Type of the party, such as `individual`, `company` or `other`. |
| `data[].respondents[].role` | string | Role of the party, normally `respondent`. |
| `data[].respondents[].cik` | string | Central Index Key of the party, if available. |
| `data[].respondents[].ticker` | string | Ticker symbol of the party, if available. |
| `data[].respondentsText` | string | The respondent line as the SEC prints it. It can carry a note such as `(Order Granting Extension of Time)`. |
| `data[].entities[]` | object[] | All parties involved in the proceeding, including related parties that are not respondents. |
| `data[].entities[].name` | string | Name of the party involved. |
| `data[].entities[].type` | string | Type of the party, such as `individual`, `company` or `other`. |
| `data[].entities[].role` | string | Role of the party, such as `respondent`, `defendant`, `affected entity` or `other`. |
| `data[].entities[].cik` | string | Central Index Key of the party, if available. |
| `data[].entities[].ticker` | string | Ticker symbol of the party, if available. |

### Charges, orders and penalties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].complaints[]` | string[] | The complaints or charges in the proceeding. One sentence each. |
| `data[].violatedSections[]` | string[] | The securities laws the respondents violated. |
| `data[].orders[]` | string[] | The orders the SEC issued in the proceeding, for example `Respondent is suspended from appearing or practicing before the Commission as an accountant`. Unique to this tool. |
| `data[].requestedRelief[]` | string[] | The relief requested, such as `cease-and-desist order`, `permanent injunctions` or `civil penalties`. |
| `data[].hasAgreedToSettlement` | boolean | True when the respondent has agreed to a settlement. |
| `data[].hasAgreedToPayPenalty` | boolean | True when the respondent has agreed to pay a penalty. |
| `data[].penaltyAmounts[]` | object[] | One object per penalty in the order. |
| `data[].penaltyAmounts[].penaltyAmount` | string | The cleaned penalty amount in USD. Digits only, no currency sign and no separators. |
| `data[].penaltyAmounts[].penaltyAmountText` | string | The penalty amount as stated in the order, for example `$75,000`. |
| `data[].penaltyAmounts[].imposedOn` | string | The party on which the penalty was imposed. |

### Documents

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].resources[]` | object[] | Links to source documents and related material, such as a submission for comments. |
| `data[].resources[].label` | string | Name of the document. The order PDF carries the label `primary`. |
| `data[].resources[].url` | string | Direct link to the document. |

### Who acted

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].investigationConductedBy[]` | string[] | The persons or entities that conducted the investigation, for example `Division of Enforcement`. |
| `data[].litigationLedBy[]` | string[] | The persons or entities that litigated the case, for example `Division of Enforcement`. |
| `data[].parallelActionsTakenBy[]` | string[] | Other agencies that took a parallel action related to the proceeding. |
| `data[].otherAgenciesInvolved[]` | object[] | Other agencies involved in the proceeding, for example `Autorité des Marchés Financiers`. |
| `data[].otherAgenciesInvolved[].name` | string | Name of the agency. |
| `data[].otherAgenciesInvolved[].country` | string | Country of the agency. |

Size behaviour. `size` defaults to 50 and cannot exceed 50. Page with `from`.
This example set `size: 1` and returned 1,824 bytes for one row, so a `size: 50`
call can reach about 90 KB.

## Example

Prompt: "What is the newest SEC administrative proceeding, and who is the
respondent?"

```json
{ "name": "sec-administrative-proceedings", "arguments": { "query": "releaseNo:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "releasedAt": "2026-08-12T20:20:38-04:00",
      "releaseNo": ["34-106124"],
      "fileNumbers": ["3-19038"],
      "respondents": [
        { "name": "Investmark Advisory Group LLC", "type": "company", "role": "respondent", "cik": "1926349", "ticker": "INVES6" }
      ],
      "respondentsText": "Investmark Advisory Group LLC",
      "resources": [
        { "label": "primary", "url": "https://www.sec.gov/files/litigation/admin/2023/34-106124.pdf" }
      ],
      "title": "ORDER AUTHORIZING THE TRANSFER OF ANY REMAINING FUNDS IN THE DISTRIBUTION FUND AND ANY FUNDS RETURNED IN THE FUTURE TO THE U.S. TREASURY AND TERMINATING THE DISTRIBUTION FUND",
      "tags": ["disclosure failures", "conflicts of interest"],
      "orders": ["The Distribution Fund is terminated."],
      "violatedSections": ["Sections 203(e) and 203(k) of the Investment Advisers Act of 1940"],
      "requestedRelief": ["disgorgement of profits"]
    }
  ]
}
```

## Limits and errors

- `total.value: 10000` with `relation: "gte"` is the search-window ceiling. It
  means 10,000 or more. Narrow the query with a date range to get an exact
  count.
- A query without a `:` fails with HTTP 400 `Invalid Lucene query string`.
- A query over 1,000 characters fails with
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- `releasedAt` is the release date of the document, not the date the case
  began. In the example a document released on 2026-08-12 links a PDF filed
  under `/admin/2023/`. `releasedAt` is also not the date of the misconduct.
- `entities[].ticker` is not always a tradable symbol. The example shows
  `INVES6` for a private LLC. Verify a ticker with [`mapping`](./mapping.md).
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`sec-litigation-releases`](./sec-litigation-releases.md). Civil cases filed in federal court.
- [`sec-enforcement-actions`](./sec-enforcement-actions.md). Announced actions, with press release text.
- [`aaers`](./aaers.md). The same orders when they concern accounting or auditing.
- [Query language](../query-language.md). Lucene syntax and field names.
- [sec-api.io Administrative Proceedings API docs](https://sec-api.io/docs/sec-administrative-proceedings-database-api)
