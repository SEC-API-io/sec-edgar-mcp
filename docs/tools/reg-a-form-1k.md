# reg-a-form-1k

Search Form 1-K annual reports, filed each year by companies with a live
Regulation A offering.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /reg-a/form-1k`                        |

## What it does

A Tier 2 Regulation A issuer files a Form 1-K every year. It reports the fiscal
year, the securities on issue, and how much of the qualified offering has sold
to date, with the fees paid to seven service-provider roles: underwriter, sales
commissions, finder, auditor, lawyer, promoter and blue-sky agent.

This tool searches only the 1-K family. One row is one filing.

The index is small. It held exactly **3,000 rows** on 2026-08-13, and the total
came back with `relation: "eq"`, so that is a real count, not a ceiling. Of
those, 2,875 are `1-K` and 125 are `1-K/A`. Coverage starts 2016-04-27.

## When to use it

- How much has this Reg A offering sold since it opened?
- Which Reg A issuers filed an annual report for fiscal 2025?
- What did this issuer pay its auditor and its lawyers?
- Which security series does this issuer have on offer?
- Which issuers flag Regulation A Rule 257?

## When to use a different tool

| Situation                                | Better tool                         | Why                                      |
| ---------------------------------------- | ----------------------------------- | ---------------------------------------- |
| You want 1-A, 1-K and 1-Z in one search  | [reg-a-search](./reg-a-search.md)   | Queries all three indices together       |
| You want the opening offering terms      | [reg-a-form-1a](./reg-a-form-1a.md) | The 1-A has the tier and the price       |
| You want the final closed-out numbers    | [reg-a-form-1z](./reg-a-form-1z.md) | The exit report is the last word         |
| You want the annual report narrative     | [extractor](./extractor.md)         | Returns the document sections as text    |
| You want a 10-K, not a 1-K               | [filing-search](./filing-search.md) | 1-K is the Reg A form, 10-K is the full one |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

| Field                              | Example                                       |
| ---------------------------------- | --------------------------------------------- |
| `cik`                               | `cik:1968039`                                 |
| `accessionNo`                       | `accessionNo:"0001213900-26-088286"`          |
| `companyName`                       | `companyName:"Arrived STR 2"`                 |
| `ticker`                            | `ticker:BSFC`                                 |
| `fileNo`                            | `fileNo:"24R-00881"`                          |
| `formType`                          | `formType:"1-K/A"`                            |
| `filedAt`                           | `filedAt:[2025-01-01 TO 2025-12-31]`          |
| `periodOfReport`                    | `periodOfReport:[2025-01-01 TO 2025-12-31]`   |
| `item1.fiscalYearEnd`               | `item1.fiscalYearEnd:*`                       |
| `item1.stateOrCountry`              | `item1.stateOrCountry:WA`                     |
| `item1Info.jurisdictionOrganization`| `item1Info.jurisdictionOrganization:DE`       |
| `item2.regArule257`                 | `item2.regArule257:true`                      |
| `summaryInfo.offeringSecuritiesSold`| `summaryInfo.offeringSecuritiesSold:[1000 TO *]` |
| `summaryInfo.issuerNetProceeds`     | `summaryInfo.issuerNetProceeds:[10000000 TO *]` |

`formType` matches the exact string. `formType:"1-K"` does not include `1-K/A`.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. On this index
`relation` is `eq`, so the count is exact.

One element of `data[]` is one filing.

### Filing

| Field                   | Type   | Meaning                                                                                                       |
| ----------------------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| `data[].id`             | string | Internal identifier of the filing record.                                                                     |
| `data[].accessionNo`    | string | EDGAR accession number of the 1-K.                                                                            |
| `data[].fileNo`         | string | SEC file number. It ties together every filing of the same process. Reg A annual reports carry a `24R-` prefix. |
| `data[].formType`       | string | `1-K` or `1-K/A`.                                                                                             |
| `data[].filedAt`        | string | Time EDGAR accepted the filing.                                                                               |
| `data[].periodOfReport` | string | Reporting period the filing covers, `YYYY-MM-DD`.                                                             |
| `data[].cik`            | string | Central Index Key of the reporting entity, no leading zeros.                                                  |
| `data[].ticker`         | string | Stock ticker, if the issuer trades publicly. Usually empty. Most Reg A issuers are unlisted.                   |
| `data[].companyName`    | string | Legal name of the issuer as given in the filing.                                                              |
| `data[].item1`          | object | Cover page of the report.                                                                                     |
| `data[].item1Info[]`    | array  | Issuer identity. One entry per issuer named on the cover page.                                                |
| `data[].item2`          | object | Rule 257 certification.                                                                                       |
| `data[].summaryInfo[]`  | array  | Offering results to date. One entry per commission file number. Optional.                                     |

### Cover page, `item1`

| Field                                 | Type     | Meaning                                                                                                          |
| ------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------- |
| `data[].item1.formIndication`         | string   | Report type. `Annual Report` or `Special Financial Report for the fiscal year`.                                   |
| `data[].item1.fiscalYearEnd`          | string   | End date of the fiscal year the filing covers, `MM-DD-YYYY`.                                                     |
| `data[].item1.street1`                | string   | Street address of the principal executive offices.                                                               |
| `data[].item1.street2`                | string   | Second address line of the principal executive offices, such as a suite.                                         |
| `data[].item1.city`                   | string   | City of the principal executive offices.                                                                         |
| `data[].item1.stateOrCountry`         | string   | State or country of the issuer address.                                                                          |
| `data[].item1.zipCode`                | string   | Postal code of the principal executive offices.                                                                  |
| `data[].item1.phoneNumber`            | string   | Contact phone number of the principal executive offices.                                                         |
| `data[].item1.issuedSecuritiesTitle[]`| string[] | Titles of the security classes issued under Regulation A. A series issuer lists one title per series, so this array can be long. |

### Issuer identity, `item1Info[]`

| Field                                       | Type   | Meaning                                                       |
| ------------------------------------------- | ------ | ------------------------------------------------------------- |
| `data[].item1Info[].issuerName`             | string | Official issuer name as registered with the SEC.              |
| `data[].item1Info[].cik`                    | string | Central Index Key of the reporting entity, zero-padded to ten digits. |
| `data[].item1Info[].jurisdictionOrganization`| string | Jurisdiction where the issuer is incorporated or organized.   |
| `data[].item1Info[].irsNum`                 | string | Internal Revenue Service identification number of the issuer. |

### Rule 257, `item2`

| Field                        | Type    | Meaning                                                        |
| ---------------------------- | ------- | -------------------------------------------------------------- |
| `data[].item2.regArule257`   | boolean | Whether the issuer complies with Regulation A Rule 257.        |

### Offering results, `summaryInfo[]`

| Field                                            | Type     | Meaning                                                              |
| ------------------------------------------------ | -------- | --------------------------------------------------------------------- |
| `data[].summaryInfo[].commissionFileNumber`       | string   | SEC commission file number of the offering statement.                |
| `data[].summaryInfo[].offeringQualificationDate`  | string   | Date the SEC qualified the offering statement, `MM-DD-YYYY`.         |
| `data[].summaryInfo[].offeringCommenceDate`       | string   | Date the offering started, `MM-DD-YYYY`.                             |
| `data[].summaryInfo[].qualifiedSecuritiesSold`    | number   | Quantity of securities qualified for sale in the offering.           |
| `data[].summaryInfo[].offeringSecuritiesSold`     | number   | Quantity of securities sold in the offering.                         |
| `data[].summaryInfo[].pricePerSecurity`           | number   | Unit price of each security offered.                                 |
| `data[].summaryInfo[].aggregrateOfferingPrice`    | number   | Total sales price of the securities the issuer sold.                 |
| `data[].summaryInfo[].aggregrateOfferingPriceHolders` | number | Total sales price of the securities sold for selling securityholders. |
| `data[].summaryInfo[].underwrittenSpName[]`       | string[] | Names of the underwriters in the offering.                           |
| `data[].summaryInfo[].underwriterFees`            | number   | Fees the underwriters charged.                                       |
| `data[].summaryInfo[].salesCommissionsSpName[]`   | string[] | Names of the providers paid sales commissions.                       |
| `data[].summaryInfo[].salesCommissionsFee`        | number   | Amount paid as sales commission.                                     |
| `data[].summaryInfo[].findersSpName[]`            | string[] | Names of the finders in the offering.                                |
| `data[].summaryInfo[].findersFees`                | number   | Amount paid for finders' services.                                   |
| `data[].summaryInfo[].auditorSpName[]`            | string[] | Names of the auditors of the financial statements.                   |
| `data[].summaryInfo[].auditorFees`                | number   | Amount paid for audit services.                                      |
| `data[].summaryInfo[].legalSpName[]`              | string[] | Names of the law firms or counsel engaged.                           |
| `data[].summaryInfo[].legalFees`                  | number   | Amount paid for legal services.                                      |
| `data[].summaryInfo[].promoterSpName[]`           | string[] | Names of the promoters in the offering.                              |
| `data[].summaryInfo[].promotersFees`              | number   | Amount paid for promotional services.                                |
| `data[].summaryInfo[].blueSkySpName[]`            | string[] | Names of the blue-sky providers. Blue sky is the state-level securities compliance work. |
| `data[].summaryInfo[].blueSkyFees`                | number   | Amount paid for blue-sky compliance.                                 |
| `data[].summaryInfo[].crdNumberBrokerDealer`      | string   | CRD number of the broker-dealer in the offering.                     |
| `data[].summaryInfo[].issuerNetProceeds`          | number   | Proceeds the issuer kept after fees and expenses.                    |
| `data[].summaryInfo[].clarificationResponses`     | string   | Free-text notes the filer added about the offering figures.          |

`summaryInfo[]` is optional. It is absent from the example row below. Guard for
it.

Note the misspelling in the API: `aggregrateOfferingPrice`, with an extra `r`.
Copy it exactly.

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.
With only 3,000 rows in the index, that ceiling never bites here.

## Example

Prompt: "Show me the newest Regulation A annual reports."

```json
{ "name": "reg-a-form-1k", "arguments": { "query": "cik:*", "size": 1 } }
```

Response, trimmed for length:

```json
{
  "total": { "value": 3000, "relation": "eq" },
  "data": [
    {
      "id": "37e34f4ccd78a155c0d51a94859de774",
      "accessionNo": "0001213900-26-088286",
      "fileNo": "24R-00881",
      "formType": "1-K/A",
      "filedAt": "2026-08-12T15:21:43-04:00",
      "periodOfReport": "2025-12-31",
      "cik": "1968039",
      "ticker": "",
      "companyName": "Arrived STR 2, LLC",
      "item1": {
        "formIndication": "Annual Report",
        "fiscalYearEnd": "12-31-2025",
        "street1": "1700 Westlake Ave North",
        "city": "Seattle",
        "stateOrCountry": "WA",
        "zipCode": "98109",
        "issuedSecuritiesTitle": ["Arrived Series Pinkshell", "Arrived Series Alta", "Arrived Series Vita"]
      },
      "item1Info": [
        { "issuerName": "ARRIVED STR 2, LLC", "cik": "0001968039", "jurisdictionOrganization": "DE", "irsNum": "92-1716225" }
      ],
      "item2": { "regArule257": true }
    }
  ]
}
```

The full `issuedSecuritiesTitle` held 12 series. Three are shown here.

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error.
- The index is small, 3,000 rows. A broad query returns an exact `total`, not a
  ceiling.
- `summaryInfo[]` is optional and was absent from the example row.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-search](./reg-a-search.md), [reg-a-form-1a](./reg-a-form-1a.md), [reg-a-form-1z](./reg-a-form-1z.md)
- [filing-search](./filing-search.md), [extractor](./extractor.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
