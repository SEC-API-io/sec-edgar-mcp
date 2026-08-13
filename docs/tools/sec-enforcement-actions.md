# sec-enforcement-actions

Search SEC enforcement actions announced by the Commission, with the parties,
the allegations and the penalties already extracted.

|                 |                                                         |
| --------------- | ------------------------------------------------------- |
| Category        | Enforcement                                             |
| Required input  | `query`                                                 |
| Returns         | `{total: {value, relation}, data[]}`                    |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`            |
| REST equivalent | `POST https://api.sec-api.io/sec-enforcement-actions`   |

## What it does

`sec-enforcement-actions` searches SEC press releases that announce an
enforcement action or a whistleblower award. One row is one press release. Each
row carries a plain-language `summary`, the `entities` involved with their CIK
and ticker, the `complaints`, the `violatedSections` and any `penaltyAmounts`.
Coverage starts in 1997 and runs to the present. New actions arrive as the SEC
publishes them. A request for `releaseNo:*` returned `total.value: 2768`. The
index held 2,768 releases on 2026-08-13.

Coverage limit. The source is the SEC newsroom, not the case docket. The SEC
does not publish a press release for every action it brings. For full case
coverage, search the two sibling tools as well.

## When to use it

- Which enforcement actions did the SEC announce this month?
- Has the SEC ever charged this company or this person?
- Which crypto fraud cases ended in a settlement and a penalty?
- Which cases ran in parallel with the Department of Justice?

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| You need every civil case, not only the announced ones | [`sec-litigation-releases`](./sec-litigation-releases.md) | Litigation releases cover federal court actions that get no press release. |
| The case was decided inside the SEC | [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | Orders, settlements and bars appear there, with a file number. |
| The subject is accounting fraud or an audit failure | [`aaers`](./aaers.md) | AAERs are the accounting and auditing subset, with their own numbering. |
| You want exchange rule changes, not misconduct | [`sro`](./sro.md) | SRO filings are proposed rules, not enforcement. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `query` | string | yes | Lucene syntax, max 1,000 characters, must contain a `:` | Example: `releaseNo:*`. |
| `from` | integer | no | 0 to 10,000 | Offset of the first row. Default 0. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"releasedAt": {"order": "desc"}}]`. |


Query fields: `releaseNo`, `releasedAt`, `entities.name`, `entities.cik`,
`entities.ticker`, `entities.role`, `entities.type`, `tags`, `complaints`,
`violatedSections`, `requestedRelief`, `hasAgreedToSettlement`,
`hasAgreedToPayPenalty`, `penaltyAmounts.penaltyAmount`,
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
| `total.value` | number | Number of releases that match the query. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[]` | array | One object per press release. |

### Release

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].id` | string | Internal unique identifier of the enforcement action. |
| `data[].releaseNo` | string | SEC release number of the enforcement action, for example `2011-279`. A string here, an array on [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). |
| `data[].releasedAt` | string | Publication date and time of the enforcement action, format `yyyy-MM-ddTHH:mm:ssXXX`. Records before 2012 carry the date part only, for example `2011-12-29`. |
| `data[].url` | string | URL of the original SEC press release. |
| `data[].title` | string | Title of the enforcement action. |
| `data[].summary` | string | Brief summary of the enforcement action. |
| `data[].tags[]` | string[] | Tags for the enforcement action, such as `bribery` or `insider trading`. |

### Parties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].entities[]` | object[] | The parties involved in the enforcement action. |
| `data[].entities[].name` | string | Name of the party involved. |
| `data[].entities[].type` | string | Type of the party, such as `individual`, `company` or `other`. |
| `data[].entities[].role` | string | Role of the party, such as `respondent`, `defendant` or `other`. |
| `data[].entities[].cik` | string | Central Index Key of the party, if available. |
| `data[].entities[].ticker` | string | Ticker symbol of the party, if available. |

