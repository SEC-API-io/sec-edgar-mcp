# form-npx

Search Form N-PX proxy-voting filings and get one metadata record per filing.

|                 |                                          |
| --------------- | ---------------------------------------- |
| Category        | Funds                                    |
| Required input  | `query`                                  |
| Returns         | `{total, data[]}`                        |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`          |
| REST equivalent | `POST /form-npx`                         |

## What it does

The tool searches Form N-PX filings. Funds and 13F managers file N-PX once a
year and report how they voted the shares they hold. One item in `data[]` is one
filing. It gives the filer, the report period, the cover page, the list of fund
series in the report, and the signature block. The data starts on
1 January 2024. Filings before that date are not in the index.

**The search result holds no votes.** The registry description says the tool
returns "issuer voted on, meeting date, ballot item, fund's vote". The response
holds none of that. Vote records come from
[`form-npx-file`](./form-npx-file.md), which takes an accession number. Treat
`form-npx` as the index, and `form-npx-file` as the content.
`proxyVotingRecordsAttached` tells you whether that follow-up call will find
records. It is `true` in the example below.

## When to use it

- Which N-PX filings did this fund company file, and for which years?
- What is the accession number of the fund's latest proxy-voting report?
- Which fund series are covered by one N-PX report?
- Is this an institutional manager report or a fund voting report?

## When to use a different tool

| Situation                                  | Better tool                                   | Why                                                              |
| ------------------------------------------ | --------------------------------------------- | ---------------------------------------------------------------- |
| You want the actual votes                  | [`form-npx-file`](./form-npx-file.md)         | Only that tool returns `proxyVotingRecords[]`.                    |
| You want what the fund holds               | [`form-nport`](./form-nport.md)               | N-PORT reports positions. N-PX reports votes.                     |
| You want the fund's service providers      | [`form-ncen`](./form-ncen.md)                 | N-CEN is the annual census report.                                |

## Input

| Parameter | Type    | Required | Constraints                                     | Notes                                                |
| --------- | ------- | -------- | ----------------------------------------------- | ---------------------------------------------------- |
| `query`   | string  | Yes      | Must contain a colon. Maximum 1,000 characters. | Lucene syntax, for example `cik:2110`.               |
| `from`    | integer | No       | 0 to 10,000                                     | Offset of the first result. Default 0.               |
| `size`    | integer | No       | 1 to 50                                         | Default 50. Ask for less if you only need the newest.|
| `sort`    | array   | No       | Elasticsearch sort array                        | Default `[{"filedAt": {"order": "desc"}}]`.          |

The query length limit is 1,000 characters here. It is 2,000 on
[`form-nport`](./form-nport.md).

Every response field is a query field. Common ones are `cik`, `companyName`,
`formType`, `filedAt`, `periodOfReport`, `accessionNo`,
`proxyVotingRecordsAttached`, `headerData.filerInfo.registrantType`,
`formData.coverPage.reportCalendarYear`,
`formData.coverPage.reportInfo.reportType`,
`formData.coverPage.reportInfo.confidentialTreatment` and
`formData.seriesPage.seriesDetails.seriesReports.nameOfSeries`. See
[query language](../query-language.md).

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`. A
`relation` of `eq` means the count is exact. A request for CIK 2110 returned
`{value: 2, relation: "eq"}`.

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | --------------------------------------------------------------- |
| `total.value`    | number | Number of N-PX filings that match the query.                     |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many.    |
| `data[]`         | array  | The matching filings. One item per filing. No vote records.      |

### Filing

| Field                          | Type    | Meaning                                                                                   |
| ------------------------------ | ------- | ------------------------------------------------------------------------------------------- |
| `data[].id`                    | string  | System-internal identifier of the filing record.                                            |
| `data[].accessionNo`           | string  | Accession number of the filing. Pass this to `form-npx-file`.                               |
| `data[].formType`              | string  | `N-PX`, or `N-PX/A` for an amendment.                                                        |
| `data[].filedAt`               | string  | Time EDGAR accepted the filing, ISO 8601 with an offset.                                     |
| `data[].periodOfReport`        | string  | End of the report period, `YYYY-MM-DD`. Usually 30 June of the filing year.                 |
| `data[].cik`                   | string  | CIK of the filer, no zero padding.                                                           |
| `data[].ticker`                | string  | Ticker of the filer, when the filer is publicly traded. Empty string in the example response. |
| `data[].companyName`           | string  | Name of the filer of this proxy voting report.                                               |
| `data[].proxyVotingRecordsAttached` | boolean | `true` when vote records exist for this accession number.                              |
| `data[].headerData`            | object  | The EDGAR submission header of the filing.                                                   |
| `data[].formData`              | object  | The form itself: cover page, summary page, series page and signature page.                   |

