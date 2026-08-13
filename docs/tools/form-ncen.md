# form-ncen

Search Form N-CEN filings, the annual census report of registered investment
companies.

|                 |                                 |
| --------------- | ------------------------------- |
| Category        | Funds                           |
| Required input  | `query`                         |
| Returns         | `{total, data[]}`               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-ncen`               |

## What it does

The tool searches parsed Form N-CEN filings. Every registered investment company
files N-CEN once a year. That includes mutual funds, ETFs, closed-end funds,
unit investment trusts and insurance separate accounts. One item in `data[]` is
one N-CEN filing. It carries the registrant identity, the share classes covered,
the chief compliance officer, the principal underwriter, the public accountant,
and the sections that fit the registrant type.

The data starts in June 2018.

The example filing is a unit investment trust, an insurance separate account.
Its type-specific data sits under `unitInvestmentTrust`. The registry
description also promises adviser, custodian, transfer agent and
securities-lending data. Those sections sit under
`managementInvestmentQuestionSeriesInfo[]`, one entry per fund series, and
belong to management investment companies. They are absent from the example
response.

## When to use it

- Who audits this fund, and who underwrites it?
- Who is the fund's chief compliance officer?
- Which share classes does this registrant cover?
- Which funds belong to one fund family?
- Which registrants are incorporated in a given state?

## When to use a different tool

| Situation                              | Better tool                             | Why                                                     |
| -------------------------------------- | --------------------------------------- | --------------------------------------------------------- |
| You want what the fund holds           | [`form-nport`](./form-nport.md)         | N-PORT reports positions every quarter. N-CEN does not.    |
| You want how the fund voted            | [`form-npx`](./form-npx.md)             | N-PX carries proxy votes.                                  |
| You want the fund adviser's own filing | [`form-adv-firms`](./form-adv-firms.md) | Form ADV covers the adviser as a firm.                     |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                                      |
| --------- | ------- | -------- | ------------------------ | ----------------------------------------------------------- |
| `query`   | string  | Yes      | Lucene syntax            | For example `entities.cik:1639553`.                          |
| `from`    | integer | No       | 0 to 10,000              | Offset of the first result. Default 0.                       |
| `size`    | integer | No       | 1 to 50                  | Default 50.                                                  |
| `sort`    | array   | No       | Elasticsearch sort array | Default `[{"filedAt": {"order": "desc"}}]`.                  |

Query fields: `entities.cik`, which returned 8 filings for `1639553`,
`registrantInfo.registrantState`, which returned 5,891 filings for `NY`,
`accessionNo`, `formType`, `filedAt`, `periodOfReport`, `entities.fileNo`,
`registrantInfo.investmentCompFileNo`, `registrantInfo.registrantFullName`,
`registrantInfo.familyInvCompFullName`,
`registrantInfo.publicAccountants.publicAccountantName`,
`registrantInfo.directors.isDirectorInterestedPerson`, the `registrantInfo.is*`
flags, and `managementInvestmentQuestionSeriesInfo.monthlyAvgNetAssets`. See
[query language](../query-language.md).

Unlike [`form-nport`](./form-nport.md) and [`form-npx`](./form-npx.md), this
tool does not reject a `query` that has no colon. Use `field:value` syntax.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`.
A request for `entities.cik:1639553` returned `{value: 8, relation: "eq"}`, an
exact count. A `relation` of `gte` means the count is capped at 10,000 and the
true count is higher.

Paths below sit inside one item of `data[]`. A filing carries only the sections
that fit its registrant type. A field is absent when the filer leaves that item
of the form blank.

### Envelope

| Field            | Type   | Meaning                                                      |
| ---------------- | ------ | ------------------------------------------------------------ |
| `total.value`    | number | Count of filings that match the query. Counting stops at 10,000. |
| `total.relation` | string | `eq` when `value` is exact, `gte` when the true count is at or above `value` |
| `data[]`         | array  | One element per N-CEN filing                                 |

### Filing identity

| Field            | Type   | Meaning                                                      |
| ---------------- | ------ | ------------------------------------------------------------ |
| `id`             | string | Internal ID of the parsed filing record                      |
| `accessionNo`    | string | EDGAR accession number of the filing, for example `0001639553-26-000002` |
| `fileNo`         | string | Investment Company Act file number of the filing, for example `811-23054` |
| `formType`       | string | Form type of the filing. `N-CEN` is the annual census report. |
| `filedAt`        | string | Time when EDGAR accepted the filing, with the offset         |
| `periodOfReport` | string | Period the filing covers, `YYYY-MM-DD`                       |

### `entities[]`

The EDGAR header entities of the filing.

| Field                             | Type   | Meaning                                                      |
| --------------------------------- | ------ | ------------------------------------------------------------ |
| `entities[].cik`                  | string | Central Index Key of the entity                              |
| `entities[].ticker`               | string | Ticker symbol of the entity                                  |
| `entities[].companyName`          | string | Legal name of the entity as filed. The role follows the name, for example `(Filer)`. |
| `entities[].irsNo`                | string | IRS employer identification number of the entity             |
| `entities[].fiscalYearEnd`        | string | Fiscal year end as month and day, `MMDD`, for example `1231` |
| `entities[].stateOfIncorporation` | string | State or country where the entity is incorporated            |
| `entities[].sic`                  | string | Standard Industrial Classification code of the entity        |
| `entities[].act`                  | string | Act under which the entity reports. `40` is the Investment Company Act of 1940. |
| `entities[].fileNo`               | string | File number that EDGAR uses to track the filings of the entity |
| `entities[].filmNo`               | string | Film number that the SEC gives the filing                    |

### `seriesClass`

The series and share classes that the report covers.

| Field                                                                      | Type            | Meaning                                                      |
| -------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `seriesClass.reportSeriesClass.rptIncludeAllSeriesFlag`                    | boolean         | True when the report covers every series of the registrant   |
| `seriesClass.reportSeriesClass.rptSeriesClassInfo[].seriesId`              | string          | Series ID of one series in the report                        |
| `seriesClass.reportSeriesClass.rptSeriesClassInfo[].includeAllClassesFlag` | boolean         | True when the report covers every class of that series       |
| `seriesClass.reportSeriesClass.rptSeriesClassInfo[].classIds[]`            | array of string | Class IDs of that series in the report                       |
| `seriesClass.reportClass[].rptIncludeAllClassesFlag`                       | boolean         | True when the report covers every class                      |
| `seriesClass.reportClass[].classIds[]`                                     | array of string | Share class IDs that the report covers                       |

### `generalInfo`

| Field                            | Type    | Meaning                                                      |
| -------------------------------- | ------- | ------------------------------------------------------------ |
| `generalInfo.reportEndingPeriod` | string  | Last day of the period that the report covers                |
| `generalInfo.isReportPeriodLt12` | boolean | True when the period is shorter than 12 months               |

### `registrantInfo`: identity and address

| Field                                  | Type            | Meaning                                                      |
| -------------------------------------- | --------------- | ------------------------------------------------------------ |
| `registrantInfo.registrantFullName`    | string          | Full legal name of the registrant that files the report      |
| `registrantInfo.investmentCompFileNo`  | string          | Investment Company Act file number of the registrant         |
| `registrantInfo.registrantCik`         | string          | Central Index Key of the registrant                          |
| `registrantInfo.registrantLei`         | string          | Legal Entity Identifier of the registrant. Twenty zeros when the registrant has none. |
| `registrantInfo.registrantStreet1`     | string          | Street line 1 of the registrant address                      |
| `registrantInfo.registrantStreet2`     | string          | Street line 2 of the registrant address                      |
| `registrantInfo.registrantCity`        | string          | City of the registrant address                               |
| `registrantInfo.registrantZipCode`     | string          | Postal code of the registrant address                        |
| `registrantInfo.registrantState`       | string          | State of the registrant address, as a two-letter code        |
| `registrantInfo.registrantCountry`     | string          | Country of the registrant address                            |
| `registrantInfo.registrantPhoneNumber` | string          | Phone number of the registrant                               |
| `registrantInfo.websites[]`            | array of string | Web addresses of the registrant                              |

