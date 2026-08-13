# form-d

Search Form D notices, the filings that private companies and funds submit for
exempt offerings under Regulation D.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, offerings[]}`                       |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-d`                               |

## What it does

The tool searches Form D and Form D/A notices. One row is one notice. A notice
tells you who is raising money, how much, from how many investors, and which
people are behind the issuer. This is the main public record of venture rounds,
private funds and other private placements.

Coverage starts 2008-09-17. The rows carry the full XML of the notice, mapped to
JSON, so nothing is summarised or rewritten.

**Read the envelope carefully. This tool returns `offerings[]`, not `data[]`.**
It is one of only two tools in the server that use that key.

## When to use it

- Which companies in my state raised over $10M in a private round this year?
- How much of the announced round has actually been sold?
- Who are the officers, directors and promoters named on this raise?
- Which venture funds filed a new Form D last week?
- Did this offering accept non-accredited investors?

## When to use a different tool

| Situation                              | Better tool                         | Why                                          |
| -------------------------------------- | ----------------------------------- | -------------------------------------------- |
| The raise is a crowdfunding campaign   | [form-c](./form-c.md)               | Reg CF offerings file Form C, not Form D     |
| The raise is a Regulation A offering   | [reg-a-form-1a](./reg-a-form-1a.md) | Reg A offerings file Form 1-A                |
| The company is going public            | [form-s1-424b4](./form-s1-424b4.md) | Registered offerings file S-1 or 424B4       |
| You only want the filing index entry   | [filing-search](./filing-search.md) | Lists the filings without parsing the XML    |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

| Field                                                    | Example                                    |
| -------------------------------------------------------- | ------------------------------------------ |
| `primaryIssuer.cik`                                       | `primaryIssuer.cik:0002149897`             |
| `primaryIssuer.entityName`                                | `primaryIssuer.entityName:"Fusion VC"`     |
| `primaryIssuer.entityType`                                | `primaryIssuer.entityType:"Limited Partnership"` |
| `primaryIssuer.issuerAddress.stateOrCountry`              | `...stateOrCountry:CA`                     |
| `submissionType`                                          | `submissionType:"D/A"`                     |
| `filedAt`                                                 | `filedAt:[2024-01-01 TO 2024-12-31]`       |
| `offeringData.offeringSalesAmounts.totalOfferingAmount`   | `...totalOfferingAmount:[1000000 TO *]`    |
| `offeringData.offeringSalesAmounts.totalAmountSold`       | `...totalAmountSold:[100000000 TO *]`      |
| `offeringData.industryGroup.industryGroupType`            | `...industryGroupType:"Pooled Investment Fund"` |
| `offeringData.investors.hasNonAccreditedInvestors`        | `...hasNonAccreditedInvestors:true`        |
| `offeringData.federalExemptionsExclusions.item`           | `...item:"06b"`                            |
| `offeringData.typeOfFiling.dateOfFirstSale.value`         | `...value:[2026-01-01 TO 2026-12-31]`      |
| `relatedPersonsList.relatedPersonInfo.relatedPersonName.lastName` | `...lastName:Musk`                 |

Two traps in this list:

1. There is **no bare `cik` field**. `cik:*` returns zero rows. The CIK lives at
   `primaryIssuer.cik`.
2. `primaryIssuer.cik` is stored zero-padded to 10 digits.
   `primaryIssuer.cik:2149897` returns nothing.
   `primaryIssuer.cik:0002149897` returns the row.

## Output

The envelope is `{total, offerings[]}`. `total` is `{value, relation}`. A
`relation` of `gte` with `value` `10000` means "10,000 or more", not exactly
10,000.