### `data[].headerData`

| Field                                                                       | Type    | Meaning                                                                                   |
| --------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------- |
| `headerData.submissionType`                                                 | string  | Submission type of the filing, `N-PX` or `N-PX/A`.                                          |
| `headerData.filerInfo.registrantType`                                       | string  | Filer class. `RMIC` is a registered management investment company. `IM` is an institutional manager. |
| `headerData.filerInfo.filer.issuerCredentials.cik`                          | string  | CIK of the filer, zero padded to 10 digits.                                                 |
| `headerData.filerInfo.flags.overrideInternetFlag`                           | boolean | Whether the filer overrides the default internet submission rules.                          |
| `headerData.filerInfo.flags.confirmingCopyFlag`                             | boolean | Whether this submission is an extra copy of a filing already sent.                          |
| `headerData.filerInfo.investmentCompanyType`                                | string  | SEC registration form of the investment company. `N-1A` is an open-end fund, `N-2` a closed-end fund. `N-3`, `N-4`, `N-6`, `S-1 or S-3` and `S-6` also appear. |
| `headerData.filerInfo.periodOfReport`                                       | string  | End of the report period, `MM/DD/YYYY`. Same date as `periodOfReport`.                      |
| `headerData.seriesClass.reportSeriesClass.rptIncludeAllSeriesFlag`          | boolean | Whether the report covers every series of the investment company.                           |
| `headerData.seriesClass.reportSeriesClass.rptSeriesClassInfo[]`             | array   | The series and classes the report covers, when it covers a subset. Up to 1,000 items.       |
| `headerData.seriesClass.reportSeriesClass.rptSeriesClassInfo[].seriesId`    | string  | Identifier of the series.                                                                   |
| `headerData.seriesClass.reportSeriesClass.rptSeriesClassInfo[].includeAllClassesFlag` | boolean | Whether every class of that series is in the report.                              |
| `headerData.seriesClass.reportSeriesClass.rptSeriesClassInfo[].classInfo[]` | array   | The classes of that series. Up to 1,000 items.                                              |
| `headerData.seriesClass.reportSeriesClass.rptSeriesClassInfo[].classInfo[].classId` | string | Identifier of the class.                                                             |
| `headerData.seriesClass.reportClass[]`                                      | array   | Classes the report covers on their own, outside a series structure.                         |
| `headerData.seriesClass.reportClass[].rptIncludeAllClassesFlag`             | boolean | Whether the report covers every class of the fund.                                          |
| `headerData.seriesClass.reportClass[].classInfo[]`                          | array   | The single classes in the report.                                                           |
| `headerData.seriesClass.reportClass[].classInfo[].classId`                  | string  | Identifier of the class.                                                                    |

### `data[].formData.coverPage`

