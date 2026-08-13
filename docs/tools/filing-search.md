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
| You want the structured items in an 8-K    | [`form-8k`](./form-8k.md)                                              | `form-8k` returns parsed item data. `filing-search` gives the item codes in `items` and in `description`. |
| You have a name and need a CIK or ticker   | [`mapping`](./mapping.md)                                              | `mapping` resolves a name to a CIK or ticker. `filing-search` queries those fields.                |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                                |
| --------- | ------- | -------- | ------------------------ | ---------------------------------------------------- |
| `query`   | string  | yes      | Lucene syntax            | Example: `ticker:AAPL AND formType:"10-K"`.          |
| `from`    | integer | no       | 0 to 9999                | Offset of the first row. Default 0.                  |
| `size`    | integer | no       | 1 to 50                  | Rows per call. Default 50. Over 50 returns an error. |
| `sort`    | array   | no       | Elasticsearch sort array | Default `[{"filedAt": {"order": "desc"}}]`.          |

Query fields: `id`, `accessionNo`, `formType`, `filedAt`, `cik`, `ticker`,
`companyName`, `companyNameLong`, `description`, `periodOfReport`, `items`,
`groupMembers`, `linkToFilingDetails`, `linkToTxt`, `linkToHtml`,
`effectivenessDate`, `effectivenessTime`, `registrationForm`,
`referenceAccessionNo`, `entities.cik`, `entities.sic`,
`entities.stateOfIncorporation`, `entities.fiscalYearEnd`, `entities.irsNo`,
`entities.fileNo`, `entities.filmNo`, `entities.act`, `entities.type`,
`documentFormatFiles.type`, `documentFormatFiles.description`,
`documentFormatFiles.documentUrl`, `documentFormatFiles.size`,
`dataFiles.type`, `dataFiles.sequence`,
`seriesAndClassesContractsInformation.series`,
`seriesAndClassesContractsInformation.classesContracts.classContract`.

## Output

The envelope has three keys.

- `total` is an object, `{value, relation}`. It is not a number.
- `query` echoes the paging that the server applied, `{from, size}`.
- `filings` is the array of rows.

[Response format](../response-format.md) groups this tool in the
`{total, filings[]}` family. The `query` echo is extra.

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | -------------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                        |
| `total.relation` | string | `eq` means exact. `gte` means at least that many.              |
| `query.from`     | number | Offset of the first row the server returned.                   |
| `query.size`     | number | Number of rows the server returned.                            |
| `filings[]`      | array  | The filing rows. 50 rows at most.                              |

### Filing

One row is one filing. The last four fields hold nested objects. Their tables
follow below.

| Field                                              | Type             | Meaning                                                                                                                 |
| -------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `filings[].id`                                     | string           | System-internal unique ID of the filing object.                                                                         |
| `filings[].accessionNo`                            | string           | Accession number of the filing. It keys the submission on EDGAR.                                                        |
| `filings[].formType`                               | string           | EDGAR filing form type, such as `10-K`.                                                                                 |
| `filings[].filedAt`                                | string           | Date and time EDGAR accepted the filing, ISO 8601 with an offset, for example `2025-10-31T06:01:26-04:00`.              |
| `filings[].cik`                                    | string           | CIK of the filing issuer, leading zeros removed.                                                                        |
| `filings[].ticker`                                 | string           | Ticker symbol of the filing company. Empty when no ticker maps.                                                         |
| `filings[].companyName`                            | string           | Name of the primary filing company or person.                                                                           |
| `filings[].companyNameLong`                        | string           | Long version of the company name. It includes the filer type, for example `Apple Inc. (Filer)`.                         |
| `filings[].description`                            | string           | Description of the form. For 8-K filings it lists the items.                                                            |
| `filings[].periodOfReport`                         | string           | Period of report, `YYYY-MM-DD`. Optional.                                                                               |
| `filings[].linkToFilingDetails`                    | string           | URL of the actual filing content on sec.gov.                                                                            |
| `filings[].linkToTxt`                              | string           | URL of the plain text `.txt` version of the filing.                                                                     |
| `filings[].linkToHtml`                             | string           | URL of the index page of the filing, also called the filing detail page.                                                |
| `filings[].linkToXbrl`                             | string           | XBRL instance URL. It is an empty string in the response, even though XBRL files exist. `dataFiles` holds those entries. |
| `filings[].items[]`                                | array of strings | Item strings as reported on form 8-K, 8-K/A, D, D/A, ABS-15G, ABS-15G/A, 1-U and 1-U/A. Optional.                       |
| `filings[].groupMembers[]`                         | array of strings | Member strings as reported on SC 13G, SC 13G/A, SC 13D and SC 13D/A filings. Optional.                                  |
| `filings[].effectivenessDate`                      | string           | Effectiveness date, `YYYY-MM-DD`. EFFECT forms only.                                                                    |
| `filings[].effectivenessTime`                      | string           | Effectiveness time, `HH:mm:ss`. EFFECT forms only.                                                                      |
| `filings[].registrationForm`                       | string           | Registration form type as reported on EFFECT forms.                                                                     |
| `filings[].referenceAccessionNo`                   | string           | Reference accession number as reported on EFFECT forms.                                                                 |
| `filings[].entities[]`                             | array            | The EDGAR header entities of the submission. One item per filer, subject or issuer.                                     |
| `filings[].documentFormatFiles[]`                  | array            | Every document of the submission.                                                                                       |
| `filings[].dataFiles[]`                            | array            | The data files of the submission, such as the XBRL files.                                                               |
| `filings[].seriesAndClassesContractsInformation[]` | array            | Fund series and class data. Empty for operating companies.                                                              |

