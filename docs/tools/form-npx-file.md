# form-npx-file

Get every proxy-voting record of one Form N-PX filing, by accession number.

|                 |                                                                     |
| --------------- | ------------------------------------------------------------------- |
| Category        | Funds                                                               |
| Required input  | `accessionNo`                                                       |
| Returns         | One filing object with `proxyVotingRecords[]`. No `total`, no wrapper. |
| Pagination      | **None.** No `from`, `size` or `sort`.                          |
| REST equivalent | `GET /form-npx/{accessionNo}`                                       |

## What it does

The tool returns the complete parsed N-PX filing for one accession number. The
object repeats the metadata that [`form-npx`](./form-npx.md) returns, and adds
`proxyVotingRecords[]`. One record is one ballot item voted at one meeting for
one fund series. The data starts on 1 January 2024.

> **Context warning.** This tool returns every vote record in one filing, in a
> single text block, and there is no way to ask for fewer. One mid-size fund
> trust returned **4,372 vote records, 1.94 MB**. A large fund family returns
> considerably more. A filing can hold up to 500,000 records, and large ones
> pass 200 MB and 420,000 records. `proxyVotingRecordsAttached` on the
> `form-npx` result states whether the filing holds vote records.

The registry description calls the result "raw ... XML / structured data". The
response is parsed JSON, not XML.

## When to use it

- How did this fund vote on a named ballot item?
- How often did the fund vote against management last year?
- Which shareholder proposals did the fund support?
- How many shares were on loan, and so not voted, at each meeting?
- Which of the trust's series voted on this issuer?

## When to use a different tool

| Situation                                     | Better tool                       | Why                                                        |
| --------------------------------------------- | --------------------------------- | ----------------------------------------------------------- |
| You do not have an accession number           | [`form-npx`](./form-npx.md)       | A hit carries the `accessionNo` this tool needs.             |
| You only need the filing metadata             | [`form-npx`](./form-npx.md)       | Same metadata, 2 KB instead of 1.94 MB.                      |
| You want fund positions, not votes            | [`form-nport`](./form-nport.md)   | N-PORT reports holdings.                                     |
| You want the document as filed                | [`get-edgar-file`](./get-edgar-file.md) | This tool returns parsed data.                         |

## Input

| Parameter     | Type   | Required | Constraints                                                       | Notes                                                    |
| ------------- | ------ | -------- | ----------------------------------------------------------------- | -------------------------------------------------------- |
| `accessionNo` | string | Yes      | `XXXXXXXXXX-YY-NNNNNN`, or the same 18 digits with no dashes.     | A `form-npx` hit carries it in its `accessionNo` field.   |

There is no `query` parameter and no Lucene syntax here.

## Output

The envelope is a **bare filing object**. There is no `total` and no `data[]`
array around it. The top-level keys are the same as one `form-npx` hit, plus
`proxyVotingRecords`.

### Filing

| Field                        | Type    | Meaning                                                                                   |
| ---------------------------- | ------- | ------------------------------------------------------------------------------------------- |
| `id`                         | string  | System-internal identifier of the filing record.                                            |
| `accessionNo`                | string  | Accession number of the filing.                                                             |
| `formType`                   | string  | `N-PX`, or `N-PX/A` for an amendment.                                                        |
| `filedAt`                    | string  | Time EDGAR accepted the filing, ISO 8601 with an offset.                                     |
| `periodOfReport`             | string  | End of the report period, `YYYY-MM-DD`. Usually 30 June of the filing year.                 |
| `cik`                        | string  | CIK of the filer, no zero padding.                                                           |
| `ticker`                     | string  | Ticker of the filer, when the filer is publicly traded.                                      |
| `companyName`                | string  | Name of the filer of this proxy voting report.                                               |
| `proxyVotingRecordsAttached` | boolean | `true` when vote records exist for this accession number.                                     |
| `headerData`                 | object  | The EDGAR submission header of the filing.                                                   |
| `formData`                   | object  | The form itself: cover page, summary page, series page and signature page.                   |
| `proxyVotingRecords[]`       | array   | The votes. One entry per ballot item. 4,372 entries in this response. Up to 500,000.         |

### `headerData`

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

### `formData.coverPage`

| Field                                                        | Type    | Meaning                                                                                    |
| ------------------------------------------------------------ | ------- | -------------------------------------------------------------------------------------------- |
| `coverPage.yearOrQuarter`                                    | string  | Whether the report covers a year or a quarter. `YEAR` or `QUARTER`.                          |
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
| `coverPage.reportInfo.reportType`                            | string  | Type of report. `FUND VOTING REPORT`, `FUND NOTICE REPORT`, `INSTITUTIONAL MANAGER VOTING REPORT`, `INSTITUTIONAL MANAGER NOTICE REPORT` or `INSTITUTIONAL MANAGER COMBINATION REPORT`. |
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

