# form-nport

Search Form N-PORT filings for the monthly portfolio holdings of U.S. mutual
funds, ETFs and closed-end funds.

|                 |                                                                    |
| --------------- | ------------------------------------------------------------------ |
| Category        | Funds                                                              |
| Required input  | `query`                                                            |
| Returns         | `{total, filings[]}`                                               |
| Pagination      | `from`, `size`, `sort`. The schema allows `size` up to 50. The server caps this tool at 10. See [Limits and errors](#limits-and-errors). |
| REST equivalent | `POST /form-nport`                                                 |

## What it does

The tool searches parsed N-PORT filings. Funds file N-PORT every quarter and
report three months of data in each filing. One item in `filings[]` is one
filing for one fund series. It carries the fund identity in `genInfo`, the
fund-level totals and flows in `fundInfo`, and the complete position list in
`invstOrSecs`. Coverage runs from 2019 to present, for form types `NPORT-P` and
`NPORT-P/A`. A request for `genInfo.regName:*`
returns `total: {value: 10000, relation: "gte"}`. That means 10,000 or more.

Each filing is large, and this is the heaviest tool on the server. A filing
carries every position the fund held. The example filing holds 70 positions in
45 KB at `size: 1`. A large fund holds thousands, and one filing alone can
exceed anything else the server returns.

## When to use it

- What securities did a fund hold at the end of the last reporting period?
- Which funds hold a given CUSIP or ISIN?
- How big is the fund, and what were its monthly flows?
- What percentage of the portfolio is one position?
- Is a position restricted, in default, or lent out?

## When to use a different tool

| Situation                                        | Better tool                                     | Why                                                                     |
| ------------------------------------------------ | ----------------------------------------------- | ----------------------------------------------------------------------- |
| You want equity positions of an institution      | [`form-13f-holdings`](./form-13f-holdings.md)   | 13F covers 13F-eligible equities of managers, not fund portfolios.       |
| You want the fund's annual census and structure  | [`form-ncen`](./form-ncen.md)                   | N-CEN reports service providers and structure, not holdings.          |
| You want how a fund voted its shares             | [`form-npx`](./form-npx.md)                     | N-PX carries proxy votes. N-PORT carries positions.                      |

## Input

| Parameter | Type    | Required | Constraints                        | Notes                                                        |
| --------- | ------- | -------- | ---------------------------------- | ------------------------------------------------------------ |
| `query`   | string  | Yes      | Must contain a colon. Maximum 2,000 characters. | Lucene syntax, for example `genInfo.regName:*`. |
| `from`    | integer | No       | Minimum 0                          | Offset of the first result. Default 0.                        |
| `size`    | integer | No       | Schema 1 to 50. Server maximum 10. | Number of filings, not holdings. Default 10.                  |
| `sort`    | array   | No       | Elasticsearch sort array           | Default `[{"filedAt": {"order": "desc"}}]`.                   |

Query fields:

- `genInfo.regName`
- `fundInfo.totAssets`, with range syntax such as `[100000000 TO *]`
- `genInfo.seriesName`, `genInfo.regCik`, `genInfo.repPdDate`,
  `genInfo.isFinalFiling`, `submissionType`, `accessionNo`, `filedAt`
- `filerInfo.seriesClassInfo.seriesId` and `filerInfo.seriesClassInfo.classId`
- The position level: `invstOrSecs.cusip`,
  `invstOrSecs.identifiers.isin.value`, `invstOrSecs.name`,
  `invstOrSecs.assetCat`, `invstOrSecs.invCountry`, `invstOrSecs.fairValLevel`,
  `invstOrSecs.isRestrictedSec`, `invstOrSecs.debtSec.isDefault`,
  `invstOrSecs.debtSec.couponKind` and
  `invstOrSecs.debtSec.areIntrstPmntsInArrs`

See [query language](../query-language.md).

## Output

The envelope is `{total, filings[]}`. It is **not** `data[]`. `total` is an
object, `{value, relation}`. A `relation` of `gte` means the count is capped.

A filing follows the parts of the form. `genInfo` is Part A, `fundInfo` is
Part B, and `invstOrSecs[]` is Part C. Most objects are optional. A fund
reports only the items that apply to it.

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | -------------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                         |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many.  |
| `filings[]`      | array  | The matching filings. One item is one filing of one fund series.|

### Filing

| Field                      | Type   | Meaning                                                       |
| -------------------------- | ------ | ------------------------------------------------------------- |
| `filings[].submissionType` | string | Form type, `NPORT-P` or `NPORT-P/A`.                          |
| `filings[].accessionNo`    | string | EDGAR accession number of the filing, such as `0000894189-26-022150`. |
| `filings[].filedAt`        | string | Date and time EDGAR accepted the filing, ISO 8601 with an offset. |
| `filings[].id`             | string | Internal identifier of the filing record.                      |
| `filings[].filerInfo`      | object | The filer, and the series and classes the filing covers.       |
| `filings[].genInfo`        | object | Part A. The filer and the reporting period.                    |
| `filings[].fundInfo`       | object | Part B. Assets, liabilities, risk, returns and flows of the fund. |
| `filings[].invstOrSecs[]`  | array  | Part C. The schedule of portfolio investments.                 |
| `filings[].explntrNotes`   | object | Notes the fund added to explain single items.                  |
| `filings[].signature`      | object | Who signed the filing, and when.                               |

### `filings[].filerInfo`

| Field                                    | Type  | Meaning                                                  |
| ---------------------------------------- | ----- | -------------------------------------------------------- |
| `filerInfo.filer.fileNumber`             | string | File number of the filer.                                |
| `filerInfo.filer.issuerCredentials.cik`  | string | CIK of the filer, zero padded to 10 digits.              |
| `filerInfo.filer.issuerCredentials.ccc`  | string | EDGAR access code of the filer. It is masked as `XXXXXXXX`. |
| `filerInfo.seriesClassInfo.seriesId`     | string | Series ID of the fund, such as `S000069692`.             |
| `filerInfo.seriesClassInfo.seriesLei`    | string | LEI of the series.                                       |
| `filerInfo.seriesClassInfo.seriesName`   | string | Name of the series.                                      |
| `filerInfo.seriesClassInfo.classId[]`    | array of string | Every class ID of the series, such as `C000222265`. |

### `filings[].genInfo`, Part A

| Field                                | Type   | Meaning                                                     |
| ------------------------------------ | ------ | ----------------------------------------------------------- |
| `genInfo.regName`                    | string | Name of the filer, that is the trust or fund company.       |
| `genInfo.regFileNumber`              | string | File number of the filer, for example `811-23928`.          |
| `genInfo.regCik`                     | string | CIK of the filer, zero padded to 10 digits.                 |
| `genInfo.regLei`                     | string | LEI of the filer.                                           |
| `genInfo.regStreet1`                 | string | Street 1 of the filer address.                              |
| `genInfo.regStreet2`                 | string | Street 2 of the filer address.                              |
| `genInfo.regCity`                    | string | City of the filer address.                                  |
| `genInfo.regZipOrPostalCode`         | string | ZIP or postal code of the filer address.                    |
| `genInfo.regCountry`                 | string | Country code of the filer address. `regStateConditional` carries the country when the address also names a state. |
| `genInfo.regStateConditional.regCountry` | string | Country code of the filer address, such as `US`.        |
| `genInfo.regStateConditional.regState`   | string | State code of the filer address, such as `US-NY`.       |
| `genInfo.regPhone`                   | string | Phone number of the filer.                                  |
| `genInfo.seriesName`                 | string | Name of the fund series this filing covers.                 |
| `genInfo.seriesId`                   | string | Series ID of that fund.                                     |
| `genInfo.seriesLei`                  | string | LEI of that series.                                         |
| `genInfo.repPdEnd`                   | string | End of the reporting period, `YYYY-MM-DD`.                  |
| `genInfo.repPdDate`                  | string | Date of the reporting period, `YYYY-MM-DD`.                 |
| `genInfo.isFinalFiling`              | string | `Y` when this is the last N-PORT filing of the fund.        |

### `filings[].fundInfo`: assets and borrowings, Items B.1 and B.2

| Field                             | Type   | Meaning                                                      |
| --------------------------------- | ------ | ------------------------------------------------------------ |
| `fundInfo.totAssets`              | number | Item B.1.a. Total assets in USD.                              |
| `fundInfo.totLiabs`               | number | Item B.1.b. Total liabilities in USD.                         |
| `fundInfo.netAssets`              | number | Item B.1.c. Net assets in USD.                                |
| `fundInfo.assetsAttrMiscSec`      | number | Item B.2.a. Assets attributable to miscellaneous securities reported in Part D. |
| `fundInfo.assetsInvested`         | number | Item B.2.b. Assets invested in a controlled foreign corporation, to hold instruments such as commodities. |
| `fundInfo.amtPayOneYrBanksBorr`   | number | Item B.2.c. Amounts payable within one year to banks or other financial institutions for borrowings. |
| `fundInfo.amtPayOneYrCtrldComp`   | number | Item B.2.c. Amounts payable within one year to controlled companies. |
| `fundInfo.amtPayOneYrOthAffil`    | number | Item B.2.c. Amounts payable within one year to other affiliates. |
| `fundInfo.amtPayOneYrOther`       | number | Item B.2.c. Amounts payable within one year to others.        |
| `fundInfo.amtPayAftOneYrBanksBorr`| number | Item B.2.c. Amounts payable after one year to banks or other financial institutions for borrowings. |
| `fundInfo.amtPayAftOneYrCtrldComp`| number | Item B.2.c. Amounts payable after one year to controlled companies. |
| `fundInfo.amtPayAftOneYrOthAffil` | number | Item B.2.c. Amounts payable after one year to other affiliates. |
| `fundInfo.amtPayAftOneYrOther`    | number | Item B.2.c. Amounts payable after one year to others.         |
| `fundInfo.delayDeliv`             | number | Item B.2.d.i. Payables for investments bought on a delayed delivery, when-issued or other firm commitment basis. |
| `fundInfo.standByCommit`          | number | Item B.2.d.ii. Payables for investments bought on a standby commitment basis. |
| `fundInfo.liquidPref`             | number | Item B.2.e. Liquidation preference of the preferred stock the fund issued. |
| `fundInfo.cshNotRptdInCorD`       | number | Item B.2.f. Cash and cash equivalents that Parts C and D do not report. |

### `filings[].fundInfo`: risk metrics, Item B.3

| Field                                        | Type   | Meaning                                      |
| -------------------------------------------- | ------ | -------------------------------------------- |
| `fundInfo.curMetrics.curMetric[]`            | array  | One item per currency in which the fund held 1% or more of its net assets. |
| `curMetric[].curCd`                          | string | ISO currency code of that currency.           |
| `curMetric[].intrstRtRiskdv01`               | object | Item B.3.a. Change in portfolio value from a 1 basis point change in interest rates. |
| `curMetric[].intrstRtRiskdv100`              | object | Item B.3.b. Change in portfolio value from a 100 basis point change in interest rates. |
| `fundInfo.creditSprdRiskInvstGrade`          | object | Item B.3.c. Change in portfolio value from a 1 basis point change in credit spreads, investment grade exposure. |
| `fundInfo.creditSprdRiskNonInvstGrade`       | object | Item B.3.c. The same measure for non-investment-grade exposure. |

Those four objects hold the same five keys, one per maturity.

| Field          | Type   | Meaning   |
| -------------- | ------ | --------- |
| `.period3Mon`  | number | 3 month.  |
| `.period1Yr`   | number | 1 year.   |
| `.period5Yr`   | number | 5 years.  |
| `.period10Yr`  | number | 10 years. |
| `.period30Yr`  | number | 30 years. |

### `filings[].fundInfo`: securities lending, Item B.4

| Field                            | Type   | Meaning                                                 |
| -------------------------------- | ------ | ------------------------------------------------------- |
| `fundInfo.borrowers[]`           | array  | Item B.4.a. One item per borrower of the fund's securities. |
| `borrowers[].name`               | string | Name of the borrower.                                    |
| `borrowers[].lei`                | string | LEI of the borrower.                                     |
| `borrowers[].aggregateValue`     | number | Value of all securities on loan to that borrower.        |
| `fundInfo.isNonCashCollateral`   | string | Item B.4.b. `Y` when a securities lending counterparty gave non-cash collateral. |
| `fundInfo.aggregateAmount`       | number | Item B.4.b.i. Aggregate principal amount of that collateral. |
| `fundInfo.aggregateCollateral`   | number | Item B.4.b.ii. Aggregate value of that collateral.       |
| `fundInfo.investmentCategory`    | string | Item B.4.b.iii. Category of investment that the collateral fits best, such as U.S. Treasuries or agency mortgage-backed securities. |

### `filings[].fundInfo`: returns and flows, Item B.5

| Field                                                  | Type   | Meaning                                 |
| ------------------------------------------------------ | ------ | --------------------------------------- |
| `fundInfo.returnInfo.monthlyTotReturns.monthlyTotReturn[]` | array | Total returns of the fund for the three months the filing covers. A multiple class fund reports one item per class. |
| `monthlyTotReturn[].classId`                           | string | Class ID the returns belong to.          |
| `monthlyTotReturn[].rtn1`                              | number | Total return of month 1, in percent.     |
| `monthlyTotReturn[].rtn2`                              | number | Total return of month 2.                 |
| `monthlyTotReturn[].rtn3`                              | number | Total return of month 3.                 |
| `fundInfo.returnInfo.monthlyReturnCategories[]`        | array  | Monthly net realized gain and net change in unrealized appreciation from derivatives, by asset category and by instrument type. Losses are negative. |
| `monthlyReturnCategories[].commodityContracts`         | object | Asset category: commodity contracts.     |
| `monthlyReturnCategories[].creditContracts`            | object | Asset category: credit contracts.        |
| `monthlyReturnCategories[].equityContracts`            | object | Asset category: equity contracts.        |
| `monthlyReturnCategories[].foreignExchgContracts`      | object | Asset category: foreign exchange contracts. |
| `monthlyReturnCategories[].interestRtContracts`        | object | Asset category: interest rate contracts. |
| `monthlyReturnCategories[].otherContracts`             | object | Asset category: other contracts.         |
| `fundInfo.returnInfo.othMon1.netRealizedGain`          | number | Net realized gain of month 1 in USD, outside the derivative categories above. A loss is negative. |
| `fundInfo.returnInfo.othMon1.netUnrealizedAppr`        | number | Net change in unrealized appreciation of month 1 in USD. Depreciation is negative. |
| `fundInfo.returnInfo.othMon2.netRealizedGain`          | number | The same figure for month 2.             |
| `fundInfo.returnInfo.othMon2.netUnrealizedAppr`        | number | The same figure for month 2.             |
| `fundInfo.returnInfo.othMon3.netRealizedGain`          | number | The same figure for month 3.             |
| `fundInfo.returnInfo.othMon3.netUnrealizedAppr`        | number | The same figure for month 3.             |
| `fundInfo.mon1Flow.sales`                              | number | Money the fund received for new shares in month 1. |
| `fundInfo.mon1Flow.redemption`                         | number | Money the fund paid out for share redemptions in month 1. |
| `fundInfo.mon1Flow.reinvestment`                       | number | Amount from dividend reinvestment in month 1. |
| `fundInfo.mon2Flow.sales`                              | number | The same figure for month 2.             |
| `fundInfo.mon2Flow.redemption`                         | number | The same figure for month 2.             |
| `fundInfo.mon2Flow.reinvestment`                       | number | The same figure for month 2.             |
| `fundInfo.mon3Flow.sales`                              | number | The same figure for month 3.             |
| `fundInfo.mon3Flow.redemption`                         | number | The same figure for month 3.             |
| `fundInfo.mon3Flow.reinvestment`                       | number | The same figure for month 3.             |
| `fundInfo.varInfo.fundsDesignatedInfo.nameDesignatedIndex` | string | Name of the index the fund designated. `N/A` when it designated none. |
| `fundInfo.varInfo.fundsDesignatedInfo.indexIdentifier` | string | Identifier of that index. `N/A` when the fund designated none. |

### `filings[].invstOrSecs[]`, Part C

One item is one position. The array holds the whole schedule of portfolio
investments.

| Field                                      | Type   | Meaning                                       |
| ------------------------------------------ | ------ | --------------------------------------------- |
| `invstOrSecs[].name`                       | string | Name of the issuer.                            |
| `invstOrSecs[].lei`                        | string | LEI of the issuer. For a holding in a series of a series trust, the LEI of the series. `N/A` when there is none. |
| `invstOrSecs[].title`                      | string | Title of the issue, or a description of the investment. |
| `invstOrSecs[].cusip`                      | string | CUSIP of the security.                         |
| `invstOrSecs[].identifiers.isin.value`     | string | ISIN of the security.                          |
| `invstOrSecs[].identifiers.ticker.value`   | string | Ticker of the security. It is used when there is no CUSIP or ISIN. |
| `invstOrSecs[].identifiers.other.value`    | string | Another identifier, used when there is no CUSIP, ISIN or ticker. |
| `invstOrSecs[].identifiers.other.otherDesc`| string | Description of that other identifier.          |
| `invstOrSecs[].balance`                    | number | Quantity held, in the unit that `units` names. |
| `invstOrSecs[].units`                      | string | Unit of the balance. `NS` is number of shares, `PA` is principal amount, `OU` is another unit. |
| `invstOrSecs[].descOthUnits`               | string | Description of the unit when `units` is another unit. |
| `invstOrSecs[].curCd`                      | string | ISO code of the currency the investment is denominated in. |
| `invstOrSecs[].currencyConditional.curCd`  | string | Currency code, reported here instead of `curCd` when the investment is not in USD. |
| `invstOrSecs[].currencyConditional.exchangeRt` | number | Exchange rate used to convert the value to USD. |
| `invstOrSecs[].valUSD`                     | number | Value of the position in USD.                  |
| `invstOrSecs[].pctVal`                     | number | Value as a percent of the fund's net assets. The 70 example positions sum to 97.6. |
| `invstOrSecs[].payoffProfile`              | string | `Long`, `Short` or `N/A`. Derivatives report `N/A` here and answer in `derivativeInfo`. |
| `invstOrSecs[].assetCat`                   | string | Asset type. The form covers equity, debt, loans, repurchase agreements, short-term investment vehicles, structured notes, derivatives by underlying, asset-backed securities, commodities, real estate and other. The example response holds `ABS-MBS`, `STIV`, `LON` and `EC`. |
| `invstOrSecs[].issuerCat`                  | string | Issuer type. The form covers corporate, U.S. Treasury, U.S. government agency, U.S. government sponsored entity, municipal, non-U.S. sovereign, private fund, registered fund and other. The example response holds `CORP`. |
| `invstOrSecs[].invCountry`                 | string | ISO code of the country where the issuer is organized. |
| `invstOrSecs[].isRestrictedSec`            | string | `Y` when the investment is a restricted security. |
| `invstOrSecs[].fairValLevel`               | string | Level in the fair value hierarchy of ASC 820, `1`, `2` or `3`. `N/A` when the position has no level, for example when net asset value is the practical expedient. |
| `invstOrSecs[].debtSec`                    | object | Item C.9. Terms of a debt security.            |
| `invstOrSecs[].repurchaseAgrmt`            | object | Item C.10. Terms of a repurchase agreement.    |
| `invstOrSecs[].derivativeInfo`             | object | Item C.11. Terms of a derivative.              |
| `invstOrSecs[].securityLending`            | object | Item C.12. Securities lending answers for the position. |

### `invstOrSecs[].debtSec`, Item C.9

Debt securities only. It covers 68 of the 70 example positions. Equity
positions carry no `debtSec`.

| Field                          | Type   | Meaning                                                 |
| ------------------------------ | ------ | ------------------------------------------------------- |
| `debtSec.maturityDt`           | string | Maturity date, `YYYY-MM-DD`.                             |
| `debtSec.couponKind`           | string | Coupon type that fits best, such as `Fixed`, `Floating` or `Variable`. The form also allows none. |
| `debtSec.annualizedRt`         | number | Annualized coupon rate, in percent.                      |
| `debtSec.isDefault`            | string | `Y` when the security is in default.                     |
| `debtSec.areIntrstPmntsInArrs` | string | `Y` when interest payments are in arrears, or the issuer legally deferred a coupon. |
| `debtSec.isPaidKind`           | string | `Y` when part of the interest is actually paid in kind.  |
| `debtSec.isMandatoryConvrtbl`  | string | `Y` when the security is a mandatory convertible.        |
| `debtSec.isContngtConvrtbl`    | string | `Y` when the security is a contingent convertible.       |
| `debtSec.dbtSecRefInstruments.dbtSecRefInstrument[]` | array | The reference instrument of a convertible. |
| `dbtSecRefInstrument[].name`   | string | Name of the issuer of the reference instrument.          |
| `dbtSecRefInstrument[].title`  | string | Title of the issue of the reference instrument.          |
| `dbtSecRefInstrument[].curCd`  | string | Currency the reference instrument is denominated in.     |
| `dbtSecRefInstrument[].identifiers.cusip` | object | CUSIP of the reference instrument.            |
| `dbtSecRefInstrument[].identifiers.isin`  | object | ISIN, used when there is no CUSIP.            |
| `dbtSecRefInstrument[].identifiers.ticker`| object | Ticker, used when there is no CUSIP or ISIN.  |
| `dbtSecRefInstrument[].identifiers.other` | object | Another identifier, used when there is no CUSIP, ISIN or ticker. |
| `debtSec.currencyInfos`        | object | Conversion ratio per 1,000 units of the bond currency. It holds one entry per ratio. |
| `debtSec.delta`                | string | Delta of the conversion feature.                         |

### `invstOrSecs[].repurchaseAgrmt`, Item C.10

Repurchase and reverse repurchase agreements only.

| Field                                              | Type   | Meaning                          |
| -------------------------------------------------- | ------ | -------------------------------- |
| `repurchaseAgrmt.transCat`                         | string | Category of the transaction. Repurchase means the fund lends cash and takes collateral. Reverse repurchase means the fund borrows cash and posts collateral. |
| `repurchaseAgrmt.clearedCentCparty.isCleared`      | string | `Y` when a central counterparty cleared the trade. |
| `repurchaseAgrmt.clearedCentCparty.centralCounterparty` | string | Name of that central counterparty. |
| `repurchaseAgrmt.notClearedCentCparty`             | object | Name and LEI of the counterparty when no central counterparty cleared the trade. |
| `repurchaseAgrmt.isTriParty`                       | string | `Y` when the agreement is tri-party. |
| `repurchaseAgrmt.repurchaseRt`                     | number | Repurchase rate.                  |
| `repurchaseAgrmt.maturityDt`                       | string | Maturity date, `YYYY-MM-DD`.      |
| `repurchaseAgrmt.repurchaseCollaterals.repurchaseCollateral[]` | array | The securities that back the agreement. |
| `repurchaseCollateral[].principalAmt`              | number | Principal amount of the collateral. |
| `repurchaseCollateral[].principalCd`               | string | Currency of that principal amount. |
| `repurchaseCollateral[].collateralVal`             | number | Value of the collateral.          |
| `repurchaseCollateral[].collateralCd`              | string | Currency of that value.           |
| `repurchaseCollateral[].invstCat`                  | string | Category of investment that the collateral fits best, such as corporate debt securities or U.S. Treasuries. |

### `invstOrSecs[].derivativeInfo`, Item C.11

Derivative positions only. One of five objects appears, by the type of the
contract.

Every type carries `counterparties[]` and `descRefInstrmnt`.

| Field                                    | Type   | Meaning                                       |
| ---------------------------------------- | ------ | --------------------------------------------- |
| `counterparties[]`                       | array  | One item per counterparty, a central counterparty included. |
| `counterparties[].counterpartyName`      | string | Name of the counterparty.                      |
| `counterparties[].counterpartyLei`       | string | LEI of the counterparty. `N/A` when there is none. |
| `descRefInstrmnt.indexBasketInfo.indexName` | string | Name of the index or basket the contract references. |
| `descRefInstrmnt.indexBasketInfo.indexIdentifier` | string | Identifier of that index.        |
| `descRefInstrmnt.indexBasketInfo.narrativeDesc`   | string | Description of the components of that basket. |
| `descRefInstrmnt.otherRefInst.issuerName` | string | Name of the issuer of a single-name reference instrument. |
| `descRefInstrmnt.otherRefInst.issueTitle` | string | Title of that issue.                          |

`derivativeInfo.fwdDeriv`, forwards.

| Field                    | Type   | Meaning                                              |
| ------------------------ | ------ | ---------------------------------------------------- |
| `fwdDeriv.amtCurSold`    | number | Amount of currency sold.                              |
| `fwdDeriv.curSold`       | string | Currency sold.                                        |
| `fwdDeriv.amtCurPur`     | number | Amount of currency purchased.                         |
| `fwdDeriv.curPur`        | string | Currency purchased.                                   |
| `fwdDeriv.settlementDt`  | string | Settlement date.                                      |
| `fwdDeriv.payOffProf`    | string | Payoff profile, `Long` or `Short`.                    |
| `fwdDeriv.expDate`       | string | Expiration date.                                      |
| `fwdDeriv.notionalAmt`   | number | Notional amount or contract value on the trade date.  |
| `fwdDeriv.curCd`         | string | Currency of that notional amount.                     |
| `fwdDeriv.unrealizedAppr`| number | Unrealized appreciation. Depreciation is negative.    |
| `fwdDeriv.derivCat`      | string | Type of derivative that fits the contract best.       |

`derivativeInfo.futrDeriv`, futures.

| Field                     | Type   | Meaning                                             |
| ------------------------- | ------ | --------------------------------------------------- |
| `futrDeriv.payOffProf`    | string | Payoff profile, `Long` or `Short`.                   |
| `futrDeriv.expDate`       | string | Expiration date.                                     |
| `futrDeriv.notionalAmt`   | number | Notional amount or contract value on the trade date. |
| `futrDeriv.curCd`         | string | Currency of that notional amount.                    |
| `futrDeriv.unrealizedAppr`| number | Unrealized appreciation. Depreciation is negative.   |
| `futrDeriv.derivCat`      | string | Type of derivative that fits the contract best.      |

`derivativeInfo.swapDeriv`, swaps.

| Field                        | Type   | Meaning                                          |
| ---------------------------- | ------ | ------------------------------------------------ |
| `swapDeriv.amtCurSold`       | number | Amount of currency sold.                          |
| `swapDeriv.curSold`          | string | Currency sold.                                    |
| `swapDeriv.amtCurPur`        | number | Amount of currency purchased.                     |
| `swapDeriv.curPur`           | string | Currency purchased.                               |
| `swapDeriv.settlementDt`     | string | Settlement date.                                  |
| `swapDeriv.swapFlag`         | string | `Y` when a swap execution facility traded the swap.|
| `swapDeriv.fixedRecDesc`     | object | The fixed leg the fund receives.                  |
| `swapDeriv.floatingRecDesc`  | object | The floating leg the fund receives.               |
| `swapDeriv.otherRecDesc`     | object | A received leg that is neither plain fixed nor floating, such as an index return. |
| `swapDeriv.fixedPmntDesc`    | object | The fixed leg the fund pays.                      |
| `swapDeriv.floatingPmntDesc` | object | The floating leg the fund pays.                   |
| `swapDeriv.otherPmntDesc`    | object | A paid leg that is neither plain fixed nor floating. |
| `swapDeriv.terminationDt`    | string | Termination or maturity date.                     |
| `swapDeriv.upfrontPmnt`      | number | Upfront payment the fund made.                    |
| `swapDeriv.pmntCurCd`        | string | Currency of that payment.                         |
| `swapDeriv.upfrontRcpt`      | number | Upfront payment the fund received.                |
| `swapDeriv.rcptCurCd`        | string | Currency of that receipt.                         |
| `swapDeriv.notionalAmt`      | number | Notional amount.                                  |
| `swapDeriv.curCd`            | string | Currency of that notional amount.                 |
| `swapDeriv.unrealizedAppr`   | number | Unrealized appreciation. Depreciation is negative.|
| `swapDeriv.derivCat`         | string | Type of derivative, `SWP` for this object.        |

The fixed legs hold `amount`, `curCd`, `fixedOrFloating` and `fixedRt`. The
floating legs hold `curCd`, `fixedOrFloating`, `floatingRtIndex`,
`floatingRtSpread`, `pmntAmt` and `rtResetTenors`.

| Field                                          | Type   | Meaning                       |
| ---------------------------------------------- | ------ | ----------------------------- |
| `[leg].amount`                                 | number | Amount of the leg.             |
| `[leg].curCd`                                  | string | Currency of the leg.           |
| `[leg].fixedOrFloating`                        | string | `Fixed` or `Floating`.         |
| `[leg].fixedRt`                                | number | Rate of a fixed leg.           |
| `[leg].floatingRtIndex`                        | string | Reference rate of a floating leg, such as `Thai Baht Interest Rate Fixing 6 Months`. |
| `[leg].floatingRtSpread`                       | number | Spread over that reference rate. |
| `[leg].pmntAmt`                                | number | Payment amount of a floating leg.|
| `[leg].rtResetTenors.rtResetTenor[]`           | array  | One item per reset rule of the leg. |
| `rtResetTenor[].rateTenor`                     | string | Period of the reference rate, such as `Month` or `Day`. |
| `rtResetTenor[].rateTenorUnit`                 | string | Number of those periods, such as `3`. |
| `rtResetTenor[].resetDt`                       | string | Period between rate resets, such as `Month`. |
| `rtResetTenor[].resetDtUnit`                   | string | Number of those periods, such as `6`. |

`derivativeInfo.optionSwaptionWarrantDeriv`, options, swaptions and warrants.

| Field                                            | Type   | Meaning                     |
| ------------------------------------------------ | ------ | --------------------------- |
| `optionSwaptionWarrantDeriv.putOrCall`           | string | `Put` or `Call`.             |
| `optionSwaptionWarrantDeriv.writtenOrPur`        | string | `Written` when the fund sold the contract. `Purchased` when it bought it. |
| `optionSwaptionWarrantDeriv.shareNo`             | number | Shares, or principal amount, of the reference instrument per contract. |
| `optionSwaptionWarrantDeriv.exercisePrice`       | number | Exercise price or rate.      |
| `optionSwaptionWarrantDeriv.exercisePriceCurCd`  | string | Currency of that exercise price. |
| `optionSwaptionWarrantDeriv.expDt`               | string | Expiration date.             |
| `optionSwaptionWarrantDeriv.delta`               | number | Delta. Only options on equities report it. |
| `optionSwaptionWarrantDeriv.unrealizedAppr`      | number | Unrealized appreciation. Depreciation is negative. |
| `optionSwaptionWarrantDeriv.derivCat`            | string | Type of derivative, `OPT` option, `SWO` swaption or `WAR` warrant. |

`derivativeInfo.othDeriv`, derivatives of any other type.

| Field                                        | Type   | Meaning                         |
| -------------------------------------------- | ------ | ------------------------------- |
| `othDeriv.terminationDt`                     | string | Termination or maturity date.    |
| `othDeriv.notionalAmts.notionalAmt[].amt`    | number | A notional amount of the contract. |
| `othDeriv.notionalAmts.notionalAmt[].curCd`  | string | Currency of that amount.         |
| `othDeriv.delta`                             | number | Delta. Only options on equities report it. |
| `othDeriv.unrealizedAppr`                    | number | Unrealized appreciation. Depreciation is negative. |
| `othDeriv.derivCat`                          | string | Type of derivative, `OTH` for this object. |
| `othDeriv.othDesc`                           | string | Description of the instrument, such as `rights`. |

### `invstOrSecs[].securityLending`, Item C.12

| Field                                | Type   | Meaning                                             |
| ------------------------------------ | ------ | --------------------------------------------------- |
| `securityLending.isCashCollateral`   | string | `Y` when the position reinvests cash collateral that the fund got for loaned securities. |
| `securityLending.isNonCashCollateral`| string | `Y` when the position is non-cash collateral that the fund got for loaned securities and treats as an asset. |
| `securityLending.isLoanByFund`       | string | `Y` when the fund lent part of the position out.     |

### `filings[].explntrNotes` and `filings[].signature`

| Field                                | Type   | Meaning                                              |
| ------------------------------------ | ------ | ---------------------------------------------------- |
| `explntrNotes.explntrNote[]`         | array  | Notes the fund added to explain its answers.          |
| `explntrNote[].note`                 | string | Text of the note.                                     |
| `explntrNote[].noteItem`             | string | Form item the note explains, such as `B.5.a`.         |
| `signature.nameOfApplicant`          | string | Name of the entity that signed the filing.            |
| `signature.signature`                | string | Signature line, such as `/s/ Michael Schoonover`.     |
| `signature.signerName`               | string | Name of the person who signed.                        |
| `signature.title`                    | string | Job title of that person, such as `President`.        |
| `signature.dateSigned`               | string | Date of the signature, `YYYY-MM-DD`.                  |

`size` counts filings, not positions. Every returned filing carries its whole
`invstOrSecs` array, so the response grows with each portfolio.

The JSON arrives as one stringified text block. See
[response format](../response-format.md).

## Example

Prompt: "Show me the newest N-PORT filing from any fund, with its holdings."

```json
{ "name": "form-nport", "arguments": { "query": "genInfo.regName:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "filings": [
    {
      "submissionType": "NPORT-P",
      "genInfo": {
        "regName": "Catalyst/Perini Strategic Income Fund",
        "repPdEnd": "2026-09-30",
        "repPdDate": "2026-06-30"
      },
      "fundInfo": { "totAssets": 52966273.23, "netAssets": 52833180.21 },
      "invstOrSecs": [
        {
          "name": "Banc of America Mortgage Secur",
          "cusip": "05955BAG4",
          "balance": 20223128.61,
          "valUSD": 252789.11,
          "pctVal": 0.4784665791,
          "assetCat": "ABS-MBS"
        }
      ],
      "accessionNo": "0000894189-26-022150"
    }
  ]
}
```

Keys were removed to fit. The values are unchanged.

## Limits and errors

- A `query` without a colon fails with HTTP 400 and `Invalid query`.
- A `query` longer than 2,000 characters fails with
  `Query too long. Maximum length: 2000 characters`.
- The server caps `size` at **10** for N-PORT, while the tool schema advertises
  50. A `size` of 11 to 50 passes the schema, then fails with
  `Maximum 'size' limit of 10 exceeded`. `size` also defaults to 10.
- Responses are heavy. A 70-position fund returns about 45 KB per filing.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-ncen`](./form-ncen.md), [`form-npx`](./form-npx.md),
  [`form-npx-file`](./form-npx-file.md)
- [`form-13f-holdings`](./form-13f-holdings.md)
- REST documentation: [Form N-PORT API](https://sec-api.io/docs/n-port-data-api)
