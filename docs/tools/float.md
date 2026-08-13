# float

Fetch the outstanding share count and the public float of one company.

|                 |                                                                                                      |
| --------------- | ---------------------------------------------------------------------------------------------------- |
| Category        | Company and entity                                                                                   |
| Required input  | `ticker` or `cik`. The schema marks neither as required, but the server rejects a call with neither. |
| Returns         | `{total, data[]}`                                                                                    |
| Pagination      | **None.** No `from`, `size` or `sort`.                                                               |
| REST equivalent | `GET /float?ticker=AAPL`                                                                             |

## What it does

The tool returns the share counts that companies print on the cover page of
their 10-K and 10-Q filings. One row is one source filing, so one reporting
period. Outstanding shares appear on both 10-K and 10-Q rows. Public float is
disclosed once a year, in the 10-K, so `publicFloat` is an empty array on most
rows. A request for Apple returns 61 rows and reaches back to a filing reported
on 2011-07-20.

The data starts in 2011. It covers listed and unlisted companies on US markets,
domestic and foreign filers, and it keeps companies that no longer trade. A row
holds the count as of the last quarterly or annual report. Shares issued or
bought back after that report are not in it.

## When to use it

- How many Apple shares are outstanding today?
- What was Apple's public float at the last fiscal year end?
- How many shares does each class of Alphabet stock have?
- How far has the share count fallen over ten years of buybacks?
- Which filing reported this share count?

## When to use a different tool

| Situation                                        | Better tool                           | Why                                                                                                              |
| ------------------------------------------------ | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| You have a company name or a CUSIP, not a ticker | [`mapping`](./mapping.md)             | `float` consumes an identifier. It does not resolve one.                                                         |
| You want the filing behind a row                 | [`filing-search`](./filing-search.md) | `float` gives you `sourceFilingAccessionNo`, not the document. `filing-search` returns filing metadata and URLs. |
| You want revenue, assets or other financials     | [`xbrl-to-json`](./xbrl-to-json.md)   | `float` covers cover-page share data only.                                                                       |

## Input

| Parameter | Type   | Required       | Constraints        | Notes                                                                            |
| --------- | ------ | -------------- | ------------------ | -------------------------------------------------------------------------------- |
| `ticker`  | string | one of the two | 1 to 10 characters | Use the plain symbol, `AAPL`.                                                    |
| `cik`     | string | one of the two | 1 to 20 characters | Send the CIK without leading zeros. `320193` works. `0000320193` returns 0 rows. |

The schema sets `additionalProperties: true`, so a client may send extra keys.
The server ignores them. If you send both `ticker` and `cik`, the server uses
`ticker`. `{ "ticker": "AAPL", "cik": "1652044" }` returns Apple.

This tool takes no `query`. There is no Lucene syntax here.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`,
not a number. One element of `data[]` is one source filing.

### Envelope

| Field            | Type   | Meaning                                                          |
| ---------------- | ------ | ------------------------------------------------------------------ |
| `total.value`    | number | Number of float records that match the request.                  |
| `total.relation` | string | `eq` for an exact count.                                         |
| `data[]`         | array  | The float records. One item is one reporting period.             |

### Record

| Field                            | Type     | Meaning                                                          |
| -------------------------------- | -------- | ------------------------------------------------------------------ |
| `data[].id`                      | string   | Unique identifier of the float record.                           |
| `data[].tickers`                 | string[] | The tickers the record applies to. One ticker for one share class, several for more. Alphabet returns `["GOOGL","GOOG","GOOGM","GOOGN"]`. |
| `data[].cik`                     | string   | CIK of the filer, no leading zeros.                              |
| `data[].reportedAt`              | string   | Time the source filing and its share data went public, ISO 8601 with offset. |
| `data[].periodOfReport`          | string   | Fiscal period the source filing covers, `YYYY-MM-DD`. Absent on some pre-2013 rows. |
| `data[].sourceFilingAccessionNo` | string   | Accession number of the 10-K or 10-Q the numbers came from.      |
| `data[].float`                   | object   | The share data for that reporting period.                        |

### Share counts and public float

| Field                                         | Type     | Meaning                                            |
| --------------------------------------------- | -------- | ---------------------------------------------------- |
| `data[].float.outstandingShares[]`            | object[] | One object per share class. Each one holds a share count and the date it applies to. |
| `data[].float.outstandingShares[].period`     | string   | The date the count applies to, `YYYY-MM-DD`. It is not the filing date. It usually falls between `periodOfReport` and `reportedAt`. |
| `data[].float.outstandingShares[].shareClass` | string   | Class label, when the filer reports one. Often an empty string. Alphabet rows use `CommonClassA`, `CommonClassB`, `CapitalClassC`. |
| `data[].float.outstandingShares[].value`      | number   | Number of shares outstanding in that class.        |
| `data[].float.publicFloat[]`                  | object[] | One object per share class. Each one holds the dollar amount of the public float and the date it applies to. Empty when the filing gave no figure. That is normal on 10-Q rows. |
| `data[].float.publicFloat[].period`           | string   | The date the amount applies to, `YYYY-MM-DD`.      |
| `data[].float.publicFloat[].shareClass`       | string   | Class label, when the filer reports one. Often an empty string. |
| `data[].float.publicFloat[].value`            | number   | The public float in US dollars, not a share count. Apple reported 3,253,431,000,000 for 2025-03-28. |

**This tool has no pagination.** `size`, `from` and `sort` are accepted by the
schema and then ignored. One call returns every period the server holds for that
company, newest first. Apple returns 61 rows and 19,434 bytes. Alphabet returns
29 rows. Budget the context before you call it, because you cannot ask for less.

## Example

Prompt: "How many Apple shares are outstanding, and what was the last public float?"

```json
{ "name": "float", "arguments": { "ticker": "AAPL" } }
```

```json
{
  "total": { "value": 61, "relation": "eq" },
  "data": [
    {
      "id": "a923804208d5bda4533272b211fd7b41",
      "tickers": ["AAPL"],
      "cik": "320193",
      "float": {
        "outstandingShares": [
          { "period": "2026-07-17", "shareClass": "", "value": 14594180000 }
        ],
        "publicFloat": []
      },
      "reportedAt": "2026-07-31T06:01:02-04:00",
      "periodOfReport": "2026-06-27",
      "sourceFilingAccessionNo": "0000320193-26-000020"
    },
    {
      "id": "736081ee32d8abd105e3d9cf4fadc5fb",
      "tickers": ["AAPL"],
      "float": {
        "outstandingShares": [
          { "period": "2025-10-17", "shareClass": "", "value": 14776353000 }
        ],
        "publicFloat": [
          { "period": "2025-03-28", "shareClass": "", "value": 3253431000000 }
        ]
      }
    }
  ]
}
```

## Limits and errors

- A call with neither identifier returns
  `sec-api error: Parameter <ticker> or <cik> invalid.`
- A zero-padded CIK returns `{"total":{"value":0},"data":[]}`. It does not
  return an error. Strip the leading zeros.
- The server caps the underlying query at 200 records. Apple has 61, so it
  stays under the cap.
- The tool description calls public float "shares held by non-affiliates".
  `publicFloat[].value` is the public float in US dollars, not a share count.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`mapping`](./mapping.md) resolves a name or CUSIP to the ticker this tool needs.
- [`edgar-entities`](./edgar-entities.md) has the filer profile behind the CIK.
- Envelope comparison across tools: [response format](../response-format.md)
- REST documentation: [Outstanding Shares & Public Float API](https://sec-api.io/docs/outstanding-shares-float-api)
