# insider-trading

Search Form 3, 4 and 5 insider ownership and trading disclosures.

|                 |                                                                        |
| --------------- | ------------------------------------------------------------------------ |
| Category        | Ownership and insiders                                                  |
| Required input  | `query`                                                                 |
| Returns         | `{total, transactions[]}`. **Not `data[]`.** One item per filing.       |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                                         |
| REST equivalent | `POST /insider-trading`                                                 |

## What it does

Officers, directors and 10% owners must report their holdings and trades on
Form 3 (initial statement), Form 4 (change in ownership) and Form 5 (annual
statement). This tool searches all three.

One item in `transactions[]` is one filing, not one trade. The trades sit two
levels down, in `nonDerivativeTable.transactions[]` and
`derivativeTable.transactions[]`. The array name is misleading. Count filings at
the top level and trades inside each item.

The capture asked for `issuer.tradingSymbol:AAPL` with `size: 1`. It returned
`total.value: 1374` and one Form 4 from Apple's general counsel in 3,296 bytes.
That filing carried two non-derivative trades, one derivative trade and three
footnotes.

## When to use it

- Which insiders bought NVDA shares in the last month?
- Did an executive sell under a 10b5-1 plan or on their own decision?
- How many shares does a director own after the latest transaction?
- Which trades were tax withholding rather than open-market sales?

## When to use a different tool

| Situation                                     | Better tool                               | Why                                                                          |
| --------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------ |
| You want notice of a sale before it happens   | [`form-144`](./form-144.md)               | Form 144 is filed before the sale. Form 4 is filed within two business days after. |
| You want stakes above 5% of a company         | [`form-13d-13g`](./form-13d-13g.md)       | 13D and 13G cover beneficial owners crossing 5%, including funds.              |
| You want salary, bonus and equity awards      | [`compensation`](./compensation.md)       | Compensation data comes from the proxy statement, not from Section 16 forms.   |

## Input

| Parameter | Type    | Required | Constraints                            | Notes                                                     |
| --------- | ------- | -------- | -------------------------------------- | ----------------------------------------------------------- |
| `query`   | string  | yes      | must contain `:`, max 2,000 characters | Lucene syntax. See [query language](../query-language.md). The 2,000 limit is double the limit on the 13F and 13D tools. |
| `from`    | integer | no       | minimum 0                              | Offset into the result set. The handler sets no ceiling.    |
| `size`    | integer | no       | 1 to 50                                | Filings per call. **Defaults to 50** when you omit it.      |
| `sort`    | array   | no       | array of sort objects                  | Defaults to `[{"filedAt": {"order": "desc"}}]`.             |

The schema sets `additionalProperties: true`. The handler also reads
`time_zone`, a string that defaults to `America/New_York` and applies to date
ranges in the query. Query fields:

- `issuer.tradingSymbol`. Confirmed. The capture used
  `issuer.tradingSymbol:AAPL`. Note the shape. Other tools use bare `ticker` or
  `entities.ticker`. This one does not.
- `documentType`. Used in the Node SDK examples as `documentType:4`,
  **unverified** through MCP.
- `issuer.cik`, `issuer.name`, `reportingOwner.cik`, `reportingOwner.name`,
  `reportingOwner.relationship.isDirector`, `filedAt`, `periodOfReport`,
  `accessionNo`. All present in the response body, all **unverified** as query
  fields.

## Output

The envelope is `{total, transactions[]}`. Read `total.value` and
`total.relation`. The SDK responses show `{"value": 10000, "relation": "gte"}`
on broad queries. That is the search-window ceiling. Read it as 10,000 or more.

