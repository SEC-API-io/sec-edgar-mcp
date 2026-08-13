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
material-weakness flag, and signed oath. Coverage starts in January 2016 and
runs to now. It includes firms that no longer trade.

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

Query fields: `accessionNo`, `formType`, `filedAt`, `periodOfReport`,
`entities.cik`, `entities.fileNo`, `entities.irsNo`,
`entities.stateOfIncorporation`, `entities.fiscalYearEnd`,
`registrantIdentification.brokerDealerName`,
`registrantIdentification.businessAddress.stateOrCountry`,
`accountantIdentification.accountantName`,
`accountantIdentification.accountantType`,
`submissionInformation.materialWeakness`,
`submissionInformation.amendmentDescription`,
`submissionInformation.typeOfRegistrant.typeOfBDRegistrant`,
`submissionInformation.typeOfRegistrant.typeOfSDRegistrant`,
`submissionInformation.subTypeOfRegistrant`,
`submissionInformation.subTypeOfBDRegistrant`,
`submissionInformation.subTypeOfSDRegistrant`,
`oathSignature.signPersonName`, `oathSignature.oathTitle`, and
`oathSignature.confirmNotarizedFlag`.

Working examples: `entities.cik:42352`, `entities.fileNo:"008-15869"`,
`formType:"X-17A-5/A"`, `filedAt:[2020-03-01 TO 2020-03-31]`,
`submissionInformation.materialWeakness:Y`. An unknown field name returns
`total: 0` and an empty `data` array. It raises no error. Check the spelling
when a query returns nothing.

## Output

The envelope is `{total, data[]}`. `total` has `value` and `relation`. `data`
holds one object per filing.

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | -------------------------------------------------------------- |
| `total.value`    | number | Number of X-17A-5 filings that match the query. It stops at `10000`. |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many.  |
| `data[]`         | array  | The matching filings. One item per submission.                  |

### Filing

| Field                            | Type   | Meaning                                                            |
| -------------------------------- | ------ | ------------------------------------------------------------------ |
| `data[].id`                      | string | System-internal identifier of the filing record.                    |
| `data[].accessionNo`             | string | EDGAR accession number of the FOCUS report submission.              |
| `data[].formType`                | string | `X-17A-5` for the original filing, `X-17A-5/A` for an amendment.    |
| `data[].filedAt`                 | string | Time when EDGAR accepted the filing.                                |
| `data[].effectivenessDate`       | string | `YYYY-MM-DD`, the date the filing became effective on EDGAR. It is missing on older records. |
| `data[].periodOfReport`          | string | `YYYY-MM-DD`, the end of the reported period.                       |
| `data[].entities[]`              | array  | Entities named in the filing, usually the broker-dealer or swap dealer registrant. The same CIK can appear twice in one filing. |
| `data[].submissionInformation`   | object | Period, registrant class, amendment text and material-weakness flag. |
| `data[].registrantIdentification`| object | Name, address and contact person of the registrant.                 |
| `data[].accountantIdentification`| object | The independent public accountant whose report goes with the filing. |
| `data[].oathSignature`           | object | Oath an officer signs to attest that the report is true and correct. |

### `data[].entities[]`

| Field                             | Type   | Meaning                                                           |
| --------------------------------- | ------ | ----------------------------------------------------------------- |
| `entities[].companyName`          | string | Legal name of the entity as registered with the SEC, with the EDGAR role suffix `(Filer)`. |
| `entities[].cik`                  | string | Central Index Key of the entity. It comes back with no zero padding, e.g. `42352`. |
| `entities[].irsNo`                | string | IRS employer identification number (EIN) of the entity.            |
| `entities[].stateOfIncorporation` | string | Two-letter state or country code where the entity is incorporated. |
| `entities[].fiscalYearEnd`        | string | Fiscal year end as `MMDD`, e.g. `1231` for 31 December.            |
| `entities[].type`                 | string | Filing type recorded for the entity on this submission.            |
| `entities[].act`                  | string | Act the entity files under. `34` is the Securities Exchange Act of 1934. |
| `entities[].fileNo`               | string | SEC file number. It tracks the registrant across filings.          |
| `entities[].filmNo`               | string | Film number the SEC assigned to this submission.                   |
| `entities[].sic`                  | string | Standard Industrial Classification code of the entity, e.g. `6162` for mortgage bankers. |
| `entities[].tickers`              | string | Ticker of the entity. It is usually empty, because most broker-dealers are not publicly traded. |

### `data[].submissionInformation`