### `registrantInfo`: status and classification

| Field                                               | Type    | Meaning                                                      |
| --------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `registrantInfo.isRegistrantFirstFiling`            | boolean | True when this is the first N-CEN filing of the registrant   |
| `registrantInfo.isRegistrantLastFiling`             | boolean | True when this is the last N-CEN filing of the registrant    |
| `registrantInfo.familyInvCompFullName`              | string  | Name of the family of investment companies. Use it to group registrants. |
| `registrantInfo.isRegistrantFamilyInvComp`          | boolean | True when the registrant belongs to a family of investment companies |
| `registrantInfo.registrantClassificationType`       | string  | Registration form of the registrant. One of `N-1A`, `N-2`, `N-3`, `N-4`, `N-5`, `N-6` or `S-6`. |
| `registrantInfo.totalSeries`                        | number  | Count of series that the registrant reports                  |
| `registrantInfo.isSecuritiesActRegistration`        | boolean | True when the registrant also registers under the Securities Act |
| `registrantInfo.terminatedSeries[].seriesName`      | string  | Name of a series that ended in the period                    |
| `registrantInfo.terminatedSeries[].seriesId`        | string  | Series ID of that series                                     |
| `registrantInfo.terminatedSeries[].terminationDate` | string  | Date the series ended, `YYYY-MM-DD`                          |

### `registrantInfo`: books, records and people

| Field                                                                     | Type            | Meaning                                                      |
| ------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `registrantInfo.locationBooksRecords[].officeName`                        | string          | Name of the office that keeps the books and records          |
| `registrantInfo.locationBooksRecords[].officeAddress1`                    | string          | Street line 1 of that office                                 |
| `registrantInfo.locationBooksRecords[].officeAddress2`                    | string          | Street line 2 of that office                                 |
| `registrantInfo.locationBooksRecords[].officeCity`                        | string          | City of that office                                          |
| `registrantInfo.locationBooksRecords[].officeState`                       | string          | State of that office                                         |
| `registrantInfo.locationBooksRecords[].officeCountry`                     | string          | Country of that office                                       |
| `registrantInfo.locationBooksRecords[].officeRecordsZipCode`              | string          | Postal code of that office                                   |
| `registrantInfo.locationBooksRecords[].officePhone`                       | string          | Phone number of that office                                  |
| `registrantInfo.locationBooksRecords[].booksRecordsDesc`                  | string          | Text that names the books and records kept at that office    |
| `registrantInfo.directors[].directorName`                                 | string          | Full name of a director of the registrant                    |
| `registrantInfo.directors[].crdNumber`                                    | string          | CRD number of the director                                   |
| `registrantInfo.directors[].isDirectorInterestedPerson`                   | boolean         | True when the director is an interested person of the registrant. An interested person has a business or family tie to the fund. |
| `registrantInfo.directors[].fileNos[]`                                    | array of string | File numbers of the funds that the director serves           |
| `registrantInfo.chiefComplianceOfficers[].ccoName`                        | string          | Full name of the chief compliance officer                    |
| `registrantInfo.chiefComplianceOfficers[].crdNumber`                      | string          | CRD number of the chief compliance officer                   |
| `registrantInfo.chiefComplianceOfficers[].ccoStreet1`                     | string          | Street line 1 of the officer address                         |
| `registrantInfo.chiefComplianceOfficers[].ccoStreet2`                     | string          | Street line 2 of the officer address                         |
| `registrantInfo.chiefComplianceOfficers[].ccoCity`                        | string          | City of the officer address                                  |
| `registrantInfo.chiefComplianceOfficers[].ccoState`                       | string          | State of the officer address                                 |
| `registrantInfo.chiefComplianceOfficers[].ccoCountry`                     | string          | Country of the officer address                               |
| `registrantInfo.chiefComplianceOfficers[].ccoZipCode`                     | string          | Postal code of the officer address                           |
| `registrantInfo.chiefComplianceOfficers[].ccoPhone`                       | string          | Phone number of the officer. The value is `XXXXXX` in the example response. |
| `registrantInfo.chiefComplianceOfficers[].isCcoChangedSinceLastFiling`    | boolean         | True when the officer changed since the last filing          |
| `registrantInfo.chiefComplianceOfficers[].ccoEmployers[].ccoEmployerName` | string          | Name of another employer of the officer. `N/A` when there is none. |
| `registrantInfo.chiefComplianceOfficers[].ccoEmployers[].ccoEmployerId`   | string          | Identifier of that employer                                  |

### `registrantInfo`: matters, proceedings and support

| Field                                                      | Type            | Meaning                                                      |
| ---------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `registrantInfo.isRegistrantSubmittedMatter`               | boolean         | True when the registrant put a matter to a vote of security holders |
| `registrantInfo.securityMatterSeriesInfo[].seriesName`     | string          | Name of a series that the matter covers                      |
| `registrantInfo.securityMatterSeriesInfo[].seriesId`       | string          | Series ID of that series                                     |
| `registrantInfo.isPreviousLegalProceeding`                 | boolean         | True when a legal proceeding ran against the registrant      |
| `registrantInfo.legalProceedingSeriesInfo[].seriesName`    | string          | Name of a series in that proceeding                          |
| `registrantInfo.legalProceedingSeriesInfo[].seriesId`      | string          | Series ID of that series                                     |
| `registrantInfo.isPreviousProceedingTerminated`            | boolean         | True when such a proceeding ended in the period              |
| `registrantInfo.previousProceedingTerminated[].seriesName` | string          | Name of a series in the proceeding that ended                |
| `registrantInfo.previousProceedingTerminated[].seriesId`   | string          | Series ID of that series                                     |
| `registrantInfo.isClaimFiled`                              | boolean         | True when someone filed a claim against the registrant       |
| `registrantInfo.totalClaimAmount`                          | number          | Total amount of that claim                                   |
| `registrantInfo.isCoveredByInsurancePolicy`                | boolean         | True when an insurance policy covers the claim               |
| `registrantInfo.isClaimFiledDuringPeriod`                  | boolean         | True when the claim came in during the period                |
| `registrantInfo.isFinancialSupportDuringPeriod`            | boolean         | True when an affiliate gave the registrant financial support in the period |
| `registrantInfo.financialSupportSeriesInfo[].seriesName`   | string          | Name of a series that got the support                        |
| `registrantInfo.financialSupportSeriesInfo[].seriesId`     | string          | Series ID of that series                                     |
| `registrantInfo.isExemptionFromAct`                        | boolean         | True when the registrant relied on an exemption from the Investment Company Act |
| `registrantInfo.releaseNumbers[]`                          | array of string | Release numbers of the exemptions the registrant relied on   |

### `registrantInfo`: underwriters and accountants

| Field                                                                                   | Type    | Meaning                                                      |
| --------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `registrantInfo.principalUnderwriters[].principalUnderwriterName`                       | string  | Name of the principal underwriter                            |
| `registrantInfo.principalUnderwriters[].principalUnderwriterFileNo`                     | string  | SEC file number of the underwriter                           |
| `registrantInfo.principalUnderwriters[].principalUnderwriterCrdNumber`                  | string  | CRD number of the underwriter                                |
| `registrantInfo.principalUnderwriters[].principalUnderwriterLei`                        | string  | Legal Entity Identifier of the underwriter                   |
| `registrantInfo.principalUnderwriters[].principalUnderWriterState`                      | string  | State where the underwriter is registered                    |
| `registrantInfo.principalUnderwriters[].principalUnderWriterCountry`                    | string  | Country where the underwriter is registered                  |
| `registrantInfo.principalUnderwriters[].isPrincipalUnderwriterAffiliatedWithRegistrant` | boolean | True when the underwriter is affiliated with the registrant  |
| `registrantInfo.isUnderwriterHiredOrTerminated`                                         | boolean | True when the registrant hired or terminated an underwriter in the period |
| `registrantInfo.publicAccountants[].publicAccountantName`                               | string  | Name of the independent public accountant                    |
| `registrantInfo.publicAccountants[].pcaobNumber`                                        | string  | PCAOB registration number of the accountant                  |
| `registrantInfo.publicAccountants[].publicAccountantLei`                                | string  | Legal Entity Identifier of the accountant                    |
| `registrantInfo.publicAccountants[].publicAccountantState`                              | string  | State where the accountant is registered                     |
| `registrantInfo.publicAccountants[].publicAccountantCountry`                            | string  | Country where the accountant is registered                   |
| `registrantInfo.isPublicAccountantChanged`                                              | boolean | True when the registrant changed accountant in the period    |

