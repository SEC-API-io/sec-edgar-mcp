# form-8k

Search 8-K current reports and get the event details of three items as
structured data.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-8k`                              |

## What it does

A public company files an 8-K when something material happens. This tool
searches an index that reads the 8-K body and returns the event as fields. One
row is one 8-K filing.

Coverage starts 2004-08-23.

**Only three 8-K items are extracted.**

| Item object | 8-K item | Subject                                            | Rows          |
| ----------- | -------- | -------------------------------------------------- | ------------- |
| `item4_01`  | 4.01     | Change of the certifying accountant                 | 10,000 or more |
| `item4_02`  | 4.02     | Non-reliance on previously issued financials        | 8,592         |
| `item5_02`  | 5.02     | Director and officer departures and appointments    | 10,000 or more |

Queries for `item1_01`, `item1_02`, `item2_01`, `item2_03`, `item3_01`,
`item5_01`, `item5_03`, `item7_01`, `item8_01` and `item9_01` all return zero
rows. The registry description names "acquisitions", "earnings releases" and
"restructurings". Those items are **not** in this index. Treat that part of the
description as wrong. Every row still carries the `items[]` array, which lists
the full item titles the filing reported, including items with no structured
object.

## When to use it

- Which companies dismissed or hired an auditor in the last month?
- Which companies restated their financial statements this quarter, and why?
- Who joined or left this company's board, and on what date?
- What sign-on package did this new CFO get?

## When to use a different tool

| Situation                                    | Better tool                                                 | Why                                              |
| -------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------ |
| You want every 8-K, or an item other than 4.01, 4.02, 5.02 | [filing-search](./filing-search.md)          | Indexes all 8-K filings by form type and date    |
| You want the 8-K text or its exhibits        | [extractor](./extractor.md), [get-edgar-file](./get-edgar-file.md) | Returns the document itself             |
| You want a phrase anywhere in an 8-K         | [full-text-search](./full-text-search.md)                   | Searches the filing body text                    |
| You want the current board roster            | [directors-and-board-members](./directors-and-board-members.md) | Proxy-based roster, not one-off changes       |
| You want annual pay tables                   | [compensation](./compensation.md)                           | Summary compensation table, not a single hire    |

## Input

| Parameter | Type    | Required | Constraints        | Notes                                        |
| --------- | ------- | -------- | ------------------ | -------------------------------------------- |
| `query`   | string  | yes      | must contain a `:` | Lucene syntax. Max 1000 characters.          |
| `from`    | integer | no       | 0 or more          | Offset. `from` above 10000 returns HTTP 400. |
| `size`    | integer | no       | 1 to 50            | Default 50. Above 50 returns HTTP 400.       |
| `sort`    | array   | no       | ES sort clause     | Default `[{"filedAt":{"order":"desc"}}]`.    |

The colon rule is strict. A bare word such as `apple` returns HTTP 400 with
`Invalid request parameter provided.`

Query fields:

| Field            | Example                                     |
| ---------------- | ------------------------------------------- |
| `ticker`         | `ticker:AAPL`                               |
| `cik`            | `cik:320193`                                |
| `companyName`    | `companyName:Apple*`                        |
| `formType`       | `formType:"8-K"`                            |
| `filedAt`        | `filedAt:[2024-01-01 TO 2024-12-31]`        |
| `periodOfReport` | `periodOfReport:[2026-01-01 TO 2026-12-31]` |
| `items`          | `items:"Item 5.02"`                         |
| `item4_01`       | `item4_01:*`                                |
| `item4_02`       | `item4_02:*`                                |
| `item5_02`       | `item5_02:*`                                |

The `itemX_YY:*` form is the idiom for "give me the filings that carry this
structured block". Combine it with a date range, for example
`item4_01:* AND filedAt:[2024-01-01 TO 2024-12-31]`. Nested fields inside the
item objects, such as `item4_01.formerAccountantName`, follow the same dotted
syntax.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

| Field            | Type   | Meaning                                              |
| ---------------- | ------ | ---------------------------------------------------- |
| `id`             | string | Internal document ID                                 |
| `accessionNo`    | string | EDGAR accession number                               |
| `formType`       | string | `8-K`                                                |
| `filedAt`        | string | Filing timestamp with offset                         |
| `periodOfReport` | string | Date of the event, `YYYY-MM-DD`                      |
| `cik`            | string | Filer CIK, no leading zeros                          |
| `ticker`         | string | Filer ticker, empty when the filer has none          |
| `companyName`    | string | Filer name                                           |
| `items[]`        | array  | Full item titles the filing reported                 |
| `item4_01`       | object | Auditor change, present only for that item           |
| `item4_02`       | object | Non-reliance and restatement, present only for that item |
| `item5_02`       | object | Officer and director change, present only for that item |

Each item object starts with `keyComponents`, a one-paragraph plain-language
summary of the event. The rest differs per item.

`item5_02` holds `personnelChanges[]`. Each entry has `type`
(`appointment` and similar), `effectiveDate`, `positions[]`, a `person` object
(`name`, `age`, `background`, `previousPositions[]`, `academicAffiliations[]`,
`positionsAtOtherCompanies[]`), a `compensation` object (`annual`, `equity`,
`equityVesting`, `noCompensation`), and the booleans `continuedConsultingRole`,
`termExtended`, `termShortened`, `compensationIncreased`,
`compensationDecreased`, `disagreements` and `interim`. `organizationChanges`
and `attachments[]` can also appear.

The other two item objects hold their own fields.

`item4_01`: `formerAccountantName`, `newAccountantName`, `engagementEndReason`,
`formerAccountantDate`, `newAccountantDate`, `opinionType`, `attachments[]`, and
the booleans `engagedNewAccountant`, `consultedNewAccountant`,
`reportedDisagreements`, `reportableEventsExist`, `reportedIcfrWeakness`,
`auditDisclaimer`, `approvedChange`.

`item4_02`: `identifiedIssues[]`, `affectedReportingPeriods[]`,
`identifiedBy[]`, `reasonsForRestatement[]`, `affectedLineItems[]`,
`impactOfError`, `eventClassification`, and the booleans
`restatementIsNecessary`, `impactYetToBeDetermined`, `impactIsMaterial`,
`materialWeaknessIdentified`, `netIncomeDecreased`, `netIncomeIncreased`,
`revenueDecreased`, `revenueIncreased`.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.

## Example

Prompt: "Show me the newest 8-K filings with a structured event block."

```json
{ "name": "form-8k", "arguments": { "query": "cik:*", "size": 1 } }
```

Response, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "id": "f9c18b5e5b1e1b7471d6adb39fdd055e",
      "accessionNo": "0001140361-26-032653",
      "formType": "8-K",
      "filedAt": "2026-08-13T08:48:37-04:00",
      "periodOfReport": "2026-08-08",
      "cik": "1130713",
      "ticker": "BBBY",
      "companyName": "BED BATH & BEYOND, INC.",
      "items": ["Item 5.02: Departure of Directors or Certain Officers; Election of Directors; Appointment of Certain Officers: Compensatory Arrangements of Certain Officers"],
      "item5_02": {
        "keyComponents": "Jill Windrum was appointed as the Company's Chief Accounting Officer and Deputy Chief Financial Officer, effective August 31, 2026. She will replace Brian LaRose as the principal accounting officer.",
        "personnelChanges": [
          {
            "type": "appointment",
            "effectiveDate": "2026-08-31",
            "positions": ["Chief Accounting Officer", "Deputy Chief Financial Officer", "principal accounting officer"],
            "person": { "name": "Jill Windrum", "age": 46 },
            "compensation": { "annual": "400,000", "equity": "Sign-on equity awards with an aggregate target value of $400,000", "noCompensation": false },
            "interim": false
          }
        ]
      }
    }
  ]
}
```

## Limits and errors

- No colon in `query` gives HTTP 400 `Invalid request parameter provided.`
- `query` longer than 1000 characters gives the same 400.
- `from` above 10000 gives the same 400.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- A query for an item outside 4.01, 4.02 and 5.02 returns zero rows, not an
  error. Zero rows here means "not covered", not "no such event happened".
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [filing-search](./filing-search.md), [full-text-search](./full-text-search.md), [extractor](./extractor.md)
- [directors-and-board-members](./directors-and-board-members.md), [compensation](./compensation.md), [audit-fees](./audit-fees.md)
- REST docs: <https://sec-api.io/docs/form-8k-data-item4-1-search-api>,
  <https://sec-api.io/docs/form-8k-data-search-api>,
  <https://sec-api.io/docs/form-8k-data-item5-2-search-api>