### `formData.summaryPage`, `formData.seriesPage` and `formData.signaturePage`

| Field                                                            | Type   | Meaning                                                                        |
| ----------------------------------------------------------------- | ------ | -------------------------------------------------------------------------------- |
| `summaryPage.otherIncludedManagersCount`                         | number | Number of other managers whose votes the report covers.                          |
| `summaryPage.otherManagers2.investmentManagers[]`                | array  | Those managers, one item each. Up to 999 items.                                  |
| `summaryPage.otherManagers2.investmentManagers[].serialNo`       | number | Sequence number of the manager inside this filing. `proxyVotingRecords[].voteManager.otherManagers[].otherManager` points to it. |
| `summaryPage.otherManagers2.investmentManagers[].form13FFileNumber` | string | Form 13F file number of the manager.                                          |
| `summaryPage.otherManagers2.investmentManagers[].crdNumber`      | string | Central Registration Depository number of the manager.                           |
| `summaryPage.otherManagers2.investmentManagers[].secFileNumber`  | string | SEC file number of the manager.                                                  |
| `summaryPage.otherManagers2.investmentManagers[].leiNumber`      | string | Legal Entity Identifier of the manager.                                          |
| `summaryPage.otherManagers2.investmentManagers[].name`           | string | Name of the manager.                                                             |
| `seriesPage.seriesCount`                                         | number | Number of fund series in the report.                                             |
| `seriesPage.seriesDetails.seriesReports[]`                       | array  | The series in the report, one item each. Up to 999 items.                        |
| `seriesPage.seriesDetails.seriesReports[].idOfSeries`            | string | Identifier of the series, for example `S000009184`.                              |
| `seriesPage.seriesDetails.seriesReports[].nameOfSeries`          | string | Official name of the series.                                                     |
| `seriesPage.seriesDetails.seriesReports[].leiOfSeries`           | string | Legal Entity Identifier of the series.                                           |
| `signaturePage.reportingPerson`                                  | string | Name of the person or entity responsible for signing.                            |
| `signaturePage.txSignature`                                      | string | Signature of the authorised person.                                              |
| `signaturePage.txPrintedSignature`                               | string | Printed name of that person.                                                     |
| `signaturePage.txTitle`                                          | string | Title or position of that person.                                                |
| `signaturePage.txAsOfDate`                                       | string | Date the person signed the form, `MM/DD/YYYY`.                                   |
| `signaturePage.secondaryRecords.secondaryRecord[]`               | array  | Further authorised signatories.                                                  |
| `signaturePage.secondaryRecords.secondaryRecord[].txSignature`   | string | Signature of the secondary signatory.                                            |
| `signaturePage.secondaryRecords.secondaryRecord[].printedSign`   | string | Printed name of that signatory.                                                  |
| `signaturePage.secondaryRecords.secondaryRecord[].txTitle`       | string | Title or role of that signatory.                                                 |
| `signaturePage.secondaryRecords.secondaryRecord[].txAsOfDate`    | string | Date that signatory signed, `MM/DD/YYYY`.                                        |

### `proxyVotingRecords[]`