### `registrantInfo`: accounting, valuation and payments

| Field                                                                                | Type    | Meaning                                                      |
| ------------------------------------------------------------------------------------ | ------- | ------------------------------------------------------------ |
| `registrantInfo.isMaterialWeakness`                                                  | boolean | True when the accountant found a material weakness           |
| `registrantInfo.isOpinionOffered`                                                    | boolean | True when the accountant gave an audit opinion               |
| `registrantInfo.auditOpinionSeries[].seriesName`                                     | string  | Name of a series that the opinion covers                     |
| `registrantInfo.auditOpinionSeries[].seriesId`                                       | string  | Series ID of that series                                     |
| `registrantInfo.isMaterialChange`                                                    | boolean | True when a valuation method changed in a material way       |
| `registrantInfo.valuationMethodsChanges[].dateOfChange`                              | string  | Date of the change to the valuation method                   |
| `registrantInfo.valuationMethodsChanges[].changeExplanation`                         | string  | Text that explains the change                                |
| `registrantInfo.valuationMethodsChanges[].assetType`                                 | string  | Asset type that the change covers, for example equities, derivatives, structured notes or loans |
| `registrantInfo.valuationMethodsChanges[].assetTypeOtherDesc`                        | string  | Text description when the asset type is other                |
| `registrantInfo.valuationMethodsChanges[].investmentType`                            | string  | Investment type inside that asset type                       |
| `registrantInfo.valuationMethodsChanges[].statutoryRegulatoryBasis`                  | string  | Statute or rule that supports the change                     |
| `registrantInfo.valuationMethodsChanges[].valuationMethodsChangeSeries[].seriesName` | string  | Name of a series that the change covers                      |
| `registrantInfo.valuationMethodsChanges[].valuationMethodsChangeSeries[].seriesId`   | string  | Series ID of that series                                     |
| `registrantInfo.isAccountingPrincipleChange`                                         | boolean | True when an accounting principle changed in the period      |
| `registrantInfo.isPaymentErrorInNetAssetValue`                                       | boolean | True when a payment error changed the net asset value        |
| `registrantInfo.paymentErrorSeries[].seriesName`                                     | string  | Name of a series that the error hit                          |
| `registrantInfo.paymentErrorSeries[].seriesId`                                       | string  | Series ID of that series                                     |
| `registrantInfo.isPaymentDividend`                                                   | boolean | True when the registrant paid a dividend in the period       |
| `registrantInfo.paymentDividendSeries[].seriesName`                                  | string  | Name of a series that paid the dividend                      |
| `registrantInfo.paymentDividendSeries[].seriesId`                                    | string  | Series ID of that series                                     |

### `managementInvestmentQuestionSeriesInfo[]`: fund and classes

The management company section. One element per fund series.

| Field                                                                                        | Type            | Meaning                                                      |
| -------------------------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].mgmtInvFundName`                                   | string          | Name of the fund series                                      |
| `managementInvestmentQuestionSeriesInfo[].mgmtInvSeriesId`                                   | string          | Series ID of the fund                                        |
| `managementInvestmentQuestionSeriesInfo[].mgmtInvLei`                                        | string          | Legal Entity Identifier of the fund                          |
| `managementInvestmentQuestionSeriesInfo[].isFirstFilingByFund`                               | boolean         | True when the fund reports for the first time                |
| `managementInvestmentQuestionSeriesInfo[].numAuthorizedClass`                                | number          | Count of share classes that the fund may issue               |
| `managementInvestmentQuestionSeriesInfo[].numAddedClass`                                     | number          | Count of share classes added in the period                   |
| `managementInvestmentQuestionSeriesInfo[].numTerminatedClass`                                | number          | Count of share classes ended in the period                   |
| `managementInvestmentQuestionSeriesInfo[].sharesOutstanding[].sharesOutstandingClassName`    | string          | Name of a share class of the fund                            |
| `managementInvestmentQuestionSeriesInfo[].sharesOutstanding[].sharesOutstandingClassId`      | string          | Class ID of that share class                                 |
| `managementInvestmentQuestionSeriesInfo[].sharesOutstanding[].sharesOutstandingTickerSymbol` | string          | Ticker symbol of that share class                            |
| `managementInvestmentQuestionSeriesInfo[].fundTypes[]`                                       | array of string | Types that fit the fund, for example `Master-Feeder`, `Index`, `ETF`, `Interval` or `Money Market` |

### `managementInvestmentQuestionSeriesInfo[]`: fund type detail

| Field                                                                                                 | Type    | Meaning                                                      |
| ----------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.feederFunds[].feederFundName`          | string  | Name of a feeder fund. A feeder fund puts its assets into a master fund. |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.feederFunds[].regFeederFundFileNo`     | string  | SEC file number of the feeder fund when it is registered     |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.feederFunds[].regFeederFundSeriesIdNo` | string  | Series ID of the registered feeder fund                      |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.feederFunds[].regFeederFundLei`        | string  | Legal Entity Identifier of the registered feeder fund        |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.feederFunds[].unregFeederFileNo`       | string  | File number of the feeder fund when it is not registered     |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.feederFunds[].unregFeederFundLei`      | string  | Legal Entity Identifier of the unregistered feeder fund      |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.masterFunds[].masterFundName`          | string  | Name of the master fund that holds the assets                |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.masterFunds[].masterFundFileNo`        | string  | File number of the master fund                               |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.masterFunds[].masterFundSECFileNo`     | string  | SEC file number of the master fund                           |
| `managementInvestmentQuestionSeriesInfo[].masterFeederFundInfo.masterFunds[].masterFundLei`           | string  | Legal Entity Identifier of the master fund                   |
| `managementInvestmentQuestionSeriesInfo[].indexFundInfo.isIndexFundAffiliated`                        | boolean | True when the index is affiliated with the fund              |
| `managementInvestmentQuestionSeriesInfo[].indexFundInfo.isIndexFundExclusive`                         | boolean | True when the fund holds exclusive rights to use the index   |
| `managementInvestmentQuestionSeriesInfo[].indexFundInfo.indexFundReturnDiffBeforeExpense`             | number  | Difference between the fund return and the index return, before expenses |
| `managementInvestmentQuestionSeriesInfo[].indexFundInfo.indexFundReturnDiffAfterExpense`              | number  | The same difference after expenses                           |
| `managementInvestmentQuestionSeriesInfo[].indexFundInfo.indexFundReturnDailyStdevBeforeExpense`       | number  | Standard deviation of the daily return difference, before expenses |
| `managementInvestmentQuestionSeriesInfo[].indexFundInfo.indexFundReturnDailyStdevAfterExpense`        | number  | Standard deviation of the daily return difference, after expenses |
| `managementInvestmentQuestionSeriesInfo[].isNonDiversifiedCompany`                                    | boolean | True when the fund is non-diversified. A non-diversified fund may hold more of its assets in fewer issuers. |
| `managementInvestmentQuestionSeriesInfo[].isForeignSubsidiary`                                        | boolean | True when the fund invests through a foreign subsidiary      |
| `managementInvestmentQuestionSeriesInfo[].foreignInvestments[].foreignSubsidiaryName`                 | string  | Name of the foreign subsidiary                               |
| `managementInvestmentQuestionSeriesInfo[].foreignInvestments[].foreignSubsidiaryLei`                  | string  | Legal Entity Identifier of the foreign subsidiary            |