| Field                                                        | Type    | Meaning                                                                                    |
| ------------------------------------------------------------ | ------- | -------------------------------------------------------------------------------------------- |
| `coverPage.yearOrQuarter`                                    | string  | Whether the report covers a year or a quarter. `YEAR` or `QUARTER`. `YEAR` in the example response. |
| `coverPage.reportCalendarYear`                               | string  | Calendar year the report covers, for example `2025`.                                         |
| `coverPage.reportQuarterYear`                                | string  | Quarter the report covers, on a quarterly report.                                            |
| `coverPage.amendmentInfo.isAmendment`                        | boolean | Whether this filing amends an earlier report.                                                |
| `coverPage.amendmentInfo.amendmentNo`                        | number  | Sequence number of the amendment.                                                            |
| `coverPage.amendmentInfo.amendmentType`                      | string  | `RESTATEMENT` or `NEW PROXY`.                                                                |
| `coverPage.amendmentInfo.confDeniedExpired`                  | boolean | Whether a request for confidential treatment was denied, or has expired.                     |
| `coverPage.amendmentInfo.dateExpiredDenied`                  | string  | Date that request was denied or expired, `MM/DD/YYYY`.                                       |
| `coverPage.amendmentInfo.dateReported`                       | string  | Date the proxy voting information was reported, `MM/DD/YYYY`.                                |
| `coverPage.amendmentInfo.reasonForNonConfidentiality`        | string  | Why the data is no longer confidential. `Denied` or `Confidential Treatment Expired`.        |
| `coverPage.reportingPerson`                                  | object  | The entity responsible for the filing: name, phone and address.                              |
| `coverPage.reportingPerson.name`                             | string  | Name of the entity responsible for the filing.                                               |
| `coverPage.reportingPerson.phoneNumber`                      | string  | Contact phone number of that entity.                                                         |
| `coverPage.reportingPerson.address.street1`                  | string  | First street line of that entity.                                                            |
| `coverPage.reportingPerson.address.street2`                  | string  | Second street line of that entity.                                                           |
| `coverPage.reportingPerson.address.city`                     | string  | City of that entity.                                                                         |
| `coverPage.reportingPerson.address.stateOrCountry`           | string  | US state or country of that entity.                                                          |
| `coverPage.reportingPerson.address.zipCode`                  | string  | Postal code of that entity.                                                                  |
| `coverPage.agentForService`                                  | object  | The agent for service of process: name and address.                                          |
| `coverPage.agentForService.name`                             | string  | Full name of the agent for service of process.                                               |
| `coverPage.agentForService.address.street1`                  | string  | First street line of the agent.                                                              |
| `coverPage.agentForService.address.street2`                  | string  | Second street line of the agent.                                                             |
| `coverPage.agentForService.address.city`                     | string  | City of the agent.                                                                           |
| `coverPage.agentForService.address.state`                    | string  | State of the agent.                                                                          |
| `coverPage.agentForService.address.country`                  | string  | Country of the agent.                                                                        |
| `coverPage.agentForService.address.stateOrCountry`           | string  | US state or country of the agent.                                                            |
| `coverPage.agentForService.address.zipCode`                  | string  | Postal code of the agent.                                                                    |
| `coverPage.reportInfo.reportType`                            | string  | Type of report. `FUND VOTING REPORT`, `FUND NOTICE REPORT`, `INSTITUTIONAL MANAGER VOTING REPORT`, `INSTITUTIONAL MANAGER NOTICE REPORT` or `INSTITUTIONAL MANAGER COMBINATION REPORT`. A notice report says the filer held no securities to vote, or cast no votes. |
| `coverPage.reportInfo.noticeExplanation`                     | string  | Why the filer sent a notice report. `ALL VOTES BY OTHER PERSONS`, `REPORTING PERSON DID NOT EXERCISE VOTING` or `REPORTING PERSON HAS POLICY TO NOT VOTE`. |
| `coverPage.reportInfo.confidentialTreatment`                 | boolean | Whether the SEC granted confidential treatment to part of the filing.                        |
| `coverPage.fileNumber`                                       | string  | SEC file number of the registrant or institutional manager, for example `811-01829`.         |
| `coverPage.reportingCrdNumber`                               | string  | Central Registration Depository number of the reporting entity.                              |
| `coverPage.reportingSecFileNumber`                           | string  | SEC file number of the reporting entity.                                                     |
| `coverPage.leiNumber`                                        | string  | Legal Entity Identifier of the filer.                                                        |
| `coverPage.otherManagersInfo.otherManager[]`                 | array   | Other managers whose proxy votes this report covers. Up to 999 items.                        |
| `coverPage.otherManagersInfo.otherManager[].icaOr13FFileNumber` | string | Investment Company Act or Form 13F file number of that manager.                            |
| `coverPage.otherManagersInfo.otherManager[].crdNumber`       | string  | Central Registration Depository number of that manager.                                      |
| `coverPage.otherManagersInfo.otherManager[].otherFileNumber` | string  | A different SEC file number of that manager.                                                 |
| `coverPage.otherManagersInfo.otherManager[].leiNumberOM`     | string  | Legal Entity Identifier of that manager.                                                     |
| `coverPage.otherManagersInfo.otherManager[].managerName`     | string  | Name of that manager.                                                                        |
| `coverPage.explanatoryInformation.explanatoryChoice`         | boolean | Whether the filer added explanatory information.                                             |
| `coverPage.explanatoryInformation.explanatoryNotes`          | string  | The explanation or comments of the filer.                                                    |

### `data[].formData.summaryPage`

