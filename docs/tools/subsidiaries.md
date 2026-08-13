# subsidiaries

Search the subsidiary lists that companies disclose in Exhibit 21 of their 10-K
filings.

|                 |                                                              |
| --------------- | ------------------------------------------------------------ |
| Category        | Company and entity                                           |
| Required input  | `query`                                                      |
| Returns         | `{total, data[]}`                                            |
| Pagination      | `from` (0 to 10000), `size` (1 to 50, default 50), `sort` (default `filedAt` descending) |
| REST equivalent | `POST /subsidiaries`                                         |

## What it does

Exhibit 21 of a 10-K lists the legal subsidiaries of the filer and the
jurisdiction each one is incorporated in. This tool searches that data with
Lucene syntax. One row is one Exhibit 21, so one company for one filing year.
The whole subsidiary list sits inside that row, in the `subsidiaries[]` array.
A company therefore returns one row per annual report. Apple returns 26 rows,
one for each year it has filed the exhibit, and the 2025 row lists 19
subsidiaries.

## When to use it

- Which legal entities does Apple own?
- Which companies list an Irish subsidiary?
- How has a company's subsidiary count changed over the years?
- Who is the parent of "Braeburn Capital, Inc."?
- Which subsidiaries did a company add or drop after an acquisition?

## When to use a different tool

| Situation                                        | Better tool                            | Why                                                                       |
| ------------------------------------------------ | -------------------------------------- | ------------------------------------------------------------------------- |
| You need a filer profile, not an ownership list  | [`edgar-entities`](./edgar-entities.md) | A subsidiary is rarely an SEC filer. `edgar-entities` covers registered filers only. |
| You want the raw Exhibit 21 document             | [`get-edgar-file`](./get-edgar-file.md) | This tool returns parsed names and jurisdictions, not the source file.    |
| You want the shareholders of a company           | [`form-13d-13g`](./form-13d-13g.md)    | Subsidiaries are companies owned by the filer, not owners of the filer.   |

## Input

| Parameter | Type    | Required | Constraints                       | Notes                                                        |
| --------- | ------- | -------- | --------------------------------- | ------------------------------------------------------------ |
| `query`   | string  | yes      | Lucene, max 1000 characters, must contain a colon | `ticker:AAPL`. A bare word is rejected.      |
| `from`    | integer | no       | 0 to 10000                        | Offset. Above 10000 the server returns an error.             |
| `size`    | integer | no       | 1 to 50, default 50               | Above 50 the server returns an error.                        |
| `sort`    | array   | no       | Elasticsearch sort clause         | Default `[{"filedAt":{"order":"desc"}}]`.                    |

Query fields confirmed to return rows:

| Field                        | Example                                  |
| ---------------------------- | ---------------------------------------- |
| `ticker`                     | `ticker:AAPL`                            |
| `cik`                        | `cik:320193`                             |
| `companyName`                | `companyName:"Apple Inc."`               |
| `accessionNo`                | `accessionNo:"0000320193-25-000079"`     |
| `filedAt`                    | `filedAt:[2025-01-01 TO 2025-12-31]`     |
| `subsidiaries.name`          | `subsidiaries.name:"Braeburn Capital, Inc."` |
| `subsidiaries.jurisdiction`  | `subsidiaries.jurisdiction:Ireland`      |

A match on `subsidiaries.name` or `subsidiaries.jurisdiction` returns the whole
parent row, with every sibling subsidiary in it. The server does not filter the
inner array.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `eq` means the count is exact. A `relation` of `gte` with value 10000 means
"10,000 or more", which is the search-window ceiling.

| Field                         | Type     | Meaning                                                     |
| ----------------------------- | -------- | ----------------------------------------------------------- |
| `id`                          | string   | Internal record ID.                                         |
| `accessionNo`                 | string   | Accession number of the 10-K that carried the exhibit.      |
| `filedAt`                     | string   | Filing date and time, ISO 8601 with offset.                 |
| `cik`                         | string   | Parent company CIK.                                         |
| `ticker`                      | string   | Parent company ticker. A single string, not an array.       |
| `companyName`                 | string   | Parent company name as filed.                               |
| `subsidiaries[]`              | object[] | The Exhibit 21 list.                                        |
| `subsidiaries[].name`         | string   | Legal name of the subsidiary.                               |
| `subsidiaries[].jurisdiction` | string   | Where it is incorporated, for example `Ireland` or `Delaware, U.S.` |

Size behaviour: page with `from` and `size`. Rows are heavy, because each row
carries a full subsidiary list. Ten rows of a broad jurisdiction query returned
184,676 bytes. All 26 Apple rows returned 18,046 bytes. Start with `size: 1` to
`5` when you do not know how large the lists are.

## Example

Prompt: "List the subsidiaries Apple disclosed in its latest 10-K."

```json
{ "name": "subsidiaries", "arguments": { "query": "ticker:AAPL", "size": 1 } }
```

```json
{
  "total": { "value": 26, "relation": "eq" },
  "data": [
    {
      "id": "53b6eca92223fed0008eae2e5e2ec8f1",
      "accessionNo": "0000320193-25-000079",
      "filedAt": "2025-10-31T06:01:26-04:00",
      "cik": "320193",
      "ticker": "AAPL",
      "companyName": "Apple Inc.",
      "subsidiaries": [
        { "name": "Apple Asia Limited", "jurisdiction": "Hong Kong" },
        { "name": "Apple Canada Inc.", "jurisdiction": "Canada" },
        { "name": "Apple Distribution International Limited", "jurisdiction": "Ireland" },
        { "name": "Apple India Private Limited", "jurisdiction": "India" },
        { "name": "Apple Operations Limited", "jurisdiction": "Ireland" },
        { "name": "Braeburn Capital, Inc.", "jurisdiction": "Nevada, U.S." },
        { "name": "iTunes K.K.", "jurisdiction": "Japan" }
      ]
    }
  ]
}
```

The real row lists 19 subsidiaries. Seven are shown here for length.

## Limits and errors

- A query without a colon returns
  `sec-api error: Invalid request parameter provided.` The same message comes
  back when `from` is above 10000.
- `size` above 50 returns
  `sec-api error: Maximum 'size' limit of 50 exceeded. ...`
- Coverage is Exhibit 21 only. A company that does not file the exhibit, or
  files it as a picture, has no rows.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`edgar-entities`](./edgar-entities.md) for the parent's SEC filer profile.
- [`mapping`](./mapping.md) to turn a company name into the ticker used here.
- [`filing-search`](./filing-search.md) to find the 10-K that carried the exhibit.
- Lucene syntax rules: [query language](../query-language.md)
- REST documentation: [Subsidiary API](https://sec-api.io/docs/subsidiary-api)
