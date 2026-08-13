# form-13f-holdings

Search the equity positions that institutional investment managers report on
Form 13F.

|                 |                                                                                   |
| --------------- | --------------------------------------------------------------------------------- |
| Category        | Ownership and insiders                                                            |
| Required input  | `query`                                                                           |
| Returns         | `{total, data[]}`. One item per 13F filing, with a nested `holdings[]` array.      |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                                                   |
| REST equivalent | `POST /form-13f/holdings`                                                         |

## What it does

Form 13F is the quarterly report of an institutional investment manager that
holds more than $100 million in US-listed equities. This tool searches the
holdings index built from those filings. One item in `data[]` is one filing, not
one position. The positions sit in the `holdings[]` array inside each item.

A request for `cik:1067983` (Berkshire Hathaway) with `size: 1` returned
`total.value: 210` and one 13F-HR filing for the period ending
2026-03-31. That single filing carried **90 holdings and 28,316 bytes**. The
registry description says the tool returns "individual holding rows". The
response contradicts this. Read `data[i].holdings` to reach the positions.

## When to use it

- What did Berkshire Hathaway hold at the end of the last quarter?
- How many shares of one CUSIP does a manager hold, and at what market value?
- Does the manager have sole or shared voting power over a position?
- How did one manager's position in one stock change over four quarters?

## When to use a different tool

| Situation                                        | Better tool                                            | Why                                                                             |
| ------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------- |
| You want one row per filing and the total value  | [`form-13f-cover-pages`](./form-13f-cover-pages.md)    | Cover pages carry `tableValueTotal` and the manager's address, with no position list. |
| You track stakes above 5% of a company           | [`form-13d-13g`](./form-13d-13g.md)                    | 13D and 13G report beneficial ownership as a percent of the class.               |
| You want mutual fund or ETF holdings             | [`form-nport`](./form-nport.md)                        | N-PORT covers registered funds and reports monthly. 13F covers managers, quarterly. |
| You only want the filing metadata or the raw XML | [`filing-search`](./filing-search.md)                  | `filing-search` returns links without the position payload.                      |

## Input

| Parameter | Type    | Required | Constraints                          | Notes                                                        |
| --------- | ------- | -------- | ------------------------------------ | ------------------------------------------------------------ |
| `query`   | string  | yes      | must contain `:`, max 1,000 characters | Lucene syntax. See [query language](../query-language.md).  |
| `from`    | integer | no       | 0 to 10000                           | Offset into the result set.                                   |
| `size`    | integer | no       | 1 to 50                              | Number of **filings**, not positions. **Defaults to 50** when you omit it. |
| `sort`    | array   | no       | array of sort objects                | Defaults to `[{"filedAt": {"order": "desc"}}]`.               |

**The server rewrites your query.** If the query string does not contain the
text `13F`, the server prepends `formType:"13F-HR" AND ` to it. The query
`cik:1067983` runs as `formType:"13F-HR" AND cik:1067983`. Put `13F` in your own
query to keep control, for example when you want amendments. This tool searches
the general filings index, the same one [`filing-search`](./filing-search.md)
uses.

Query fields: `cik`, `ticker`, `companyName`, `formType`, `accessionNo`,
`periodOfReport`, `filedAt`, `holdings.ticker`, `holdings.cusip` and
`holdings.nameOfIssuer`. All are present in the response body. The example uses
`cik:1067983`.

## Output

The envelope is `{total, data[]}`. `total` is an object, not a number. Read
`total.value`. `total.relation` is `"eq"` for an exact count, `"gte"` at 10000
for 10,000 or more.

