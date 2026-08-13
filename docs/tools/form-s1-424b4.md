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

Coverage starts 2000-01-04. The index holds the registration statements `S-1`,
`S-1/A`, `F-1`, `F-1/A`, `S-11` and `S-11/A`, plus the `424B4` prospectus. The
queries `formType:"424B3"` and `formType:"424B5"` both return zero rows, so
424B3 and 424B5 prospectuses are not in this index.

The registry description promises "use of proceeds" and a "risk factors
summary". No such field exists in any response. Treat that part of the
description as wrong. For prospectus prose, use [extractor](./extractor.md).

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

Query fields:

Every field of the extracted data is searchable. The fields used most are
`ticker`, `cik`, `entityName`, `formType`, `filedAt`, `accessionNo`,
`tickers.type`, `tickers.exchange`, `securities.name`,
`publicOfferingPrice.total`, `underwritingDiscount.total`,
`proceedsBeforeExpenses.total`, `underwriters.name`, `lawFirms.name`,
`lawFirms.location`, `auditors.name`, `management.name`, `management.age`,
`management.position` and `employees.total`. Range syntax works on the numeric
fields, for example `publicOfferingPrice.total:[100000000 TO *]`.

`entityName`, `underwriters.name` and `auditors.name` are analysed text fields.
A quoted phrase matches loosely, so check the rows you get back.

## Output

The envelope is `{total, data[]}`. `total` is an object, `{value, relation}`.
When `relation` is `gte` and `value` is `10000`, read it as "10,000 or more".
Elasticsearch stops counting there.

### Envelope

| Field            | Type   | Meaning                                                          |
| ---------------- | ------ | ---------------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                           |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many.     |
| `data[]`         | array  | The matching filings. One item per filing, up to 50 per request.   |

### Filing and issuer

| Field                    | Type    | Meaning                                                     |
| ------------------------ | ------- | ----------------------------------------------------------- |
| `data[].id`              | string  | System-internal identifier of the record.                    |
| `data[].accessionNo`     | string  | EDGAR accession number of the filing.                        |
| `data[].formType`        | string  | EDGAR form type. `S-1`, `S-1/A`, `F-1`, `F-1/A`, `S-11`, `S-11/A` or `424B4`. |
| `data[].filedAt`         | string  | Date and time EDGAR accepted the filing for processing. ISO 8601 with an offset. |
| `data[].cik`             | string  | Central Index Key of the issuer. Leading zeros are removed.  |
| `data[].ticker`          | string  | Trading symbol of the issuer when the filing was indexed. Empty when there is none. |
| `data[].entityName`      | string  | Name of the issuer.                                          |
| `data[].filingUrl`       | string  | URL of the filing document.                                  |

### Securities

| Field                     | Type    | Meaning                                                    |
| ------------------------- | ------- | ---------------------------------------------------------- |
| `data[].tickers[]`        | array   | One item per security the issuer offers, for example common stock, preferred stock, warrants or debt securities. |
| `data[].tickers[].ticker` | string  | Trading symbol of that security.                            |
| `data[].tickers[].type`   | string  | Type of that security, for example `Common Stock`.          |
| `data[].tickers[].exchange` | string | Exchange the security trades on, or is being listed on.    |
| `data[].securities[]`     | array   | One item per security the filing offers or refers to.       |
| `data[].securities[].name` | string | The security, with the number of shares, warrants or other units, for example `Up to 3,409,091 Shares of Common Stock`. |

### Offering economics

The three objects below share the same four keys.

