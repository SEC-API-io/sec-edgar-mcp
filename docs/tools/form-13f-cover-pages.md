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

A request for `cik:1067983` (Berkshire Hathaway) with `size: 1` returns
`total.value: 60` and the 13F-HR cover page for the period ending
2026-03-31, in 2,774 bytes. The counts differ between tools. The same query
returns 60 here and 210 in [`form-13f-holdings`](./form-13f-holdings.md). Both
totals count filings, not positions. The two indices do not hold the same number
of records.

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

Unlike [`form-13f-holdings`](./form-13f-holdings.md), this tool does not
rewrite your query.

Query fields: `cik`, `crdNumber`, `periodOfReport`, `filedAt`, `formType`,
`accessionNo`, `reportType`, `filingManager.name`,
`filingManager.address.stateOrCountry`, `amendmentInfo.amendmentType`,
`otherIncludedManagersCount` and `tableValueTotal`. The example uses
`cik:1067983`.

## Output

The envelope is `{total, data[]}`. `total` is an object, not a number. The count
is in `total.value`.

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | -------------------------------------------------------------- |
| `total.value`    | number | Number of cover pages that match the query.                    |
| `total.relation` | string | `eq` for an exact count. `gte` at 10000 for 10,000 or more.    |
| `data[]`         | array  | The cover pages. One item is one 13F filing.                   |

### Filing

| Field                             | Type    | Meaning                                                                                            |
| --------------------------------- | ------- | ---------------------------------------------------------------------------------------------------- |
| `data[].id`                       | string  | Unique identifier of the cover page record.                                                        |
| `data[].accessionNo`              | string  | Accession number of the 13F filing. It joins to `form-13f-holdings`.                               |
| `data[].filedAt`                  | string  | Time the manager submitted the filing, ISO 8601 with offset.                                       |
| `data[].formType`                 | string  | Form type of the filing, `13F-HR` in this response.                                                |
| `data[].periodOfReport`           | string  | End of the quarter the filing covers, `YYYY-MM-DD`.                                                |
| `data[].reportType`               | string  | `13F HOLDINGS REPORT`, `13F NOTICE` or `13F COMBINATION REPORT`. A notice carries no position list. |
| `data[].isAmendment`              | boolean | `true` when the filing amends an earlier report.                                                   |
| `data[].provideInfoForInstruction5` | boolean | The manager gives the extra information that Instruction 5 of the form asks for.                 |

### Filing manager

| Field                                     | Type   | Meaning                                                                                            |
| ----------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------- |
| `data[].cik`                              | string | CIK of the filing manager.                                                                         |
| `data[].crdNumber`                        | string | CRD number of the filing manager. It joins to `form-adv-firms`. **An empty string in this response.** Other filers populate it, for example Bridgewater. |
| `data[].secFileNumber`                    | string | SEC file number of the filing manager. **An empty string in this response.** Presence varies by filer. |
| `data[].form13FFileNumber`                | string | The 13F file number, for example `028-04545`.                                                      |
| `data[].filingManager.name`               | string | Name of the manager.                                                                               |
| `data[].filingManager.address.street`     | string | Street of the manager address.                                                                     |
| `data[].filingManager.address.city`       | string | City of the manager address.                                                                       |
| `data[].filingManager.address.stateOrCountry` | string | State or country of the manager address, for example `NE`.                                     |
| `data[].filingManager.address.zipCode`    | number | ZIP code of the manager address. A **number**, not a string. `68131`. A ZIP with a leading zero loses it, for example Westport CT arrives as `6880`. |

### Amendment

`amendmentInfo` is present only when `isAmendment` is `true`. The key is absent
from this response.

| Field                              | Type   | Meaning                                              |
| ---------------------------------- | ------ | ------------------------------------------------------ |
| `data[].amendmentInfo`             | object | Detail of the amendment.                             |
| `data[].amendmentInfo.amendmentNo` | number | Number of the amendment.                             |
| `data[].amendmentInfo.amendmentType` | string | `RESTATEMENT` replaces the earlier report. `NEW HOLDINGS` adds positions to it. |

### Totals

