# form-ncen

Search Form N-CEN filings, the annual census report of registered investment
companies.

|                 |                                 |
| --------------- | ------------------------------- |
| Category        | Funds                           |
| Required input  | `query`                         |
| Returns         | `{total, data[]}`               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-ncen`               |

## What it does

The tool searches parsed Form N-CEN filings. Every registered investment company
files N-CEN once a year. That includes mutual funds, ETFs, closed-end funds,
unit investment trusts and insurance separate accounts. One item in `data[]` is
one N-CEN filing. It carries the registrant identity, the share classes covered,
the chief compliance officer, the principal underwriter, the public accountant,
and the sections that fit the registrant type.

The captured filing is a unit investment trust, an insurance separate account.
Its type-specific data sits under `unitInvestmentTrust`. The registry
description also promises adviser, custodian, transfer agent and
securities-lending data. Those sections belong to management investment
companies. They are absent from the capture and from the SDK example, which is
the same filing. **Their field names are not verified.**

## When to use it

- Who audits this fund, and who underwrites it?
- Who is the fund's chief compliance officer?
- Which share classes does this registrant cover?
- Which funds belong to one fund family?
- Which registrants are incorporated in a given state?

## When to use a different tool

| Situation                              | Better tool                             | Why                                                     |
| -------------------------------------- | --------------------------------------- | --------------------------------------------------------- |
| You want what the fund holds           | [`form-nport`](./form-nport.md)         | N-PORT reports positions every quarter. N-CEN does not.    |
| You want how the fund voted            | [`form-npx`](./form-npx.md)             | N-PX carries proxy votes.                                  |
| You want the fund adviser's own filing | [`form-adv-firms`](./form-adv-firms.md) | Form ADV covers the adviser as a firm.                     |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                                      |
| --------- | ------- | -------- | ------------------------ | ----------------------------------------------------------- |
| `query`   | string  | Yes      | Lucene syntax            | For example `entities.cik:1639553`.                          |
| `from`    | integer | No       | Minimum 0                | Offset of the first result. Default 0.                       |
| `size`    | integer | No       | 1 to 50                  | Default 50.                                                  |
| `sort`    | array   | No       | Elasticsearch sort array | Default `[{"filedAt": {"order": "desc"}}]`.                  |

Query fields confirmed to return rows: `entities.cik`, which returned 8 filings
for `1639553`, and `registrantInfo.registrantState`, which returned 5,891
filings for `NY`.

Fields taken from the response shape and the SDK README, all **unverified**:
`accessionNo`, `formType`, `periodOfReport`, `fileNo`,
`registrantInfo.registrantFullName`, `registrantInfo.familyInvCompFullName`. See
[query language](../query-language.md).

Unlike [`form-nport`](./form-nport.md) and [`form-npx`](./form-npx.md), this
handler does not reject a `query` that has no colon. The result of a bare term
query was not verified. Use `field:value` syntax.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`.
The capture returned `{value: 8, relation: "eq"}`, an exact count. A `relation`
of `gte` means the count is capped at 10,000 and the true count is higher.

| Field                                        | Type    | Meaning                                                        |
| -------------------------------------------- | ------- | -------------------------------------------------------------- |
| `accessionNo`                                | string  | Accession number. `id` is the internal record ID.               |
| `fileNo`                                     | string  | Investment Company Act file number, for example `811-23054`.    |
| `formType`, `filedAt`                        | string  | `N-CEN`, and the filing timestamp with offset.                  |
| `periodOfReport`                             | string  | Fiscal year end, `YYYY-MM-DD`.                                  |
| `entities[]`                                 | array   | EDGAR header entities. `cik`, `companyName`, `irsNo`, `fiscalYearEnd`, `stateOfIncorporation`, `act`, `fileNo`, `filmNo`. |
| `seriesClass.reportClass[].classIds[]`       | array   | Share class IDs covered by the report.                          |
| `generalInfo.reportEndingPeriod`             | string  | End of the reporting period. `isReportPeriodLt12` is `true` for a short year. |
| `registrantInfo.registrantFullName`          | string  | Registrant name. `registrantCik` holds the CIK.                 |
| `registrantInfo.registrantLei`               | string  | LEI. Twenty zeros when the registrant has none.                 |
| `registrantInfo.registrantState`             | string  | Two-letter state code. Confirmed as a query field.              |
| `registrantInfo.registrantClassificationType`| string  | Registration form of the registrant, `N-4` in the capture.      |
| `registrantInfo.familyInvCompFullName`       | string  | Fund family name. Use it to group registrants.                  |
| `registrantInfo.locationBooksRecords[]`      | array   | Where the books and records are kept, with address and phone.   |
| `registrantInfo.chiefComplianceOfficers[]`   | array   | `ccoName`, `crdNumber`, address, `isCcoChangedSinceLastFiling`. |
| `registrantInfo.principalUnderwriters[]`     | array   | Name, file number, CRD, LEI, and an affiliation flag.           |
| `registrantInfo.publicAccountants[]`         | array   | `publicAccountantName`, `pcaobNumber`, LEI, state, country.     |
| `registrantInfo.is*` flags                   | boolean | Yes or no answers, for example `isPreviousLegalProceeding`, `isMaterialChange`, `isPublicAccountantChanged`. |
| `unitInvestmentTrust`                        | object  | UIT section. `depositors[]`, `uitAdmins[]`, `numOfContracts`, `contractSecurities[]` with per-contract assets, premiums and redemptions, and rule-reliance flags. |
| `attachmentsTab`                             | object  | Flags for the attachments filed, for example `isLegalProceedings`. |
| `signature`                                  | object  | `registrantSignedName`, `signedDate`, `signature`, `title`.      |

The record is wide but not heavy. The capture was 4.8 KB for one filing. `size`
counts filings. The JSON arrives as one stringified text block. See
[response format](../response-format.md).

## Example

Prompt: "Get the latest N-CEN filing for CIK 1639553."

```json
{ "name": "form-ncen", "arguments": { "query": "entities.cik:1639553", "size": 1 } }
```

```json
{
  "total": { "value": 8, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001639553-26-000002",
      "fileNo": "811-23054",
      "formType": "N-CEN",
      "filedAt": "2026-03-16T17:18:30-04:00",
      "periodOfReport": "2025-12-31",
      "entities": [
        {
          "cik": "1639553",
          "companyName": "Variable Annuity-8 Series Account (of Empower Life & Annuity Insurance Co of New York) (Filer)"
        }
      ],
      "generalInfo": { "reportEndingPeriod": "2025-12-31", "isReportPeriodLt12": false },
      "registrantInfo": {
        "registrantFullName": "Variable Annuity-8 Series Account (of Empower Life & Annuity Insurance Co of New York)",
        "registrantState": "NY",
        "familyInvCompFullName": "Empower Funds, Inc.",
        "registrantClassificationType": "N-4"
      }
    }
  ]
}
```

Keys were removed to fit. The values are unchanged.

## Limits and errors

- `size` above 50 fails with HTTP 400 and `Maximum 'size' limit of 50 exceeded`.
- Omitting `size` returns 50 records, not 10.
- Sections vary by registrant type. A management company filing does not carry
  `unitInvestmentTrust`. Read the keys you get, do not assume them.
- The `ccoPhone` value was masked as `XXXXXX` in the capture.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-nport`](./form-nport.md), [`form-npx`](./form-npx.md),
  [`form-npx-file`](./form-npx-file.md)
- REST documentation:
  [Form N-CEN API](https://sec-api.io/docs/form-ncen-api-annual-reports-investment-companies)
