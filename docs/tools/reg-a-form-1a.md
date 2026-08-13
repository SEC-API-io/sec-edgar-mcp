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
| `accessionNo`                              | `accessionNo:"0001493152-26-037452"`           |
| `companyName`                              | `companyName:"Blue Star Foods"`                |
| `ticker`                                   | `ticker:BSFC`                                  |
| `fileNo`                                   | `fileNo:"024-12712"`                           |
| `formType`                                 | `formType:"1-A POS"`                           |
| `filedAt`                                  | `filedAt:[2024-01-01 TO 2024-12-31]`           |
| `summaryInfo.indicateTier1Tier2Offering`   | `...indicateTier1Tier2Offering:Tier2`          |
| `summaryInfo.financialStatementAuditStatus`| `...financialStatementAuditStatus:Audited`     |
| `summaryInfo.securitiesOfferedTypes`       | `...securitiesOfferedTypes:Equity*`            |
| `summaryInfo.offerDelayedContinuousFlag`   | `...offerDelayedContinuousFlag:true`           |
| `summaryInfo.totalAggregateOffering`       | `...totalAggregateOffering:[10000000 TO *]`    |
| `summaryInfo.auditorServiceProviderName`   | `...auditorServiceProviderName:"GreenGrowth*"` |
| `summaryInfo.legalServiceProviderName`     | `...legalServiceProviderName:"Capital*"`       |
| `summaryInfo.brokerDealerCrdNumber`        | `...brokerDealerCrdNumber:*`                   |
| `issuerInfo.stateOrCountry`                | `issuerInfo.stateOrCountry:CA`                 |
| `issuerInfo.nameAuditor`                   | `issuerInfo.nameAuditor:"GreenGrowth CPAs"`    |
| `issuerInfo.totalAssets`                   | `issuerInfo.totalAssets:[1000000 TO *]`        |
| `employeesInfo.jurisdictionOrganization`   | `...jurisdictionOrganization:DE`               |
| `employeesInfo.yearIncorporation`          | `...yearIncorporation:2017`                    |
| `employeesInfo.sicCode`                    | `...sicCode:3510`                              |
| `juridictionSecuritiesOffered.issueJuridicationSecuritiesOffering` | `...issueJuridicationSecuritiesOffering:NV` |

`formType` matches the exact string. `formType:"1-A"` does not include `1-A/A`.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

| Field            | Type    | Meaning                                                 |
| ---------------- | ------- | ------------------------------------------------------- |
| `total.value`    | integer | Count of 1-A filings that match the query. It stops at 10,000. |
| `total.relation` | string  | `eq` marks an exact count. `gte` marks the 10,000 ceiling. |
| `data[]`         | array   | The matched filings. One element is one filing.         |

Every field below sits inside one `data[]` row. A row carries only the fields
the filer completed.

### Filing metadata

| Field            | Type   | Meaning                                                     |
| ---------------- | ------ | ----------------------------------------------------------- |
| `id`             | string | Internal ID of the filing record                            |
| `accessionNo`    | string | EDGAR accession number of the filing                        |
| `fileNo`         | string | SEC file number. Every filing of the same offering process shares it, for example `024-12712`. |
| `formType`       | string | `1-A`, `1-A/A`, `1-A POS` or `1-A-W`                        |
| `filedAt`        | string | Moment EDGAR accepted the filing                            |
| `periodOfReport` | string | Period the submission covers, `YYYY-MM-DD`                  |
| `cik`            | string | CIK of the issuer, no leading zeros                         |
| `ticker`         | string | Stock ticker of the issuer at filing time. Usually empty. Most Reg A issuers are unlisted. |
| `companyName`    | string | Legal name of the issuer as given in the filing             |

### `employeesInfo[]`: issuer identity and headcount

| Field                                      | Type    | Meaning                                       |
| ------------------------------------------ | ------- | --------------------------------------------- |
| `employeesInfo[].issuerName`               | string  | Name of the issuer as stated in its charter   |
| `employeesInfo[].jurisdictionOrganization` | string  | Jurisdiction that the issuer is organized or incorporated under |
| `employeesInfo[].yearIncorporation`        | string  | Year of incorporation                         |
| `employeesInfo[].cik`                      | string  | CIK of the issuer, padded to 10 digits        |
| `employeesInfo[].sicCode`                  | integer | SIC code of the main business of the issuer   |
| `employeesInfo[].irsNum`                   | string  | IRS Employer Identification Number            |
| `employeesInfo[].fullTimeEmployees`        | integer | Count of full-time employees the issuer reports |
| `employeesInfo[].partTimeEmployees`        | integer | Count of part-time employees the issuer reports |