| Field                              | Type   | Meaning                                                        |
| ---------------------------------- | ------ | -------------------------------------------------------------- |
| `data[].tableEntryTotal`           | number | Number of position lines in the information table. `90` in this response. |
| `data[].tableValueTotal`           | number | Total value of the holdings in US dollars. `263095703570` in this response. |
| `data[].tableEntryTotalAsReported` | number | The entry count exactly as the manager filed it.               |
| `data[].tableValueTotalAsReported` | number | The total value exactly as the manager filed it.               |
| `data[].otherIncludedManagersCount` | number | Number of other managers this report covers. `14` in this response. |

The two `AsReported` fields match the plain fields in this response. They differ
when the filed value needs a correction, such as a value reported in thousands.

### Other managers

`otherIncludedManagers[]` names the managers this report covers.
`otherManagersReportingForThisManager[]` names the managers that report this
manager's holdings in their own filing. The second array is empty in this
response.

| Field                                                        | Type   | Meaning                                                        |
| ------------------------------------------------------------ | ------ | -------------------------------------------------------------- |
| `data[].otherIncludedManagers[].sequenceNumber`              | number | Position of the manager in the list. The holdings reference it. |
| `data[].otherIncludedManagers[].name`                        | string | Name of the manager.                                           |
| `data[].otherIncludedManagers[].cik`                         | string | CIK of the manager.                                            |
| `data[].otherIncludedManagers[].crdNumber`                   | string | CRD number of the manager.                                     |
| `data[].otherIncludedManagers[].secFileNumber`               | string | SEC file number of the manager.                                |
| `data[].otherIncludedManagers[].form13FFileNumber`           | string | 13F file number of the manager, for example `28-2226`.         |
| `data[].otherManagersReportingForThisManager[].name`         | string | Name of the reporting manager.                                 |
| `data[].otherManagersReportingForThisManager[].cik`          | string | CIK of the reporting manager.                                  |
| `data[].otherManagersReportingForThisManager[].crdNumber`    | string | CRD number of the reporting manager.                           |
| `data[].otherManagersReportingForThisManager[].secFileNumber` | string | SEC file number of the reporting manager.                     |
| `data[].otherManagersReportingForThisManager[].form13FFileNumber` | string | 13F file number of the reporting manager.                 |

### Signature

| Field                             | Type   | Meaning                                                        |
| --------------------------------- | ------ | -------------------------------------------------------------- |
| `data[].signature.name`           | string | Name of the person who signed the form.                        |
| `data[].signature.title`          | string | Title of that person, for example `Senior Vice President`.     |
| `data[].signature.phone`          | string | Contact phone number for the filing.                           |
| `data[].signature.signature`      | string | The signature text as typed on the form.                       |
| `data[].signature.city`           | string | City where the person signed.                                  |
| `data[].signature.stateOrCountry` | string | State or country where the person signed.                      |
| `data[].signature.signatureDate`  | string | Date of the signature, `MM-DD-YYYY`, for example `05-15-2026`. Every other date on this record is ISO. |

`otherIncludedManagers[].sequenceNumber` matches the `otherManager` string on
each holding in [`form-13f-holdings`](./form-13f-holdings.md).
Berkshire's holding `"otherManager": "4"` means Buffett Warren E. The `cik` of
these sub-managers is an empty string in this response.

Size behaviour: cover pages are small. This example returned 2,774 bytes for one
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

Trimmed. The full response lists 13 more sub-managers, each with empty `cik`,
`crdNumber` and `secFileNumber`.

## Limits and errors

- A missing `query`, a query without `:`, a query over 1,000 characters, or
  `from` above 10000 all fail with HTTP 400 `Invalid request parameter
  provided.` One message covers four causes.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- `tableValueTotal` is what the manager reported. It is not audited.
- Empty strings are common on this record. `crdNumber`, `secFileNumber` and
  sub-manager `cik` are optional.
- The index covers 13F-HR filings since 1998, and 13F-E filings from 1994 to
  1998.
- One query returns at most 10,000 cover pages. Narrow the query to reach more.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-13f-holdings`](./form-13f-holdings.md)
- [`form-adv-firms`](./form-adv-firms.md)
- [`form-13d-13g`](./form-13d-13g.md)
- REST docs: [Form 13F API](https://sec-api.io/docs/form-13-f-filings-institutional-holdings-api)
