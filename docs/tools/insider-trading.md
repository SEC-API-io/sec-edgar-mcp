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
statement). This tool searches all three and their amendments, `3/A`, `4/A` and
`5/A`. Coverage runs from 2003 to present.

One item in `transactions[]` is one filing, not one trade. The trades sit two
levels down, in `nonDerivativeTable.transactions[]` and
`derivativeTable.transactions[]`. The array name is misleading. The top level
counts filings. The nested arrays count trades.

A request for `issuer.tradingSymbol:AAPL` with `size: 1` returned
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
| `from`    | integer | no       | 0 to 10000                             | Offset into the result set. One query reaches 10,000 filings. |
| `size`    | integer | no       | 1 to 50                                | Filings per call. **Defaults to 50** when you omit it.      |
| `sort`    | array   | no       | array of sort objects                  | Defaults to `[{"filedAt": {"order": "desc"}}]`.             |

Query fields:

- `issuer.tradingSymbol`. The example uses `issuer.tradingSymbol:AAPL`. Other
  tools use bare `ticker` or `entities.ticker`. This one does not.
- `documentType`, for example `documentType:4`.
- `issuer.cik`, `issuer.name`, `reportingOwner.cik`, `reportingOwner.name`,
  `reportingOwner.relationship.isDirector`,
  `reportingOwner.relationship.isOfficer`,
  `reportingOwner.relationship.isTenPercentOwner`,
  `reportingOwner.relationship.isOther`, `filedAt`, `periodOfReport`,
  `accessionNo`. All present in the response body.
- The trade level is searchable too:
  `nonDerivativeTable.transactions.coding.code`,
  `derivativeTable.transactions.coding.code`,
  `derivativeTable.transactions.coding.equitySwapInvolved` and
  `nonDerivativeTable.transactions.timeliness`.
- `aff10b5One`, `remarks`, `footnotes.id` and `footnotes.text`.

## Output

The envelope is `{total, transactions[]}`. `total` holds `value` and
`relation`. Broad queries return `{"value": 10000, "relation": "gte"}`.
That is the search-window ceiling. It means 10,000 or more.

| Field            | Type     | Meaning                                                                              |
| ---------------- | -------- | ------------------------------------------------------------------------------------ |
| `total.value`    | integer  | Number of filings that match the query. It stops at 10000.                           |
| `total.relation` | string   | `eq` for an exact count. `gte` when the count hit the 10000 ceiling.                 |
| `transactions[]` | object[] | The matched filings, at most 50 per response. One item is one filing, not one trade. |

One item in `transactions[]` is one Form 3, 4 or 5 filing. The paths below
are relative to a `transactions[]` item. A field is absent when the form
leaves it blank.

### Filing identity

| Field                      | Type    | Meaning                                                                                                                                                                                     |
| -------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                       | string  | System-internal identifier of the filing record.                                                                                                                                            |
| `accessionNo`              | string  | Accession number that EDGAR gave to the original filing.                                                                                                                                    |
| `filedAt`                  | string  | Time when EDGAR accepted the filing, in ISO 8601.                                                                                                                                           |
| `documentType`             | string  | Form type. `3`, `3/A`, `4`, `4/A`, `5` or `5/A`.                                                                                                                                            |
| `periodOfReport`           | string  | `YYYY-MM-DD`. On Form 3 the date of the event that made the statement necessary. On Form 4 the date of the earliest transaction in the filing. On Form 5 the fiscal year end of the issuer. |
| `dateOfOriginalSubmission` | string  | `YYYY-MM-DD` date of the filing that this one amends. Only on `3/A`, `4/A` and `5/A`.                                                                                                       |
| `schemaVersion`            | string  | Version tag of the SEC ownership XML document, such as `X0609`. Older filings carry lower tags.                                                                                             |
| `notSubjectToSection16`    | boolean | True when the filer checks the box that says the person is no longer subject to Section 16.                                                                                                 |
| `aff10b5One`               | boolean | True when the filer affirms that the trades were made under a Rule 10b5-1(c) trading plan.                                                                                                  |