| Field                                                            | Type   | Meaning                                                                        |
| ----------------------------------------------------------------- | ------ | -------------------------------------------------------------------------------- |
| `summaryPage.otherIncludedManagersCount`                         | number | Number of other managers whose votes the report covers.                          |
| `summaryPage.otherManagers2.investmentManagers[]`                | array  | Those managers, one item each. Up to 999 items.                                  |
| `summaryPage.otherManagers2.investmentManagers[].serialNo`       | number | Sequence number of the manager inside this filing.                               |
| `summaryPage.otherManagers2.investmentManagers[].form13FFileNumber` | string | Form 13F file number of the manager.                                          |
| `summaryPage.otherManagers2.investmentManagers[].crdNumber`      | string | Central Registration Depository number of the manager.                           |
| `summaryPage.otherManagers2.investmentManagers[].secFileNumber`  | string | SEC file number of the manager.                                                  |
| `summaryPage.otherManagers2.investmentManagers[].leiNumber`      | string | Legal Entity Identifier of the manager.                                          |
| `summaryPage.otherManagers2.investmentManagers[].name`           | string | Name of the manager.                                                             |

### `data[].formData.seriesPage`

| Field                                                | Type   | Meaning                                                                |
| ----------------------------------------------------- | ------ | ------------------------------------------------------------------------ |
| `seriesPage.seriesCount`                             | number | Number of fund series in the report. 5 in the example response.          |
| `seriesPage.seriesDetails.seriesReports[]`           | array  | The series in the report, one item each. Up to 999 items.                |
| `seriesPage.seriesDetails.seriesReports[].idOfSeries`| string | Identifier of the series, for example `S000009184`.                      |
| `seriesPage.seriesDetails.seriesReports[].nameOfSeries` | string | Official name of the series.                                          |
| `seriesPage.seriesDetails.seriesReports[].leiOfSeries` | string | Legal Entity Identifier of the series.                                 |

### `data[].formData.signaturePage`

| Field                                                    | Type   | Meaning                                                            |
| --------------------------------------------------------- | ------ | -------------------------------------------------------------------- |
| `signaturePage.reportingPerson`                          | string | Name of the person or entity responsible for signing.                |
| `signaturePage.txSignature`                              | string | Signature of the authorised person.                                  |
| `signaturePage.txPrintedSignature`                       | string | Printed name of that person.                                         |
| `signaturePage.txTitle`                                  | string | Title or position of that person.                                    |
| `signaturePage.txAsOfDate`                               | string | Date the person signed the form, `MM/DD/YYYY`.                       |
| `signaturePage.secondaryRecords.secondaryRecord[]`       | array  | Further authorised signatories.                                      |
| `signaturePage.secondaryRecords.secondaryRecord[].txSignature` | string | Signature of the secondary signatory.                          |
| `signaturePage.secondaryRecords.secondaryRecord[].printedSign` | string | Printed name of that signatory.                                |
| `signaturePage.secondaryRecords.secondaryRecord[].txTitle` | string | Title or role of that signatory.                                  |
| `signaturePage.secondaryRecords.secondaryRecord[].txAsOfDate` | string | Date that signatory signed, `MM/DD/YYYY`.                      |

`seriesPage` is absent on an institutional manager filer. Expect it only on fund
filers. The response is small, 2 KB for one record. The JSON arrives as one
stringified text block. See [response format](../response-format.md).

## Example

Prompt: "Find the latest Form N-PX filing for CIK 2110."

```json
{ "name": "form-npx", "arguments": { "query": "cik:2110", "size": 1 } }
```

```json
{
  "total": { "value": 2, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001021408-25-002642",
      "formType": "N-PX",
      "filedAt": "2025-08-21T13:52:49-04:00",
      "periodOfReport": "2025-06-30",
      "cik": "2110",
      "ticker": "",
      "companyName": "COLUMBIA ACORN TRUST",
      "proxyVotingRecordsAttached": true,
      "formData": {
        "coverPage": {
          "yearOrQuarter": "YEAR",
          "reportCalendarYear": "2025",
          "reportInfo": { "reportType": "FUND VOTING REPORT" },
          "fileNumber": "811-01829"
        },
        "seriesPage": { "seriesCount": 5 }
      }
    }
  ]
}
```

Keys were removed to fit. The values are unchanged.

## Limits and errors

- A `query` without a colon fails with HTTP 400 and
  `Invalid Lucene query string`.
- A `query` longer than 1,000 characters fails with
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded`.
- Omitting `size` returns 50 records, not 10.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-npx-file`](./form-npx-file.md) is the required second step for votes.
- [`form-nport`](./form-nport.md), [`form-ncen`](./form-ncen.md)
- REST documentation:
  [Form N-PX Proxy Voting Records API](https://sec-api.io/docs/form-npx-proxy-voting-records-api)