### Charges, relief and penalties

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].complaints[]` | string[] | The complaints or charges filed in the enforcement action. One sentence each. |
| `data[].violatedSections[]` | string[] | The securities laws the defendants violated, such as `Section 10(b) of the Securities Exchange Act of 1934` or `Foreign Corrupt Practices Act (FCPA)`. |
| `data[].requestedRelief[]` | string[] | The relief the SEC requests, such as `disgorgement of profits`, `injunction` or `civil penalty`. |
| `data[].hasAgreedToSettlement` | boolean | True when the defendant has agreed to a settlement. |
| `data[].hasAgreedToPayPenalty` | boolean | True when the defendant has agreed to pay a penalty. |
| `data[].penaltyAmounts[]` | object[] | One object per penalty in the release. |
| `data[].penaltyAmounts[].penaltyAmount` | string | The cleaned penalty amount in USD. Digits only, no currency sign and no separators. |
| `data[].penaltyAmounts[].penaltyAmountText` | string | The penalty amount as stated in the enforcement action, for example `$31.2 million`. |
| `data[].penaltyAmounts[].imposedOn` | string | The party on which the penalty was imposed. |

### Documents

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].resources[]` | object[] | Links to related documents, such as litigation releases and complaints. |
| `data[].resources[].label` | string | Name of the related document, for example `SEC Complaint`. |
| `data[].resources[].url` | string | Direct link to the document. |

### Who acted

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `data[].investigationConductedBy[]` | string[] | The persons or entities that conducted the investigation that led to the enforcement action. Holds staff names, or a division such as `Division of Enforcement`. |
| `data[].litigationLedBy[]` | string[] | The persons or entities that litigated the case. |
| `data[].parallelActionsTakenBy[]` | string[] | Other agencies that took a parallel action, such as `U.S. Department of Justice` for criminal charges. |
| `data[].otherAgenciesInvolved[]` | object[] | Other agencies involved in the investigation or the enforcement action. |
| `data[].otherAgenciesInvolved[].name` | string | Name of the agency. |
| `data[].otherAgenciesInvolved[].country` | string | Country of the agency. |

Size behaviour. `size` defaults to 50 and cannot exceed 50. Page with `from`.
This example set `size: 1` and returned 2,550 bytes for one row, so a `size: 50`
call can reach about 125 KB.

## Example

Prompt: "Show me the newest SEC enforcement action and who it charges."

```json
{ "name": "sec-enforcement-actions", "arguments": { "query": "releaseNo:*", "size": 1 } }
```

```json
{
  "total": { "value": 2768, "relation": "eq" },
  "data": [
    {
      "releaseNo": "2026-73",
      "releasedAt": "2026-08-10T14:00:00Z",
      "url": "https://www.sec.gov/newsroom/press-releases/2026-73-sec-charges-private-fund-adviser-adit-ventures-management-its-ceo-affiliated-general-partners",
      "title": "SEC Charges Private Fund Adviser Adit Ventures Management, Its CEO and Affiliated General Partners in Alleged Fraud",
      "summary": "The SEC has charged Adit Ventures Management, its CEO Eric Munson, and affiliated general partners with defrauding investors by misappropriating funds, charging undisclosed fees, and violating fiduciary duties in connection with pre-IPO investments.",
      "tags": ["fraud", "investment advisory", "misappropriation", "undisclosed fees"],
      "entities": [
        { "name": "Adit Ventures Management LLC", "type": "company", "role": "defendant", "cik": "1852894", "ticker": "UPFRON" },
        { "name": "Eric Munson", "type": "individual", "role": "defendant" }
      ],
      "hasAgreedToSettlement": false,
      "penaltyAmounts": [],
      "violatedSections": ["antifraud provisions of the Securities Act of 1933"],
      "investigationConductedBy": ["Division of Enforcement"]
    }
  ]
}
```

## Limits and errors

- A query without a `:` fails with HTTP 400 `Invalid Lucene query string`.
- A query over 1,000 characters fails with
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- `entities[].ticker` is not always a tradable symbol. Values include `UPFRON`
  and `HSTVEN` for private LLCs. Check one with [`mapping`](./mapping.md) before
  you use it.
- Empty arrays are normal. `penaltyAmounts` and `requestedRelief` were empty in
  the example response, on a case that was still contested.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`sec-litigation-releases`](./sec-litigation-releases.md). Civil cases filed in federal court.
- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). Cases decided inside the SEC.
- [`aaers`](./aaers.md). The accounting and auditing subset.
- [Query language](../query-language.md). Lucene syntax and field names.
- [sec-api.io Enforcement Actions API docs](https://sec-api.io/docs/sec-enforcement-actions-database-api)
