# form-x-17a-5

Search structured Form X-17A-5 filings, the FOCUS reports of SEC-registered
broker-dealers.

|                 |                                          |
| --------------- | ---------------------------------------- |
| Category        | Broker-dealers                           |
| Required input  | `query`                                  |
| Returns         | `{total, data[]}`                        |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`          |
| REST equivalent | `POST /form-x-17a-5`                     |

## What it does

Form X-17A-5 is the FOCUS report. Every SEC-registered broker-dealer and
security-based swap dealer files it under Rule 17a-5. One row is one X-17A-5
submission on EDGAR. The row carries the facing-page data: filer entity,
reporting period, registrant name and address, independent accountant,
material-weakness flag, and signed oath. Coverage starts in January 2016. The
earliest record seen has `filedAt` `2016-01-28`.

The row holds **no financial figures**. There is no net capital computation, no
balance sheet, and no income statement. The registry description says the tool
returns "financial condition data". The response contradicts that. The audited
statements sit in the filing documents. Read them with
[`get-edgar-file`](./get-edgar-file.md).

## When to use it

- Which broker-dealers filed a FOCUS report for the period ending 2025-12-31?
- Which firms disclosed a material weakness in internal control?
- Who audits this broker-dealer, and did the audit firm change?
- Which broker-dealers are also registered as security-based swap dealers?

## When to use a different tool

| Situation                                       | Better tool                             | Why                                                          |
| ----------------------------------------------- | --------------------------------------- | ------------------------------------------------------------ |
| You want the audited financial statements       | [`get-edgar-file`](./get-edgar-file.md) | The numbers live in the filing documents, not in this index.   |
| You want every filing a broker-dealer made      | [`filing-search`](./filing-search.md)   | It searches all EDGAR form types, not X-17A-5 alone.           |
| The firm is an investment adviser, not a dealer | [`form-adv-firms`](./form-adv-firms.md) | Advisers file Form ADV. They file no FOCUS report.             |

## Input

| Parameter | Type    | Required | Constraints        | Notes                                                      |
| --------- | ------- | -------- | ------------------ | ---------------------------------------------------------- |
| `query`   | string  | yes      | Lucene syntax      | `field:value`. `AND`, `OR`, and ranges work.               |
| `from`    | integer | no       | minimum 0          | Offset. Default 0. `from` plus `size` must be 10,000 or less. |
| `size`    | integer | no       | 1 to 50            | Default 50. Above 50 the server returns HTTP 400.          |
| `sort`    | array   | no       | Elasticsearch sort | Default `[{"filedAt": {"order": "desc"}}]`.                |

These query fields returned rows on 2026-08-13. The probe confirmed `*:*`
through MCP. The rest were confirmed on the REST endpoint, which serves the same
index: `entities.cik`,
`entities.fileNo`, `formType`, `filedAt`, `periodOfReport`,
`registrantIdentification.brokerDealerName`,
`accountantIdentification.accountantName`,
`submissionInformation.materialWeakness`, and
`submissionInformation.typeOfRegistrant.typeOfBDRegistrant`.

Working examples: `entities.cik:42352`, `entities.fileNo:"008-15869"`,
`formType:"X-17A-5/A"`, `filedAt:[2020-03-01 TO 2020-03-31]`,
`submissionInformation.materialWeakness:Y`. An unknown field name returns
`total: 0` and an empty `data` array. It raises no error. Check the spelling
when a query returns nothing.

## Output

The envelope is `{total, data[]}`. `total` has `value` and `relation`. `data`
holds one object per filing.

| Field                                       | Type   | Meaning                                                     |
| ------------------------------------------- | ------ | ----------------------------------------------------------- |
| `accessionNo`                               | string | EDGAR accession number of the submission.                   |
| `formType`                                  | string | `X-17A-5`, or `X-17A-5/A` for an amendment.                 |
| `filedAt`                                   | string | Acceptance timestamp with a UTC offset. `effectivenessDate` is the EDGAR effective date, and is missing on older records. |
| `periodOfReport`                            | string | `YYYY-MM-DD`, the end of the reported period.               |
| `entities[]`                                | array  | Filer blocks. The same CIK can appear twice in one filing.  |
| `entities[].cik`                            | string | CIK as filed, with no zero padding, e.g. `42352`.           |
| `entities[].companyName`                    | string | Legal name with the EDGAR role suffix `(Filer)`.            |
| `entities[].fileNo`                         | string | SEC file number. Each block also has `irsNo`, `stateOfIncorporation`, `fiscalYearEnd`, `act`, `filmNo`, `type`. |
| `submissionInformation.periodBegin`         | string | Period start, format `MM-DD-YYYY`. `periodEnd` closes it.   |
| `submissionInformation.materialWeakness`    | string | `Y` or `N`. `amendmentDescription` says what an `X-17A-5/A` changed. |
| `submissionInformation.typeOfRegistrant`    | object | `typeOfBDRegistrant` and `typeOfSDRegistrant`.              |
| `registrantIdentification.brokerDealerName` | string | Legal name of the registrant. `businessAddress`, `contactPersonName`, and `contactPersonPhoneNumber` sit beside it. |
| `accountantIdentification.accountantName`   | string | Audit firm, e.g. `PricewaterhouseCoopers LLP`. `accountantType` is `Certified Public Accountant` or the non-resident variant. |
| `oathSignature`                             | object | `signPersonName`, `signature`, `oathTitle`, `signDate`, `confirmNotarizedFlag`, `explanation`. |

The sec-api.io reference also lists `entities[].sic`, `entities[].tickers`, and
the `submissionInformation.subTypeOf*` fields. None appeared in the sampled
records, so treat them as unverified.

`size` defaults to 50 and caps at 50. Page with `from`. `total.value` stops at
`10000` with `relation: "gte"`, so a broad query reports "10,000 or more", not a
true count. A `from` plus `size` above 10,000 returns `total: 0` and an empty
array, with no error. The dataset holds about 16,500 filings.

## Example

Prompt: "Show the newest FOCUS report filed by any broker-dealer."

```json
{ "name": "form-x-17a-5", "arguments": { "query": "*:*", "size": 1 } }
```

Trimmed response from the capture:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "accessionNo": "0001876509-26-000001",
      "formType": "X-17A-5",
      "filedAt": "2026-08-12T17:50:20-04:00",
      "periodOfReport": "2026-06-30",
      "effectivenessDate": "2026-08-12",
      "entities": [
        { "companyName": "RMB SECURITIES (USA) INC. (Filer)", "cik": "1876509", "fileNo": "008-70765" }
      ],
      "submissionInformation": {
        "periodBegin": "07-01-2025", "periodEnd": "06-30-2026", "materialWeakness": "N",
        "typeOfRegistrant": { "typeOfBDRegistrant": "Broker-dealer" }
      },
      "registrantIdentification": { "brokerDealerName": "RMB SECURITIES (USA) INC." },
      "accountantIdentification": { "accountantName": "Rayfield & Licata, PC" }
    }
  ]
}
```

The address blocks and the `oathSignature` object were removed to fit.

## Limits and errors

- `size` above 50 returns HTTP 400 with `Maximum 'size' limit of 50 exceeded.`
- The REST route allows 3 requests per second.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`filing-search`](./filing-search.md). Find X-17A-5 filings among all form types.
- [`get-edgar-file`](./get-edgar-file.md). Read the statements attached to the filing.
- [`sro`](./sro.md). Rule filings by FINRA and the exchanges.
- [`form-adv-firms`](./form-adv-firms.md). The adviser-side register.
- [Form X-17A-5 API reference](https://sec-api.io/docs/form-x-17a-5-focus-report-api)