### `managementInvestmentQuestionSeriesInfo[]`: securities lending

| Field                                                                                                                           | Type            | Meaning                                                      |
| ------------------------------------------------------------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].isFundSecuritiesLending`                                                              | boolean         | True when the fund may lend portfolio securities             |
| `managementInvestmentQuestionSeriesInfo[].didFundLendSecurities`                                                                | boolean         | True when the fund lent securities in the period             |
| `managementInvestmentQuestionSeriesInfo[].fundLendSecurities.isFundLiquidated`                                                  | boolean         | True when the lending arrangement was liquidated             |
| `managementInvestmentQuestionSeriesInfo[].fundLendSecurities.isFundAdverselyImpacted`                                           | boolean         | True when the lending arrangement harmed the fund            |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].securitiesAgentName`                                               | string          | Name of the securities lending agent. The agent places the loans for the fund. |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].securitiesAgentLei`                                                | string          | Legal Entity Identifier of the agent                         |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].isSecuritiesAgentAffiliated`                                       | boolean         | True when the agent is affiliated with the fund              |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].isSecurityAgentIndemnity`                                          | boolean         | True when the agent gives the fund an indemnity against loss |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].securityAgentIndemnity.didIndemnificationRights`                   | boolean         | True when the arrangement gives the fund indemnification rights |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].securityAgentIndemnity.indemnityProviders[].indemnityProviderName` | string          | Name of the party that provides the indemnity                |
| `managementInvestmentQuestionSeriesInfo[].securityLendings[].securityAgentIndemnity.indemnityProviders[].indemnityProviderLei`  | string          | Legal Entity Identifier of that party                        |
| `managementInvestmentQuestionSeriesInfo[].collateralManagers[].collateralManagerName`                                           | string          | Name of the collateral manager. The manager invests the collateral that borrowers post. |
| `managementInvestmentQuestionSeriesInfo[].collateralManagers[].collateralManagerLei`                                            | string          | Legal Entity Identifier of the collateral manager            |
| `managementInvestmentQuestionSeriesInfo[].collateralManagers[].isCollateralManagerAffiliated`                                   | boolean         | True when the collateral manager is an affiliated person     |
| `managementInvestmentQuestionSeriesInfo[].collateralManagers[].isCollateralManagerAffiliatedWithFund`                           | boolean         | True when the collateral manager is affiliated with the fund itself |
| `managementInvestmentQuestionSeriesInfo[].paymentToAgentManagerTypes[]`                                                         | array of string | Kinds of payment that went to the lending agent and the collateral manager, for example revenue sharing or an administrative fee |
| `managementInvestmentQuestionSeriesInfo[].paymentToAgentManagerOtherFeeDesc`                                                    | string          | Text description when the payment kind is other              |
| `managementInvestmentQuestionSeriesInfo[].avgPortfolioSecuritiesValue`                                                          | number          | Average value of the portfolio securities of the fund in the period |
| `managementInvestmentQuestionSeriesInfo[].netIncomeSecuritiesLending`                                                           | number          | Net income that securities lending earned for the fund       |

### `managementInvestmentQuestionSeriesInfo[]`: rules, fees and waivers

| Field                                                                 | Type            | Meaning                                                      |
| --------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].relyOnRules[]`              | array of string | Rules under the Investment Company Act that the fund relied on |
| `managementInvestmentQuestionSeriesInfo[].isExpenseLimitationInPlace` | boolean         | True when an expense limit applies to the fund               |
| `managementInvestmentQuestionSeriesInfo[].isExpenseReducedOrWaived`   | boolean         | True when the fund had expenses reduced or waived            |
| `managementInvestmentQuestionSeriesInfo[].isFeesWaivedRecoupable`     | boolean         | True when the adviser may recoup the waived fees later       |
| `managementInvestmentQuestionSeriesInfo[].isExpenseWaivedRecoupable`  | boolean         | True when the adviser may recoup the waived expenses later   |

### `managementInvestmentQuestionSeriesInfo[]`: advisers

| Field                                                                                                        | Type    | Meaning                                                      |
| ------------------------------------------------------------------------------------------------------------ | ------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserName`                        | string  | Name of the investment adviser of the fund                   |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserFileNo`                      | string  | SEC file number of the adviser                               |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserCrdNo`                       | string  | CRD number of the adviser                                    |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserLei`                         | string  | Legal Entity Identifier of the adviser                       |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserState`                       | string  | State where the adviser is registered                        |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserCountry`                     | string  | Country where the adviser is registered                      |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].investmentAdviserStartDate`                   | string  | Date the fund engaged the adviser                            |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisers[].isInvestmentAdviserHired`                     | boolean | True when the fund engages the adviser                       |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investmentAdviserTerminatedName`    | string  | Name of an adviser that the fund terminated                  |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investAdviserTerminatedFileNo`      | string  | SEC file number of that adviser                              |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investAdviserTerminatedCrdNo`       | string  | CRD number of that adviser                                   |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investAdviserTerminatedLei`         | string  | Legal Entity Identifier of that adviser                      |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investmentAdviserTerminatedState`   | string  | State of that adviser                                        |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investmentAdviserTerminatedCountry` | string  | Country of that adviser                                      |
| `managementInvestmentQuestionSeriesInfo[].investmentAdvisersTerminated[].investAdviserTerminationDate`       | string  | Date the engagement ended                                    |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserName`                                      | string  | Name of a sub-adviser. A sub-adviser manages part of the fund for the adviser. |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserFileNo`                                    | string  | SEC file number of the sub-adviser                           |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserCrdNo`                                     | string  | CRD number of the sub-adviser                                |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserLei`                                       | string  | Legal Entity Identifier of the sub-adviser                   |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].isSubAdviserAffiliated`                              | boolean | True when the sub-adviser is affiliated with the adviser     |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserState`                                     | string  | State where the sub-adviser is registered                    |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserCountry`                                   | string  | Country where the sub-adviser is registered                  |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].subAdviserStartDate`                                 | string  | Date the fund engaged the sub-adviser                        |
| `managementInvestmentQuestionSeriesInfo[].subAdvisers[].isSubAdviserHired`                                   | boolean | True when the fund engages the sub-adviser                   |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminatedName`                  | string  | Name of a sub-adviser that the fund terminated               |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminatedFileNo`                | string  | SEC file number of that sub-adviser                          |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminatedCrdNo`                 | string  | CRD number of that sub-adviser                               |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminatedLei`                   | string  | Legal Entity Identifier of that sub-adviser                  |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminatedState`                 | string  | State of that sub-adviser                                    |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminatedCountry`               | string  | Country of that sub-adviser                                  |
| `managementInvestmentQuestionSeriesInfo[].subAdvisersTerminated[].subAdviserTerminationDate`                 | string  | Date the engagement ended                                    |

### `managementInvestmentQuestionSeriesInfo[]`: other service providers

