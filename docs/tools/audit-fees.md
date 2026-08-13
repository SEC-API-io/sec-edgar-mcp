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

One row is one filing, not one fee figure. Each row carries a `records[]` array
with one entry per fiscal year, so a single proxy statement gives you the
current year and the prior year side by side. The capture asked for NVIDIA and
got 22 filings, the newest holding fees for 2026 and 2025, both audited by PwC.

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
| `from`    | integer | No       | minimum 0          | Offset. Default 0.                                           |
| `size`    | integer | No       | 1 to 50            | Default 50. Above 50 the server returns HTTP 400.            |
| `sort`    | array   | No       | Elasticsearch sort | Default `[{"filedAt": {"order": "desc"}}]`.                  |

Query fields verified live on 2026-08-13: `entities.ticker`, `entities.cik`,
`formType`, `records.auditor`, `records.totalFees`.

Note the shape. The company identifiers sit under `entities`, so the ticker
field is `entities.ticker`, not bare `ticker`. Other tools differ. See
[query language](../query-language.md).

Ranges work on the fee fields. `records.totalFees:[10000000 TO *]` returned
3,271 filings.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`,
not a number.

| Field                        | Type   | Meaning                                                              |
| ---------------------------- | ------ | -------------------------------------------------------------------- |
| `total.value`                | number | Number of matching filings. `10000` with `relation: "gte"` means "10,000 or more". |
| `data[].accessionNo`         | string | Accession number of the proxy statement.                              |
| `data[].formType`            | string | Form that carried the disclosure, usually `DEF 14A`.                  |
| `data[].filedAt`             | string | Filing timestamp, ISO 8601 with an offset.                            |
| `data[].periodOfReport`      | string | Period of the filing, `YYYY-MM-DD`. For a proxy this is the meeting date. |
| `data[].entities[]`          | array  | Filer blocks. Each holds `cik`, `ticker`, `companyName`, `irsNo`, `fiscalYearEnd`, `stateOfIncorporation`, `sic`. |
| `data[].records[]`           | array  | One entry per fiscal year reported in that filing.                    |
| `records[].year`             | number | Fiscal year the fees belong to.                                       |
| `records[].auditFees`        | number | Audit fees in dollars.                                                |
| `records[].auditRelatedFees` | number | Audit-related fees. `null` when the company reported none.            |
| `records[].taxFees`          | number | Tax fees. `null` when none.                                           |
| `records[].allOtherFees`     | number | All other fees. `null` when none.                                     |
| `records[].totalFees`        | number | Sum reported by the company.                                          |
| `records[].auditor`          | string | Audit firm name as filed. Spelling is not normalized. `PwC` and `Ernst & Young` both appear. |

`null` means "not reported", not zero. The `sic` string keeps the HTML entity
`&amp;`, so it can read `3674 Semiconductors &amp; Related Devices`.

`size` defaults to 50 and caps at 50. Page with `from`. One filing with two fee
records was 706 bytes, so a `size: 50` call stays small.

## Example

Prompt: "What has NVIDIA paid its auditor?"

```json
{ "name": "audit-fees", "arguments": { "query": "entities.ticker:NVDA", "size": 1 } }
```

Trimmed response from the capture:

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
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`compensation`](./compensation.md),
  [`directors-and-board-members`](./directors-and-board-members.md). The other
  proxy-statement datasets.
- [`aaers`](./aaers.md). Enforcement actions about accounting and auditing.
- [Query language](../query-language.md). Lucene syntax and field names.
- REST documentation:
  [Audit Fees API](https://sec-api.io/docs/audit-fees-api)
