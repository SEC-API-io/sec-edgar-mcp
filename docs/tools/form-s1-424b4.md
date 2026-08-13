# form-s1-424b4

Search S-1, S-1/A, F-1 and 424B4 registration filings and get the offering terms
as structured data.

|                 |                                                          |
| --------------- | -------------------------------------------------------- |
| Category        | Offerings and registrations                              |
| Required input  | `query`                                                  |
| Returns         | `{total, data[]}`                                        |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`             |
| REST equivalent | `POST /form-s1-424b4`                                    |

## What it does

The tool searches an index of IPO and follow-on registration documents. One row
is one filing. The server reads the prospectus and returns the deal terms as
fields: price per share, gross proceeds, underwriting discount, underwriters,
law firms, auditors, named management and headcount.

Coverage starts 2000-01-04. A 50-row sample of the newest filings held these
form types: `S-1`, `S-1/A`, `424B4`, `F-1`, `F-1/A` and `S-11`. The queries
`formType:"424B3"` and `formType:"424B5"` both return zero rows, so 424B3 and
424B5 prospectuses are not in this index.

The registry description promises "use of proceeds" and a "risk factors
summary". No such field exists in the capture or in any of the 50 sampled rows.
Treat that part of the description as wrong. For prospectus prose, use
[extractor](./extractor.md).

## When to use it

- What price did this company price its IPO at, and how much did it raise?
- Which banks underwrote the IPOs that priced last month?
- Which law firm and which audit firm worked on this offering?
- Who runs the company, and how old are they, at the time of the IPO?
- How many staff did the issuer report in its prospectus?

## When to use a different tool

| Situation                                     | Better tool                            | Why                                                     |
| --------------------------------------------- | -------------------------------------- | ------------------------------------------------------- |
| You want the list of S-1 filings and their URLs | [filing-search](./filing-search.md)  | Covers every form type and every filing, not just deals |
| You want the risk factors or use-of-proceeds text | [extractor](./extractor.md)        | Returns the prospectus sections as text                 |
| The offering is a Regulation A offering       | [reg-a-form-1a](./reg-a-form-1a.md)    | Reg A offerings file Form 1-A, not S-1                  |
| The offering is a private placement           | [form-d](./form-d.md)                  | Reg D offerings file Form D                             |

## Input

| Parameter | Type    | Required | Constraints        | Notes                                             |
| --------- | ------- | -------- | ------------------ | ------------------------------------------------- |
| `query`   | string  | yes      | must contain a `:` | Lucene syntax. Max 1000 characters.               |
| `from`    | integer | no       | 0 or more          | Offset. `from` above 10000 returns HTTP 400.      |
| `size`    | integer | no       | 1 to 50            | Default 50. Above 50 returns HTTP 400.            |
| `sort`    | array   | no       | ES sort clause     | Default `[{"filedAt":{"order":"desc"}}]`.         |

The colon rule is strict. A bare word such as `apple` returns HTTP 400 with
`Invalid request parameter provided.`

Query fields confirmed to return rows:

`ticker`, `cik`, `entityName`, `formType`, `filedAt`, `accessionNo`,
`tickers.exchange`, `underwriters.name`, `auditors.name`,
`publicOfferingPrice.total`. Range syntax works on the numeric fields, for
example `publicOfferingPrice.total:[100000000 TO *]`.

`entityName`, `underwriters.name` and `auditors.name` are analysed text fields.
A quoted phrase matches loosely, so check the rows you get back.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`.
When `relation` is `gte` and `value` is `10000`, read it as "10,000 or more".
Elasticsearch stops counting there.

| Field                    | Type    | Meaning                                                     |
| ------------------------ | ------- | ----------------------------------------------------------- |
| `id`                     | string  | Internal document ID                                        |
| `accessionNo`            | string  | EDGAR accession number of the filing                        |
| `formType`               | string  | `S-1`, `S-1/A`, `424B4`, `F-1`, `F-1/A`, `S-11`             |
| `filedAt`                | string  | Filing timestamp with offset                                |
| `cik`                    | string  | Issuer CIK, no leading zeros                                |
| `ticker`                 | string  | Primary ticker, empty when the issuer has none              |
| `entityName`             | string  | Issuer name as filed                                        |
| `filingUrl`              | string  | Link to the document on sec.gov                             |
| `tickers[]`              | array   | `ticker`, `type` (share class), `exchange`                  |
| `securities[]`           | array   | `name`, the share counts and classes named in the prospectus |
| `publicOfferingPrice`    | object  | `perShare`, `perShareText`, `total`, `totalText`            |
| `underwritingDiscount`   | object  | Same four keys. The banks' fee.                             |
| `proceedsBeforeExpenses` | object  | Same four keys. Net to the issuer before expenses.          |
| `underwriters[]`         | array   | `name`                                                      |
| `lawFirms[]`             | array   | `name`, `location`                                          |
| `auditors[]`             | array   | `name`                                                      |
| `management[]`           | array   | `name`, `age`, `position`                                   |
| `employees`              | object  | `total`, `asOfDate`, `perDivision[]`, `perRegion[]`         |

The `*Text` twins carry the currency symbol. `perShare` and `total` are plain
numbers. Use the numbers for maths, the text for display.

Fields are omitted, not nulled, when the prospectus does not state them. An S-1
filed before pricing has no `publicOfferingPrice`.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.

## Example

Prompt: "What price did Rivian price its IPO at, and who underwrote it?"

```json
{ "name": "form-s1-424b4", "arguments": { "query": "ticker:RIVN", "size": 1 } }
```

Response from the capture, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "id": "936bcafb9b9d9697b90862e305f9c03f",
      "filedAt": "2026-08-12T21:32:28-04:00",
      "accessionNo": "0001213900-26-088617",
      "formType": "424B4",
      "cik": "2101240",
      "ticker": "SNSC",
      "entityName": "SunScout Holding Ltd",
      "filingUrl": "https://www.sec.gov/Archives/edgar/data/2101240/000121390026088617/ea0268485-18.htm",
      "tickers": [{ "ticker": "SNSC", "type": "Class A Ordinary Shares", "exchange": "NYSE American" }],
      "publicOfferingPrice": { "perShare": 5, "perShareText": "US$5.00", "total": 15500000, "totalText": "US$15,500,000" },
      "underwritingDiscount": { "perShare": 0.35, "perShareText": "US$0.35", "total": 1085000, "totalText": "US$1,085,000" },
      "proceedsBeforeExpenses": { "perShare": 4.65, "perShareText": "US$4.65", "total": 14415000, "totalText": "US$14,415,000" },
      "underwriters": [{ "name": "Dominari Securities LLC" }, { "name": "Revere Securities LLC" }],
      "auditors": [{ "name": "Fruci & Associates II, PLLC" }],
      "management": [{ "name": "Mr. Edwin Cywinski", "age": 66, "position": "Chief Executive Officer, Chairman and Executive Director" }]
    }
  ]
}
```

The capture used `query: "cik:*"`, which is why the row is SunScout, not Rivian.
`ticker:RIVN` returns 5 rows.

## Limits and errors

- No colon in `query` gives HTTP 400 `Invalid request parameter provided.`
- `query` longer than 1000 characters gives the same 400.
- `from` above 10000 gives the same 400.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [filing-search](./filing-search.md), [extractor](./extractor.md)
- [form-d](./form-d.md), [form-c](./form-c.md), [reg-a-form-1a](./reg-a-form-1a.md)
- REST docs: <https://sec-api.io/docs/form-s1-424b4-data-search-api>