### `issuerInfo`: address and contact

| Field                                | Type   | Meaning                                             |
| ------------------------------------ | ------ | --------------------------------------------------- |
| `issuerInfo.street1`                 | string | First street line of the principal executive offices |
| `issuerInfo.street2`                 | string | Second street line of the principal executive offices |
| `issuerInfo.city`                    | string | City of the principal office                        |
| `issuerInfo.stateOrCountry`          | string | State or country of the principal office            |
| `issuerInfo.zipCode`                 | string | Postal code of the principal office                 |
| `issuerInfo.phoneNumber`             | string | Telephone number of the principal office            |
| `issuerInfo.connectionName`          | string | Primary contact for correspondence about the filing |
| `issuerInfo.connectionStreet1`       | string | First street line of the contact                    |
| `issuerInfo.connectionStreet2`       | string | Second street line of the contact                   |
| `issuerInfo.connectionCity`          | string | City of the contact                                 |
| `issuerInfo.connectionStateOrCountry`| string | State or country of the contact                     |
| `issuerInfo.connectionZipCode`       | string | Postal code of the contact                          |
| `issuerInfo.connectionPhoneNumber`   | string | Telephone number of the contact                     |
| `issuerInfo.commentsEmailAddress`    | string | Email address for comments or questions about the filing |
| `issuerInfo.industryGroup`           | string | Industry group of the issuer: `Banking`, `Insurance` or `Other` |

### `issuerInfo`: cover-page financials

Plain numbers, not strings. The issuer reports them on the cover page of the
offering statement.

| Field                                       | Type   | Meaning                                    |
| ------------------------------------------- | ------ | ------------------------------------------ |
| `issuerInfo.cashEquivalents`                | number | Cash and cash equivalents on the balance sheet |
| `issuerInfo.investmentSecurities`           | number | Value of the investment securities the issuer holds |
| `issuerInfo.totalInvestments`               | number | Total value of all investments the issuer reports |
| `issuerInfo.accountsReceivable`             | number | Total accounts receivable                  |
| `issuerInfo.loans`                          | number | Value of the loans the issuer reports      |
| `issuerInfo.propertyPlantEquipment`         | number | Property, plant and equipment on the balance sheet |
| `issuerInfo.propertyAndEquipment`           | number | Property and equipment, combined value     |
| `issuerInfo.totalAssets`                    | number | Total assets                               |
| `issuerInfo.accountsPayable`                | number | Total accounts payable                     |
| `issuerInfo.policyLiabilitiesAndAccruals`   | number | Total policy liabilities and accruals      |
| `issuerInfo.deposits`                       | number | Deposits on the balance sheet              |
| `issuerInfo.longTermDebt`                   | number | Total long-term debt                       |
| `issuerInfo.totalLiabilities`               | number | Total liabilities                          |
| `issuerInfo.totalStockholderEquity`         | number | Total stockholder equity. It goes negative when liabilities exceed assets. |
| `issuerInfo.totalLiabilitiesAndEquity`      | number | Liabilities plus stockholder equity        |
| `issuerInfo.totalRevenues`                  | number | Total revenues of the period               |
| `issuerInfo.totalInterestIncome`            | number | Total interest income                      |
| `issuerInfo.costAndExpensesApplToRevenues`  | number | Costs and expenses that apply to revenues  |
| `issuerInfo.totalInterestExpenses`          | number | Total interest expense                     |
| `issuerInfo.depreciationAndAmortization`    | number | Depreciation and amortization expense      |
| `issuerInfo.netIncome`                      | number | Net income of the period                   |
| `issuerInfo.earningsPerShareBasic`          | number | Basic earnings per share                   |
| `issuerInfo.earningsPerShareDiluted`        | number | Diluted earnings per share                 |
| `issuerInfo.nameAuditor`                    | string | Auditor of the financial statements        |

### `commonEquity[]`, `preferredEquity[]` and `debtSecurities[]`

One element per class of securities the issuer has outstanding.

