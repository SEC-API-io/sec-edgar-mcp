# form-13f-cover-pages

Search the cover page of each Form 13F filing, the filing-level summary of an
institutional manager's quarterly report.

|                 |                                                            |
| --------------- | ------------------------------------------------------------ |
| Category        | Ownership and insiders                                       |
| Required input  | `query`                                                      |
| Returns         | `{total, data[]}`. One item per 13F filing. No position list. |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                              |
| REST equivalent | `POST /form-13f/cover-pages`                                 |

## What it does

Every Form 13F carries a cover page. It names the filing manager, the report
period, the number of reported positions and the total portfolio value. This
tool searches those cover pages. One item in `data[]` is one filing. The
holdings themselves are not included.

The capture asked for `cik:1067983` (Berkshire Hathaway) with `size: 1`. It
returned `total.value: 60` and the 13F-HR cover page for the period ending
2026-03-31, in 2,774 bytes. Note the count difference. The same query returns 60
here and 210 in [`form-13f-holdings`](./form-13f-holdings.md). The two indices
do not hold the same number of records, and the cause is not verified. Do not
use one total as a proxy for the other.

## When to use it

- Which managers filed a 13F for the quarter ending 2026-03-31?
- What was the total portfolio value a manager reported?
- How many positions did the manager report on the cover page?
- Which other managers are included in this manager's report?

## When to use a different tool

| Situation                                  | Better tool                                          | Why                                                                    |
| ------------------------------------------ | ---------------------------------------------------- | ----------------------------------------------------------------------- |
| You need the actual positions              | [`form-13f-holdings`](./form-13f-holdings.md)        | Cover pages carry totals only. Holdings carry CUSIP, shares and value.  |
| You want the adviser's registration record | [`form-adv-firms`](./form-adv-firms.md)              | Form ADV holds the firm profile behind the CRD number.                  |
| You want stakes above 5% of one company    | [`form-13d-13g`](./form-13d-13g.md)                  | 13D and 13G report percent ownership of a single issuer.                |

## Input

| Parameter | Type    | Required | Constraints                          | Notes                                                     |
| --------- | ------- | -------- | ------------------------------------ | ----------------------------------------------------------- |
| `query`   | string  | yes      | must contain `:`, max 1,000 characters | Lucene syntax. See [query language](../query-language.md). |
| `from`    | integer | no       | 0 to 10000                           | Offset into the result set.                                 |
| `size`    | integer | no       | 1 to 50                              | Cover pages per call. **Defaults to 50** when you omit it.  |
| `sort`    | array   | no       | array of sort objects                | Defaults to `[{"filedAt": {"order": "desc"}}]`.             |

The schema sets `additionalProperties: true`. The handler also reads
`time_zone`, a string that defaults to `America/New_York` and applies to date
ranges in the query. Unlike [`form-13f-holdings`](./form-13f-holdings.md), this
handler does not rewrite your query. Query fields:

- `cik`. Confirmed. The capture used `cik:1067983`.
- `formType`, `periodOfReport`, `filedAt`, `accessionNo`, `crdNumber`,
  `secFileNumber`, `form13FFileNumber`, `isAmendment`, `filingManager.name`. All
  present in the response body, all **unverified** as query fields.

## Output

The envelope is `{total, data[]}`. `total` is an object. Read `total.value`.
`total.relation` is `"eq"` for an exact count, `"gte"` at 10000 for 10,000 or
more.

