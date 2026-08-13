# audit-fees

Search the auditor and audit-fee disclosures that public companies publish in
their proxy statements.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Governance and compensation                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /audit-fees`                           |

## What it does

Public companies report what they paid their audit firm. The disclosure sits in
the proxy statement, usually a DEF 14A. This tool searches those disclosures.

The data covers DEF 14A filings from 2001 to today. It holds more than 139,000
fee records from 69,000 filings. Records remain after a company delists or stops reporting to the SEC.

One row is one filing, not one fee figure. Each row carries a `records[]` array
with one entry per fiscal year, so a single proxy statement gives you the
current year and the prior year side by side. A filing that names more than one
audit firm gets one entry per firm and year. A request for NVIDIA returns 22
filings, the newest holding fees for 2026 and 2025, both audited by PwC.

## When to use it

- What did this company pay its auditor last year?
- Who audits this company, and did the audit firm change?
- Which companies pay more than $10 million in audit fees?
- How large are tax fees next to audit fees at this issuer?
- Which issuers does Deloitte audit?

## When to use a different tool

| Situation                                    | Better tool                                             | Why                                                        |
| -------------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------- |
| You want the proxy statement itself          | [`filing-search`](./filing-search.md)                   | Search `formType:"DEF 14A"` and read the document URL.      |
| You want executive pay from the same proxy   | [`compensation`](./compensation.md)                     | Pay tables are a separate dataset.                          |
| You want the board and its audit committee   | [`directors-and-board-members`](./directors-and-board-members.md) | Committee membership is a separate dataset.       |
| You want accounting enforcement cases        | [`aaers`](./aaers.md)                                   | AAERs cover audit failures and reporting misconduct.        |

## Input

| Parameter | Type    | Required | Constraints        | Notes                                                        |
| --------- | ------- | -------- | ------------------ | ------------------------------------------------------------ |
| `query`   | string  | Yes      | Lucene syntax      | `field:value`. `AND`, `OR` and ranges work.                  |
| `from`    | integer | No       | 0 to 10,000        | Offset. Default 0.                                           |
| `size`    | integer | No       | 1 to 50            | Default 50. Above 50 the server returns HTTP 400.            |
| `sort`    | array   | No       | Elasticsearch sort | Default `[{"filedAt": {"order": "desc"}}]`.                  |

Query fields: `accessionNo`, `formType`, `filedAt`, `periodOfReport`, `cik`,
`entities.cik`, `entities.ticker`, `entities.companyName`, `entities.fileNo`,
`entities.sic`, `entities.stateOfIncorporation`, `entities.fiscalYearEnd`,
`records.year`, `records.auditor`, `records.auditFees`,
`records.auditRelatedFees`, `records.taxFees`, `records.allOtherFees`,
`records.totalFees`.

Note the shape. The company identifiers sit under `entities`, so the ticker
field is `entities.ticker`, not bare `ticker`. The CIK works both ways, as
`cik` and as `entities.cik`. Other tools differ. See
[query language](../query-language.md).

Ranges work on the fee fields. `records.totalFees:[10000000 TO *]` returned
3,271 filings.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`,
not a number.

A row has 27 fields across three levels. Every one is listed below.

### Envelope

| Field            | Type   | Meaning                                                          |
| ---------------- | ------ | ----------------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                          |
| `total.relation` | string | `eq` for an exact count. `gte` at 10000 means "10,000 or more".  |
| `data[]`         | array  | The matching filings. One item is one proxy statement.           |

### Filing

| Field                   | Type   | Meaning                                                            |
| ----------------------- | ------ | ------------------------------------------------------------------- |
| `data[].id`             | string | System-internal unique identifier of the filing record.            |
| `data[].accessionNo`    | string | Accession number of the Form DEF 14A filing.                       |
| `data[].formType`       | string | Form type of the SEC filing, usually `DEF 14A`.                    |
| `data[].filedAt`        | string | Timestamp when EDGAR accepted the filing. ISO 8601 with an offset. |
| `data[].periodOfReport` | string | Reporting period the filing covers, `YYYY-MM-DD`.                  |
| `data[].entities[]`     | array  | Filer blocks. One item per entity in the filing header.            |
| `data[].records[]`      | array  | Fee records. One item per fiscal year, and per audit firm.         |

### Entity