| Field                                           | Type    | Meaning                            |
| ----------------------------------------------- | ------- | ---------------------------------- |
| `commonEquity[].commonEquityClassName`          | string  | Name of the common equity class    |
| `commonEquity[].outstandingCommonEquity`        | integer | Shares of that class outstanding   |
| `commonEquity[].commonCusipEquity`              | string  | CUSIP of the class. `000000000` when the class has none. |
| `commonEquity[].publiclyTradedCommonEquity`     | string  | Where the class trades, for example `OTCID`. `N/A` when it does not trade. |
| `preferredEquity[].preferredEquityClassName`    | string  | Name of the preferred equity class |
| `preferredEquity[].outstandingPreferredEquity`  | integer | Shares of that class outstanding   |
| `preferredEquity[].preferredCusipEquity`        | string  | CUSIP of the class. `000000000` when the class has none. |
| `preferredEquity[].publiclyTradedPreferredEquity` | string | Where the class trades. `N/A` when it does not trade. |
| `debtSecurities[].debtSecuritiesClassName`      | string  | Name of the debt securities class  |
| `debtSecurities[].outstandingDebtSecurities`    | integer | Amount of that class outstanding   |
| `debtSecurities[].cusipDebtSecurities`          | string  | CUSIP of the class. `000000000` when the class has none. |
| `debtSecurities[].publiclyTradedDebtSecurities` | string  | Where the class trades. `N/A` when it does not trade. |

### `issuerEligibility` and `applicationRule262`

| Field                                       | Type    | Meaning                                   |
| ------------------------------------------- | ------- | ----------------------------------------- |
| `issuerEligibility.certifyIfTrue`           | boolean | The issuer certifies that it meets the SEC eligibility rules |
| `applicationRule262.certifyIfNotDisqualified` | boolean | The issuer certifies that SEC Rule 262 does not disqualify it |
| `applicationRule262.certifyIfBadActor`      | boolean | Marks a bad actor disqualification under the SEC rules |

### `summaryInfo`: offering terms

| Field                                        | Type    | Meaning                                  |
| -------------------------------------------- | ------- | ---------------------------------------- |
| `summaryInfo.indicateTier1Tier2Offering`     | string  | Tier of the offering: `Tier1` or `Tier2` |
| `summaryInfo.financialStatementAuditStatus`  | string  | `Audited` or `Unaudited` financial statements |
| `summaryInfo.securitiesOfferedTypes[]`       | array   | Types of securities in the offering: `Equity (common or preferred stock)`, `Debt`, `Option, warrant or other right to acquire another security`, `Security to be acquired upon exercise of option, warrant or other right`, `Tenant-in-common securities`, `Other(describe)` |
| `summaryInfo.securitiesOfferedOtherDesc`     | string  | Description of the securities when the type is `Other(describe)` |
| `summaryInfo.offerDelayedContinuousFlag`     | boolean | The offering is delayed or continuous    |
| `summaryInfo.offeringYearFlag`               | boolean | The offering is tied to a set offering year |
| `summaryInfo.offeringAfterQualifFlag`        | boolean | The offering starts after qualification  |
| `summaryInfo.offeringBestEffortsFlag`        | boolean | The offering runs on a best-efforts basis |
| `summaryInfo.solicitationProposedOfferingFlag` | boolean | The offering uses solicitation         |
| `summaryInfo.resaleSecuritiesAffiliatesFlag` | boolean | The offering includes securities that affiliates resell |
| `summaryInfo.securitiesOffered`              | number  | Count of securities in the offering      |
| `summaryInfo.outstandingSecurities`          | number  | Count of securities outstanding          |
| `summaryInfo.pricePerSecurity`               | number  | Offering price of one security           |
| `summaryInfo.issuerAggregateOffering`        | number  | Aggregate value of the securities the issuer offers |
| `summaryInfo.securityHolderAggegate`         | number  | Aggregate value of the securities that existing holders offer |
| `summaryInfo.qualificationOfferingAggregate` | number  | Aggregate amount of the securities offered in the qualification process |
| `summaryInfo.concurrentOfferingAggregate`    | number  | Aggregate amount of any concurrent offering |
| `summaryInfo.totalAggregateOffering`         | number  | Total aggregate value of the offering    |
| `summaryInfo.brokerDealerCrdNumber`          | string  | CRD number of the broker-dealer in the offering |
| `summaryInfo.estimatedNetAmount`             | number  | Estimated net proceeds to the issuer after the deductions |
| `summaryInfo.clarificationResponses`         | string  | Free text the filer adds about the offering answers |