| Field                                   | Type    | Meaning                                                        |
| --------------------------------------- | ------- | ---------------------------------------------------------------- |
| `accessionNo`                           | string  | EDGAR accession number.                                           |
| `filedAt`, `periodOfReport`             | string  | Filing timestamp in ISO 8601, and the `YYYY-MM-DD` date of the earliest reported transaction. |
| `documentType`                          | string  | `3`, `4` or `5`.                                                  |
| `notSubjectToSection16`, `aff10b5One`   | boolean | Not a Section 16 insider, and traded under a Rule 10b5-1 plan. `aff10b5One` is on the Form 4 in the capture and absent from the SDK Form 3 and Form 5 responses. |
| `issuer`                                | object  | `cik`, `name`, `tradingSymbol` of the company.                    |
| `reportingOwner`                        | object  | `cik`, `name`, `address`, `relationship` of the insider.          |
| `reportingOwner.relationship`           | object  | `isDirector`, `isOfficer`, `officerTitle`, `isTenPercentOwner`, `isOther`. `officerTitle` only appears when `isOfficer` is true. |
| `nonDerivativeTable.transactions[]`     | array   | Trades in the stock itself.                                       |
| `nonDerivativeTable.holdings[]`         | array   | Positions held without a trade. This is the only table a Form 3 populates. |
| `derivativeTable.transactions[]`        | array   | Trades in options, RSUs and other derivatives.                    |
| `...securityTitle`, `...transactionDate` | string | The security traded, and the `YYYY-MM-DD` trade date.            |
| `...coding.code`                        | string  | Transaction code. Observed: `M` derivative settled, `F` shares withheld for tax, `S` open-market sale, `A` grant. The full SEC code table is larger. |
| `...amounts.shares`, `.acquiredDisposedCode` | number, string | Share count, which can be fractional such as `2264.5`, and `A` acquired or `D` disposed. |
| `...amounts.pricePerShare`              | number  | Price per share. **Optional.** When a footnote sets the price instead, the field is absent and `pricePerShareFootnoteId` appears. The first trade in the capture has no `pricePerShare`. |
| `...postTransactionAmounts.sharesOwnedFollowingTransaction` | number | Shares held after the trade.                  |
| `...ownershipNature.directOrIndirectOwnership` | string | `D` direct, `I` indirect. `natureOfOwnership` explains an `I`. |
| `...timeliness`                         | string  | Late-filing flag, for example `L`. Seen on a Form 5 in the SDK response, absent from the capture. |
| `footnotes[]`                           | array   | `id` and `text`. Any `*FootnoteId` field points here. Footnote text can run to several hundred words. `ownerSignatureName` and `ownerSignatureNameDate` close the record. |

Read the footnotes before you interpret a trade. In the capture, a 16,238 share
disposal at $296.42 looks like a sale. Footnote F2 says Apple withheld the
shares for tax on vesting RSUs and that no shares were sold. The `F` code says
the same thing.

Size behaviour: one Form 4 was 3,296 bytes, and most of that was footnote text.
`size` defaults to 50, so a call without `size` on an active issuer can return
well over 100 KB. Set `size` and filter by `documentType` and a date range.

## Example

Prompt: "Show me the most recent insider transaction at Apple."

```json
{ "name": "insider-trading", "arguments": { "query": "issuer.tradingSymbol:AAPL", "size": 1 } }
```

```json
{
  "total": { "value": 1374, "relation": "eq" },
  "transactions": [
    {
      "accessionNo": "0001140361-26-025622",
      "documentType": "4",
      "issuer": { "cik": "320193", "name": "Apple Inc.", "tradingSymbol": "AAPL" },
      "reportingOwner": {
        "name": "Newstead Jennifer",
        "relationship": { "isDirector": false, "isOfficer": true, "officerTitle": "SVP, GC and Secretary" }
      },
      "nonDerivativeTable": {
        "transactions": [
          {
            "securityTitle": "Common Stock",
            "transactionDate": "2026-06-15",
            "coding": { "formType": "4", "code": "F", "equitySwapInvolved": false },
            "amounts": { "shares": 16238, "pricePerShare": 296.42, "acquiredDisposedCode": "D" },
            "postTransactionAmounts": { "sharesOwnedFollowingTransaction": 41546 }
          }
        ]
      }
    }
  ]
}
```

Trimmed. The capture also holds a second non-derivative trade, one derivative
trade, three footnotes and the signature block.

## Limits and errors

- The array is called `transactions` but holds filings. This is the most common
  mistake with this tool.
- A price of `0` on a grant or a settlement is real, not missing data. An absent
  `pricePerShare` is also normal. Check the matching footnote.
- A missing `query`, or a query without `:`, fails with HTTP 400 `Invalid
  query`. A query over 2,000 characters fails with `Query too long. Maximum
  length: 2000 characters`. `size` above 50 fails with `Maximum 'size' limit of
  50 exceeded.` These texts come from the server handler, not from the capture.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-144`](./form-144.md)
- [`form-13d-13g`](./form-13d-13g.md)
- [`directors-and-board-members`](./directors-and-board-members.md)
- REST docs: [Insider Trading API](https://sec-api.io/docs/insider-ownership-trading-api)