| Field                                                                                                       | Type    | Meaning                                                      |
| ----------------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].transferAgentName`                               | string  | Name of the transfer agent. The agent keeps the shareholder register. |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].transferAgentFileNo`                             | string  | SEC file number of the transfer agent                        |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].transferAgentLei`                                | string  | Legal Entity Identifier of the transfer agent                |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].transferAgentState`                              | string  | State where the transfer agent is registered                 |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].transferAgentCountry`                            | string  | Country where the transfer agent is registered               |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].isTransferAgentAffiliated`                       | boolean | True when the transfer agent is affiliated with the fund     |
| `managementInvestmentQuestionSeriesInfo[].transferAgents[].isTransferAgentSubAgent`                         | boolean | True when the entity acts as a sub-transfer agent            |
| `managementInvestmentQuestionSeriesInfo[].isTransferAgentHiredOrTerminated`                                 | boolean | True when the fund hired or terminated a transfer agent in the period |
| `managementInvestmentQuestionSeriesInfo[].pricingServices[].pricingServiceName`                             | string  | Name of the pricing service. The service supplies prices for the portfolio. |
| `managementInvestmentQuestionSeriesInfo[].pricingServices[].pricingServiceLei`                              | string  | Legal Entity Identifier of the pricing service               |
| `managementInvestmentQuestionSeriesInfo[].pricingServices[].pricingServiceIdNumberDesc`                     | string  | Other identifier of the pricing service, with a description  |
| `managementInvestmentQuestionSeriesInfo[].pricingServices[].pricingServiceState`                            | string  | State where the pricing service is registered                |
| `managementInvestmentQuestionSeriesInfo[].pricingServices[].pricingServiceCountry`                          | string  | Country where the pricing service is registered              |
| `managementInvestmentQuestionSeriesInfo[].pricingServices[].isPricingServiceAffiliated`                     | boolean | True when the pricing service is affiliated with the fund    |
| `managementInvestmentQuestionSeriesInfo[].isPricingServiceHiredOrTerminated`                                | boolean | True when the fund hired or terminated a pricing service in the period |
| `managementInvestmentQuestionSeriesInfo[].custodians[].custodianName`                                       | string  | Name of the custodian. The custodian holds the fund assets.  |
| `managementInvestmentQuestionSeriesInfo[].custodians[].custodianLei`                                        | string  | Legal Entity Identifier of the custodian                     |
| `managementInvestmentQuestionSeriesInfo[].custodians[].custodianState`                                      | string  | State where the custodian is registered                      |
| `managementInvestmentQuestionSeriesInfo[].custodians[].custodianCountry`                                    | string  | Country where the custodian is registered                    |
| `managementInvestmentQuestionSeriesInfo[].custodians[].isCustodianAffiliated`                               | boolean | True when the custodian is affiliated with the fund          |
| `managementInvestmentQuestionSeriesInfo[].custodians[].isSubCustodian`                                      | boolean | True when the entity acts as a sub-custodian                 |
| `managementInvestmentQuestionSeriesInfo[].custodians[].custodyType`                                         | string  | Kind of custody, for example a bank, a securities depository or a foreign custodian |
| `managementInvestmentQuestionSeriesInfo[].custodians[].otherCustodianDesc`                                  | string  | Text description when the kind of custody is other           |
| `managementInvestmentQuestionSeriesInfo[].isCustodianHiredOrTerminated`                                     | boolean | True when the fund hired or terminated a custodian in the period |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].shareholderServiceAgentName`         | string  | Name of the shareholder servicing agent. The agent answers shareholder requests. |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].shareholderServiceAgentLei`          | string  | Legal Entity Identifier of that agent                        |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].shareholderServiceIdNumberDesc`      | string  | Other identifier of that agent, with a description           |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].shareholderServiceAgentState`        | string  | State where that agent is registered                         |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].shareholderServiceAgentCountry`      | string  | Country where that agent is registered                       |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].isShareholderServiceAgentAffiliated` | boolean | True when that agent is affiliated with the fund             |
| `managementInvestmentQuestionSeriesInfo[].shareholderServicingAgents[].isShareholderServiceAgentSubShare`   | boolean | True when the entity acts as a sub-agent                     |
| `managementInvestmentQuestionSeriesInfo[].isShareholderServiceHiredTerminated`                              | boolean | True when the fund hired or terminated such an agent in the period |
| `managementInvestmentQuestionSeriesInfo[].admins[].adminName`                                               | string  | Name of the administrator. The administrator runs the day to day operations of the fund. |
| `managementInvestmentQuestionSeriesInfo[].admins[].adminLei`                                                | string  | Legal Entity Identifier of the administrator                 |
| `managementInvestmentQuestionSeriesInfo[].admins[].idNumberDesc`                                            | string  | Other identifier of the administrator, with a description    |
| `managementInvestmentQuestionSeriesInfo[].admins[].adminState`                                              | string  | State where the administrator is registered                  |
| `managementInvestmentQuestionSeriesInfo[].admins[].adminCountry`                                            | string  | Country where the administrator is registered                |
| `managementInvestmentQuestionSeriesInfo[].admins[].isAdminAffiliated`                                       | boolean | True when the administrator is affiliated with the fund      |
| `managementInvestmentQuestionSeriesInfo[].admins[].isAdminSubAdmin`                                         | boolean | True when the entity acts as a sub-administrator             |
| `managementInvestmentQuestionSeriesInfo[].isAdminHiredOrTerminated`                                         | boolean | True when the fund hired or terminated an administrator in the period |

### `managementInvestmentQuestionSeriesInfo[]`: brokers and dealers

| Field                                                                                         | Type    | Meaning                                                      |
| --------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerName`                   | string  | Name of a broker-dealer that the fund used                   |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerFileNo`                 | string  | SEC file number of the broker-dealer                         |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerCrdNo`                  | string  | CRD number of the broker-dealer                              |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerLei`                    | string  | Legal Entity Identifier of the broker-dealer                 |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerState`                  | string  | State where the broker-dealer is registered                  |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerCountry`                | string  | Country where the broker-dealer is registered                |
| `managementInvestmentQuestionSeriesInfo[].brokerDealers[].brokerDealerCommission`             | number  | Commission that the broker-dealer got from the fund          |
| `managementInvestmentQuestionSeriesInfo[].brokers[].brokerName`                               | string  | Name of a broker that the fund used                          |
| `managementInvestmentQuestionSeriesInfo[].brokers[].brokerFileNo`                             | string  | SEC file number of the broker                                |
| `managementInvestmentQuestionSeriesInfo[].brokers[].brokerCrdNo`                              | string  | CRD number of the broker                                     |
| `managementInvestmentQuestionSeriesInfo[].brokers[].brokerLei`                                | string  | Legal Entity Identifier of the broker                        |
| `managementInvestmentQuestionSeriesInfo[].brokers[].brokerState`                              | string  | State of the broker                                          |
| `managementInvestmentQuestionSeriesInfo[].brokers[].brokerCountry`                            | string  | Country of the broker                                        |
| `managementInvestmentQuestionSeriesInfo[].brokers[].grossCommission`                          | number  | Gross commission that the broker got                         |
| `managementInvestmentQuestionSeriesInfo[].aggregateCommission`                                | number  | Total commission that all brokers got from the fund          |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalName`              | string  | Name of a party in a principal transaction. A principal trades from its own book. |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalFileNo`            | string  | SEC file number of that party                                |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalCrdNo`             | string  | CRD number of that party                                     |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalLei`               | string  | Legal Entity Identifier of that party                        |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalState`             | string  | State of that party                                          |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalCountry`           | string  | Country of that party                                        |
| `managementInvestmentQuestionSeriesInfo[].principalTransactions[].principalTotalPurchaseSale` | number  | Total value of the purchases and sales with that party       |
| `managementInvestmentQuestionSeriesInfo[].principalAggregatePurchase`                         | number  | Total value of the principal transactions across all parties |
| `managementInvestmentQuestionSeriesInfo[].isBrokerageResearchPayment`                         | boolean | True when the fund paid for research with brokerage commissions |

### `managementInvestmentQuestionSeriesInfo[]`: assets, credit and pricing

