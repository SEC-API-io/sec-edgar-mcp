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
series in the report, and the signature block.

**The search result holds no votes.** The registry description says the tool
returns "issuer voted on, meeting date, ballot item, fund's vote". The capture
returns none of that. Vote records come from
[`form-npx-file`](./form-npx-file.md), which takes an accession number. Treat
`form-npx` as the index, and `form-npx-file` as the content.
`proxyVotingRecordsAttached` tells you whether that follow-up call will find
records. It was `true` in the capture.

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
| `from`    | integer | No       | Minimum 0                                       | Offset of the first result. Default 0.               |
| `size`    | integer | No       | 1 to 50                                         | Default 50. Ask for less if you only need the newest.|
| `sort`    | array   | No       | Elasticsearch sort array                        | Default `[{"filedAt": {"order": "desc"}}]`.          |

The query length limit is 1,000 characters here. It is 2,000 on
[`form-nport`](./form-nport.md).

Query field confirmed to return rows: `cik`.

Fields taken from the response shape, all **unverified**: `companyName`,
`formType`, `periodOfReport`, `accessionNo`, `proxyVotingRecordsAttached`,
`formData.coverPage.reportCalendarYear`. See
[query language](../query-language.md).

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`. A
`relation` of `eq` means the count is exact. The capture returned
`{value: 2, relation: "eq"}` for CIK 2110.

| Field                                       | Type    | Meaning                                                             |
| ------------------------------------------- | ------- | ------------------------------------------------------------------- |
| `accessionNo`                               | string  | Accession number. Pass this to `form-npx-file`. `id` is the internal record ID. |
| `formType`, `filedAt`                       | string  | `N-PX`, and the filing timestamp with offset.                        |
| `periodOfReport`                            | string  | End of the report period, `YYYY-MM-DD`.                              |
| `cik`                                       | string  | Filer CIK, no zero padding.                                          |
| `ticker`                                    | string  | Ticker. Empty string in the capture. Most N-PX filers have none.     |
| `companyName`                               | string  | Filer name.                                                          |
| `proxyVotingRecordsAttached`                | boolean | `true` when vote records exist for this accession number.            |
| `headerData.filerInfo.registrantType`       | string  | `RMIC` in the capture. The SDK example shows `IM`.                   |
| `headerData.filerInfo.investmentCompanyType`| string  | Registration form, `N-1A` in the capture.                            |
| `formData.coverPage.yearOrQuarter`          | string  | `YEAR` in the capture.                                               |
| `formData.coverPage.reportCalendarYear`     | string  | Calendar year covered, for example `2025`.                           |
| `formData.coverPage.reportingPerson`        | object  | Name, phone and address. `agentForService` has the same shape.       |
| `formData.coverPage.reportInfo.reportType`  | string  | `FUND VOTING REPORT` in the capture. The SDK example shows `INSTITUTIONAL MANAGER VOTING REPORT`. |
| `formData.coverPage.fileNumber`             | string  | File number, for example `811-01829`. `leiNumber` holds the LEI.     |
| `formData.summaryPage.otherIncludedManagersCount` | number | Count of other managers included in the report.                |
| `formData.seriesPage.seriesCount`           | number  | Number of fund series in the report. 5 in the capture.               |
| `formData.seriesPage.seriesDetails.seriesReports[]` | array | `idOfSeries`, `nameOfSeries`, `leiOfSeries` per series.        |
| `formData.signaturePage`                    | object  | `txSignature`, `txPrintedSignature`, `txTitle`, `txAsOfDate`.        |

`seriesPage` is absent from the SDK example for an institutional manager filer.
Expect it only on fund filers. The response is small. The capture was 2 KB for
one record. The JSON arrives as one stringified text block. See
[response format](../response-format.md).

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
