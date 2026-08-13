# reg-a-form-1a

Search Form 1-A offering statements, the document that opens a Regulation A
offering.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /reg-a/form-1a`                        |

## What it does

Form 1-A is the offering statement a company files to open a Regulation A
offering. This tool searches only the 1-A family. One row is one filing.

A row gives you the tier, the size of the offering, the price per security, the
issuer's balance sheet and income statement as filed on the cover, the service
providers and their fees, and the securities the issuer sold in the last year
without registration.

Coverage starts 2015-06-22. Form types and counts on 2026-08-13:
`1-A` 2,326, `1-A/A` 4,608, `1-A POS` 2,591, `1-A-W` 492. The index reports
"10,000 or more" in total.

The registry description promises "use of proceeds" and "risk factors". Neither
field exists in the response. For the narrative sections, use
[extractor](./extractor.md) on the filing URL.

## When to use it

- Which companies opened a Tier 2 Reg A offering this quarter?
- How large is this offering, and at what price per share?
- What are the issuer's total assets, revenue and net income on the cover page?
- Which auditor and which law firm worked on this offering, and for what fee?
- Which Reg A issuers are incorporated in Delaware?

## When to use a different tool

| Situation                                | Better tool                         | Why                                       |
| ---------------------------------------- | ----------------------------------- | ----------------------------------------- |
| You want 1-A, 1-K and 1-Z in one search  | [reg-a-search](./reg-a-search.md)   | Queries all three indices together        |
| You want the yearly report of the issuer | [reg-a-form-1k](./reg-a-form-1k.md) | Annual results, not the opening statement |
| You want to know if the offering closed  | [reg-a-form-1z](./reg-a-form-1z.md) | The exit report holds the final numbers   |
| You want the risk factors text           | [extractor](./extractor.md)         | Returns the document sections as text     |
| The company is doing a full IPO          | [form-s1-424b4](./form-s1-424b4.md) | Registered offerings file S-1 or 424B4    |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

| Field                                     | Example                                        |
| ----------------------------------------- | ---------------------------------------------- |
| `cik`                                      | `cik:1730773`                                  |
| `companyName`                              | `companyName:"Blue Star Foods"`                |
| `ticker`                                   | `ticker:BSFC`                                  |
| `fileNo`                                   | `fileNo:"024-12712"`                           |
| `formType`                                 | `formType:"1-A POS"`                           |
| `filedAt`                                  | `filedAt:[2024-01-01 TO 2024-12-31]`           |
| `summaryInfo.indicateTier1Tier2Offering`   | `...indicateTier1Tier2Offering:Tier2`          |
| `summaryInfo.financialStatementAuditStatus`| `...financialStatementAuditStatus:Audited`     |
| `summaryInfo.totalAggregateOffering`       | `...totalAggregateOffering:[10000000 TO *]`    |
| `issuerInfo.stateOrCountry`                | `issuerInfo.stateOrCountry:CA`                 |
| `issuerInfo.nameAuditor`                   | `issuerInfo.nameAuditor:"GreenGrowth CPAs"`    |
| `employeesInfo.jurisdictionOrganization`   | `...jurisdictionOrganization:DE`               |

`formType` matches the exact string. `formType:"1-A"` does not include `1-A/A`.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

| Field                                       | Type   | Meaning                                          |
| ------------------------------------------- | ------ | ------------------------------------------------ |
| `id`                                         | string | Internal document ID                             |
| `accessionNo`                                | string | EDGAR accession number                           |
| `fileNo`                                     | string | SEC file number, for example `024-12712`         |
| `formType`                                   | string | `1-A`, `1-A/A`, `1-A POS`, `1-A-W`               |
| `filedAt`                                    | string | Filing timestamp with offset                     |
| `cik`                                        | string | Issuer CIK, no leading zeros                     |
| `ticker`                                     | string | Usually empty. Most Reg A issuers are unlisted.  |
| `companyName`                                | string | Issuer name                                      |
| `employeesInfo[]`                            | array  | `issuerName`, `jurisdictionOrganization`, `yearIncorporation`, `cik`, `sicCode`, `irsNum`, `fullTimeEmployees`, `partTimeEmployees` |
| `issuerInfo`                                 | object | Address, phone, `industryGroup`, `nameAuditor`, and the cover-page financials |
| `commonEquity[]`                             | array  | `commonEquityClassName`, `outstandingCommonEquity`, `commonCusipEquity`, `publiclyTradedCommonEquity` |
| `preferredEquity[]`                          | array  | Same four keys with `preferred` prefixes         |
| `debtSecurities[]`                           | array  | `debtSecuritiesClassName`, `outstandingDebtSecurities`, `cusipDebtSecurities`, `publiclyTradedDebtSecurities` |
| `issuerEligibility.certifyIfTrue`            | bool   | Issuer certifies it is eligible for Reg A        |
| `applicationRule262.certifyIfNotDisqualified`| bool   | Issuer certifies it is not a bad actor           |
| `summaryInfo`                                | object | Offering terms and fees, see below               |
| `juridictionSecuritiesOffered`               | object | `jurisdictionsOfSecOfferedNone`, `issueJuridicationSecuritiesOffering[]` state codes |
| `securitiesIssued[]`                         | array  | `securitiesIssuerName`, `securitiesIssuerTitle`, `securitiesIssuedTotalAmount`, `securitiesPrincipalHolderAmount`, `securitiesIssuedAggregateAmount` |
| `unregisteredSecuritiesAct.securitiesActExcemption` | string | The exemption claimed for past unregistered sales |

`issuerInfo` carries the cover-page financials as plain numbers:
`cashEquivalents`, `investmentSecurities`, `accountsReceivable`,
`propertyPlantEquipment`, `totalAssets`, `accountsPayable`, `longTermDebt`,
`totalLiabilities`, `totalStockholderEquity`, `totalLiabilitiesAndEquity`,
`totalRevenues`, `costAndExpensesApplToRevenues`, `depreciationAndAmortization`,
`netIncome`, `earningsPerShareBasic`, `earningsPerShareDiluted`.

`summaryInfo` carries `indicateTier1Tier2Offering` (`Tier1` or `Tier2`),
`financialStatementAuditStatus` (`Audited` or `Unaudited`),
`securitiesOfferedTypes[]`, `securitiesOffered`, `outstandingSecurities`,
`pricePerSecurity`, `issuerAggregateOffering`, `securityHolderAggegate`,
`qualificationOfferingAggregate`, `concurrentOfferingAggregate`,
`totalAggregateOffering`, six boolean flags (`offerDelayedContinuousFlag`,
`offeringYearFlag`, `offeringAfterQualifFlag`, `offeringBestEffortsFlag`,
`solicitationProposedOfferingFlag`, `resaleSecuritiesAffiliatesFlag`), and three
service-provider pairs: `auditorServiceProviderName` with `auditorFees`,
`legalServiceProviderName` with `legalFees`, `blueSkyServiceProviderName` with
`blueSkyFees`. `estimatedNetAmount` and `clarificationResponses` also appear.

Copy the spellings exactly. Several are misspelled in the API and will not match
a corrected guess: `juridictionSecuritiesOffered`,
`issueJuridicationSecuritiesOffering`, `securityHolderAggegate`,
`securitiesActExcemption`. Some rows also carry an `unregisteredSecurities` key,
distinct from `unregisteredSecuritiesAct`.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.

## Example

Prompt: "Show me the newest Regulation A offering statements."

```json
{ "name": "reg-a-form-1a", "arguments": { "query": "cik:*", "size": 1 } }
```

Response, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "id": "7d83a0661e42e659f1e1650c12ec049f",
      "accessionNo": "0001493152-26-037452",
      "fileNo": "024-12712",
      "formType": "1-A/A",
      "filedAt": "2026-08-12T17:24:40-04:00",
      "cik": "1730773",
      "ticker": "BSFC",
      "companyName": "Blue Star Foods Corp.",
      "issuerInfo": {
        "city": "Miami",
        "stateOrCountry": "FL",
        "industryGroup": "Other",
        "totalAssets": 1288041,
        "totalLiabilities": 4261146,
        "totalRevenues": 782676,
        "netIncome": -800438,
        "nameAuditor": "GreenGrowth CPAs"
      },
      "summaryInfo": {
        "indicateTier1Tier2Offering": "Tier2",
        "financialStatementAuditStatus": "Audited",
        "securitiesOffered": 500000000,
        "pricePerSecurity": 0.001,
        "totalAggregateOffering": 500000,
        "auditorServiceProviderName": "GreenGrowth CPAs",
        "auditorFees": 25000
      }
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- The 1-A index reports `total.value` 10000 with `relation: "gte"` on broad
  queries. That is a counting ceiling, not a real count.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-search](./reg-a-search.md), [reg-a-form-1k](./reg-a-form-1k.md), [reg-a-form-1z](./reg-a-form-1z.md)
- [form-d](./form-d.md), [form-c](./form-c.md), [form-s1-424b4](./form-s1-424b4.md), [extractor](./extractor.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
