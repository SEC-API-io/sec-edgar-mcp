# directors-and-board-members

Search the boards of US public companies, funds and trusts, as their proxy
statements list them.

|                 |                                                                 |
| --------------- | --------------------------------------------------------------- |
| Category        | Governance and compensation                                     |
| Required input  | `query`                                                         |
| Returns         | `{total, data[]}`                                               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` (default `filedAt` descending) |
| REST equivalent | `POST /directors-and-board-members`                             |

## What it does

A proxy statement names the board, the committees, and the experience each
member brings. This tool searches that parsed data with Lucene syntax. The data
runs from 2007 to today and holds more than 100,000 people from more than
10,000 companies. Listed issuers, unlisted entities such as trusts, funds and
business development companies, and delisted companies all appear.

One row is one proxy statement, so one company for one year. The whole board
sits inside that row, in the `directors[]` array. Apple returns 20 rows, one per
proxy statement, and the row filed on 2026-01-08 lists 12 people.

The array mixes outside directors with named executive officers. Read `position`
and `committeeMemberships` before you call someone a director. In the Apple row,
Tim Cook and the CFO sit beside the independent board members.

## When to use it

- Who sits on Apple's board?
- Who chairs the audit committee?
- Which boards does this person sit on?
- How has this board changed over ten years?
- Which directors does the company mark as independent?

## When to use a different tool

| Situation                                  | Better tool                                       | Why                                                          |
| ------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------ |
| You want what the board was paid           | [`compensation`](./compensation.md)               | Pay rows carry the person, the year and the amounts.          |
| You want one company's full pay history    | [`compensation-by-key`](./compensation-by-key.md) | One CIK or ticker, up to 200 rows.                            |
| You want the proxy statement itself        | [`filing-search`](./filing-search.md)             | Search `formType:"DEF 14A"` and read the document URL.        |
| You want share trades by an officer        | [`insider-trading`](./insider-trading.md)         | Forms 3, 4 and 5 report trades. This dataset reports seats.   |
| You want the audit fees in the same proxy  | [`audit-fees`](./audit-fees.md)                   | Fees are a separate dataset.                                  |

## Input

| Parameter | Type    | Required | Constraints                                       | Notes                                                      |
| --------- | ------- | -------- | ------------------------------------------------- | ----------------------------------------------------------- |
| `query`   | string  | Yes      | Lucene, max 1000 characters, must contain a colon | `ticker:AAPL`. A bare word is rejected.                     |
| `from`    | integer | No       | 0 to 10,000                                       | Offset. Default 0.                                          |
| `size`    | integer | No       | 1 to 50                                           | Default 50. Above 50 the server returns HTTP 400.           |
| `sort`    | array   | No       | Elasticsearch sort clause                         | Default `[{"filedAt": {"order": "desc"}}]`.                 |

Query fields: `id`, `ticker`, `cik`, `entityName`, `accessionNo`, `filedAt`,
`directors.name`, `directors.position`, `directors.isIndependent`,
`directors.committeeMemberships`, `directors.qualificationsAndExperience`,
`directors.directorClass`, `directors.dateFirstElected`.

The company identifier is bare `ticker` here, as on
[`compensation`](./compensation.md). On [`audit-fees`](./audit-fees.md) the same
idea is `entities.ticker`. See [query language](../query-language.md).

A match on a `directors.*` field returns the whole row, with every other board
member in it. The server does not filter the inner array. That behaviour mirrors
[`subsidiaries`](./subsidiaries.md). `directors.name:"Alex Gorsky"` returned 31
filings, which is how you find interlocking directorates.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `eq` means the count is exact.

A row has 18 fields across three levels. Every one is listed below.

### Envelope

| Field            | Type   | Meaning                                                          |
| ---------------- | ------ | ----------------------------------------------------------------- |
| `total.value`    | number | Number of records that match the query.                          |
| `total.relation` | string | `eq` for an exact count. `gte` at 10000 means "10,000 or more".  |
| `data[]`         | array  | The matching records. One item is one proxy statement.           |

### Record

| Field                 | Type     | Meaning                                                       |
| --------------------- | -------- | --------------------------------------------------------------- |
| `data[].id`           | string   | Unique identifier of the record in the dataset.                |
| `data[].filedAt`      | string   | Date and time the information was disclosed to the SEC. ISO 8601 with an offset. |
| `data[].accessionNo`  | string   | Accession number of the filing the information was taken from. |
| `data[].cik`          | string   | Central Index Key of the company or entity the directors belong to. No leading zeros. |
| `data[].ticker`       | string   | Ticker of that company or entity. A single string, not an array. Empty when the entity has no stock exchange listing. |
| `data[].entityName`   | string   | Name of the company or entity the directors belong to.         |
| `data[].directors[]`  | object[] | The directors and board members of that company, as the proxy statement listed them. |

### Director

| Field                                            | Type     | Meaning                                       |
| ------------------------------------------------ | -------- | ----------------------------------------------- |
| `data[].directors[].name`                        | string   | Name of the director.                          |
| `data[].directors[].position`                    | string   | Positions or titles of the director, for example `Former Chair and CEO, Johnson & Johnson; Director`. Free text. |
| `data[].directors[].age`                         | string   | Age of the director. It is a **string**, not a number. It can be empty. |
| `data[].directors[].directorClass`               | string   | Class of the director on a staggered board, `I`, `II` or `III`. Empty when the filing gives no class. Some filings put free text here, for example `Executive Director`. |
| `data[].directors[].dateFirstElected`            | string   | Date the director was first elected. Free text. `2021`, `November 2017` and an empty string all appear. |
| `data[].directors[].isIndependent`               | boolean  | Whether the director is independent. `true`, `false` or `null`. It is `null` when the status is unknown. |
| `data[].directors[].committeeMemberships`        | string[] | Committees the director is a member of, for example `Audit Committee`. A chair is marked inside the string, `People and Compensation Committee (Chair)`. |
| `data[].directors[].qualificationsAndExperience` | string[] | Qualifications and experience of the director, as short phrases. |

`null` in `isIndependent` means the filing did not disclose the status. It does
not mean "not independent". `false` marks a person the filing reports as not
independent, such as the CEO. Ten of the 12 people in the Apple row are `null`.

Rows carry no document URL. Take `accessionNo` to
[`filing-search`](./filing-search.md) to open the proxy statement.

Size behaviour: page with `from` and `size`. Rows are heavy, because each one
holds a whole board with its experience text. One Apple row was 6,265 bytes, so
a `size: 50` call can pass 300 KB. Start with `size: 1` to `5`.

## Example

Prompt: "Who sits on Apple's board?"

```json
{ "name": "directors-and-board-members", "arguments": { "query": "ticker:AAPL", "size": 1 } }
```

Trimmed response:

```json
{
  "total": { "value": 20, "relation": "eq" },
  "data": [
    {
      "id": "42fe18db08211769589dc61fbd461443",
      "filedAt": "2026-01-08T16:31:36-05:00",
      "accessionNo": "0001308179-26-000008",
      "cik": "320193",
      "ticker": "AAPL",
      "entityName": "Apple Inc.",
      "directors": [
        {
          "name": "Alex Gorsky",
          "position": "Former Chair and CEO, Johnson & Johnson; Director",
          "age": "65", "directorClass": "II", "dateFirstElected": "2021", "isIndependent": false,
          "committeeMemberships": ["Nominating Committee", "People and Compensation Committee"],
          "qualificationsAndExperience": ["executive leadership experience", "brand marketing expertise"]
        },
        { "name": "Art Levinson", "position": "Board Chair; Chair of the Board", "age": "75", "committeeMemberships": ["Audit Committee", "People and Compensation Committee"] },
        { "name": "Tim Cook", "position": "CEO; Chief Executive Officer", "age": "65", "committeeMemberships": [] }
      ]
    }
  ]
}
```

The real row lists 12 people. Three are shown here for length.

## Limits and errors

- A query without a colon returns `Invalid Lucene query string`.
- A query longer than 1,000 characters returns
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 returns HTTP 400 with `Maximum 'size' limit of 50 exceeded.`
- Coverage is proxy statements from 2007 on. A company that files none has no
  rows.
- One query reaches 10,000 rows at most, because `from` stops at 10,000. Split a
  larger search with a `filedAt` range.
- Boards repeat year to year. The same person appears once per proxy statement,
  so deduplicate on `cik` plus `directors[].name` when you count people.
- Ages, titles and dates are free text taken from the filing. Parse them with
  care.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`compensation`](./compensation.md),
  [`compensation-by-key`](./compensation-by-key.md),
  [`audit-fees`](./audit-fees.md). The other proxy-statement datasets.
- [`filing-search`](./filing-search.md). Find the DEF 14A behind a row.
- [Query language](../query-language.md). Lucene syntax and field names.
- REST documentation:
  [Directors & Board Members API](https://sec-api.io/docs/directors-and-board-members-data-api)
