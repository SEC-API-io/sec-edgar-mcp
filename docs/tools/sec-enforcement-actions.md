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
A request for `releaseNo:*` returned `total.value: 2768`. The index held 2,768
releases on 2026-08-13.

Coverage limit. The source is the SEC newsroom, not the case docket. The SEC
does not publish a press release for every action it brings. The server also
adds `tags:* AND ` in front of your query, so only releases with extracted tags
can match. For full case coverage, search the two sibling tools as well.

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
| `from` | integer | no | minimum 0 | Offset of the first row. Default 0. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"releasedAt": {"order": "desc"}}]`. |


Query fields: `releaseNo`, `releasedAt`, `tags`, `entities.name`,
`entities.cik`, `entities.ticker`, `violatedSections` and
`hasAgreedToSettlement`. Every other name in the Output table below is a
response field. The sec-api REST docs treat those as searchable too. A date
range looks like `releasedAt:[2024-01-01 TO 2024-12-31]`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of matching releases. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[].id` | string | Internal record hash. |
| `data[].releaseNo` | string | Press release number, for example `2026-73`. A string here, an array on [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). |
| `data[].releasedAt` | string | Publication timestamp, ISO 8601. |
| `data[].url` | string | URL of the press release on sec.gov. |
| `data[].title` | string | Headline of the release. |
| `data[].summary` | string | One-paragraph summary of the action. |
| `data[].tags[]` | array | Short subject labels, for example `fraud`, `investment advisory`. |
| `data[].entities[]` | array | Parties. Each has `name`, `type`, `role`, and `cik` and `ticker` when known. |
| `data[].complaints[]` | array | One sentence per allegation. |
| `data[].violatedSections[]` | array | Statutes and rules the SEC says were violated. |
| `data[].requestedRelief[]` | array | Relief the SEC asks for, such as `civil penalties`. |
| `data[].hasAgreedToSettlement`, `data[].hasAgreedToPayPenalty` | boolean | True when the release states a settlement, or a penalty payment. |
| `data[].penaltyAmounts[]` | array | Each has `penaltyAmount` (a numeric string), `penaltyAmountText` and `imposedOn`. |
| `data[].resources[]` | array | Attached documents. Each has `label` and `url`. |
| `data[].investigationConductedBy[]` | array | Staff names, or a division such as `Division of Enforcement`. |
| `data[].otherAgenciesInvolved[]` | array | Each has `name` and `country`. |

Two more arrays name the staff and bodies involved: `parallelActionsTakenBy`
and `litigationLedBy`. Both were empty in the example response.

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
