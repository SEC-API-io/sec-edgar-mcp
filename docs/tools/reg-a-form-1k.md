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
to date, with the fees paid to the underwriter, auditor, lawyer and blue-sky
agent.

This tool searches only the 1-K family. One row is one filing.

The index is small. It held exactly **3,000 rows** on 2026-08-13, and the total
came back with `relation: "eq"`, so that is a real count, not a ceiling. Of
those, 2,875 are `1-K` and 125 are `1-K/A`. Coverage starts 2016-04-27.

## When to use it

- How much has this Reg A offering sold since it opened?
- Which Reg A issuers filed an annual report for fiscal 2025?
- What did this issuer pay its auditor and its lawyers?
- Which security series does this issuer have on offer?
- Did the issuer rely on the Rule 257 reporting relief?

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
| `companyName`                       | `companyName:"Arrived STR 2"`                 |
| `formType`                          | `formType:"1-K/A"`                            |
| `filedAt`                           | `filedAt:[2025-01-01 TO 2025-12-31]`          |
| `periodOfReport`                    | `periodOfReport:[2025-01-01 TO 2025-12-31]`   |
| `item1.fiscalYearEnd`               | `item1.fiscalYearEnd:*`                       |
| `item1.stateOrCountry`              | `item1.stateOrCountry:WA`                     |
| `item1Info.jurisdictionOrganization`| `item1Info.jurisdictionOrganization:DE`       |
| `item2.regArule257`                 | `item2.regArule257:true`                      |
| `summaryInfo.issuerNetProceeds`     | `summaryInfo.issuerNetProceeds:[10000000 TO *]` |

`formType` matches the exact string. `formType:"1-K"` does not include `1-K/A`.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. On this index
`relation` is `eq`, so the count is exact.

| Field            | Type   | Meaning                                              |
| ---------------- | ------ | ---------------------------------------------------- |
| `id`             | string | Internal document ID                                 |
| `accessionNo`    | string | EDGAR accession number                               |
| `fileNo`         | string | SEC file number, `24R-` prefixed                     |
| `formType`       | string | `1-K` or `1-K/A`                                     |
| `filedAt`        | string | Filing timestamp with offset                         |
| `periodOfReport` | string | Fiscal year end, `YYYY-MM-DD`                        |
| `cik`            | string | Issuer CIK, no leading zeros                         |
| `ticker`         | string | Usually empty. Most Reg A issuers are unlisted.      |
| `companyName`    | string | Issuer name                                          |
| `item1`          | object | Cover page, see below                                |
| `item1Info[]`    | array  | `issuerName`, `cik` zero-padded, `jurisdictionOrganization`, `irsNum` |
| `item2`          | object | `regArule257`, true when the issuer claims the Rule 257 relief |
| `summaryInfo[]`  | array  | Offering results to date, see below                  |

`item1` holds `formIndication` (for example `Annual Report`), `fiscalYearEnd` in
`MM-DD-YYYY`, the address fields `street1`, `street2`, `city`, `stateOrCountry`,
`zipCode`, `phoneNumber`, and `issuedSecuritiesTitle[]`, the list of security
series on issue. Series issuers list one title per series, so this array can be
long.

`summaryInfo[]` is one entry per commission file number. It holds
`commissionFileNumber`, `offeringQualificationDate`,
`offeringCommenceDate`, `qualifiedSecuritiesSold`, `offeringSecuritiesSold`,
`pricePerSecurity`, `aggregrateOfferingPrice`,
`aggregrateOfferingPriceHolders`, the service-provider pairs
`underwrittenSpName` and `underwriterFees`, `auditorSpName` and `auditorFees`,
`legalSpName` and `legalFees`, `blueSkySpName` and `blueSkyFees`, plus
`crdNumberBrokerDealer`, `issuerNetProceeds` and `clarificationResponses`.

`summaryInfo[]` is optional. It is absent from the example row below. Guard for
it.

Note the misspelling in the API: `aggregrateOfferingPrice`, with an extra `r`.
Copy it exactly.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.
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