| Field                                                       | Type   | Meaning                                            |
| ----------------------------------------------------------- | ------ | -------------------------------------------------- |
| `submissionInformation.periodBegin`                         | string | Start of the reported period, format `MM-DD-YYYY`.  |
| `submissionInformation.periodEnd`                           | string | End of the reported period, format `MM-DD-YYYY`.    |
| `submissionInformation.materialWeakness`                    | string | `Y` or `N`. It says whether the accountant found a material weakness. |
| `submissionInformation.amendmentDescription`                | string | Free text that says what an `X-17A-5/A` changed against the earlier submission. |
| `submissionInformation.subTypeOfRegistrant`                 | string | Sub-class of the registrant when the filing does not split it by dealer type, e.g. `OTC derivatives dealer`. |
| `submissionInformation.subTypeOfBDRegistrant`               | string | Sub-class of the broker-dealer registrant, e.g. `OTC derivatives dealer`. |
| `submissionInformation.subTypeOfSDRegistrant`               | string | Sub-class of the security-based swap dealer registrant, e.g. `Filing pursuant to a Commission substituted compliance order`. |
| `submissionInformation.typeOfRegistrant`                    | object | Class of the registrant, split between broker-dealer and swap dealer. |
| `submissionInformation.typeOfRegistrant.typeOfBDRegistrant` | string | Broker-dealer registrant type. It holds `Broker-dealer`. |
| `submissionInformation.typeOfRegistrant.typeOfSDRegistrant` | string | Swap dealer registrant type. It holds `Security-based swap dealer`. |

### `data[].registrantIdentification`

| Field                                                | Type   | Meaning                                                    |
| ---------------------------------------------------- | ------ | ---------------------------------------------------------- |
| `registrantIdentification.brokerDealerName`          | string | Legal name of the broker-dealer that filed the FOCUS report. |
| `registrantIdentification.businessAddress`           | object | Primary business address of the registrant.                 |
| `registrantIdentification.businessAddress.street1`   | string | First line of the registrant street address.                |
| `registrantIdentification.businessAddress.street2`   | string | Second line of the street address, such as a floor or suite. |
| `registrantIdentification.businessAddress.city`      | string | City of the business address.                               |
| `registrantIdentification.businessAddress.stateOrCountry` | string | Two-letter state or country code of the business address. |
| `registrantIdentification.businessAddress.zipCode`   | string | Postal code of the business address.                        |
| `registrantIdentification.contactPersonName`         | string | Person at the registrant who answers questions about the report. |
| `registrantIdentification.contactPersonPhoneNumber`  | string | Phone number of that contact person.                        |

### `data[].accountantIdentification`

| Field                                                    | Type   | Meaning                                                |
| -------------------------------------------------------- | ------ | ------------------------------------------------------ |
| `accountantIdentification.accountantName`                | string | Independent public accountant or audit firm, e.g. `PricewaterhouseCoopers LLP`. |
| `accountantIdentification.accountantType`                | string | `Certified Public Accountant`, or `Certified Public Accountant not resident in United States or any of its possessions`. |
| `accountantIdentification.accountantAddress`             | object | Mailing address of the accountant.                      |
| `accountantIdentification.accountantAddress.street1`     | string | First line of the accountant street address.            |
| `accountantIdentification.accountantAddress.street2`     | string | Second line of the street address, such as a floor or suite. |
| `accountantIdentification.accountantAddress.city`        | string | City of the accountant office.                          |
| `accountantIdentification.accountantAddress.stateOrCountry` | string | Two-letter state or country code of the accountant office. |
| `accountantIdentification.accountantAddress.zipCode`     | string | Postal code of the accountant office.                   |

### `data[].oathSignature`

| Field                                | Type   | Meaning                                                        |
| ------------------------------------ | ------ | -------------------------------------------------------------- |
| `oathSignature.signPersonName`       | string | Officer who signed the oath.                                    |
| `oathSignature.signature`            | string | Typed signature string on the oath. It usually repeats `signPersonName`. |
| `oathSignature.entityName`           | string | Entity the oath is given for.                                   |
| `oathSignature.oathTitle`            | string | Title of the officer who executed the oath, e.g. `Chief Financial Officer`. |
| `oathSignature.signDate`             | string | Date the officer signed the oath, format `MM-DD-YYYY`.          |
| `oathSignature.confirmNotarizedFlag` | string | `Y` or `N`. It says whether a notary witnessed the oath.        |
| `oathSignature.explanation`          | string | Free text beside the oath. It often names an exception, or reads `None`. |

A field with no value in the filing is absent from the object. `sic`,
`tickers`, `amendmentDescription`, `explanation` and the three `subType` fields
drop out most often.

`size` defaults to 50 and caps at 50. Page with `from`. `total.value` stops at
`10000` with `relation: "gte"`, so a broad query reports "10,000 or more", not a
true count. A `from` plus `size` above 10,000 returns `total: 0` and an empty
array, with no error. The dataset holds more than 16,500 filings.

## Example

Prompt: "Show the newest FOCUS report filed by any broker-dealer."

```json
{ "name": "form-x-17a-5", "arguments": { "query": "formType:\"X-17A-5\"", "size": 1 } }
```

Trimmed response:

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