### `issuer`

The company whose securities the insider reports.

| Field                  | Type   | Meaning                                                                     |
| ---------------------- | ------ | --------------------------------------------------------------------------- |
| `issuer.cik`           | string | CIK of the company whose securities were traded. Leading zeros are removed. |
| `issuer.name`          | string | Name of that company.                                                       |
| `issuer.tradingSymbol` | string | Ticker of the traded security.                                              |

### `reportingOwner`

The insider who files the form. It can be a person or a company.

| Field                                           | Type    | Meaning                                                                      |
| ----------------------------------------------- | ------- | ---------------------------------------------------------------------------- |
| `reportingOwner.cik`                            | string  | CIK of the insider who files the form. Leading zeros are removed.            |
| `reportingOwner.name`                           | string  | Name of the insider. A person appears surname first.                         |
| `reportingOwner.address.street1`                | string  | First line of the address the insider gives. It is often care of the issuer. |
| `reportingOwner.address.street2`                | string  | Second line of that address.                                                 |
| `reportingOwner.address.city`                   | string  | City of that address.                                                        |
| `reportingOwner.address.state`                  | string  | Two-letter code of the state. Absent on many addresses outside the US.       |
| `reportingOwner.address.stateDescription`       | string  | Full name of the state or the country.                                       |
| `reportingOwner.address.zipCode`                | string  | Postal code.                                                                 |
| `reportingOwner.relationship.isDirector`        | boolean | True when the insider sits on the board of the issuer.                       |
| `reportingOwner.relationship.isOfficer`         | boolean | True when the insider is an officer of the issuer.                           |
| `reportingOwner.relationship.officerTitle`      | string  | Job title of the officer. Present when `isOfficer` is true.                  |
| `reportingOwner.relationship.isTenPercentOwner` | boolean | True when the insider owns 10% or more of the issuer.                        |
| `reportingOwner.relationship.isOther`           | boolean | True when the insider has another relationship to the issuer.                |
| `reportingOwner.relationship.otherText`         | string  | Text that names that relationship. Present when `isOther` is true.           |

### `nonDerivativeTable`

Table I of the form. It covers the stock itself. `transactions[]` holds
trades. `holdings[]` holds positions that the insider reports without a
trade. A Form 3 fills only `holdings[]`.

Trades:

