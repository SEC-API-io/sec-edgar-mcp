# full-text-search

Search the full text of SEC EDGAR filings and their exhibits.

|                 |                                                                               |
| --------------- | ----------------------------------------------------------------------------- |
| Category        | Search and discovery                                                          |
| Required input  | `query`                                                                       |
| Returns         | `{total, filings[]}`                                                          |
| Pagination      | `page` only, 1-based, 100 rows per page. **No `size`, no `from`, no `sort`.** |
| REST equivalent | `POST https://api.sec-api.io/full-text-search`                                |

## What it does

`full-text-search` reads the body of EDGAR filings and every attached exhibit,
from 2001 to today. One row is one **document**, not one filing. A single 8-K
with four exhibits produces up to five rows that share the same `accessionNo`.
A search for `"quantum computing"` returned 100 rows that held only 84 distinct
accession numbers. Results come back by relevance, never by date, so `filedAt`
values arrive out of order. The search covers only the last 12 months unless you
give a date range.

## When to use it

- Which 8-K filings mention quantum computing?
- Who names a specific drug candidate, product or counterparty in a filing?
- In which exhibit does a phrase first appear?
- Which filings from these CIKs contain a term in a given year?
- Which companies use a particular contract clause wording?

## When to use a different tool

| Situation                                               | Better tool                           | Why                                                                                     |
| ------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------- |
| You filter by ticker, form type or date, not by wording | [`filing-search`](./filing-search.md) | `filing-search` sorts by date, pages up to 10,000 rows and returns far richer metadata. |
| You need filings older than 2001                        | [`filing-search`](./filing-search.md) | Full-text coverage starts in 2001. Metadata coverage starts in 1993.                    |
| You already have the filing and want a section          | [`extractor`](./extractor.md)         | `full-text-search` returns a document URL, never the matched text.                      |
| You want the 8-K item codes                             | [`form-8k`](./form-8k.md)             | Item codes are structured data, not free text.                                          |

## Input

| Parameter   | Type             | Required | Constraints      | Notes                                                                          |
| ----------- | ---------------- | -------- | ---------------- | ------------------------------------------------------------------------------ |
| `query`     | string           | yes      | Search operators | Quotes make an exact phrase. `OR` and wildcards also work.                     |
| `formTypes` | array of strings | no       | Form names       | Example `["8-K", "10-Q"]`. Filters the form, not the exhibit type.             |
| `ciks`      | array of strings | no       | CIK numbers      | The server pads each value to 10 digits for you.                               |
| `startDate` | string           | no       | `YYYY-MM-DD`     | It applies only **with** `endDate`. See the warning below.                     |
| `endDate`   | string           | no       | `YYYY-MM-DD`     | It applies only **with** `startDate`.                                          |
| `page`      | integer          | no       | 1 or higher      | 1-based. Page 2 starts at row 101.                                             |

`query` is a search expression, not Lucene field syntax. You cannot write
`ticker:AAPL` here. `ciks` and `formTypes` carry the filters. Quotes around a
phrase make an exact match, for example `"quantum computing"`. The `OR` and
wildcard operators also work.

**The date range needs both dates.** The server applies your range only when
`startDate` and `endDate` are both set. If either one is missing it overwrites
both with a window of the last 12 months, silently. With no dates at all, the
100 rows span `2025-08-13` to `2026-08-12` for a request made on `2026-08-13`.

## Output

The envelope has two keys, and no `query` echo, unlike
[`filing-search`](./filing-search.md). `total` is an object, `{value, relation}`,
not a number. `filings` is the array of document rows.

| Field                       | Type           | Meaning                                                                                                                                       |
| --------------------------- | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `total.value`               | number         | Number of matching documents.                                                                                                                 |
| `total.relation`            | string         | `eq` means exact. `gte` means at least that many.                                                                                             |
| `filings[].accessionNo`     | string         | Accession number of the filing that holds the document. Repeats across rows.                                                                  |
| `filings[].cik`             | string         | Filer CIK, leading zeros removed.                                                                                                             |
| `filings[].companyNameLong` | string         | Filer name, for example `Quantum Computing Inc. (Filer)`. An older form also appears, `Lipocine Inc. (LPCN) (CIK 0001535955)`. |
| `filings[].ticker`          | string or null | Ticker of the filer. It is `null` when no ticker maps. 7 of the 100 rows were `null`.                                                |
| `filings[].description`     | string         | Document description as filed, for example `INVESTOR PRESENTATION, DATED MAY 13, 2026`. Free text, not normalized.                            |
| `filings[].formType`        | string         | Form type of the parent filing, for example `8-K`.                                                                                            |
| `filings[].type`            | string         | Document type inside that filing, for example `EX-99.1`. This is the field that separates the exhibit rows.                                   |
| `filings[].filingUrl`       | string         | Direct URL of the matched document on sec.gov.                                                                                                |
| `filings[].filedAt`         | string         | Filing date, `YYYY-MM-DD`. Date only. `filing-search` returns a full timestamp instead.                                                       |

That is the whole row. There are nine fields and no others. The response carries
no relevance score, no matched snippet and no highlight.

**Pagination is `page` and nothing else.** There is no `size`, so you always get
100 rows per call. There is no `sort`, so you cannot ask for the newest match
first. The order is relevance in every response. A 100-row page came to
31,531 bytes, so a page costs about 32 KB.

## Example

Prompt: "Which 8-K filings mention quantum computing?"

```json
{
  "name": "full-text-search",
  "arguments": { "query": "\"quantum computing\"", "formTypes": ["8-K"] }
}
```

```json
{
  "total": { "value": 413, "relation": "eq" },
  "filings": [
    {
      "accessionNo": "0001213900-26-058515",
      "cik": "1758009",
      "companyNameLong": "Quantum Computing Inc. (Filer)",
      "ticker": "QUBT",
      "description": "INVESTOR PRESENTATION, DATED MAY 13, 2026",
      "formType": "8-K",
      "type": "EX-99.1",
      "filingUrl": "https://www.sec.gov/Archives/edgar/data/1758009/000121390026058515/ea029123901ex99-1.htm",
      "filedAt": "2026-05-18"
    },
    {
      "accessionNo": "0001907982-26-000052",
      "ticker": "QBTS",
      "formType": "8-K",
      "type": "EX-99.1",
      "filedAt": "2026-05-05"
    }
  ]
}
```

The second row is trimmed here. Every row carries the same nine fields. 413
matches at 100 rows per page means 5 calls. `page: 2` returns the next 100.

## Limits and errors

- Coverage starts in 2001. Older filings are invisible here.
- The default date window is the last 12 months. A request with both `startDate`
  and `endDate` reaches further back.
- A row is a document, not a filing. `total.value` counts documents, so it is
  higher than the number of matching filings. The unique `accessionNo` values
  give the filing count.
- The query string has a 2,000 character ceiling, with the message
  `Query too long. Maximum length: 2000 characters`. This is a tighter limit
  than the 3,500 characters of [`filing-search`](./filing-search.md).
- The endpoint proxies EDGAR full-text search.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`filing-search`](./filing-search.md). Filter by metadata, sort by date, page deeper.
- [`extractor`](./extractor.md), [`get-edgar-file`](./get-edgar-file.md). Read a document you found here.
- [Query language](../query-language.md). Why this tool does not take Lucene fields.
- [sec-api.io Full-Text Search API docs](https://sec-api.io/docs/full-text-search-api)