| Field                                        | Type    | Meaning                                                          |
| -------------------------------------------- | ------- | ---------------------------------------------------------------- |
| `proxyVotingRecords[].issuerName`            | string  | Company that issued the security and held the shareholder meeting. |
| `proxyVotingRecords[].cusip`                 | string  | CUSIP of the security voted.                                     |
| `proxyVotingRecords[].isin`                  | string  | ISIN of the security voted. Present on every record in this response. |
| `proxyVotingRecords[].figi`                  | string  | Financial Instrument Global Identifier of the security voted.    |
| `proxyVotingRecords[].meetingDate`           | string  | Meeting date, as the filer typed it. Usually `MM/DD/YYYY`. A few filings use another format. The format differs from `filedAt`. |
| `proxyVotingRecords[].voteDescription`       | string  | Text of the proposal voted on, for example "Reelect Jeff Horing as Director". |
| `proxyVotingRecords[].voteCategories.voteCategory[]` | array | Categories the proposal falls under. Up to 14 items.       |
| `proxyVotingRecords[].voteCategories.voteCategory[].categoryType` | string | SEC vote category. The full set is `DIRECTOR ELECTIONS`, `SECTION 14A SAY-ON-PAY VOTES`, `AUDIT-RELATED`, `INVESTMENT COMPANY MATTERS`, `SHAREHOLDER RIGHTS AND DEFENSES`, `EXTRAORDINARY TRANSACTIONS`, `CAPITAL STRUCTURE`, `COMPENSATION`, `CORPORATE GOVERNANCE`, `ENVIRONMENT OR CLIMATE`, `HUMAN RIGHTS OR HUMAN CAPITAL/WORKFORCE`, `DIVERSITY, EQUITY, AND INCLUSION`, `OTHER SOCIAL ISSUES` and `OTHER`. This response holds `DIRECTOR ELECTIONS` (1,895), `CORPORATE GOVERNANCE` (815), `CAPITAL STRUCTURE` (668), `COMPENSATION` (599), `AUDIT-RELATED` (343), `SECTION 14A SAY-ON-PAY VOTES` (98) and seven more. |
| `proxyVotingRecords[].otherVoteDescription`  | string  | Extra text for a ballot item that fits no set category, or that needs more explanation. Present on 10 of the 4,372 records. |
| `proxyVotingRecords[].voteSource`            | string  | Who put the item on the ballot. `ISSUER` is the company, `SECURITY HOLDER` is a shareholder. `ISSUER` on 4,353 records, `SECURITY HOLDER` on 19. |
| `proxyVotingRecords[].sharesVoted`           | number  | Total shares voted on the item.                                  |
| `proxyVotingRecords[].sharesOnLoan`          | number  | Shares out on loan that the fund did not recall to vote.         |
| `proxyVotingRecords[].vote.voteRecord[]`     | array   | How the shares were voted. Up to 999 items.                      |
| `proxyVotingRecords[].vote.voteRecord[].howVoted` | string | The vote: `FOR`, `AGAINST`, `ABSTAIN` or `WITHHOLD`. `WITHHOLD` is used in director elections. This response holds `FOR` (4,067), `AGAINST` (233), `WITHHOLD` (34), `ABSTAIN` (15) and `ONE YEAR` (13). |
| `proxyVotingRecords[].vote.voteRecord[].sharesVoted` | number | Shares voted in that direction.                            |
| `proxyVotingRecords[].vote.voteRecord[].managementRecommendation` | string | What management asked for. `FOR` (4,078), `AGAINST` (269), `NONE` (15). A record where `howVoted` differs from this value is a vote against management. |
| `proxyVotingRecords[].voteManager.otherManagers[]` | array | Other managers that took part in the vote decision. Up to 25 items. |
| `proxyVotingRecords[].voteManager.otherManagers[].otherManager` | number | Sequence number of one such manager. It matches `serialNo` in `formData.summaryPage.otherManagers2.investmentManagers[]`. |
| `proxyVotingRecords[].voteSeries`            | string  | Series ID that cast the vote, for example `S000009184`. It matches a series ID in `formData.seriesPage.seriesDetails.seriesReports[]`. |
| `proxyVotingRecords[].voteOtherInfo`         | string  | Further remarks the filer added about the vote.                  |

`vote` is missing on 10 of the 4,372 records, so `vote.voteRecord[0]` is absent
on those. Those same 10 carry `otherVoteDescription`. `voteRecord` is an array,
because one ballot item can hold more than one vote row.

**This tool has no pagination at all.** You get the whole filing or nothing. The
call above returned 1.94 MB in under a second. The JSON arrives as one
stringified text block. See [response format](../response-format.md).

## Example

Prompt: "Show me how Columbia Acorn Trust voted in its 2025 N-PX filing."

```json
{ "name": "form-npx-file", "arguments": { "accessionNo": "0001021408-25-002642" } }
```

```json
{
  "accessionNo": "0001021408-25-002642",
  "companyName": "COLUMBIA ACORN TRUST",
  "proxyVotingRecords": [
    {
      "issuerName": "monday.com Ltd.",
      "cusip": "M7S64H106",
      "isin": "IL0011762130",
      "meetingDate": "07/31/2024",
      "voteDescription": "Reelect Jeff Horing as Director",
      "voteCategories": {
        "voteCategory": [{ "categoryType": "DIRECTOR ELECTIONS" }]
      },
      "voteSource": "ISSUER",
      "sharesVoted": 96858,
      "sharesOnLoan": 0,
      "vote": {
        "voteRecord": [
          { "howVoted": "FOR", "sharesVoted": 96858, "managementRecommendation": "FOR" }
        ]
      },
      "voteSeries": "S000009184"
    }
  ]
}
```

This is the first of 4,372 records. Metadata keys were removed to fit. The
values are unchanged.

## Limits and errors

- A malformed accession number fails with HTTP 400 and
  `Invalid accession number. Must be in format "XXXXXXXXXX-YY-NNNNNN"`.
- A filing with no stored N-PX file fails with HTTP 404 and `File not found.`
  `proxyVotingRecordsAttached` on the `form-npx` hit states whether a file
  exists.
- Size is the real limit. 1.94 MB in one response, with no way to reduce it.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-npx`](./form-npx.md) is the search step. Its hits carry the accession number.
- [`form-nport`](./form-nport.md), [`form-ncen`](./form-ncen.md)
- REST documentation:
  [Form N-PX Proxy Voting Records API](https://sec-api.io/docs/form-npx-proxy-voting-records-api)