| Field                                                                                                | Type     | Meaning                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `nonDerivativeTable.transactions[].securityTitle`                                                    | string   | Name of the security class that was traded, such as `Common Stock`.                                                                                                                                                                                                                      |
| `nonDerivativeTable.transactions[].securityTitleFootnoteId`                                          | string[] | Footnote IDs for `securityTitle`. Every field that ends in `FootnoteId` points into `footnotes[]`.                                                                                                                                                                                       |
| `nonDerivativeTable.transactions[].transactionDate`                                                  | string   | `YYYY-MM-DD` date the trade took place.                                                                                                                                                                                                                                                  |
| `nonDerivativeTable.transactions[].deemedExecutionDate`                                              | string   | `YYYY-MM-DD` date the SEC treats as the execution date. It appears when it differs from `transactionDate`.                                                                                                                                                                               |
| `nonDerivativeTable.transactions[].coding.formType`                                                  | string   | Form that carries this line. A Form 5 can carry a line coded `4` when it reports a Form 4 trade late.                                                                                                                                                                                    |
| `nonDerivativeTable.transactions[].coding.code`                                                      | string   | SEC transaction code. `P` purchase, `S` sale, `A` grant or award, `F` securities delivered or withheld to pay the exercise price or the tax, `M` settlement of an exempt derivative, `X` exercise of a derivative, `C` conversion, `G` gift, `J` other. The full table holds 18 letters. |
| `nonDerivativeTable.transactions[].coding.equitySwapInvolved`                                        | boolean  | True when an equity swap or a similar instrument is part of the trade.                                                                                                                                                                                                                   |
| `nonDerivativeTable.transactions[].coding.footnoteId`                                                | string[] | Footnote IDs for the coding block.                                                                                                                                                                                                                                                       |
| `nonDerivativeTable.transactions[].timeliness`                                                       | string   | `E` early, `L` late. Empty or absent when the report was on time.                                                                                                                                                                                                                        |
| `nonDerivativeTable.transactions[].amounts.shares`                                                   | number   | Number of shares acquired or disposed. It can be fractional, such as `2264.5`.                                                                                                                                                                                                           |
| `nonDerivativeTable.transactions[].amounts.pricePerShare`                                            | number   | Price of one share in the trade. A `0` on a grant or a settlement is a real price.                                                                                                                                                                                                       |
| `nonDerivativeTable.transactions[].amounts.pricePerShareFootnoteId`                                  | string[] | Footnote IDs for the price. They appear when a footnote carries the price in place of the field.                                                                                                                                                                                         |
| `nonDerivativeTable.transactions[].amounts.acquiredDisposedCode`                                     | string   | `A` when the insider got the securities, `D` when the insider gave them up.                                                                                                                                                                                                              |
| `nonDerivativeTable.transactions[].postTransactionAmounts.sharesOwnedFollowingTransaction`           | number   | Shares of this class the insider owns after the trade.                                                                                                                                                                                                                                   |
| `nonDerivativeTable.transactions[].postTransactionAmounts.sharesOwnedFollowingTransactionFootnoteId` | string[] | Footnote IDs for that share count.                                                                                                                                                                                                                                                       |
| `nonDerivativeTable.transactions[].postTransactionAmounts.valueOwnedFollowingTransaction`            | number   | Value the insider owns after the trade. Filers use it in place of a share count.                                                                                                                                                                                                         |
| `nonDerivativeTable.transactions[].ownershipNature.directOrIndirectOwnership`                        | string   | `D` when the insider holds the securities in their own name. `I` when another party holds them for the insider.                                                                                                                                                                          |
| `nonDerivativeTable.transactions[].ownershipNature.natureOfOwnership`                                | string   | Text that names the indirect holder, such as a trust, a partnership or a spouse. It goes with an `I`.                                                                                                                                                                                    |

Holdings:

| Field                                                                                  | Type     | Meaning                                                                                                          |
| -------------------------------------------------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------- |
| `nonDerivativeTable.holdings[].securityTitle`                                          | string   | Name of the security class the insider holds.                                                                    |
| `nonDerivativeTable.holdings[].coding.formType`                                        | string   | Form that carries this line. The `coding` object is often empty on a Form 3, because a holding reports no trade. |
| `nonDerivativeTable.holdings[].coding.footnoteId`                                      | string[] | Footnote IDs for the coding block.                                                                               |
| `nonDerivativeTable.holdings[].postTransactionAmounts.sharesOwnedFollowingTransaction` | number   | Shares the insider owns. On a Form 3 it is the holding on the day the person became an insider.                  |
| `nonDerivativeTable.holdings[].postTransactionAmounts.valueOwnedFollowingTransaction`  | number   | Value the insider owns, when the filer gives a value in place of a share count.                                  |
| `nonDerivativeTable.holdings[].ownershipNature.directOrIndirectOwnership`              | string   | `D` direct, `I` indirect.                                                                                        |
| `nonDerivativeTable.holdings[].ownershipNature.natureOfOwnership`                      | string   | Text that names the indirect holder.                                                                             |
| `nonDerivativeTable.holdings[].ownershipNature.natureOfOwnershipFootnoteId`            | string[] | Footnote IDs for that text.                                                                                      |

### `derivativeTable`

Table II of the form. It covers options, restricted stock units, warrants
and other securities that take their value from the stock. The two arrays
split the same way as in Table I.

Trades:

| Field                                                                                   | Type     | Meaning                                                                                                                       |
| --------------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `derivativeTable.transactions[].securityTitle`                                          | string   | Name of the derivative, such as `Restricted Stock Unit` or `Employee Stock Option (right to buy)`.                            |
| `derivativeTable.transactions[].conversionOrExercisePrice`                              | number   | Price per share the insider pays to turn the derivative into stock. It is `0` for a restricted stock unit.                    |
| `derivativeTable.transactions[].conversionOrExercisePriceFootnoteId`                    | string[] | Footnote IDs for that price.                                                                                                  |
| `derivativeTable.transactions[].transactionDate`                                        | string   | `YYYY-MM-DD` date the trade took place.                                                                                       |
| `derivativeTable.transactions[].deemedExecutionDate`                                    | string   | `YYYY-MM-DD` date the SEC treats as the execution date.                                                                       |
| `derivativeTable.transactions[].coding.formType`                                        | string   | Form that carries this line.                                                                                                  |
| `derivativeTable.transactions[].coding.code`                                            | string   | SEC transaction code. The letters match Table I. `M` and `X` are common here, because they settle or exercise the derivative. |
| `derivativeTable.transactions[].coding.equitySwapInvolved`                              | boolean  | True when an equity swap or a similar instrument is part of the trade.                                                        |
| `derivativeTable.transactions[].coding.footnoteId`                                      | string[] | Footnote IDs for the coding block.                                                                                            |
| `derivativeTable.transactions[].timeliness`                                             | string   | `E` early, `L` late. Empty or absent when the report was on time.                                                             |
| `derivativeTable.transactions[].amounts.shares`                                         | number   | Number of derivative securities acquired or disposed.                                                                         |
| `derivativeTable.transactions[].amounts.pricePerShare`                                  | number   | Price of one derivative security. It is not the price of the underlying share.                                                |
| `derivativeTable.transactions[].amounts.pricePerShareFootnoteId`                        | string[] | Footnote IDs for that price.                                                                                                  |
| `derivativeTable.transactions[].amounts.acquiredDisposedCode`                           | string   | `A` acquired, `D` disposed.                                                                                                   |
| `derivativeTable.transactions[].exerciseDate`                                           | string   | `YYYY-MM-DD` first day the insider can exercise the derivative.                                                               |
| `derivativeTable.transactions[].exerciseDateFootnoteId`                                 | string[] | Footnote IDs for that date. They replace the date when a vesting schedule sets it.                                            |
| `derivativeTable.transactions[].expirationDate`                                         | string   | `YYYY-MM-DD` last day the derivative can be exercised.                                                                        |
| `derivativeTable.transactions[].expirationDateFootnoteId`                               | string[] | Footnote IDs for that date.                                                                                                   |
| `derivativeTable.transactions[].underlyingSecurity.title`                               | string   | Name of the stock the derivative converts into.                                                                               |
| `derivativeTable.transactions[].underlyingSecurity.shares`                              | number   | Number of shares of that stock the derivative covers.                                                                         |
| `derivativeTable.transactions[].underlyingSecurity.value`                               | number   | Value of those shares, when the filer gives a value in place of a count.                                                      |
| `derivativeTable.transactions[].postTransactionAmounts.sharesOwnedFollowingTransaction` | number   | Derivative securities of this class the insider owns after the trade.                                                         |
| `derivativeTable.transactions[].postTransactionAmounts.valueOwnedFollowingTransaction`  | number   | Value the insider owns after the trade, in place of a count.                                                                  |
| `derivativeTable.transactions[].ownershipNature.directOrIndirectOwnership`              | string   | `D` direct, `I` indirect.                                                                                                     |
| `derivativeTable.transactions[].ownershipNature.natureOfOwnership`                      | string   | Text that names the indirect holder.                                                                                          |

Holdings:

| Field                                                                               | Type     | Meaning                                                         |
| ----------------------------------------------------------------------------------- | -------- | --------------------------------------------------------------- |
| `derivativeTable.holdings[].securityTitle`                                          | string   | Name of the derivative the insider holds.                       |
| `derivativeTable.holdings[].conversionOrExercisePrice`                              | number   | Price per share to turn the derivative into stock.              |
| `derivativeTable.holdings[].coding.formType`                                        | string   | Form that carries this line.                                    |
| `derivativeTable.holdings[].coding.code`                                            | string   | SEC transaction code, when the filer sets one on a holding.     |
| `derivativeTable.holdings[].coding.equitySwapInvolved`                              | boolean  | True when an equity swap is involved.                           |
| `derivativeTable.holdings[].coding.footnoteId`                                      | string[] | Footnote IDs for the coding block.                              |
| `derivativeTable.holdings[].exerciseDate`                                           | string   | `YYYY-MM-DD` first day the insider can exercise the derivative. |
| `derivativeTable.holdings[].expirationDate`                                         | string   | `YYYY-MM-DD` last day the derivative can be exercised.          |
| `derivativeTable.holdings[].underlyingSecurity.title`                               | string   | Name of the stock the derivative converts into.                 |
| `derivativeTable.holdings[].underlyingSecurity.shares`                              | number   | Number of shares of that stock the derivative covers.           |
| `derivativeTable.holdings[].underlyingSecurity.value`                               | number   | Value of those shares, in place of a count.                     |
| `derivativeTable.holdings[].postTransactionAmounts.sharesOwnedFollowingTransaction` | number   | Derivative securities of this class the insider owns.           |
| `derivativeTable.holdings[].postTransactionAmounts.valueOwnedFollowingTransaction`  | number   | Value the insider owns, in place of a count.                    |
| `derivativeTable.holdings[].ownershipNature.directOrIndirectOwnership`              | string   | `D` direct, `I` indirect.                                       |
| `derivativeTable.holdings[].ownershipNature.natureOfOwnership`                      | string   | Text that names the indirect holder.                            |

### Footnotes and signature

| Field                    | Type   | Meaning                                                                                           |
| ------------------------ | ------ | ------------------------------------------------------------------------------------------------- |
| `footnotes[].id`         | string | Label of the footnote, `F1`, `F2` and so on. Every `*FootnoteId` value names one of these labels. |
| `footnotes[].text`       | string | The footnote text. It can run to several hundred words, and it can change what a line means.      |
| `remarks`                | array  | Remarks that the filer writes under the tables.                                                   |
| `ownerSignatureName`     | string | Name on the signature line. An attorney-in-fact often signs for the insider.                      |
| `ownerSignatureNameDate` | string | `YYYY-MM-DD` date of the signature. It can precede `filedAt`.                                     |

A footnote changes what a trade means. In the example response, a
16,238 share disposal at $296.42 looks like a sale. Footnote F2 says Apple
withheld the shares for tax on vesting RSUs and that no shares were sold. The
`F` code says the same thing.

Size behaviour: one Form 4 was 3,296 bytes, and most of that was footnote text.
`size` defaults to 50, so a call without `size` on an active issuer can return
well over 100 KB. `size`, `documentType` and a date range each narrow the
response.

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

Trimmed. The full response also holds a second non-derivative trade, one
derivative trade, three footnotes and the signature block.

## Limits and errors

- The array is called `transactions` but holds filings. This is the most common
  mistake with this tool.
- A price of `0` on a grant or a settlement is real, not missing data. An absent
  `pricePerShare` is also normal. `pricePerShareFootnoteId` points to the
  footnote that holds the price.
- A missing `query`, or a query without `:`, fails with HTTP 400 `Invalid
  query`. A query over 2,000 characters fails with `Query too long. Maximum
  length: 2000 characters`. `size` above 50 fails with `Maximum 'size' limit of
  50 exceeded.`
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-144`](./form-144.md)
- [`form-13d-13g`](./form-13d-13g.md)
- [`directors-and-board-members`](./directors-and-board-members.md)
- REST docs: [Insider Trading API](https://sec-api.io/docs/insider-ownership-trading-api)
