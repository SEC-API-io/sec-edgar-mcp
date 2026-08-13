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
one fund series.

> **Context warning.** This tool returns every vote record in one filing, in a
> single text block, and there is no way to ask for fewer. One mid-size fund
> trust returned **4,372 vote records, 1.94 MB**. A large fund family returns
> considerably more. `proxyVotingRecordsAttached` on the `form-npx` result
> states whether the filing holds vote records.

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

| Field                                        | Type    | Meaning                                                          |
| -------------------------------------------- | ------- | ---------------------------------------------------------------- |
| `accessionNo`, `cik`, `companyName`          | string  | Filing and filer identity, as in `form-npx`.                     |
| `periodOfReport`, `filedAt`                  | string  | Report period end and filing timestamp.                          |
| `headerData`, `formData`                     | object  | Cover page, series page and signature page. Same shape as `form-npx`. |
| `proxyVotingRecords[]`                       | array   | One entry per ballot item. 4,372 entries in this response.       |
| `proxyVotingRecords[].issuerName`            | string  | Company voted on.                                                |
| `proxyVotingRecords[].cusip`                 | string  | CUSIP of the security voted.                                     |
| `proxyVotingRecords[].isin`                  | string  | ISIN. Present on every record in this response. Other filings omit it. |
| `proxyVotingRecords[].meetingDate`           | string  | Meeting date, `MM/DD/YYYY`. The format differs from `filedAt`. |
| `proxyVotingRecords[].voteDescription`       | string  | The ballot item text, for example "Reelect Jeff Horing as Director". |
| `proxyVotingRecords[].voteCategories.voteCategory[].categoryType` | string | SEC vote category. This response holds `DIRECTOR ELECTIONS` (1,895), `CORPORATE GOVERNANCE` (815), `CAPITAL STRUCTURE` (668), `COMPENSATION` (599), `AUDIT-RELATED` (343), `SECTION 14A SAY-ON-PAY VOTES` (98) and seven more. |
| `proxyVotingRecords[].voteSource`            | string  | Who put the item on the ballot. `ISSUER` on 4,353 records, `SECURITY HOLDER` on 19. |
| `proxyVotingRecords[].sharesVoted`           | number  | Shares voted on the item.                                        |
| `proxyVotingRecords[].sharesOnLoan`          | number  | Shares out on loan, and therefore not voted.                     |
| `proxyVotingRecords[].vote.voteRecord[].howVoted` | string | The vote. This response holds `FOR` (4,067), `AGAINST` (233), `WITHHOLD` (34), `ABSTAIN` (15), `ONE YEAR` (13). |
| `proxyVotingRecords[].vote.voteRecord[].managementRecommendation` | string | What management asked for. `FOR` (4,078), `AGAINST` (269), `NONE` (15). A record where `howVoted` differs from this value is a vote against management. |
| `proxyVotingRecords[].voteSeries`            | string  | Series ID that cast the vote, for example `S000009184`. It matches a series ID in `formData.seriesPage.seriesDetails.seriesReports[]`. |
| `proxyVotingRecords[].otherVoteDescription`  | string  | Free text. Present on 10 of the 4,372 records.                   |

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