### `summaryInfo`: service providers and fees

Seven roles, each a name and a fee. A row carries only the pairs the filer
filled in.

| Field                                                | Type   | Meaning                          |
| ---------------------------------------------------- | ------ | -------------------------------- |
| `summaryInfo.underwritersServiceProviderName`        | string | Underwriter of the offering      |
| `summaryInfo.underwritersFees`                       | number | Fee of the underwriter           |
| `summaryInfo.salesCommissionsServiceProviderName`    | string | Provider of the sales commission services |
| `summaryInfo.salesCommissionsServiceProviderFees`    | number | Fee for the sales commission services |
| `summaryInfo.findersFeesServiceProviderName`         | string | Provider of the finder services  |
| `summaryInfo.finderFeesFee`                          | number | Fee for the finder services      |
| `summaryInfo.auditorServiceProviderName`             | string | Provider of the audit services   |
| `summaryInfo.auditorFees`                            | number | Fee for the audit services       |
| `summaryInfo.legalServiceProviderName`               | string | Provider of the legal services   |
| `summaryInfo.legalFees`                              | number | Fee for the legal services       |
| `summaryInfo.promotersServiceProviderName`           | string | Provider of the promotion services |
| `summaryInfo.promotersFees`                          | number | Fee for the promotion services   |
| `summaryInfo.blueSkyServiceProviderName`             | string | Provider of the Blue Sky compliance services |
| `summaryInfo.blueSkyFees`                            | number | Fee for the Blue Sky compliance services |

### `juridictionSecuritiesOffered`

| Field                                                            | Type    | Meaning        |
| ---------------------------------------------------------------- | ------- | -------------- |
| `juridictionSecuritiesOffered.jurisdictionsOfSecOfferedNone`     | boolean | The filer names no jurisdiction for the offering |
| `juridictionSecuritiesOffered.jurisdictionsOfSecOfferedSame`     | boolean | The same jurisdictions apply across the offering |
| `juridictionSecuritiesOffered.issueJuridicationSecuritiesOffering[]` | array | Jurisdictions of the issuance, as state or country codes |
| `juridictionSecuritiesOffered.dealersJuridicationSecuritiesOffering[]` | array | Jurisdictions of the dealers, as state or country codes |

### `securitiesIssued[]` and the unregistered-sales blocks

The securities the issuer sold in the last year without registration.

| Field                                                | Type    | Meaning                         |
| ---------------------------------------------------- | ------- | ------------------------------- |
| `securitiesIssued[].securitiesIssuerName`            | string  | Issuer of the securities sold   |
| `securitiesIssued[].securitiesIssuerTitle`           | string  | Title of the securities sold    |
| `securitiesIssued[].securitiesIssuedTotalAmount`     | number  | Total amount of the securities sold |
| `securitiesIssued[].securitiesPrincipalHolderAmount` | number  | Amount of those securities that the principal holder keeps |
| `securitiesIssued[].securitiesIssuedAggregateAmount` | string  | Aggregate consideration for the sale, as free text |
| `securitiesIssued[].aggregateConsiderationBasis`     | string  | Basis of the calculation of that aggregate consideration |
| `unregisteredSecurities.ifUnregsiteredNone`          | boolean | The issuer sold no unregistered securities |
| `unregisteredSecuritiesAct.securitiesActExcemption`  | string  | Securities Act exemption the issuer claims for the sale |

Copy the spellings exactly. Several are misspelled in the API and will not match
a corrected guess: `juridictionSecuritiesOffered`,
`issueJuridicationSecuritiesOffering`, `dealersJuridicationSecuritiesOffering`,
`securityHolderAggegate`, `securitiesActExcemption`, `ifUnregsiteredNone`. Note
that `unregisteredSecurities` and `unregisteredSecuritiesAct` are two separate
keys.

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.

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
- A `1-A-W` withdrawal returns only the first seven metadata fields. EDGAR
  holds no structured data for it.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-search](./reg-a-search.md), [reg-a-form-1k](./reg-a-form-1k.md), [reg-a-form-1z](./reg-a-form-1z.md)
- [form-d](./form-d.md), [form-c](./form-c.md), [form-s1-424b4](./form-s1-424b4.md), [extractor](./extractor.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
