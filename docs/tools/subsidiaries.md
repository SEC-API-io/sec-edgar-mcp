# subsidiaries

Search the subsidiary lists that companies disclose in Exhibit 21 of their SEC
filings.

|                 |                                                              |
| --------------- | ------------------------------------------------------------ |
| Category        | Company and entity                                           |
| Required input  | `query`                                                      |
| Returns         | `{total, data[]}`                                            |
| Pagination      | `from` (0 to 10000), `size` (1 to 50, default 50), `sort` (default `filedAt` descending) |
| REST equivalent | `POST /subsidiaries`                                         |

## What it does

Exhibit 21 lists the legal subsidiaries of the filer and the jurisdiction each
one is incorporated in. Companies attach it to filings such as the 10-K, 10-Q,
S-1 and 20-F. This tool searches that data with Lucene syntax. One row is one
Exhibit 21, so one company for one filing. The whole subsidiary list sits
inside that row, in the `subsidiaries[]` array. A company therefore returns one
row per exhibit it has filed. Apple returns 26 rows, and the 2025 row lists 19
subsidiaries.

The data starts in 2003 and holds more than 100,000 lists. About 7,000 new
lists arrive each year. It keeps companies that merged, went private or stopped
filing.

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

Query fields:

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

### Envelope

| Field            | Type   | Meaning                                                          |
| ---------------- | ------ | ------------------------------------------------------------------ |
| `total.value`    | number | Number of subsidiary lists that match the query.                 |
| `total.relation` | string | `eq` for an exact count. `gte` at 10000 for 10,000 or more.      |
| `data[]`         | array  | The subsidiary lists, up to 50 per response. One item is one Exhibit 21. |

### Exhibit 21

| Field                                 | Type     | Meaning                                             |
| ------------------------------------- | -------- | ----------------------------------------------------- |
| `data[].id`                           | string   | Unique identifier of the subsidiary list.           |
| `data[].accessionNo`                  | string   | Accession number of the EDGAR filing the exhibit is attached to. |
| `data[].filedAt`                      | string   | Date and time the list was disclosed, ISO 8601 with offset. |
| `data[].cik`                          | string   | CIK of the parent company.                          |
| `data[].ticker`                       | string   | Ticker symbol of the parent company. A single string, not an array. |
| `data[].companyName`                  | string   | Name of the parent company as filed.                |
| `data[].subsidiaries[]`               | object[] | The subsidiaries of the parent company.             |
| `data[].subsidiaries[].name`          | string   | Legal name of the subsidiary.                       |
| `data[].subsidiaries[].jurisdiction`  | string   | Jurisdiction the subsidiary is incorporated in, for example `Ireland` or `Delaware, U.S.` |

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
- Coverage is Exhibit 21 only, and it starts in 2003. A company that does not
  file the exhibit, or files it as a picture, has no rows.
- The data is machine-extracted from a free-form exhibit. Fewer than 0.1% of
  the lists carry an error.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`edgar-entities`](./edgar-entities.md) for the parent's SEC filer profile.
- [`mapping`](./mapping.md) to turn a company name into the ticker used here.
- [`filing-search`](./filing-search.md) to find the filing that carried the
  exhibit.
- Lucene syntax rules: [query language](../query-language.md)
- REST documentation: [Subsidiary API](https://sec-api.io/docs/subsidiary-api)
