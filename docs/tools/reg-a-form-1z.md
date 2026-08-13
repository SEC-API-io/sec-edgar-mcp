# reg-a-form-1z

Search Form 1-Z exit reports, the filing that closes a Regulation A offering.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /reg-a/form-1z`                        |

## What it does

A Regulation A issuer files a Form 1-Z when the offering ends. The form reports
the final result: how many securities were qualified, how many actually sold, at
what price, what every service provider was paid, and how much the issuer kept.
It also certifies the suspension of the reporting duty and gives the record
holder count.

This tool searches only the 1-Z family. One row is one filing.

The index is the smallest in this category. It held **648 rows** on 2026-08-13,
with `relation: "eq"`, so that is a real count. Of those, 630 are `1-Z` and 12
are `1-Z/A`. Coverage starts 2015-11-02.

Because a 1-Z is the last word on an offering, it is the cleanest source for
"how much did this Reg A raise in the end".

## When to use it

- Which Reg A offerings closed this year, and how much did each raise?
- What share of the qualified securities actually sold?
- What did this issuer keep after underwriter, legal and blue-sky fees?
- How many record holders does the issuer have at close?
- Which auditors sign off on the most Reg A exits?

## When to use a different tool

| Situation                                | Better tool                         | Why                                        |
| ---------------------------------------- | ----------------------------------- | ------------------------------------------ |
| You want 1-A, 1-K and 1-Z in one search  | [reg-a-search](./reg-a-search.md)   | Queries all three indices together         |
| You want the opening offering terms      | [reg-a-form-1a](./reg-a-form-1a.md) | The 1-A has the tier, size and price       |
| You want progress before the close       | [reg-a-form-1k](./reg-a-form-1k.md) | The annual report tracks sales year by year |
| The offering has not closed              | [reg-a-form-1a](./reg-a-form-1a.md) | No 1-Z exists until the offering ends      |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

| Field                                        | Example                                      |
| -------------------------------------------- | -------------------------------------------- |
| `cik`                                         | `cik:2000719`                                |
| `accessionNo`                                 | `accessionNo:"0001096906-26-001033"`         |
| `companyName`                                 | `companyName:"PARADYME FUND A II"`           |
| `ticker`                                      | `ticker:BSFC`                                |
| `fileNo`                                      | `fileNo:"24R-00955"`                         |
| `formType`                                    | `formType:"1-Z"`                             |
| `filedAt`                                     | `filedAt:[2025-01-01 TO 2025-12-31]`         |
| `item1.stateOrCountry`                        | `item1.stateOrCountry:AZ`                    |
| `item1.commissionFileNumber`                  | `item1.commissionFileNumber:"024-12449"`     |
| `summaryInfoOffering.issuerNetProceeds`       | `...issuerNetProceeds:[1000000 TO *]`        |
| `summaryInfoOffering.auditorSpName`           | `...auditorSpName:"IndigoSpire"`             |
| `certificationSuspension.approxRecordHolders` | `...approxRecordHolders:[100 TO *]`          |

`formType` matches the exact string. `formType:"1-Z"` returns 630 rows and does
not include the 12 `1-Z/A` amendments.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. On this index
`relation` is `eq`, so the count is exact.

One element of `data[]` is one filing.

### Filing

| Field                              | Type   | Meaning                                                                                                  |
| ---------------------------------- | ------ | -------------------------------------------------------------------------------------------------------- |
| `data[].id`                        | string | Internal identifier of the filing record.                                                                |
| `data[].accessionNo`               | string | EDGAR accession number of the 1-Z.                                                                       |
| `data[].fileNo`                    | string | SEC file number. It ties together every filing of the same process. Values carry a `24R-` or `024-` prefix. |
| `data[].formType`                  | string | `1-Z` or `1-Z/A`.                                                                                        |
| `data[].filedAt`                   | string | Time EDGAR accepted the filing.                                                                          |
| `data[].periodOfReport`            | string | Reporting period the filing covers, `YYYY-MM-DD`. Optional.                                              |
| `data[].cik`                       | string | Central Index Key of the reporting entity, no leading zeros.                                             |
| `data[].ticker`                    | string | Stock ticker, if the issuer trades publicly. Usually empty. Most Reg A issuers are unlisted.              |
| `data[].companyName`               | string | Legal name of the issuer as given in the filing.                                                         |
| `data[].item1`                     | object | Cover page of the exit report.                                                                           |
| `data[].summaryInfoOffering[]`     | array  | Final result of the offering. One entry per offering.                                                    |
| `data[].certificationSuspension[]` | array  | Certification that the reporting duty stops. One entry per class of securities.                          |
| `data[].signatureTab[]`            | array  | Signature block. One entry per signatory.                                                                |

### Cover page, `item1`

| Field                                 | Type     | Meaning                                                             |
| ------------------------------------- | -------- | -------------------------------------------------------------------- |
| `data[].item1.issuerName`             | string   | Legal name of the issuer as written in its charter.                 |
| `data[].item1.street1`                | string   | Street address of the principal executive office.                   |
| `data[].item1.street2`                | string   | Second address line of the principal executive office, such as a suite. |
| `data[].item1.city`                   | string   | City of the principal executive office.                             |
| `data[].item1.stateOrCountry`         | string   | State or country of the principal executive office.                 |
| `data[].item1.zipCode`                | string   | Postal code of the principal executive office.                      |
| `data[].item1.phone`                  | string   | Contact phone number of the principal executive office.             |
| `data[].item1.commissionFileNumber[]` | string[] | Commission file numbers the SEC assigned to the issuer.             |

The key is `phone` here, not `phoneNumber` as on
[reg-a-form-1k](./reg-a-form-1k.md).

### Final offering result, `summaryInfoOffering[]`

| Field                                                          | Type     | Meaning                                                        |
| -------------------------------------------------------------- | -------- | --------------------------------------------------------------- |
| `data[].summaryInfoOffering[].offeringQualificationDate`        | string   | Date the SEC qualified the offering statement, `MM-DD-YYYY`.   |
| `data[].summaryInfoOffering[].offeringCommenceDate`             | string   | Date the offering started, `MM-DD-YYYY`.                       |
| `data[].summaryInfoOffering[].offeringSecuritiesQualifiedSold`  | number   | Quantity of securities qualified for sale in the offering.     |
| `data[].summaryInfoOffering[].offeringSecuritiesSold`           | number   | Quantity of securities sold in the offering.                   |
| `data[].summaryInfoOffering[].pricePerSecurity`                 | number   | Price of each security in the offering.                        |
| `data[].summaryInfoOffering[].portionSecuritiesSoldIssuer`      | number   | Share of the sales that came from the issuer's own selling effort. |
| `data[].summaryInfoOffering[].portionSecuritiesSoldSecurityholders` | number | Share of the sales that came from selling securityholders.    |
| `data[].summaryInfoOffering[].underwrittenSpName[]`             | string[] | Names of the underwriters in the offering.                     |
| `data[].summaryInfoOffering[].underwriterFees`                  | number   | Fees the underwriters charged.                                 |
| `data[].summaryInfoOffering[].salesCommissionsSpName[]`         | string[] | Names of the providers paid sales commissions.                 |
| `data[].summaryInfoOffering[].salesCommissionsFee`              | number   | Amount paid as sales commission.                               |
| `data[].summaryInfoOffering[].findersSpName[]`                  | string[] | Names of the finders in the offering.                          |
| `data[].summaryInfoOffering[].findersFees`                      | number   | Amount paid for finders' services.                             |
| `data[].summaryInfoOffering[].auditorSpName[]`                  | string[] | Names of the auditors engaged for the offering.                |
| `data[].summaryInfoOffering[].auditorFees`                      | number   | Amount paid for audit services.                                |
| `data[].summaryInfoOffering[].legalSpName[]`                    | string[] | Names of the law firms or counsel engaged.                     |
| `data[].summaryInfoOffering[].legalFees`                        | number   | Amount paid for legal services.                                |
| `data[].summaryInfoOffering[].promoterSpName[]`                 | string[] | Names of the promoters in the offering.                        |
| `data[].summaryInfoOffering[].promotersFees`                    | number   | Amount paid for promotional services.                          |
| `data[].summaryInfoOffering[].blueSkySpName[]`                  | string[] | Names of the blue-sky providers. Blue sky is the state-level securities compliance work. |
| `data[].summaryInfoOffering[].blueSkyFees`                      | number   | Amount paid for blue-sky compliance.                           |
| `data[].summaryInfoOffering[].crdNumberBrokerDealer`            | string   | CRD number of the broker-dealer in the offering.               |
| `data[].summaryInfoOffering[].issuerNetProceeds`                | number   | Proceeds the issuer kept after fees and expenses.              |
| `data[].summaryInfoOffering[].clarificationResponses`           | string   | Free-text notes the filer added about the offering figures.    |

Filers write `"None"` or `"-"` in the `*SpName[]` arrays when a role was unused.
Those two values are not company names.

### Suspension certificate, `certificationSuspension[]`

| Field                                                     | Type     | Meaning                                                             |
| --------------------------------------------------------- | -------- | -------------------------------------------------------------------- |
| `data[].certificationSuspension[].securitiesClassTitle`    | string   | Title of the class of securities the exit report covers.            |
| `data[].certificationSuspension[].certificationFileNumber[]`| string[] | Commission file numbers tied to the certification of suspension.    |
| `data[].certificationSuspension[].approxRecordHolders`     | number   | Approximate number of record holders on the date of certification.  |

### Signatures, `signatureTab[]`

| Field                                            | Type   | Meaning                                                     |
| ------------------------------------------------ | ------ | ------------------------------------------------------------ |
| `data[].signatureTab[].cik`                      | string | Central Index Key of the reporting entity, zero-padded to ten digits. |
| `data[].signatureTab[].regulationIssuerName1`    | string | Primary issuer name as written in the signature block.      |
| `data[].signatureTab[].regulationIssuerName2`    | string | Secondary issuer name, if the filer gave one.               |
| `data[].signatureTab[].signatureBy`              | string | Name of the person who signed for the issuer.               |
| `data[].signatureTab[].date`                     | string | Date of signature, `MM-DD-YYYY`.                            |
| `data[].signatureTab[].title`                    | string | Role of the signatory inside the issuer.                    |

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.
With 648 rows in the index, that ceiling never bites here.

## Example

Prompt: "Show me the most recent Regulation A exit reports."

```json
{ "name": "reg-a-form-1z", "arguments": { "query": "cik:*", "size": 1 } }
```

Response, trimmed for length:

```json
{
  "total": { "value": 648, "relation": "eq" },
  "data": [
    {
      "id": "df8a55ecc5caa1fd32be00f615d80ce1",
      "accessionNo": "0001096906-26-001033",
      "fileNo": "24R-00955",
      "formType": "1-Z",
      "filedAt": "2026-07-01T18:34:16-04:00",
      "cik": "2000719",
      "ticker": "",
      "companyName": "PARADYME FUND A II, LLC",
      "item1": {
        "issuerName": "PARADYME FUND A II, LLC",
        "city": "Lake Havasu City",
        "stateOrCountry": "AZ",
        "commissionFileNumber": ["024-12449"]
      },
      "summaryInfoOffering": [
        {
          "offeringQualificationDate": "08-12-2024",
          "offeringSecuritiesQualifiedSold": 15000,
          "offeringSecuritiesSold": 2382,
          "pricePerSecurity": 1000,
          "auditorSpName": ["IndigoSpire"],
          "auditorFees": 12500,
          "legalFees": 50000,
          "blueSkyFees": 12000,
          "issuerNetProceeds": 2382000
        }
      ],
      "certificationSuspension": [
        { "securitiesClassTitle": "Class A Series Interests", "approxRecordHolders": 124 }
      ]
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error.
- The index holds 648 rows. A broad query returns an exact `total`.
- Dates inside `summaryInfoOffering[]` and `signatureTab[]` use `MM-DD-YYYY`.
  Only the top-level `filedAt` is ISO. The two formats differ.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-search](./reg-a-search.md), [reg-a-form-1a](./reg-a-form-1a.md), [reg-a-form-1k](./reg-a-form-1k.md)
- [form-d](./form-d.md), [form-c](./form-c.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
