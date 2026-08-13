# reg-a-search

Search every Regulation A filing at once: offering statements, annual reports
and exit reports.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /reg-a/search`                         |

## What it does

Regulation A lets a small company sell shares to the public without a full
registration. The company files a Form 1-A to open the offering, a Form 1-K each
year, and a Form 1-Z when it closes.

This tool searches all three form families in one call. One row is one filing.
The server queries the 1-A, 1-K and 1-Z indices together, so **the shape of a row
depends on its `formType`**. A 1-A row has `summaryInfo` and `issuerInfo`. A 1-K
row has `item1` and `periodOfReport`. A 1-Z row has `summaryInfoOffering` and
`certificationSuspension`. Check `formType` before you read a field.

Coverage starts 2015-06-22, the first year of Regulation A+.

## When to use it

- Show me every Reg A filing this company has ever made.
- Which Reg A issuers filed anything last week?
- Trace one offering from its 1-A through its 1-K reports to its 1-Z close.
- Which Reg A issuers are in this state?

## When to use a different tool

| Situation                           | Better tool                         | Why                                            |
| ----------------------------------- | ----------------------------------- | ---------------------------------------------- |
| You only want offering statements   | [reg-a-form-1a](./reg-a-form-1a.md) | One row shape, no `formType` guard needed      |
| You only want annual reports        | [reg-a-form-1k](./reg-a-form-1k.md) | Same reason                                    |
| You only want exit reports          | [reg-a-form-1z](./reg-a-form-1z.md) | Same reason                                    |
| The raise is a private placement    | [form-d](./form-d.md)               | Reg D offerings file Form D                    |
| The raise is a crowdfunding campaign| [form-c](./form-c.md)               | Reg CF offerings file Form C                   |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

| Field                                  | Example                                          | Applies to  |
| -------------------------------------- | ------------------------------------------------ | ----------- |
| `cik`                                   | `cik:1730773`                                    | all forms   |
| `accessionNo`                           | `accessionNo:"0001493152-26-037452"`             | all forms   |
| `companyName`                           | `companyName:"Blue Star Foods"`                  | all forms   |
| `ticker`                                | `ticker:BSFC`                                    | all forms   |
| `fileNo`                                | `fileNo:"024-12712"`                             | all forms   |
| `formType`                              | `formType:"1-K"`                                 | all forms   |
| `filedAt`                               | `filedAt:[2024-01-01 TO 2024-12-31]`             | all forms   |
| `summaryInfo.indicateTier1Tier2Offering`| `summaryInfo.indicateTier1Tier2Offering:Tier1`   | 1-A only    |

A field that belongs to one form family simply filters the others out. That is
useful: `summaryInfo.indicateTier1Tier2Offering:Tier1` returns 2,009 rows, all
of them 1-A.

`formType` matches the exact string. `formType:"1-K"` returns 2,875 rows and
does **not** include the 125 `1-K/A` amendments. Query both, or use a wildcard
such as `formType:1-K*`.

Counts by form type, measured on 2026-08-13:

| `formType` | Rows   |
| ---------- | ------ |
| `1-A`      | 2,326  |
| `1-A/A`    | 4,608  |
| `1-A POS`  | 2,591  |
| `1-A-W`    | 492    |
| `1-K`      | 2,875  |
| `1-K/A`    | 125    |
| `1-Z`      | 630    |
| `1-Z/A`    | 12     |

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

### Envelope

| Field            | Type    | Meaning                                                     |
| ---------------- | ------- | ----------------------------------------------------------- |
| `total.value`    | integer | Number of filings that match the query                      |
| `total.relation` | string  | `eq` for an exact count, `gte` when the count stops at 10000 |
| `data[]`         | array   | The matched filings, at most 50 per response                |

### Filing identity

Every row carries these fields, whatever the form type.

| Field            | Type   | Meaning                                                                    |
| ---------------- | ------ | -------------------------------------------------------------------------- |
| `id`             | string | Internal document ID                                                       |
| `accessionNo`    | string | EDGAR accession number                                                     |
| `fileNo`         | string | SEC file number. It ties together all filings of one offering process. `024-` on an offering statement, `24R-` on an annual or exit report. |
| `formType`       | string | See the table above                                                        |
| `filedAt`        | string | Time EDGAR accepted the filing, with offset                                |
| `periodOfReport` | string | Period the filing covers, `YYYY-MM-DD`. On a 1-K it is the fiscal year end. |
| `cik`            | string | Issuer CIK, no leading zeros                                               |
| `ticker`         | string | Ticker of the issuer at the time of filing. Usually empty. Most Reg A issuers are unlisted. |
| `companyName`    | string | Issuer legal name as given in the filing                                   |

The rest of the row depends on `formType`. Each block below appears only on the
form family named in its heading.

| Block                                                                                                                                                                                                              | Form |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---- |
| `employeesInfo[]`, `issuerInfo`, `commonEquity[]`, `preferredEquity[]`, `debtSecurities[]`, `issuerEligibility`, `applicationRule262`, `summaryInfo`, `juridictionSecuritiesOffered`, `securitiesIssued[]`, `unregisteredSecurities`, `unregisteredSecuritiesAct` | 1-A  |
| `item1`, `item1Info[]`, `item2`, `summaryInfo[]`                                                                                                                                                                     | 1-K  |
| `item1`, `summaryInfoOffering[]`, `certificationSuspension[]`, `signatureTab[]`                                                                                                                                      | 1-Z  |

Note two collisions. `summaryInfo` is an **object** on a 1-A row and an
**array** on a 1-K row. `item1` is on both 1-K and 1-Z rows, and the two shapes
share only the address keys. Branch on `formType` first.

### 1-A `employeesInfo[]`

| Field                                       | Type   | Meaning                                                          |
| ------------------------------------------- | ------ | ---------------------------------------------------------------- |
| `employeesInfo[].issuerName`                | string | Registered name of the issuer, as stated in its charter          |
| `employeesInfo[].jurisdictionOrganization`  | string | State or country under which the issuer is organized             |
| `employeesInfo[].yearIncorporation`         | string | Year the issuer was incorporated                                 |
| `employeesInfo[].cik`                       | string | CIK of the issuer, padded to 10 digits                           |
| `employeesInfo[].sicCode`                   | number | Standard Industrial Classification code of the main business of the issuer |
| `employeesInfo[].irsNum`                    | string | IRS Employer Identification Number of the issuer                 |
| `employeesInfo[].fullTimeEmployees`         | number | Number of full-time employees the issuer reports                 |
| `employeesInfo[].partTimeEmployees`         | number | Number of part-time employees the issuer reports                 |

### 1-A `issuerInfo`

The address block, the contact block and the cover-page financials. All amounts
are in dollars.

| Field                                       | Type   | Meaning                                                          |
| ------------------------------------------- | ------ | ---------------------------------------------------------------- |
| `issuerInfo.street1`                        | string | First line of the principal executive office address             |
| `issuerInfo.street2`                        | string | Second line of that address                                      |
| `issuerInfo.city`                           | string | City of the principal office                                     |
| `issuerInfo.stateOrCountry`                 | string | State or country of the principal office                         |
| `issuerInfo.zipCode`                        | string | Postal code of the principal office                              |
| `issuerInfo.phoneNumber`                    | string | Telephone number of the principal office                         |
| `issuerInfo.connectionName`                 | string | Name of the contact person for questions about the filing        |
| `issuerInfo.connectionStreet1`              | string | First line of the contact address                                |
| `issuerInfo.connectionStreet2`              | string | Second line of the contact address                               |
| `issuerInfo.connectionCity`                 | string | City of the contact                                              |
| `issuerInfo.connectionStateOrCountry`       | string | State or country of the contact                                  |
| `issuerInfo.connectionZipCode`              | string | Postal code of the contact                                       |
| `issuerInfo.connectionPhoneNumber`          | string | Telephone number of the contact                                  |
| `issuerInfo.commentsEmailAddress`           | string | Email address for comments and questions about the filing        |
| `issuerInfo.industryGroup`                  | string | Industry group of the issuer: `Banking`, `Insurance` or `Other`  |
| `issuerInfo.cashEquivalents`                | number | Cash and cash equivalents on the balance sheet                   |
| `issuerInfo.investmentSecurities`           | number | Investment securities the issuer holds                           |
| `issuerInfo.totalInvestments`               | number | Total of all investments the issuer reports                      |
| `issuerInfo.accountsReceivable`             | number | Total accounts receivable                                        |
| `issuerInfo.loans`                          | number | Loans the issuer reports                                         |
| `issuerInfo.propertyPlantEquipment`         | number | Property, plant and equipment on the balance sheet               |
| `issuerInfo.propertyAndEquipment`           | number | Property and equipment together, as the issuer reports them      |
| `issuerInfo.totalAssets`                    | number | Total assets                                                     |
| `issuerInfo.accountsPayable`                | number | Total accounts payable                                           |
| `issuerInfo.policyLiabilitiesAndAccruals`   | number | Total policy liabilities and accruals                            |
| `issuerInfo.deposits`                       | number | Deposits on the balance sheet                                    |
| `issuerInfo.longTermDebt`                   | number | Total long-term debt                                             |
| `issuerInfo.totalLiabilities`               | number | Total liabilities                                                |
| `issuerInfo.totalStockholderEquity`         | number | Total stockholder equity. A deficit is negative.                 |
| `issuerInfo.totalLiabilitiesAndEquity`      | number | Liabilities plus stockholder equity                              |
| `issuerInfo.totalRevenues`                  | number | Total revenue for the period                                     |
| `issuerInfo.totalInterestIncome`            | number | Total interest income                                            |
| `issuerInfo.costAndExpensesApplToRevenues`  | number | Costs and expenses that apply to the revenue                     |
| `issuerInfo.totalInterestExpenses`          | number | Total interest expense                                           |
| `issuerInfo.depreciationAndAmortization`    | number | Depreciation and amortization expense                            |
| `issuerInfo.netIncome`                      | number | Net income. A loss is negative.                                  |
| `issuerInfo.earningsPerShareBasic`          | number | Basic earnings per share                                         |
| `issuerInfo.earningsPerShareDiluted`        | number | Diluted earnings per share                                       |
| `issuerInfo.nameAuditor`                    | string | Name of the auditor of the financial statements                  |

### 1-A `commonEquity[]`, `preferredEquity[]` and `debtSecurities[]`

One entry per class of security the issuer already has out.

| Field                                             | Type   | Meaning                                                     |
| ------------------------------------------------- | ------ | ----------------------------------------------------------- |
| `commonEquity[].commonEquityClassName`            | string | Name of the common equity class                             |
| `commonEquity[].outstandingCommonEquity`          | number | Shares of that class outstanding                            |
| `commonEquity[].commonCusipEquity`                | string | CUSIP of the class. Filers write `000000000` when there is none. |
| `commonEquity[].publiclyTradedCommonEquity`       | string | Whether the class trades in public. Filers name the market, for example `OTCID`, or write `N/A`. |
| `preferredEquity[].preferredEquityClassName`      | string | Name of the preferred equity class                          |
| `preferredEquity[].outstandingPreferredEquity`    | number | Shares of that class outstanding                            |
| `preferredEquity[].preferredCusipEquity`          | string | CUSIP of the class                                          |
| `preferredEquity[].publiclyTradedPreferredEquity` | string | Whether the class trades in public                          |
| `debtSecurities[].debtSecuritiesClassName`        | string | Name of the debt securities class                           |
| `debtSecurities[].outstandingDebtSecurities`      | number | Amount of debt securities outstanding                       |
| `debtSecurities[].cusipDebtSecurities`            | string | CUSIP of the debt securities                                |
| `debtSecurities[].publiclyTradedDebtSecurities`   | string | Whether the debt securities trade in public                 |

### 1-A `issuerEligibility` and `applicationRule262`

| Field                                       | Type    | Meaning                                                         |
| ------------------------------------------- | ------- | ---------------------------------------------------------------- |
| `issuerEligibility.certifyIfTrue`           | boolean | The issuer certifies that it meets the SEC eligibility rules for Regulation A |
| `applicationRule262.certifyIfNotDisqualified` | boolean | The issuer certifies that Rule 262 does not disqualify it     |
| `applicationRule262.certifyIfBadActor`      | boolean | Bad actor disqualification indicator                            |

### 1-A `summaryInfo`

The terms of the offering and the fees. This block is an object on a 1-A row.

| Field                                              | Type             | Meaning                                                   |
| -------------------------------------------------- | ---------------- | ---------------------------------------------------------- |
| `summaryInfo.indicateTier1Tier2Offering`           | string           | `Tier1` or `Tier2`                                         |
| `summaryInfo.financialStatementAuditStatus`        | string           | `Audited` or `Unaudited`                                   |
| `summaryInfo.securitiesOfferedTypes`               | array of strings | Types of security offered: `Equity (common or preferred stock)`, `Debt`, `Option, warrant or other right to acquire another security`, `Security to be acquired upon exercise of option, warrant or other right to acquire security`, `Tenant-in-common securities` or `Other(describe)` |
| `summaryInfo.securitiesOfferedOtherDesc`           | string           | The security in words when the type is `Other(describe)`   |
| `summaryInfo.offerDelayedContinuousFlag`           | boolean          | The offering is delayed or continuous                      |
| `summaryInfo.offeringYearFlag`                     | boolean          | The offering ties to a set offering year                   |
| `summaryInfo.offeringAfterQualifFlag`              | boolean          | The offering starts after qualification                    |
| `summaryInfo.offeringBestEffortsFlag`              | boolean          | The offering runs on a best efforts basis                  |
| `summaryInfo.solicitationProposedOfferingFlag`     | boolean          | The offering involves solicitation                         |
| `summaryInfo.resaleSecuritiesAffiliatesFlag`       | boolean          | The offering includes securities that affiliates resell    |
| `summaryInfo.securitiesOffered`                    | number           | Number of securities in the offering                       |
| `summaryInfo.outstandingSecurities`                | number           | Number of securities outstanding                           |
| `summaryInfo.pricePerSecurity`                     | number           | Offering price per security                                |
| `summaryInfo.issuerAggregateOffering`              | number           | Dollar value of the securities the issuer offers           |
| `summaryInfo.securityHolderAggegate`               | number           | Dollar value of the securities that existing holders offer |
| `summaryInfo.qualificationOfferingAggregate`       | number           | Dollar value of the securities offered as part of the qualification |
| `summaryInfo.concurrentOfferingAggregate`          | number           | Dollar value of any concurrent offering                    |
| `summaryInfo.totalAggregateOffering`               | number           | Total dollar value of the offering                         |
| `summaryInfo.underwritersServiceProviderName`      | string           | Name of the underwriter                                    |
| `summaryInfo.underwritersFees`                     | number           | Fee the underwriter charges                                |
| `summaryInfo.salesCommissionsServiceProviderName`  | string           | Name of the provider that handles the sales commissions    |
| `summaryInfo.salesCommissionsServiceProviderFees`  | number           | Fee for the sales commission service                       |
| `summaryInfo.findersFeesServiceProviderName`       | string           | Name of the finder                                         |
| `summaryInfo.finderFeesFee`                        | number           | Fee the finder charges                                     |
| `summaryInfo.auditorServiceProviderName`           | string           | Name of the auditor                                        |
| `summaryInfo.auditorFees`                          | number           | Fee for the audit                                          |
| `summaryInfo.legalServiceProviderName`             | string           | Name of the legal provider                                 |
| `summaryInfo.legalFees`                            | number           | Fee for the legal work                                     |
| `summaryInfo.promotersServiceProviderName`         | string           | Name of the promoter                                       |
| `summaryInfo.promotersFees`                        | number           | Fee for the promotion                                      |
| `summaryInfo.blueSkyServiceProviderName`           | string           | Name of the blue sky compliance provider                   |
| `summaryInfo.blueSkyFees`                          | number           | Fee for blue sky compliance                                |
| `summaryInfo.brokerDealerCrdNumber`                | string           | CRD number of the broker-dealer in the offering            |
| `summaryInfo.estimatedNetAmount`                   | number           | Net proceeds the issuer expects after the deductions above |
| `summaryInfo.clarificationResponses`               | string           | Free text the filer adds to explain the numbers above      |

### 1-A `juridictionSecuritiesOffered`

| Field                                                             | Type             | Meaning                                              |
| ----------------------------------------------------------------- | ---------------- | ----------------------------------------------------- |
| `juridictionSecuritiesOffered.jurisdictionsOfSecOfferedNone`      | boolean          | No jurisdiction is named for the offering            |
| `juridictionSecuritiesOffered.jurisdictionsOfSecOfferedSame`      | boolean          | The same jurisdictions apply across the offering     |
| `juridictionSecuritiesOffered.issueJuridicationSecuritiesOffering` | array of strings | Codes of the states and territories where the issuer offers the securities |
| `juridictionSecuritiesOffered.dealersJuridicationSecuritiesOffering` | array of strings | Codes of the states and territories where the dealers work |

### 1-A `securitiesIssued[]` and the unregistered blocks

The securities the issuer sold without registration in the past year, and the
exemption it claims for them.

| Field                                              | Type    | Meaning                                                            |
| -------------------------------------------------- | ------- | ------------------------------------------------------------------- |
| `securitiesIssued[].securitiesIssuerName`          | string  | Name of the issuer of those securities                             |
| `securitiesIssued[].securitiesIssuerTitle`         | string  | Title of the class of securities issued                            |
| `securitiesIssued[].securitiesIssuedTotalAmount`   | number  | Total amount of securities issued                                  |
| `securitiesIssued[].securitiesPrincipalHolderAmount` | number | Amount the principal security holder holds                        |
| `securitiesIssued[].securitiesIssuedAggregateAmount` | string | Aggregate consideration for the securities issued. Free text.      |
| `securitiesIssued[].aggregateConsiderationBasis`   | string  | Basis on which the issuer works out that aggregate consideration   |
| `unregisteredSecurities.ifUnregsiteredNone`        | boolean | The issuer has issued no unregistered securities                   |
| `unregisteredSecuritiesAct.securitiesActExcemption` | string | The Securities Act exemption the issuer claims for those sales     |

### 1-K `item1`, `item1Info[]` and `item2`

| Field                                    | Type             | Meaning                                                        |
| ---------------------------------------- | ---------------- | --------------------------------------------------------------- |
| `item1.formIndication`                   | string           | `Annual Report` or `Special Financial Report for the fiscal year` |
| `item1.fiscalYearEnd`                    | string           | End date of the fiscal year the report covers, `MM-DD-YYYY`    |
| `item1.street1`                          | string           | First line of the principal executive office address           |
| `item1.street2`                          | string           | Second line of that address                                    |
| `item1.city`                             | string           | City of the principal office                                   |
| `item1.stateOrCountry`                   | string           | State or country of the principal office                       |
| `item1.zipCode`                          | string           | Postal code of the principal office                            |
| `item1.phoneNumber`                      | string           | Telephone number of the principal office                       |
| `item1.issuedSecuritiesTitle`            | array of strings | Titles of the classes of securities issued under Regulation A. A series issuer lists one title per series. |
| `item1Info[].issuerName`                 | string           | Name of the issuer as registered with the SEC                  |
| `item1Info[].cik`                        | string           | CIK of the issuer, padded to 10 digits                         |
| `item1Info[].jurisdictionOrganization`   | string           | State or country under which the issuer is organized           |
| `item1Info[].irsNum`                     | string           | IRS Employer Identification Number of the issuer               |
| `item2.regArule257`                      | boolean          | The issuer states that it meets Regulation A Rule 257          |

### 1-K `summaryInfo[]`

Offering results to date. One entry per commission file number. This block is an
array on a 1-K row.

| Field                                       | Type             | Meaning                                                       |
| ------------------------------------------- | ---------------- | -------------------------------------------------------------- |
| `summaryInfo[].commissionFileNumber`        | string           | SEC file number of the offering statement the entry reports on |
| `summaryInfo[].offeringQualificationDate`   | string           | Date the SEC qualified the offering, `MM-DD-YYYY`             |
| `summaryInfo[].offeringCommenceDate`        | string           | Date the offering started, `MM-DD-YYYY`                       |
| `summaryInfo[].qualifiedSecuritiesSold`     | number           | Number of securities qualified for sale                       |
| `summaryInfo[].offeringSecuritiesSold`      | number           | Number of securities sold                                     |
| `summaryInfo[].pricePerSecurity`            | number           | Price per security in the offering                            |
| `summaryInfo[].aggregrateOfferingPrice`     | number           | Total sales price of the securities the issuer sold           |
| `summaryInfo[].aggregrateOfferingPriceHolders` | number        | Total sales price of the securities that selling security holders sold |
| `summaryInfo[].underwrittenSpName`          | array of strings | Name of the underwriter                                       |
| `summaryInfo[].underwriterFees`             | number           | Fee the underwriter charges                                   |
| `summaryInfo[].salesCommissionsSpName`      | array of strings | Name of the provider that handles the sales commissions       |
| `summaryInfo[].salesCommissionsFee`         | number           | Fee for the sales commission service                          |
| `summaryInfo[].findersSpName`               | array of strings | Name of the finder                                            |
| `summaryInfo[].findersFees`                 | number           | Fee the finder charges                                        |
| `summaryInfo[].auditorSpName`               | array of strings | Name of the auditor                                           |
| `summaryInfo[].auditorFees`                 | number           | Fee for the audit                                             |
| `summaryInfo[].legalSpName`                 | array of strings | Name of the legal provider                                    |
| `summaryInfo[].legalFees`                   | number           | Fee for the legal work                                        |
| `summaryInfo[].promoterSpName`              | array of strings | Name of the promoter                                          |
| `summaryInfo[].promotersFees`               | number           | Fee for the promotion                                         |
| `summaryInfo[].blueSkySpName`               | array of strings | Name of the blue sky compliance provider                      |
| `summaryInfo[].blueSkyFees`                 | number           | Fee for blue sky compliance                                   |
| `summaryInfo[].crdNumberBrokerDealer`       | string           | CRD number of the broker or dealer in the offering            |
| `summaryInfo[].issuerNetProceeds`           | number           | Proceeds the issuer keeps after the fees and expenses         |
| `summaryInfo[].clarificationResponses`      | string           | Free text the filer adds to explain the numbers above         |

### 1-Z `item1`

| Field                          | Type             | Meaning                                                              |
| ------------------------------ | ---------------- | --------------------------------------------------------------------- |
| `item1.issuerName`             | string           | Legal name of the issuer, as stated in its charter                   |
| `item1.street1`                | string           | First line of the principal executive office address                 |
| `item1.street2`                | string           | Second line of that address                                          |
| `item1.city`                   | string           | City of the principal office                                         |
| `item1.stateOrCountry`         | string           | State or country of the principal office                             |
| `item1.zipCode`                | string           | Postal code of the principal office                                  |
| `item1.phone`                  | string           | Telephone number of the principal office. The key is `phone` here, not `phoneNumber` as on a 1-K row. |
| `item1.commissionFileNumber`   | array of strings | Commission file numbers the SEC assigned to the issuer               |

### 1-Z `summaryInfoOffering[]`

The final result of the offering.

| Field                                                    | Type             | Meaning                                              |
| -------------------------------------------------------- | ---------------- | ----------------------------------------------------- |
| `summaryInfoOffering[].offeringQualificationDate`        | string           | Date the SEC qualified the offering, `MM-DD-YYYY`    |
| `summaryInfoOffering[].offeringCommenceDate`             | string           | Date the offering started, `MM-DD-YYYY`              |
| `summaryInfoOffering[].offeringSecuritiesQualifiedSold`  | number           | Number of securities qualified for sale              |
| `summaryInfoOffering[].offeringSecuritiesSold`           | number           | Number of securities sold                            |
| `summaryInfoOffering[].pricePerSecurity`                 | number           | Price per security in the offering                   |
| `summaryInfoOffering[].portionSecuritiesSoldIssuer`      | number           | Part of the sales that comes from the issuer         |
| `summaryInfoOffering[].portionSecuritiesSoldSecurityholders` | number       | Part of the sales that comes from selling security holders |
| `summaryInfoOffering[].underwrittenSpName`               | array of strings | Name of the underwriter                              |
| `summaryInfoOffering[].underwriterFees`                  | number           | Fee the underwriter charges                          |
| `summaryInfoOffering[].salesCommissionsSpName`           | array of strings | Name of the provider that handles the sales commissions |
| `summaryInfoOffering[].salesCommissionsFee`              | number           | Fee for the sales commission service                 |
| `summaryInfoOffering[].findersSpName`                    | array of strings | Name of the finder                                   |
| `summaryInfoOffering[].findersFees`                      | number           | Fee the finder charges                               |
| `summaryInfoOffering[].auditorSpName`                    | array of strings | Name of the auditor                                  |
| `summaryInfoOffering[].auditorFees`                      | number           | Fee for the audit                                    |
| `summaryInfoOffering[].legalSpName`                      | array of strings | Name of the legal provider                           |
| `summaryInfoOffering[].legalFees`                        | number           | Fee for the legal work                               |
| `summaryInfoOffering[].promoterSpName`                   | array of strings | Name of the promoter                                 |
| `summaryInfoOffering[].promotersFees`                    | number           | Fee for the promotion                                |
| `summaryInfoOffering[].blueSkySpName`                    | array of strings | Name of the blue sky compliance provider             |
| `summaryInfoOffering[].blueSkyFees`                      | number           | Fee for blue sky compliance                          |
| `summaryInfoOffering[].crdNumberBrokerDealer`            | string           | CRD number of the broker-dealer in the offering      |
| `summaryInfoOffering[].issuerNetProceeds`                | number           | Proceeds the issuer keeps after the fees and expenses |
| `summaryInfoOffering[].clarificationResponses`           | string           | Free text the filer adds to explain the numbers above |

Filers write `"None"` or `"-"` in the `*SpName` arrays when a role was unused.
Those two values are not company names.

### 1-Z `certificationSuspension[]` and `signatureTab[]`

| Field                                            | Type             | Meaning                                                   |
| ------------------------------------------------ | ---------------- | ---------------------------------------------------------- |
| `certificationSuspension[].securitiesClassTitle` | string           | Title of the class of securities the exit report covers   |
| `certificationSuspension[].certificationFileNumber` | array of strings | Commission file number that the suspension certification refers to |
| `certificationSuspension[].approxRecordHolders`  | number           | Approximate number of record holders at the certification date |
| `signatureTab[].cik`                             | string           | CIK of the issuer, padded to 10 digits                    |
| `signatureTab[].regulationIssuerName1`           | string           | Primary issuer name recorded in the filing                |
| `signatureTab[].regulationIssuerName2`           | string           | Second issuer name, when the filing gives one             |
| `signatureTab[].signatureBy`                     | string           | Name of the person who signs for the issuer               |
| `signatureTab[].date`                            | string           | Date the person signed, `MM-DD-YYYY`                      |
| `signatureTab[].title`                           | string           | Title of that person in the issuer organization           |

Dates inside the parsed blocks use `MM-DD-YYYY`. Only the top-level `filedAt`
and `periodOfReport` use ISO order. The two formats are not interchangeable.

Copy the spellings exactly. Several keys are misspelled in the API and will not
match a corrected guess: `juridictionSecuritiesOffered`,
`issueJuridicationSecuritiesOffering`, `dealersJuridicationSecuritiesOffering`,
`securityHolderAggegate`, `securitiesActExcemption`, `ifUnregsiteredNone` and
`aggregrateOfferingPrice`.

Each form family also has its own page: [reg-a-form-1a](./reg-a-form-1a.md),
[reg-a-form-1k](./reg-a-form-1k.md) and [reg-a-form-1z](./reg-a-form-1z.md).
Those pages list the query fields for one family.

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.

## Example

Prompt: "Show me the newest Regulation A filings."

```json
{ "name": "reg-a-search", "arguments": { "query": "cik:*", "size": 1 } }
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
      "employeesInfo": [
        { "issuerName": "Blue Star Foods Corp.", "jurisdictionOrganization": "DE", "yearIncorporation": "2017", "sicCode": 3510, "fullTimeEmployees": 6, "partTimeEmployees": 0 }
      ],
      "summaryInfo": {
        "indicateTier1Tier2Offering": "Tier2",
        "financialStatementAuditStatus": "Audited",
        "securitiesOffered": 500000000,
        "pricePerSecurity": 0.001,
        "totalAggregateOffering": 500000
      }
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- The registry description says the tool returns "status". No `status` field
  exists. Infer status from `formType`: a 1-Z means the offering closed.
- The registry says Reg A offerings are capped at $75M. That is the legal Tier 2
  ceiling, not a field in the response.
- A bare word query is accepted but the analysis is strict. `apple` returned
  zero rows. Use a field query.
- A `1-A-W` withdrawal returns only the first seven metadata fields. EDGAR
  holds no structured data for it.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-form-1a](./reg-a-form-1a.md), [reg-a-form-1k](./reg-a-form-1k.md), [reg-a-form-1z](./reg-a-form-1z.md)
- [form-d](./form-d.md), [form-c](./form-c.md), [form-s1-424b4](./form-s1-424b4.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
