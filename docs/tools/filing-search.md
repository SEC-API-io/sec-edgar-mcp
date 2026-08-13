# filing-search

Search SEC EDGAR filings from 1993 to today and get their metadata.

|                 |                                                          |
| --------------- | -------------------------------------------------------- |
| Category        | Search and discovery                                     |
| Required input  | `query`                                                  |
| Returns         | `{total, query, filings[]}`                              |
| Pagination      | `from` (0 to 9999), `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST https://api.sec-api.io` (Query API)                |

## What it does

`filing-search` searches the EDGAR filing metadata index with a Lucene query. It
covers all filings from 1993 to today. One row is one filing, not one document.
Each row carries the filer identity, the form type, the dates, and the URL of
every file in the submission. It does not search the text inside the documents.
[`full-text-search`](./full-text-search.md) does that.

## When to use it

- What are the last 10 10-K filings from Apple?
- Which 8-K filings did Tesla file in the first quarter of 2024?
- What is the accession number of the newest 10-Q from a given CIK?
- Where is the primary document URL for a filing I want to read next?
- Which exhibits came with this annual report, and how big is each one?

## When to use a different tool

| Situation                                  | Better tool                                                            | Why                                                                                                |
| ------------------------------------------ | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| You look for filings that mention a phrase | [`full-text-search`](./full-text-search.md)                            | `filing-search` filters on metadata fields only. It never reads the document body.                 |
| You have the filing and want its content   | [`extractor`](./extractor.md), [`get-edgar-file`](./get-edgar-file.md) | `filing-search` returns URLs, not text. Those tools take `linkToFilingDetails` as input.           |
| You want the financial statements          | [`xbrl-to-json`](./xbrl-to-json.md)                                    | `filing-search` gives you the XBRL file URLs. It does not parse them.                              |
| You want the structured items in an 8-K    | [`form-8k`](./form-8k.md)                                              | `form-8k` returns parsed item data. `filing-search` only gives the item list inside `description`. |
| You have a name and need a CIK or ticker   | [`mapping`](./mapping.md)                                              | `mapping` resolves a name to a CIK or ticker. `filing-search` queries those fields.                |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                                |
| --------- | ------- | -------- | ------------------------ | ---------------------------------------------------- |
| `query`   | string  | yes      | Lucene syntax            | Example: `ticker:AAPL AND formType:"10-K"`.          |
| `from`    | integer | no       | 0 to 9999                | Offset of the first row. Default 0.                  |
| `size`    | integer | no       | 1 to 50                  | Rows per call. Default 50. Over 50 returns an error. |
| `sort`    | array   | no       | Elasticsearch sort array | Default `[{"filedAt": {"order": "desc"}}]`.          |

Query fields: `ticker`, `formType`, `cik`, `companyName`, `companyNameLong`,
`accessionNo`, `filedAt`, `periodOfReport`, `description`, `id`, `entities.cik`,
`entities.sic`, `entities.stateOfIncorporation`, `entities.fiscalYearEnd`,
`entities.fileNo`, `documentFormatFiles.type`.

## Output

The envelope has three keys.

- `total` is an object, `{value, relation}`. It is not a number.
- `query` echoes the paging that the server applied, `{from, size}`.
- `filings` is the array of rows.

[Response format](../response-format.md) groups this tool in the
`{total, filings[]}` family. The `query` echo is extra.