| Field                                                                                                    | Type            | Meaning                                                      |
| -------------------------------------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `managementInvestmentQuestionSeriesInfo[].monthlyAvgNetAssets`                                           | number          | Monthly average net assets of the fund. Also a query field.  |
| `managementInvestmentQuestionSeriesInfo[].dailyAvgNetAssets`                                             | number          | Daily average net assets of the fund                         |
| `managementInvestmentQuestionSeriesInfo[].hasLineOfCredit`                                               | boolean         | True when the fund had a line of credit in the period        |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].isCreditLineCommitted`                          | string          | `Committed` or `Uncommitted`. A committed line binds the lender to lend. |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].lineOfCreditSize`                               | number          | Size of the line of credit                                   |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].lineOfCreditInstitutions[]`                     | array of string | Names of the institutions that give the line                 |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].soleCreditType`                                 | string          | `Sole` when only this fund uses the line, `Shared` when other funds share it |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].sharedCreditType.creditType`                    | string          | Type of the credit, `Sole` or `Shared`                       |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].sharedCreditType.creditUser[].fundName`         | string          | Name of another fund that draws on the shared line           |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].sharedCreditType.creditUser[].secFileNo`        | string          | SEC file number of that fund                                 |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].isCreditLineUsed`                               | boolean         | True when the fund drew on the line                          |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].creditLineUsed.averageCreditLineUsed`           | number          | Average amount that the fund drew                            |
| `managementInvestmentQuestionSeriesInfo[].lineOfCredit[].creditLineUsed.daysCreditUsed`                  | number          | Count of days that the draw stayed outstanding               |
| `managementInvestmentQuestionSeriesInfo[].isInterFundLending`                                            | boolean         | True when the fund lent to other funds in the group          |
| `managementInvestmentQuestionSeriesInfo[].interFundLendingDetails[].interFundLendingLoanAverage`         | number          | Average size of those loans                                  |
| `managementInvestmentQuestionSeriesInfo[].interFundLendingDetails[].interFundLendingDaysOutstanding`     | number          | Average count of days that those loans stayed outstanding    |
| `managementInvestmentQuestionSeriesInfo[].isInterFundBorrowing`                                          | boolean         | True when the fund borrowed from other funds in the group    |
| `managementInvestmentQuestionSeriesInfo[].interFundBorrowingDetails[].interFundBorrowingLoanAverage`     | number          | Average size of those loans                                  |
| `managementInvestmentQuestionSeriesInfo[].interFundBorrowingDetails[].interFundBorrowingDaysOutstanding` | number          | Average count of days that those loans stayed outstanding    |
| `managementInvestmentQuestionSeriesInfo[].isSwingPricing`                                                | boolean         | True when the fund used swing pricing. Swing pricing moves the share price to put trading costs on the investors who trade. |
| `managementInvestmentQuestionSeriesInfo[].swingPriceUpperLimit`                                          | number          | Upper limit that the fund set for the swing pricing adjustment |

### `closedEndManagementInvestment`: securities and offerings

The closed-end fund section.

| Field                                                                                          | Type            | Meaning                                                      |
| ---------------------------------------------------------------------------------------------- | --------------- | ------------------------------------------------------------ |
| `closedEndManagementInvestment.securityRelatedItems[].description`                             | string          | Kind of security that the fund issued, for example common stock, preferred stock, warrants or bonds |
| `closedEndManagementInvestment.securityRelatedItems[].securityClassTitle`                      | string          | Title of that class of security                              |
| `closedEndManagementInvestment.securityRelatedItems[].commonStocks[].commonStockExchange`      | string          | Exchange where the common stock is listed                    |
| `closedEndManagementInvestment.securityRelatedItems[].commonStocks[].commonStockTickerSymbol`  | string          | Ticker symbol of the common stock                            |
| `closedEndManagementInvestment.securityRelatedItems[].otherSecurityDescription`                | string          | Text description when the security is not common stock       |
| `closedEndManagementInvestment.isRightsOffering`                                               | boolean         | True when the fund made a rights offering. A rights offering lets current holders buy new shares. |
| `closedEndManagementInvestment.rightsOfferingFunds[].rightsOfferingTypes.rightsOfferingType[]` | array of string | Types of the rights offering                                 |
| `closedEndManagementInvestment.rightsOfferingFunds[].rightsOfferingTypes.rightsOfferingDesc`   | string          | Text description of the offering type                        |
| `closedEndManagementInvestment.rightsOfferingFunds[].rightsOfferingParticipationPercent`       | number          | Percentage of holders that took part in the offering         |
| `closedEndManagementInvestment.isSecondaryOffering`                                            | boolean         | True when the fund made a secondary offering                 |
| `closedEndManagementInvestment.secondaryOfferings.secondaryOfferingType[]`                     | array of string | Types of the secondary offering                              |
| `closedEndManagementInvestment.secondaryOfferings.otherSecondaryOfferingDesc`                  | string          | Text description when the type is other                      |
| `closedEndManagementInvestment.isRepurchaseSecurity`                                           | boolean         | True when the fund repurchased its own securities            |
| `closedEndManagementInvestment.repurchaseSecurities.repurchaseSecurityType[]`                  | array of string | Types of the securities that the fund repurchased            |
| `closedEndManagementInvestment.repurchaseSecurities.otherRepurchaseSecurityDesc`               | string          | Text description when the type is other                      |
| `closedEndManagementInvestment.isSecuritiesModified`                                           | boolean         | True when the fund changed the rights of a class of its securities |

### `closedEndManagementInvestment`: defaults, dividends and value

| Field                                                                               | Type    | Meaning                                                      |
| ----------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `closedEndManagementInvestment.isDefaultLongTermDebt`                               | boolean | True when the fund defaulted on long-term debt               |
| `closedEndManagementInvestment.longTermDebtDefaults[].defaultNature`                | string  | Text that states the nature of the default                   |
| `closedEndManagementInvestment.longTermDebtDefaults[].defaultDate`                  | string  | Date of the default                                          |
| `closedEndManagementInvestment.longTermDebtDefaults[].defaultPerThousandAmount`     | number  | Amount in default for each 1,000 units of the debt           |
| `closedEndManagementInvestment.longTermDebtDefaults[].defaultTotalAmount`           | number  | Total amount in default                                      |
| `closedEndManagementInvestment.isDividendsInArrears`                                | boolean | True when dividends on a class of the fund securities are in arrears |
| `closedEndManagementInvestment.dividendsInArrears[].dividendIssueTitle`             | string  | Title of the issue with dividends in arrears                 |
| `closedEndManagementInvestment.dividendsInArrears[].dividendAmountPerShareInArrear` | number  | Amount in arrears for each share                             |
| `closedEndManagementInvestment.managementFee`                                       | number  | Management fee that the fund paid                            |
| `closedEndManagementInvestment.netOperatingExpenses`                                | number  | Net operating expenses of the fund                           |
| `closedEndManagementInvestment.marketPricePerShare`                                 | number  | Market price of one share                                    |
| `closedEndManagementInvestment.netAssetValuePerShare`                               | number  | Net asset value of one share                                 |

### `closedEndManagementInvestment`: service providers of small companies

| Field                                                                                                | Type    | Meaning                                                      |
| ---------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserName`                 | string  | Name of the investment adviser                               |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserFileNo`               | string  | SEC file number of the adviser                               |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserCrdNo`                | string  | CRD number of the adviser                                    |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserLei`                  | string  | Legal Entity Identifier of the adviser                       |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserState`                | string  | State of the adviser                                         |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserCountry`              | string  | Country of the adviser                                       |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].smallInvestmentAdviserStartDate`            | string  | Date the engagement began                                    |
| `closedEndManagementInvestment.smallInvestmentAdvisers[].isSmallInvestmentAdviserHired`              | boolean | True when the company engages the adviser                    |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedName`    | string  | Name of an adviser that the company terminated               |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedFileNo`  | string  | SEC file number of that adviser                              |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedCrdNo`   | string  | CRD number of that adviser                                   |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedLei`     | string  | Legal Entity Identifier of that adviser                      |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedState`   | string  | State of that adviser                                        |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedCountry` | string  | Country of that adviser                                      |
| `closedEndManagementInvestment.terminatedSmallInvAdvisers[].smallInvestmentAdviserTerminatedDate`    | string  | Date the engagement ended                                    |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserName`                               | string  | Name of the sub-adviser                                      |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserFileNo`                             | string  | SEC file number of the sub-adviser                           |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserCrdNo`                              | string  | CRD number of the sub-adviser                                |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserLei`                                | string  | Legal Entity Identifier of the sub-adviser                   |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserState`                              | string  | State of the sub-adviser                                     |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserCountry`                            | string  | Country of the sub-adviser                                   |
| `closedEndManagementInvestment.smallSubAdvisers[].isSmallSubAdviserAffiliated`                       | boolean | True when the sub-adviser is affiliated with the adviser     |
| `closedEndManagementInvestment.smallSubAdvisers[].smallSubAdviserStartDate`                          | string  | Date the engagement began                                    |
| `closedEndManagementInvestment.smallSubAdvisers[].isSmallSubAdviserHired`                            | boolean | True when the company engages the sub-adviser                |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedName`           | string  | Name of a sub-adviser that the company terminated            |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedFileNo`         | string  | SEC file number of that sub-adviser                          |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedCrdNo`          | string  | CRD number of that sub-adviser                               |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedLei`            | string  | Legal Entity Identifier of that sub-adviser                  |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedState`          | string  | State of that sub-adviser                                    |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedCountry`        | string  | Country of that sub-adviser                                  |
| `closedEndManagementInvestment.terminatedSmallSubAdvisers[].smallSubAdviserTerminatedDate`           | string  | Date the engagement ended                                    |
| `closedEndManagementInvestment.smallTransferAgents[].smallTransferAgentName`                         | string  | Name of the transfer agent                                   |
| `closedEndManagementInvestment.smallTransferAgents[].smallTransferAgentFileNo`                       | string  | SEC file number of the transfer agent                        |
| `closedEndManagementInvestment.smallTransferAgents[].smallTransferAgentLei`                          | string  | Legal Entity Identifier of the transfer agent                |
| `closedEndManagementInvestment.smallTransferAgents[].smallTransferAgentState`                        | string  | State of the transfer agent                                  |
| `closedEndManagementInvestment.smallTransferAgents[].smallTransferAgentCountry`                      | string  | Country of the transfer agent                                |
| `closedEndManagementInvestment.smallTransferAgents[].isSmallTransferAgentAffiliated`                 | boolean | True when the transfer agent is affiliated with the company  |
| `closedEndManagementInvestment.smallTransferAgents[].isSmallTransferAgentSubAdmin`                   | boolean | True when the entity acts as a sub-administrator             |
| `closedEndManagementInvestment.isSmallTransferAgentHiredOrTerminated`                                | boolean | True when the company hired or terminated a transfer agent in the period |
| `closedEndManagementInvestment.smallCustodians[].smallCustodianName`                                 | string  | Name of the custodian                                        |
| `closedEndManagementInvestment.smallCustodians[].smallCustodianLei`                                  | string  | Legal Entity Identifier of the custodian                     |
| `closedEndManagementInvestment.smallCustodians[].smallCustodianState`                                | string  | State of the custodian                                       |
| `closedEndManagementInvestment.smallCustodians[].smallCustodianCountry`                              | string  | Country of the custodian                                     |
| `closedEndManagementInvestment.smallCustodians[].isSmallCustodianAffiliated`                         | boolean | True when the custodian is affiliated with the company       |
| `closedEndManagementInvestment.smallCustodians[].isSmallCustodianSub`                                | boolean | True when the entity acts as a sub-custodian                 |
| `closedEndManagementInvestment.smallCustodians[].custodyType`                                        | string  | Kind of custody that the custodian gives                     |
| `closedEndManagementInvestment.smallCustodians[].otherSmallCustodianDesc`                            | string  | Text description when the kind of custody is other           |
| `closedEndManagementInvestment.isSmallCustodianHiredTerminated`                                      | boolean | True when the company hired or terminated a custodian in the period |

