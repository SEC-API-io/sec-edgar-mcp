# directors-and-board-members

Search the boards of US public companies, as their proxy statements list them.

|                 |                                                                 |
| --------------- | --------------------------------------------------------------- |
| Category        | Governance and compensation                                     |
| Required input  | `query`                                                         |
| Returns         | `{total, data[]}`                                               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` (default `filedAt` descending) |
| REST equivalent | `POST /directors-and-board-members`                             |

## What it does

A proxy statement names the board, the committees, and the experience each
member brings. This tool searches that parsed data with Lucene syntax.

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
| `from`    | integer | No       | minimum 0                                         | Offset. Default 0.                                          |
| `size`    | integer | No       | 1 to 50                                           | Default 50. Above 50 the server returns HTTP 400.           |
| `sort`    | array   | No       | Elasticsearch sort clause                         | Default `[{"filedAt": {"order": "desc"}}]`.                 |

Query fields verified live on 2026-08-13: `ticker`, `cik`, `entityName`,
`directors.name`, `directors.isIndependent`.

Taken from the response shape and not verified: `accessionNo`, `filedAt`,
`directors.position`, `directors.committeeMemberships`,
`directors.directorClass`, `directors.dateFirstElected`.

The company identifier is bare `ticker` here, as on
[`compensation`](./compensation.md). On [`audit-fees`](./audit-fees.md) the same
idea is `entities.ticker`. See [query language](../query-language.md).

A match on a `directors.*` field returns the whole row, with every other board
member in it. The server does not filter the inner array. That behaviour mirrors
[`subsidiaries`](./subsidiaries.md) and was not verified live. The field itself
was: `directors.name:"Alex Gorsky"` returned 31 filings, which is how you find
interlocking directorates.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `eq` means the count is exact.

| Field                                    | Type     | Meaning                                                     |
| ---------------------------------------- | -------- | ----------------------------------------------------------- |
| `id`                                     | string   | Internal record ID.                                          |
| `filedAt`                                | string   | When the proxy statement was filed. ISO 8601 with an offset. |
| `accessionNo`                            | string   | Accession number of that proxy statement.                    |
| `cik`                                    | string   | Company CIK, no leading zeros.                               |
| `ticker`                                 | string   | Company ticker. A single string, not an array.               |
| `entityName`                             | string   | Company name as filed.                                       |
| `directors[]`                            | object[] | The board as that proxy statement listed it.                 |
| `directors[].name`                       | string   | Person's name as filed.                                      |
| `directors[].position`                   | string   | Role text as filed, for example `Former Chair and CEO, Johnson & Johnson; Director`. Free text. |
| `directors[].age`                        | string   | Age. It is a **string**, not a number.                       |
| `directors[].directorClass`              | string   | Class on a staggered board, `I`, `II` or `III`. Empty for executives. |
| `directors[].dateFirstElected`           | string   | Free text. `2021`, `November 2017` and an empty string all appear. |
| `directors[].isIndependent`              | boolean  | `true`, `false` or `null`.                                   |
| `directors[].committeeMemberships`       | string[] | Committee names. A chair is marked inside the string, `People and Compensation Committee (Chair)`. |
| `directors[].qualificationsAndExperience` | string[] | Short phrases the company gives for that person.            |

Do not read `null` in `isIndependent` as "not independent". It means the parser
found no statement. Ten of the 12 people in the Apple row are `null`.

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

Trimmed response from the capture:

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
- Coverage is proxy statements. A company that files none has no rows.
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