| Field                                            | Type   | Meaning                                        |
| ------------------------------------------------ | ------ | ---------------------------------------------- |
| `id`                                              | string | Internal document ID                           |
| `accessionNo`                                     | string | EDGAR accession number                         |
| `filedAt`                                         | string | Filing timestamp with offset                   |
| `submissionType`                                  | string | `D` for a new notice, `D/A` for an amendment   |
| `testOrLive`                                      | string | `LIVE` for a real filing                       |
| `schemaVersion`                                   | string | Form D XML schema version, for example `X0708` |
| `primaryIssuer.cik`                               | string | Issuer CIK, zero-padded to 10 digits           |
| `primaryIssuer.entityName`                        | string | Issuer name                                    |
| `primaryIssuer.entityType`                        | string | Legal form, for example `Limited Liability Company` |
| `primaryIssuer.jurisdictionOfInc`                 | string | State or country of incorporation              |
| `primaryIssuer.issuerAddress`                     | object | `street1`, `street2`, `city`, `stateOrCountry`, `zipCode` |
| `primaryIssuer.yearOfInc`                         | object | `withinFiveYears`, `value`                     |
| `relatedPersonsList.relatedPersonInfo[]`          | array  | Each has `relatedPersonName`, `relatedPersonAddress`, `relatedPersonRelationshipList.relationship[]` |
| `offeringData.industryGroup`                      | object | `industryGroupType`, plus `investmentFundInfo` for funds |
| `offeringData.issuerSize.revenueRange`            | string | Self-reported revenue band                     |
| `offeringData.federalExemptionsExclusions.item[]` | array  | Rule codes claimed, for example `06b`, `3C`    |
| `offeringData.typeOfFiling`                       | object | `newOrAmendment.isAmendment`, `dateOfFirstSale.value` |
| `offeringData.offeringSalesAmounts`               | object | `totalOfferingAmount`, `totalAmountSold`, `totalRemaining` |
| `offeringData.investors`                          | object | `hasNonAccreditedInvestors`, `totalNumberAlreadyInvested` |
| `offeringData.minimumInvestmentAccepted`          | number | Minimum cheque size in dollars                 |
| `offeringData.salesCommissionsFindersFees`        | object | `salesCommissions.dollarAmount`, `findersFees.dollarAmount` |
| `offeringData.signatureBlock.signature[]`         | array  | `nameOfSigner`, `signatureTitle`, `signatureDate` |

Further fields:
`offeringData.typeOfFiling.newOrAmendment.previousAccessionNumber`,
`offeringData.useOfProceeds.grossProceedsUsed.isEstimate` and
`offeringData.useOfProceeds.clarificationOfResponse`. Those three are absent
from the example response.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.

## Example

Prompt: "Show me the newest private placements that raised at least $1 million."

```json
{
  "name": "form-d",
  "arguments": {
    "query": "offeringData.offeringSalesAmounts.totalOfferingAmount:[1000000 TO *]",
    "size": 1
  }
}
```

Response, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "offerings": [
    {
      "schemaVersion": "X0708",
      "submissionType": "D",
      "testOrLive": "LIVE",
      "primaryIssuer": {
        "cik": "0002149897",
        "entityName": "Fusion VC SPV Skapion LLC",
        "jurisdictionOfInc": "DELAWARE",
        "entityType": "Limited Liability Company",
        "yearOfInc": { "withinFiveYears": true, "value": "2026" }
      },
      "offeringData": {
        "industryGroup": { "industryGroupType": "Pooled Investment Fund" },
        "minimumInvestmentAccepted": 0,
        "offeringSalesAmounts": { "totalOfferingAmount": 1402000, "totalAmountSold": 1402000, "totalRemaining": 0 },
        "investors": { "hasNonAccreditedInvestors": false, "totalNumberAlreadyInvested": 11 }
      },
      "accessionNo": "0002149897-26-000001",
      "filedAt": "2026-08-13T09:13:20-04:00",
      "id": "5be684f8ec109058266aca6744ee6997"
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"offerings":[]}`
  with no error. An empty result there does not mean there is no more data.
- `total.value` of exactly 10000 with `relation: "gte"` is a counting ceiling.
  Narrow the query if you need a real count.
- Unlike [form-s1-424b4](./form-s1-424b4.md) and [form-8k](./form-8k.md), this
  tool does not demand a colon in `query`.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [form-c](./form-c.md), [reg-a-form-1a](./reg-a-form-1a.md), [form-s1-424b4](./form-s1-424b4.md)
- [filing-search](./filing-search.md), [edgar-entities](./edgar-entities.md)
- REST docs: <https://sec-api.io/docs/form-d-xml-json-api>