### `exchangeSeriesInfo[]`

The exchange-traded fund section. One element per listed series.

| Field                                                                              | Type    | Meaning                                                      |
| ---------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `exchangeSeriesInfo[].fundName`                                                    | string  | Name of the listed fund series                               |
| `exchangeSeriesInfo[].etfSeriesId`                                                 | string  | Series ID of the exchange-traded fund                        |
| `exchangeSeriesInfo[].securityExchanges[].fundExchange`                            | string  | Exchange where the fund shares trade, as a four-letter ISO 10383 market code |
| `exchangeSeriesInfo[].securityExchanges[].fundsTickerSymbol`                       | string  | Ticker symbol of the fund on that exchange                   |
| `exchangeSeriesInfo[].authorizedParticipants[].authorizedParticipantName`          | string  | Name of an authorized participant. A participant creates and redeems fund shares. |
| `exchangeSeriesInfo[].authorizedParticipants[].authorizedParticipantFileNo`        | string  | SEC file number of the participant                           |
| `exchangeSeriesInfo[].authorizedParticipants[].authorizedParticipantCrdNo`         | string  | CRD number of the participant                                |
| `exchangeSeriesInfo[].authorizedParticipants[].authorizedParticipantLei`           | string  | Legal Entity Identifier of the participant                   |
| `exchangeSeriesInfo[].authorizedParticipants[].authorizedParticipantPurchaseValue` | number  | Value that the participant bought in the period              |
| `exchangeSeriesInfo[].authorizedParticipants[].authorizedParticipantRedeemValue`   | number  | Value that the participant redeemed in the period            |
| `exchangeSeriesInfo[].isCollateralRequired`                                        | boolean | True when the fund makes a participant post collateral       |
| `exchangeSeriesInfo[].creationUnitNumOfShares`                                     | number  | Count of fund shares in one creation unit                    |
| `exchangeSeriesInfo[].creationUnitRedeemedNumOfShares`                             | number  | Count of fund shares redeemed                                |
| `exchangeSeriesInfo[].averagePercentagePurchased`                                  | number  | Average percentage of the creation units purchased           |
| `exchangeSeriesInfo[].standardDeviationPurchased`                                  | number  | Standard deviation of that purchase percentage               |
| `exchangeSeriesInfo[].creationUnitPurchasedInKind`                                 | number  | Percentage of the creation units purchased in kind, not in cash |
| `exchangeSeriesInfo[].creationUnitPurchasedSDInKind`                               | number  | Standard deviation of that in-kind purchase percentage       |
| `exchangeSeriesInfo[].averagePercentageRedeemed`                                   | number  | Average percentage of the creation units redeemed            |
| `exchangeSeriesInfo[].percentSDRedeemed`                                           | number  | Standard deviation of that redemption percentage             |
| `exchangeSeriesInfo[].creationUnitPercentageRedeemedInKind`                        | number  | Percentage of the creation units redeemed in kind            |
| `exchangeSeriesInfo[].creationUnitSDRedeemedInKind`                                | number  | Standard deviation of that in-kind redemption percentage     |
| `exchangeSeriesInfo[].creationUnitTransactionFeePerUnit`                           | number  | Transaction fee for one creation unit                        |
| `exchangeSeriesInfo[].creationUnitTransactionFeeManyUnits`                         | number  | Transaction fee for several creation units in one order      |
| `exchangeSeriesInfo[].creationUnitTransactionFeePercentagePerUnit`                 | number  | Transaction fee for one creation unit, as a percentage       |
| `exchangeSeriesInfo[].creationUnitTransactionFeeCashPerUnit`                       | number  | Cash transaction fee for one creation unit                   |
| `exchangeSeriesInfo[].creationUnitTransactionFeeCashManyUnits`                     | number  | Cash transaction fee for several creation units in one order |
| `exchangeSeriesInfo[].creationUnitTransactionFeeCashPercentagePerUnit`             | number  | Cash transaction fee for one creation unit, as a percentage  |
| `exchangeSeriesInfo[].purchaseCreationUnitDollarPerUnit`                           | number  | Amount that a participant pays for one creation unit         |
| `exchangeSeriesInfo[].purchaseCreationUnitDollarPerMoreUnits`                      | number  | Amount for each further creation unit in the same order      |
| `exchangeSeriesInfo[].purchaseCreationUnitDollarPercentagePerUnit`                 | number  | Purchase amount for one creation unit, as a percentage       |
| `exchangeSeriesInfo[].purchaseCreationUnitCashPerUnit`                             | number  | Cash amount for one creation unit                            |