### Entities

| Field                                       | Type   | Meaning                                                                                            |
| ------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------- |
| `filings[].entities[].companyName`          | string | Company name of the entity, with the filer type appended.                                          |
| `filings[].entities[].cik`                  | string | CIK of the entity.                                                                                 |
| `filings[].entities[].irsNo`                | string | IRS number of the entity.                                                                          |
| `filings[].entities[].stateOfIncorporation` | string | State of incorporation of the entity, for example `CA`.                                            |
| `filings[].entities[].fiscalYearEnd`        | string | Fiscal year end of the entity, `MMDD`. `0927` is 27 September.                                     |
| `filings[].entities[].sic`                  | string | SIC of the entity. Code plus industry name. HTML entities such as `&amp;` can appear.              |
| `filings[].entities[].type`                 | string | Type of the filing being filed.                                                                    |
| `filings[].entities[].act`                  | string | The SEC act pursuant to which the filing was filed, for example `34`.                              |
| `filings[].entities[].fileNo`               | string | Filer number of the entity.                                                                        |
| `filings[].entities[].filmNo`               | string | Film number of the entity.                                                                         |
| `filings[].entities[].undefined`            | string | Not a real field. It holds a leftover fragment of the EDGAR office text, such as `06 Technology)`. Ignore it. |

### Files

`documentFormatFiles[]` lists the readable documents. `dataFiles[]` lists the
machine-readable files of the same submission. Both carry the same five keys.

| Field                                          | Type   | Meaning                                                        |
| ---------------------------------------------- | ------ | -------------------------------------------------------------- |
| `filings[].documentFormatFiles[].sequence`     | string | Sequence number of the file inside the submission.             |
| `filings[].documentFormatFiles[].description`  | string | Description of the file.                                       |
| `filings[].documentFormatFiles[].documentUrl`  | string | URL of the file on sec.gov.                                    |
| `filings[].documentFormatFiles[].type`         | string | Type of the file, for example `EX-4.1`.                        |
| `filings[].documentFormatFiles[].size`         | string | Size of the file in bytes.                                     |
| `filings[].dataFiles[].sequence`               | string | Sequence number of the file inside the submission.             |
| `filings[].dataFiles[].description`            | string | Description of the file, for example `XBRL TAXONOMY EXTENSION SCHEMA DOCUMENT`. |
| `filings[].dataFiles[].documentUrl`            | string | URL of the file on sec.gov.                                    |
| `filings[].dataFiles[].type`                   | string | Type of the file, for example `EX-101.SCH`.                    |
| `filings[].dataFiles[].size`                   | string | Size of the file in bytes.                                     |

### Series and classes

Investment company filings carry this array. It maps the fund series and the
share classes that the filing covers.

| Field                                                                     | Type   | Meaning                                 |
| ------------------------------------------------------------------------- | ------ | ----------------------------------------- |
| `filings[].seriesAndClassesContractsInformation[].series`                 | string | Series ID.                              |
| `filings[].seriesAndClassesContractsInformation[].name`                   | string | Name of the entity.                     |
| `filings[].seriesAndClassesContractsInformation[].classesContracts[]`     | array  | List of the classes or contracts.       |
| `filings[].seriesAndClassesContractsInformation[].classesContracts[].classContract` | string | Class or contract ID.         |
| `filings[].seriesAndClassesContractsInformation[].classesContracts[].name` | string | Name of the class or contract.         |
| `filings[].seriesAndClassesContractsInformation[].classesContracts[].ticker` | string | Ticker of the class or contract.      |

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