| Field                                    | Type    | Meaning                                     |
| ---------------------------------------- | ------- | ------------------------------------------- |
| `data[].publicOfferingPrice`             | object  | The public offering price. Not every S-1 states it. A 424B4 does as a rule. |
| `data[].publicOfferingPrice.perShare`    | number  | Public offering price per share. It is US dollars in most filings. |
| `data[].publicOfferingPrice.perShareText` | string | The same price with its currency symbol, for example `$10.00`. Read it to learn the currency. |
| `data[].publicOfferingPrice.total`       | number  | Total public offering price.                 |
| `data[].publicOfferingPrice.totalText`   | string  | The same total with its currency symbol and thousands separators. |
| `data[].underwritingDiscount`            | object  | The underwriting discount, which is the banks' fee. Not every S-1 states it. A 424B4 does. |
| `data[].underwritingDiscount.perShare`   | number  | Underwriting discount per share.             |
| `data[].underwritingDiscount.perShareText` | string | The same amount with its currency symbol.  |
| `data[].underwritingDiscount.total`      | number  | Total underwriting discount.                 |
| `data[].underwritingDiscount.totalText`  | string  | The same total with its currency symbol and thousands separators. |
| `data[].proceedsBeforeExpenses`          | object  | The proceeds to the company before expenses, which is the offering price minus the discount. Not every S-1 states it. A 424B4 does. |
| `data[].proceedsBeforeExpenses.perShare` | number  | Proceeds before expenses per share.          |
| `data[].proceedsBeforeExpenses.perShareText` | string | The same amount with its currency symbol. |
| `data[].proceedsBeforeExpenses.total`    | number  | Total proceeds before expenses to the company. |
| `data[].proceedsBeforeExpenses.totalText` | string | The same total with its currency symbol and thousands separators. |

### Deal participants

| Field                          | Type    | Meaning                                               |
| ------------------------------ | ------- | ----------------------------------------------------- |
| `data[].underwriters[]`        | array   | One item per underwriter on the offering. The first item is the lead underwriter as a rule. A registration statement may name none, and a later S-1/A adds them as they become known. |
| `data[].underwriters[].name`   | string  | Name of the underwriter.                               |
| `data[].lawFirms[]`            | array   | One item per legal counsel or law firm on the offering. |
| `data[].lawFirms[].name`       | string  | Name of the law firm.                                  |
| `data[].lawFirms[].location`   | string  | Location of the law firm, for example `California, USA`. It can be empty. |
| `data[].auditors[]`            | array   | One item per auditor on the offering.                  |
| `data[].auditors[].name`       | string  | Name of the audit firm.                                |
| `data[].management[]`          | array   | One item per member of the management team the filing names. |
| `data[].management[].name`     | string  | Name of the person.                                    |
| `data[].management[].age`      | number  | Age of the person.                                     |
| `data[].management[].position` | string  | Position of the person at the company. The titles are not standardised, so the same job reads as `President`, `Chief Executive Officer` or `CEO`. |

### Employees

| Field                                   | Type           | Meaning                                |
| --------------------------------------- | -------------- | -------------------------------------- |
| `data[].employees`                      | object         | The headcount the filing states, in total, per business unit and per region. |
| `data[].employees.total`                | number         | Total number of employees.              |
| `data[].employees.asOfDate`             | string or null | The date the total headcount refers to. It is `null` when the filing gives no date. |
| `data[].employees.perDivision[]`        | array          | One item per business division. Empty when the filing gives no split. |
| `data[].employees.perDivision[].division` | string       | Name of the business division, for example `Research and Development`. |
| `data[].employees.perDivision[].employees` | number      | Number of employees in that division.   |
| `data[].employees.perRegion[]`          | array          | One item per geographical region. Empty when the filing gives no split. |
| `data[].employees.perRegion[].region`   | string         | Name of the region, for example `Europe` or `Los Angeles`. |
| `data[].employees.perRegion[].employees` | number        | Number of employees in that region.     |

The `*Text` twins carry the currency symbol. `perShare` and `total` are plain
numbers. Use the numbers for maths, the text for display.

Fields are omitted, not nulled, when the prospectus does not state them. An S-1
filed before pricing has no `publicOfferingPrice`.

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.

## Example

Prompt: "What price did Rivian price its IPO at, and who underwrote it?"

```json
{ "name": "form-s1-424b4", "arguments": { "query": "ticker:RIVN", "size": 1 } }
```

Response, trimmed for length:

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

The response above came from `query: "cik:*"`, which is why the row is SunScout,
not Rivian. `ticker:RIVN` returns 5 rows.

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
