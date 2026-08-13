# form-144

Search Form 144 notices, the intent-to-sell filings that insiders submit before
they sell restricted stock.

|                 |                                                  |
| --------------- | -------------------------------------------------- |
| Category        | Ownership and insiders                            |
| Required input  | `query`                                           |
| Returns         | `{total, data[]}`. One item per Form 144 notice.  |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                   |
| REST equivalent | `POST /form-144`                                  |

## What it does

Rule 144 requires an affiliate to file a notice before selling restricted or
control securities above a threshold. The notice states how many shares the
person plans to sell, through which broker, on roughly which date, and what they
already sold in the past three months. This tool searches those notices.

One item in `data[]` is one Form 144 notice. It is a plan, not a completed
trade. The sale can be smaller than stated, or never happen.

The capture asked for `issuerInfo.issuerTicker:TSLA` with `size: 1`. It returned
`total.value: 75` and one notice from a Tesla officer in 2,066 bytes. He planned
to sell 2,606 shares worth $1,048,124.08.

## When to use it

- Which Tesla insiders filed notice to sell this month?
- How many shares does an executive plan to sell, and at what market value?
- What did this insider already sell in the past three months?
- Were the shares acquired from options, RSUs or an open-market purchase?

## When to use a different tool

| Situation                                   | Better tool                                 | Why                                                                        |
| ------------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------- |
| You want the sale that actually happened    | [`insider-trading`](./insider-trading.md)   | Form 4 reports the executed trade with the real price. Form 144 is only intent. |
| You want the full board and officer roster  | [`directors-and-board-members`](./directors-and-board-members.md) | Only insiders who sell file a Form 144. |
| You want stakes above 5% of a company       | [`form-13d-13g`](./form-13d-13g.md)         | 13D and 13G report beneficial ownership, not planned sales.                  |

## Input

| Parameter | Type    | Required | Constraints           | Notes                                                     |
| --------- | ------- | -------- | --------------------- | ----------------------------------------------------------- |
| `query`   | string  | yes      | none                  | Lucene syntax. See [query language](../query-language.md). The handler does not check the query at all. |
| `from`    | integer | no       | minimum 0             | Offset into the result set. The handler sets no ceiling.    |
| `size`    | integer | no       | 1 to 50               | Notices per call. **Defaults to 50** when you omit it.      |
| `sort`    | array   | no       | array of sort objects | Defaults to `[{"filedAt": {"order": "desc"}}]`.             |

The schema sets `additionalProperties: true`. The handler also reads
`time_zone`, a string that is passed to the search engine. Unlike its siblings,
this handler applies no default time zone. Query fields:

- `issuerInfo.issuerTicker`. Confirmed. The capture used
  `issuerInfo.issuerTicker:TSLA`.
- `entities.ticker`. Used in the Node SDK example, **unverified** through MCP.
  Both spellings appear in the response, which is why the confusion exists.
  Prefer the confirmed one.
- `issuerInfo.issuerCik`, `issuerInfo.issuerName`,
  `issuerInfo.relationshipsToIssuer`, `formType`, `filedAt`, `accessionNo`,
  `securitiesInformation.approxSaleDate`. All present in the response body, all
  **unverified** as query fields.

## Output

The envelope is `{total, data[]}`. Read `total.value` and `total.relation`.
`"eq"` is an exact count. `"gte"` at 10000 means 10,000 or more.