| Field                                     | Type    | Meaning                                                       |
| ----------------------------------------- | ------- | -------------------------------------------------------------- |
| `accessionNo`                             | string  | EDGAR accession number. Use it to join to `form-13f-holdings`.  |
| `filedAt`                                 | string  | ISO 8601 with offset.                                           |
| `formType`                                | string  | `13F-HR` in the capture. Use `isAmendment` to spot amendments.   |
| `cik`                                     | string  | CIK of the filing manager.                                      |
| `crdNumber`, `secFileNumber`              | string  | Adviser identifiers. **Both are empty strings in the capture.** The canonical SDK response populates them for Bridgewater. Do not assume a value. |
| `form13FFileNumber`                       | string  | The 13F file number, for example `028-04545`.                   |
| `periodOfReport`                          | string  | Quarter end, `YYYY-MM-DD`.                                      |
| `isAmendment`                             | boolean | `false` for an original report. `amendmentInfo` holds the detail and is an empty object here. |
| `filingManager`                           | object  | `name`, and `address` with `street`, `city`, `stateOrCountry`, `zipCode`. |
| `filingManager.address.zipCode`           | number  | A **number**, not a string. `68131`. A ZIP with a leading zero loses it, for example Westport CT arrives as `6880`. |
| `reportType`                              | string  | `13F HOLDINGS REPORT` in the capture.                           |
| `signature`                               | object  | `name`, `title`, `phone`, `signature`, `city`, `stateOrCountry`, `signatureDate`. |
| `signature.signatureDate`                 | string  | Format `MM-DD-YYYY`, for example `05-15-2026`. Every other date on this record is ISO. |
| `tableEntryTotal`                         | number  | Number of position lines in the information table. `90` in the capture. |
| `tableValueTotal`                         | number  | Total reported value in US dollars. `263095703570` in the capture. |
| `tableEntryTotalAsReported`, `tableValueTotalAsReported` | number | The values exactly as filed. They matched in the capture. |
| `otherIncludedManagersCount`              | number  | Count of managers covered by this report. `14` in the capture.  |
| `otherIncludedManagers[]`                 | array   | `sequenceNumber`, `name`, `cik`, `crdNumber`, `secFileNumber`, `form13FFileNumber`. |

Use `otherIncludedManagers[].sequenceNumber` to resolve the `otherManager`
string on each holding in [`form-13f-holdings`](./form-13f-holdings.md).
Berkshire's holding `"otherManager": "4"` means Buffett Warren E. The `cik` of
these sub-managers is an empty string in the capture.

Size behaviour: cover pages are small. The capture returned 2,774 bytes for one
record with 14 sub-managers. `size: 50` is safe here, unlike on
`form-13f-holdings`.

## Example

Prompt: "What was the total value Berkshire Hathaway reported on its latest 13F cover page?"

```json
{ "name": "form-13f-cover-pages", "arguments": { "query": "cik:1067983", "size": 1 } }
```

```json
{
  "total": { "value": 60, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001193125-26-226661",
      "filedAt": "2026-05-15T16:06:05-04:00",
      "formType": "13F-HR",
      "cik": "1067983",
      "form13FFileNumber": "028-04545",
      "periodOfReport": "2026-03-31",
      "isAmendment": false,
      "filingManager": {
        "name": "Berkshire Hathaway Inc",
        "address": { "street": "3555 Farnam Street", "city": "Omaha", "stateOrCountry": "NE", "zipCode": 68131 }
      },
      "signature": { "name": "Marc D. Hamburg", "title": "Senior Vice President", "signatureDate": "05-15-2026" },
      "tableEntryTotal": 90,
      "tableValueTotal": 263095703570,
      "otherIncludedManagersCount": 14,
      "otherIncludedManagers": [
        { "sequenceNumber": 1, "name": "Berkshire Hathaway Homestate Insurance Co.", "form13FFileNumber": "28-2226" }
      ]
    }
  ]
}
```

Trimmed. The capture lists 13 more sub-managers, each with empty `cik`,
`crdNumber` and `secFileNumber`.

## Limits and errors

- A missing `query`, a query without `:`, a query over 1,000 characters, or
  `from` above 10000 all fail with HTTP 400 `Invalid request parameter
  provided.` One message covers four causes, so check all four.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.` These error
  texts come from the server handler. The capture did not trigger them.
- `tableValueTotal` is what the manager reported. It is not audited.
- Empty strings are common on this record. Treat `crdNumber`, `secFileNumber`
  and sub-manager `cik` as optional.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-13f-holdings`](./form-13f-holdings.md)
- [`form-adv-firms`](./form-adv-firms.md)
- [`form-13d-13g`](./form-13d-13g.md)
- REST docs: [Form 13F API](https://sec-api.io/docs/form-13-f-filings-institutional-holdings-api)
