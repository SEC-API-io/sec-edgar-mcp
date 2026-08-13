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
| `companyName`                                 | `companyName:"PARADYME FUND A II"`           |
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

| Field                      | Type   | Meaning                                             |
| -------------------------- | ------ | --------------------------------------------------- |
| `id`                        | string | Internal document ID                                |
| `accessionNo`               | string | EDGAR accession number                              |
| `fileNo`                    | string | SEC file number, `24R-` prefixed                    |
| `formType`                  | string | `1-Z` or `1-Z/A`                                    |
| `filedAt`                   | string | Filing timestamp with offset                        |
| `cik`                       | string | Issuer CIK, no leading zeros                        |
| `ticker`                    | string | Usually empty. Most Reg A issuers are unlisted.     |
| `companyName`               | string | Issuer name                                         |
| `item1`                     | object | Cover page, see below                               |
| `summaryInfoOffering[]`     | array  | Final offering result, see below                    |
| `certificationSuspension[]` | array  | `securitiesClassTitle`, `certificationFileNumber[]`, `approxRecordHolders` |
| `signatureTab[]`            | array  | `cik`, `regulationIssuerName1`, `regulationIssuerName2`, `signatureBy`, `date`, `title` |

`item1` holds `issuerName`, `street1`, `city`, `stateOrCountry`, `zipCode`,
`phone` and `commissionFileNumber[]`. The key is `phone` here, not
`phoneNumber` as on [reg-a-form-1k](./reg-a-form-1k.md).

`summaryInfoOffering[]` is the substance of the form:

| Field                                   | Type   | Meaning                                    |
| --------------------------------------- | ------ | ------------------------------------------ |
| `offeringQualificationDate`              | string | `MM-DD-YYYY`, when the SEC qualified it    |
| `offeringCommenceDate`                   | string | `MM-DD-YYYY`, when selling started         |
| `offeringSecuritiesQualifiedSold`         | number | Securities qualified for sale              |
| `offeringSecuritiesSold`                  | number | Securities actually sold                   |
| `pricePerSecurity`                        | number | Price per unit                             |
| `portionSecuritiesSoldIssuer`             | number | Dollars sold on the issuer's behalf        |
| `portionSecuritiesSoldSecurityholders`    | number | Dollars sold on selling holders' behalf    |
| `issuerNetProceeds`                       | number | What the issuer kept                       |

Seven service-provider roles each get a name array and a fee number:
`underwrittenSpName` with `underwriterFees`, `salesCommissionsSpName` with
`salesCommissionsFee`, `findersSpName` with `findersFees`, `auditorSpName` with
`auditorFees`, `legalSpName` with `legalFees`, `promoterSpName` with
`promotersFees`, and `blueSkySpName` with `blueSkyFees`.

This block also holds `crdNumberBrokerDealer` and `clarificationResponses`.

Filers write `"None"` or `"-"` in the `*SpName[]` arrays when a role was unused.
Those two values are not company names.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.
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
- Dates inside `item1`, `summaryInfoOffering[]` and `signatureTab[]` use
  `MM-DD-YYYY`. Only the top-level `filedAt` is ISO. The two formats differ.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-search](./reg-a-search.md), [reg-a-form-1a](./reg-a-form-1a.md), [reg-a-form-1k](./reg-a-form-1k.md)
- [form-d](./form-d.md), [form-c](./form-c.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