| Field                                          | Type    | Meaning                                                  |
| ---------------------------------------------- | ------- | ---------------------------------------------------------- |
| `accessionNo`, `fileNo`                        | string  | EDGAR accession number, and the SEC file number of the issuer. |
| `formType`, `filedAt`                          | string  | `144`, and the filing timestamp in ISO 8601.                |
| `entities[]`                                   | array   | EDGAR header rows. The issuer is tagged `(Subject)`, the insider `(Reporting)`. Only the issuer row carries `ticker`. `sic` holds HTML entities such as `&amp;`. |
| `issuerInfo.issuerCik`, `.issuerTicker`, `.issuerName` | string | The company whose stock is being sold. `.issuerAddress` and `.issuerContactPhone` hold its contact block. |
| `issuerInfo.nameOfPersonForWhoseAccountTheSecuritiesAreToBeSold` | string | The selling insider.               |
| `issuerInfo.relationshipsToIssuer`             | string  | `Officer` in the capture, `Director` in the SDK response.   |
| `securitiesInformation[]`                      | array   | The planned sale. One item per class of security.           |
| `...numberOfUnitsToBeSold`                     | number  | Shares the insider plans to sell.                           |
| `...aggregateMarketValue`                      | number  | Market value of the planned sale in US dollars.             |
| `...noOfUnitsOutstanding`                      | number  | Shares outstanding, for context.                            |
| `...approxSaleDate`                            | string  | Approximate sale date, `YYYY-MM-DD`. It is an estimate.     |
| `...securitiesClassTitle`, `.securitiesExchangeName` | string | `Common` and `NASDAQ` in the capture.                 |
| `...brokerOrMarketMakerDetails`                | object  | `name` and `address` of the executing broker.               |
| `securitiesToBeSold[]`                         | array   | How the shares were acquired. One item per acquisition lot. |
| `...natureOfAcquisitionTransaction`            | string  | `Restricted Stock` in the capture. `Exercise of Stock Options` and `Previously Exercised Stock Options` in the SDK response. |
| `...acquiredDate`, `.paymentDate`, `.natureOfPayment` | string | Acquisition and payment detail.                    |
| `...amountOfSecuritiesAcquired`, `.isGiftTransaction` | number, boolean | Shares in the lot, and whether they came as a gift. |
| `nothingToReportFlagOnSecuritiesSoldInPast3Months` | boolean | `true` means the array below is empty by declaration.  |
| `securitiesSoldInPast3Months[]`                | array   | Prior sales. Each holds `sellerDetails`, `securitiesClassTitle`, `saleDate`, `amountOfSecuritiesSold` and `grossProceeds`. |
| `noticeSignature`                              | object  | `noticeDate` and `signature`. Also `planAdoptionDates[]`, the Rule 10b5-1 plan dates, which are **absent in the capture** and present in the canonical SDK response. Treat that field as optional. |

`securitiesSoldInPast3Months[].grossProceeds` is a completed dollar amount. It
is the only realised number on the record. Everything in
`securitiesInformation[]` is a plan.

Size behaviour: one notice was 2,066 bytes. Records are small and evenly sized.
`size` defaults to 50, and 50 notices are safe to hold in context.

## Example

Prompt: "Which Tesla insiders recently filed notice to sell shares?"

```json
{ "name": "form-144", "arguments": { "query": "issuerInfo.issuerTicker:TSLA", "size": 1 } }
```

```json
{
  "total": { "value": 75, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001950047-26-005795",
      "formType": "144",
      "filedAt": "2026-06-08T17:30:09-04:00",
      "issuerInfo": {
        "issuerCik": "1318605",
        "issuerTicker": "TSLA",
        "nameOfPersonForWhoseAccountTheSecuritiesAreToBeSold": "VAIBHAV TANEJA",
        "relationshipsToIssuer": "Officer"
      },
      "securitiesInformation": [
        {
          "numberOfUnitsToBeSold": 2606,
          "aggregateMarketValue": 1048124.08,
          "approxSaleDate": "2026-06-08",
          "securitiesExchangeName": "NASDAQ"
        }
      ],
      "noticeSignature": { "noticeDate": "2026-06-08", "signature": "/s/ Vaibhav Taneja" }
    }
  ]
}
```

Trimmed. The capture also holds `entities[]`, the broker block,
`securitiesToBeSold[]` and one prior sale of 3,000 shares for $1,350,000 gross.

## Limits and errors

- A Form 144 is an intention. Never report it as a completed sale. Confirm the
  outcome with [`insider-trading`](./insider-trading.md).
- `approxSaleDate` is approximate by design and often equals the filing date.
- The ticker field is `issuerInfo.issuerTicker` here. On other tools it is
  `ticker`, `entities.ticker` or `issuer.tradingSymbol`.
- The handler validates only `size`. A value above 50 fails with `Maximum 'size'
  limit of 50 exceeded.` A malformed query is passed straight to the search
  engine, so the failure mode for a bad query is not verified. Test your query
  with `size: 1` first.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`insider-trading`](./insider-trading.md)
- [`form-13d-13g`](./form-13d-13g.md)
- [`directors-and-board-members`](./directors-and-board-members.md)
- REST docs: [Form 144 API](https://sec-api.io/docs/form-144-restricted-sales-api)
