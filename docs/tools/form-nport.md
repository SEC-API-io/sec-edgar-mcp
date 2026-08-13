# form-nport

Search Form N-PORT filings for the monthly portfolio holdings of U.S. mutual
funds, ETFs and closed-end funds.

|                 |                                                                    |
| --------------- | ------------------------------------------------------------------ |
| Category        | Funds                                                              |
| Required input  | `query`                                                            |
| Returns         | `{total, filings[]}`                                               |
| Pagination      | `from`, `size`, `sort`. The schema allows `size` up to 50. The server caps this tool at 10. See [Limits and errors](#limits-and-errors). |
| REST equivalent | `POST /form-nport`                                                 |

## What it does

The tool searches parsed N-PORT filings. Funds file N-PORT every quarter and
report three months of data in each filing. One item in `filings[]` is one
filing for one fund series. It carries the fund identity in `genInfo`, the
fund-level totals and flows in `fundInfo`, and the complete position list in
`invstOrSecs`. The sec-api SDK coverage table gives the period as 2019 to
present, for form types NPORT and NPORT/A. The capture returned
`total: {value: 10000, relation: "gte"}`. Read that as "10,000 or more".

Each filing is large. The captured filing holds 70 positions, and the raw
response was 45 KB for `size: 1`. Funds with thousands of positions return more.

## When to use it

- What securities did a fund hold at the end of the last reporting period?
- Which funds hold a given CUSIP or ISIN?
- How big is the fund, and what were its monthly flows?
- What percentage of the portfolio is one position?
- Is a position restricted, in default, or lent out?

## When to use a different tool

| Situation                                        | Better tool                                     | Why                                                                     |
| ------------------------------------------------ | ----------------------------------------------- | ----------------------------------------------------------------------- |
| You want equity positions of an institution      | [`form-13f-holdings`](./form-13f-holdings.md)   | 13F covers 13F-eligible equities of managers, not fund portfolios.       |
| You want the fund's annual census and structure  | [`form-ncen`](./form-ncen.md)                   | N-CEN reports service providers and structure, not holdings.          |
| You want how a fund voted its shares             | [`form-npx`](./form-npx.md)                     | N-PX carries proxy votes. N-PORT carries positions.                      |

## Input

| Parameter | Type    | Required | Constraints                        | Notes                                                        |
| --------- | ------- | -------- | ---------------------------------- | ------------------------------------------------------------ |
| `query`   | string  | Yes      | Must contain a colon. Maximum 2,000 characters. | Lucene syntax, for example `genInfo.regName:*`. |
| `from`    | integer | No       | Minimum 0                          | Offset of the first result. Default 0.                        |
| `size`    | integer | No       | Schema 1 to 50. Server maximum 10. | Number of filings, not holdings. Default 10.                  |
| `sort`    | array   | No       | Elasticsearch sort array           | Default `[{"filedAt": {"order": "desc"}}]`.                   |

Query fields confirmed to return rows:

- `genInfo.regName`

Fields taken from the SDK examples and the response shape, all **unverified**:
`fundInfo.totAssets` with range syntax such as `[100000000 TO *]`,
`genInfo.seriesName`, `genInfo.regCik`, `invstOrSecs.cusip`, `accessionNo`,
`filedAt`. See [query language](../query-language.md).

## Output

The envelope is `{total, filings[]}`. It is **not** `data[]`. `total` is an
object, `{value, relation}`. A `relation` of `gte` means the count is capped.

| Field                              | Type   | Meaning                                                     |
| ---------------------------------- | ------ | ----------------------------------------------------------- |
| `submissionType`                   | string | Form type. `NPORT-P` in the capture.                        |
| `genInfo.regName`                  | string | Registrant, that is the trust or fund company.              |
| `genInfo.regCik`                   | string | Registrant CIK, zero padded to 10 digits.                   |
| `genInfo.regFileNumber`            | string | Investment Company Act file number, for example `811-23928`.|
| `genInfo.seriesName`               | string | Name of the fund series this filing covers.                 |
| `genInfo.repPdEnd`                 | string | End of the fiscal period, `YYYY-MM-DD`.                     |
| `genInfo.repPdDate`                | string | Date the holdings are measured on.                          |
| `fundInfo.totAssets`, `.netAssets` | number | Total and net assets in USD. `totLiabs` holds liabilities.  |
| `fundInfo.returnInfo`              | object | Monthly total returns per share class, and realized gains.  |
| `fundInfo.mon1Flow`                | object | Month 1 `sales`, `redemption`, `reinvestment`. Also `mon2Flow`, `mon3Flow`. |
| `fundInfo.curMetrics`              | object | Interest rate risk per currency, DV01 and DV100, by tenor.  |
| `invstOrSecs[].name`, `.title`     | string | Issuer name and security title. One entry per position.     |
| `invstOrSecs[].cusip`              | string | CUSIP. `identifiers.isin.value` holds the ISIN.             |
| `invstOrSecs[].balance`, `.units`  | number, string | Quantity and unit. `PA` is principal amount, `NS` is number of shares. |
| `invstOrSecs[].valUSD`             | number | Fair value in USD.                                          |
| `invstOrSecs[].pctVal`             | number | Percent of net assets. The 70 captured positions sum to 97.6.|
| `invstOrSecs[].assetCat`           | string | Asset category. The capture holds `ABS-MBS`, `STIV`, `LON`, `EC`. |
| `invstOrSecs[].fairValLevel`       | string | Fair value hierarchy, `1`, `2` or `3`.                      |
| `invstOrSecs[].debtSec`            | object | Maturity, coupon kind, rate, default flags. Present on 68 of the 70 captured positions. Equity positions have no `debtSec`. |

`size` counts filings, not positions. Every returned filing carries its whole
`invstOrSecs` array, so raise `size` with care. `filerInfo` held only the filer
CIK in the capture. The SDK example also shows `filerInfo.seriesClassInfo` and a
top-level `explntrNotes`. Neither appeared in the capture.

The JSON arrives as one stringified text block. See
[response format](../response-format.md).

## Example

Prompt: "Show me the newest N-PORT filing from any fund, with its holdings."

```json
{ "name": "form-nport", "arguments": { "query": "genInfo.regName:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "filings": [
    {
      "submissionType": "NPORT-P",
      "genInfo": {
        "regName": "Catalyst/Perini Strategic Income Fund",
        "repPdEnd": "2026-09-30",
        "repPdDate": "2026-06-30"
      },
      "fundInfo": { "totAssets": 52966273.23, "netAssets": 52833180.21 },
      "invstOrSecs": [
        {
          "name": "Banc of America Mortgage Secur",
          "cusip": "05955BAG4",
          "balance": 20223128.61,
          "valUSD": 252789.11,
          "pctVal": 0.4784665791,
          "assetCat": "ABS-MBS"
        }
      ],
      "accessionNo": "0000894189-26-022150"
    }
  ]
}
```

Keys were removed to fit. The values are unchanged.

## Limits and errors

- A `query` without a colon fails with HTTP 400 and `Invalid query`.
- A `query` longer than 2,000 characters fails with
  `Query too long. Maximum length: 2000 characters`.
- The server caps `size` at **10** for N-PORT, while the tool schema advertises
  50. A `size` of 11 to 50 passes the schema, then fails with
  `Maximum 'size' limit of 10 exceeded`. The cap is set in the API handler, which
  also defaults `size` to 10. The capture used `size: 1`, so it never triggered
  the error. Keep `size` at 10 or less.
- Responses are heavy. Budget about 45 KB per filing for a 70-position fund.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-ncen`](./form-ncen.md), [`form-npx`](./form-npx.md),
  [`form-npx-file`](./form-npx-file.md)
- [`form-13f-holdings`](./form-13f-holdings.md)
- REST documentation: [Form N-PORT API](https://sec-api.io/docs/n-port-data-api)