| Field                                              | Type   | Meaning                                                                                                                       |
| -------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `total.value`                                      | number | Number of matching filings.                                                                                                   |
| `total.relation`                                   | string | `eq` means exact. `gte` means at least that many.                                                                             |
| `filings[].accessionNo`                            | string | EDGAR accession number, the filing key.                                                                                       |
| `filings[].formType`                               | string | Form type, such as `10-K`.                                                                                                    |
| `filings[].filedAt`                                | string | Filing timestamp, ISO 8601 with an offset, for example `2025-10-31T06:01:26-04:00`.                                           |
| `filings[].periodOfReport`                         | string | Period the filing reports on, `YYYY-MM-DD`.                                                                                   |
| `filings[].cik`                                    | string | Filer CIK, leading zeros removed.                                                                                             |
| `filings[].ticker`                                 | string | Ticker of the filer, when one maps.                                                                                           |
| `filings[].companyName`                            | string | Clean company name.                                                                                                           |
| `filings[].companyNameLong`                        | string | Name with the EDGAR role, for example `Apple Inc. (Filer)`.                                                                   |
| `filings[].description`                            | string | Form description. For 8-K filings it lists the items.                                                                         |
| `filings[].linkToFilingDetails`                    | string | URL of the primary document.                                                                                                  |
| `filings[].linkToHtml`                             | string | URL of the EDGAR filing index page.                                                                                           |
| `filings[].linkToTxt`                              | string | URL of the complete submission text file.                                                                                     |
| `filings[].linkToXbrl`                             | string | XBRL instance URL. It is an empty string in the response, even though XBRL files exist. `dataFiles` holds those entries.      |
| `filings[].documentFormatFiles[]`                  | array  | Every document in the submission. Each has `sequence`, `size`, `documentUrl`, `description`, `type`.                          |
| `filings[].dataFiles[]`                            | array  | XBRL and other data files. Same shape as `documentFormatFiles`.                                                               |
| `filings[].entities[]`                             | array  | One entry per filer. Holds `cik`, `sic`, `stateOfIncorporation`, `fiscalYearEnd`, `fileNo`, `irsNo`, `filmNo`, `act`, `type`. |
| `filings[].id`                                     | string | Internal record hash.                                                                                                         |
| `filings[].seriesAndClassesContractsInformation[]` | array  | Fund series and class data. Empty for operating companies.                                                                    |

Size behaviour. `size` defaults to 50 and cannot exceed 50. `from` pages the
results and stops at 9999. One Apple 10-K row with its full file lists was 4,202
bytes, so a `size: 50` call can reach roughly 200 KB.

## Example

Prompt: "Get the newest Apple 10-K and its exhibit URLs."

```json
{
  "name": "filing-search",
  "arguments": { "query": "ticker:AAPL AND formType:\"10-K\"", "size": 1 }
}
```

```json
{
  "total": { "value": 33, "relation": "eq" },
  "query": { "from": 0, "size": 1 },
  "filings": [
    {
      "ticker": "AAPL",
      "formType": "10-K",
      "accessionNo": "0000320193-25-000079",
      "cik": "320193",
      "companyName": "Apple Inc.",
      "linkToFilingDetails": "https://www.sec.gov/Archives/edgar/data/320193/000032019325000079/aapl-20250927.htm",
      "filedAt": "2025-10-31T06:01:26-04:00",
      "periodOfReport": "2025-09-27",
      "documentFormatFiles": [
        {
          "sequence": "1",
          "size": "1520208",
          "documentUrl": "https://www.sec.gov/ix?doc=/Archives/edgar/data/320193/000032019325000079/aapl-20250927.htm",
          "description": "10-K",
          "type": "10-K"
        }
      ]
    }
  ]
}
```

## Limits and errors

- `size` above 50 fails with HTTP 400: `Maximum 'size' limit of 50 exceeded.`
- A `total.value` of exactly `10000` is the search-window ceiling, not a true
  count. It means 10,000 or more. A narrower query returns an exact count. The
  query string has a 3,500 character ceiling.
- Two data quirks are visible in the response. Each `entities[]` object can carry
  a key literally named `undefined`. The
  complete submission file entry uses a non-breaking space, U+00A0, for
  `sequence` and `type`.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`full-text-search`](./full-text-search.md). Search the document text instead.
- [`extractor`](./extractor.md), [`get-edgar-file`](./get-edgar-file.md). Read a filing you found here.
- [Query language](../query-language.md). Lucene syntax and field names.
- [sec-api.io Query API docs](https://sec-api.io/docs/query-api)