| Field                                | Type   | Meaning                                                       |
| ------------------------------------ | ------ | ------------------------------------------------------------- |
| `accessionNo`                        | string | EDGAR accession number. Joins to `form-13f-cover-pages`.       |
| `cik`, `ticker`, `companyName`       | string | The filing manager, not the held company.                      |
| `formType`                           | string | `13F-HR`, for example.                                         |
| `filedAt`                            | string | ISO 8601 with offset.                                          |
| `periodOfReport`                     | string | Quarter end the holdings describe, `YYYY-MM-DD`.               |
| `effectivenessDate`                  | string | Date the filing became effective.                              |
| `linkToTxt`, `linkToHtml`, `linkToFilingDetails`, `documentFormatFiles[]` | string, array | EDGAR URLs. `linkToXbrl` is empty for 13F. |
| `entities[]`                         | array  | EDGAR header data. `sic` holds HTML entities such as `&amp;`.  |
| `holdings[]`                         | array  | The positions. One item per line of the information table.     |
| `holdings[].nameOfIssuer`, `.cusip`, `.titleOfClass` | string | The held security.                             |
| `holdings[].value`                   | number | Market value in US dollars. `value / sshPrnamt` gives a plausible per-share price in this response. |
| `holdings[].shrsOrPrnAmt`            | object | `sshPrnamt` is the amount held. `sshPrnamtType` is `SH`, for example. |
| `holdings[].investmentDiscretion`    | string | `DFND`, for example. The form defines other codes. |
| `holdings[].votingAuthority`         | object | `Sole`, `Shared` and `None` share counts. Note the capital keys. |
| `holdings[].otherManager`            | string | Comma-separated sequence numbers of other managers, such as `"2,4,11"`. Resolve them through `form-13f-cover-pages`. |
| `holdings[].ticker`, `.cik`          | string | The held company, added by sec-api. Not in the raw filing.     |

The same issuer can appear several times in `holdings[]`. Berkshire reports
`ALLY FINL INC` four times, once per `otherManager` group. Sum the lines before
you report a position.

Size behaviour: `size` counts filings, and **it defaults to 50**. One filing can
hold hundreds of positions. The Berkshire filing returned 90 positions in 28 KB.
A Bridgewater cover page reports 1,040 entries, so that filing returns far more.
**Always set `size` on this tool. Start with `size: 1`.** A default call can
fill a context window in one shot.

## Example

Prompt: "What were Berkshire Hathaway's largest holdings in its most recent 13F?"

```json
{ "name": "form-13f-holdings", "arguments": { "query": "cik:1067983", "size": 1 } }
```

```json
{
  "total": { "value": 210, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001193125-26-226661",
      "cik": "1067983",
      "formType": "13F-HR",
      "filedAt": "2026-05-15T16:06:05-04:00",
      "periodOfReport": "2026-03-31",
      "holdings": [
        {
          "nameOfIssuer": "ALLY FINL INC",
          "cusip": "02005N100",
          "value": 498992850,
          "shrsOrPrnAmt": { "sshPrnamt": 12719675, "sshPrnamtType": "SH" },
          "investmentDiscretion": "DFND",
          "votingAuthority": { "Sole": 12719675, "Shared": 0, "None": 0 },
          "otherManager": "4",
          "ticker": "ALLY",
          "cik": "40729"
        }
      ]
    }
  ]
}
```

Trimmed. The full response holds 89 more `holdings[]` entries, plus `entities[]`,
`documentFormatFiles[]` and the link fields.

## Limits and errors

- Response size is the real risk here, not errors. See the size warning above.
- A missing `query`, a query without `:`, a query over 1,000 characters, or
  `from` above 10000 all fail with HTTP 400 `Invalid request parameter
  provided.` One message covers four causes, so check all four.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- Holdings are reported quarterly, up to 45 days after the quarter ends. The
  filing above covers the quarter ending 2026-03-31 and was filed 2026-05-15.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-13f-cover-pages`](./form-13f-cover-pages.md)
- [`form-13d-13g`](./form-13d-13g.md)
- [`form-nport`](./form-nport.md)
- REST docs: [Form 13F API](https://sec-api.io/docs/form-13-f-filings-institutional-holdings-api)