| Field                                    | Type   | Meaning                                               |
| ---------------------------------------- | ------ | ------------------------------------------------------- |
| `data[].entities[].cik`                  | string | Central Index Key of the reporting entity, no leading zeros. |
| `data[].entities[].ticker`               | string | Stock ticker that identifies the entity. It can be an empty string. |
| `data[].entities[].companyName`          | string | Legal name of the issuer as filed. A role follows it, for example `NVIDIA CORP (Filer)`. |
| `data[].entities[].irsNo`                | string | IRS Employer Identification Number of the entity, digits only. |
| `data[].entities[].fiscalYearEnd`        | string | Fiscal year end as four digits, month then day. `0131` is 31 January. |
| `data[].entities[].stateOfIncorporation` | string | US state or country where the entity is incorporated, for example `DE`. |
| `data[].entities[].sic`                  | string | Standard Industrial Classification code of the primary industry, and its name. The name keeps the HTML entity `&amp;`, so it can read `3674 Semiconductors &amp; Related Devices`. |
| `data[].entities[].act`                  | string | Act the entity files its reports under. `34` is the Securities Exchange Act of 1934. |
| `data[].entities[].fileNo`               | string | SEC file number. It ties together the filings of one registration process, for example `001-38377`. |
| `data[].entities[].filmNo`               | string | Film number the SEC assigns to the one filing.        |

`act`, `fileNo` and `filmNo` are absent from some filer blocks. Test for the key
before you read it.

### Fee record

| Field                               | Type   | Meaning                                                       |
| ----------------------------------- | ------ | --------------------------------------------------------------- |
| `data[].records[].year`             | number | Fiscal year of the audit fee record.                          |
| `data[].records[].auditFees`        | number | Fees for the audit of the financial statements and of internal control over financial reporting. |
| `data[].records[].auditRelatedFees` | number | Fees for assurance work reasonably related to the audit or review of the financial statements. `null` when the company reported none. |
| `data[].records[].taxFees`          | number | Fees for tax compliance, tax advice and tax planning. `null` when none. |
| `data[].records[].allOtherFees`     | number | Fees for work outside the audit, audit-related and tax groups. `null` when none. |
| `data[].records[].totalFees`        | number | Sum of all fee groups for the year. `null` when the company reported no total. |
| `data[].records[].auditor`          | string | Name of the independent accounting firm that did the audit, as filed. Spelling is not normalized. `PwC` and `Ernst & Young` both appear. It can be an empty string. |

`null` means "not reported", not zero.

`size` defaults to 50 and caps at 50. Page with `from`. One filing with two fee
records was 706 bytes, so a `size: 50` call stays small.

## Example

Prompt: "What has NVIDIA paid its auditor?"

```json
{ "name": "audit-fees", "arguments": { "query": "entities.ticker:NVDA", "size": 1 } }
```

Trimmed response:

```json
{
  "total": { "value": 22, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001045810-26-000036",
      "formType": "DEF 14A",
      "filedAt": "2026-05-12T16:42:13-04:00",
      "periodOfReport": "2026-06-24",
      "entities": [
        { "cik": "1045810", "ticker": "NVDA", "companyName": "NVIDIA CORP (Filer)", "fiscalYearEnd": "0131" }
      ],
      "records": [
        { "year": 2026, "auditFees": 10166400, "auditRelatedFees": 1716820, "taxFees": 1076103, "allOtherFees": 401999, "totalFees": 13361322, "auditor": "PwC" },
        { "year": 2025, "auditFees": 8067106, "auditRelatedFees": 724806, "taxFees": 856439, "allOtherFees": 354000, "totalFees": 10002351, "auditor": "PwC" }
      ]
    }
  ]
}
```

## Limits and errors

- `size` above 50 returns HTTP 400 with `Maximum 'size' limit of 50 exceeded.`
- Fee history repeats. The same fiscal year appears in this year's proxy and the
  next one. Deduplicate on `entities[].cik` plus `records[].year`.
- Auditor names are free text. Match with a wildcard or a phrase, not an exact
  string.
- One query reaches 10,000 filings at most, because `from` stops at 10,000.
  Split a larger search with a `filedAt` range.
- Small filers can carry an empty `entities.ticker`. Query them by
  `entities.cik`.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`compensation`](./compensation.md),
  [`directors-and-board-members`](./directors-and-board-members.md). The other
  proxy-statement datasets.
- [`aaers`](./aaers.md). Enforcement actions about accounting and auditing.
- [Query language](../query-language.md). Lucene syntax and field names.
- REST documentation:
  [Audit Fees API](https://sec-api.io/docs/audit-fees-api)