### `unitInvestmentTrust`

The unit investment trust section. An insurance separate account uses it.

| Field                                                                                         | Type    | Meaning                                                      |
| --------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------ |
| `unitInvestmentTrust.depositors[].depositorName`                                              | string  | Name of the depositor. The depositor creates the trust and puts the assets in it. |
| `unitInvestmentTrust.depositors[].depositorCrdNo`                                             | string  | CRD number of the depositor. `N/A` when it has none.         |
| `unitInvestmentTrust.depositors[].depositorLei`                                               | string  | Legal Entity Identifier of the depositor                     |
| `unitInvestmentTrust.depositors[].depositorState`                                             | string  | State of the depositor                                       |
| `unitInvestmentTrust.depositors[].depositorCountry`                                           | string  | Country of the depositor                                     |
| `unitInvestmentTrust.depositors[].depositorUltimateParentName`                                | string  | Name of the ultimate parent company of the depositor         |
| `unitInvestmentTrust.uitAdmins[].uitAdminName`                                                | string  | Name of the administrator of the trust                       |
| `unitInvestmentTrust.uitAdmins[].uitAdminLei`                                                 | string  | Legal Entity Identifier of the administrator                 |
| `unitInvestmentTrust.uitAdmins[].uitAdminState`                                               | string  | State of the administrator                                   |
| `unitInvestmentTrust.uitAdmins[].uitAdminCountry`                                             | string  | Country of the administrator                                 |
| `unitInvestmentTrust.uitAdmins[].isUitAdminAffiliated`                                        | boolean | True when the administrator is affiliated with the trust     |
| `unitInvestmentTrust.uitAdmins[].isUitAdminSubAdmin`                                          | boolean | True when the entity acts as a sub-administrator             |
| `unitInvestmentTrust.isUitAdminHiredTerminated`                                               | boolean | True when the trust hired or terminated an administrator in the period |
| `unitInvestmentTrust.registrantSeparateInsuranceAccount.isRegistrantSeparateInsuranceAccount` | boolean | True when the registrant is a separate account of an insurance company |
| `unitInvestmentTrust.registrantSeparateInsuranceAccount.separateAccountSeriesId`              | string  | Series ID of that separate account                           |
| `unitInvestmentTrust.numOfContracts`                                                          | number  | Count of contracts that the separate account offers          |
| `unitInvestmentTrust.contractSecurities[].separateAccountSecurityName`                        | string  | Name of a contract that the separate account offers          |
| `unitInvestmentTrust.contractSecurities[].separateAccountContractId`                          | string  | Class ID of that contract                                    |
| `unitInvestmentTrust.contractSecurities[].separateAccountTotalAsset`                          | number  | Total assets held under that contract                        |
| `unitInvestmentTrust.contractSecurities[].numContractsSold`                                   | number  | Count of those contracts sold in the period                  |
| `unitInvestmentTrust.contractSecurities[].grossPremiumReceived`                               | number  | Gross premiums received for that contract                    |
| `unitInvestmentTrust.contractSecurities[].grossPremiumReceivedSection1035`                    | number  | Part of those premiums that came from a section 1035 exchange. A section 1035 exchange swaps one contract for another with no tax. |
| `unitInvestmentTrust.contractSecurities[].numContractsAffected`                               | number  | Count of contracts in those exchanges                        |
| `unitInvestmentTrust.contractSecurities[].contractValueRedeemed`                              | number  | Contract value redeemed in the period                        |
| `unitInvestmentTrust.contractSecurities[].contractValueRedeemedSection1035`                   | number  | Part of that redeemed value that went into a section 1035 exchange |
| `unitInvestmentTrust.contractSecurities[].numContractsAffectedRedeemed`                       | number  | Count of contracts in those redemptions                      |
| `unitInvestmentTrust.isRule6C7Reliance`                                                       | boolean | True when the trust relied on rule 6c-7. That rule covers variable annuity contracts sold to Texas Optional Retirement Program participants. |
| `unitInvestmentTrust.isRule11A2Reliance`                                                      | boolean | True when the trust relied on rule 11a-2. That rule covers exchange offers by insurance company separate accounts. |
| `unitInvestmentTrust.isRule12D1Dash4Reliance`                                                 | boolean | True when the trust relied on rule 12d1-4. That rule lets a fund invest in other funds above the section 12(d)(1) limits. |
| `unitInvestmentTrust.isRule12D1GReliance`                                                     | boolean | True when the trust relied on rule 12d1-G. That rule covers a fund that invests in other funds under section 12(d)(1)(G). |

### `attachmentsTab`

Flags for the documents that the filer attached.

| Field                                        | Type    | Meaning                                                      |
| -------------------------------------------- | ------- | ------------------------------------------------------------ |
| `attachmentsTab.isLegalProceedings`          | boolean | True when the filing attaches material on a legal proceeding |
| `attachmentsTab.isProvisionFinancialSupport` | boolean | True when the filing attaches material on financial support to the fund |
| `attachmentsTab.isIPAReportInternalControl`  | boolean | True when the filing attaches the accountant report on internal control |
| `attachmentsTab.isChangeAccPrinciples`       | boolean | True when the filing attaches material on a change of accounting principle |
| `attachmentsTab.isInfoRequiredEO`            | boolean | True when the filing attaches information that an exemptive order requires |
| `attachmentsTab.isOtherInfoRequired`         | boolean | True when the filing attaches other required information     |

### `signature`

| Field                            | Type   | Meaning                                                      |
| -------------------------------- | ------ | ------------------------------------------------------------ |
| `signature.registrantSignedName` | string | Name of the registrant on the signature block                |
| `signature.signedDate`           | string | Date of the signature, `YYYY-MM-DD`                          |
| `signature.signature`            | string | Signature text of the officer who signed                     |
| `signature.title`                | string | Title of the officer who signed                              |

The record is wide but not heavy. One filing was 4.8 KB. `size`
counts filings. The JSON arrives as one stringified text block. See
[response format](../response-format.md).

## Example

Prompt: "Get the latest N-CEN filing for CIK 1639553."

```json
{ "name": "form-ncen", "arguments": { "query": "entities.cik:1639553", "size": 1 } }
```

```json
{
  "total": { "value": 8, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001639553-26-000002",
      "fileNo": "811-23054",
      "formType": "N-CEN",
      "filedAt": "2026-03-16T17:18:30-04:00",
      "periodOfReport": "2025-12-31",
      "entities": [
        {
          "cik": "1639553",
          "companyName": "Variable Annuity-8 Series Account (of Empower Life & Annuity Insurance Co of New York) (Filer)"
        }
      ],
      "generalInfo": { "reportEndingPeriod": "2025-12-31", "isReportPeriodLt12": false },
      "registrantInfo": {
        "registrantFullName": "Variable Annuity-8 Series Account (of Empower Life & Annuity Insurance Co of New York)",
        "registrantState": "NY",
        "familyInvCompFullName": "Empower Funds, Inc.",
        "registrantClassificationType": "N-4"
      }
    }
  ]
}
```

Keys were removed to fit. The values are unchanged.

## Limits and errors

- `size` above 50 fails with HTTP 400 and `Maximum 'size' limit of 50 exceeded`.
- Omitting `size` returns 50 records, not 10.
- Sections vary by registrant type. A management company filing does not carry
  `unitInvestmentTrust`. Read the keys you get, do not assume them.
- The `ccoPhone` value was masked as `XXXXXX` in the example response.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-nport`](./form-nport.md), [`form-npx`](./form-npx.md),
  [`form-npx-file`](./form-npx-file.md)
- REST documentation:
  [Form N-CEN API](https://sec-api.io/docs/form-ncen-api-annual-reports-investment-companies)
