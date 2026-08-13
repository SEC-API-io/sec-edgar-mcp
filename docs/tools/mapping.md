# mapping

Resolve one company identifier into the full identifier set: CIK, ticker, CUSIP
and name.

|                 |                                                     |
| --------------- | ----------------------------------------------------- |
| Category        | Company and entity                                    |
| Required input  | `param`, `value`                                      |
| Returns         | A bare JSON array. No envelope, no `total`.           |
| Pagination      | **None.** No `from`, `size` or `sort`.            |
| REST equivalent | `GET /mapping/{param}/{value}`                        |

## What it does

You give the tool one identifier type and one value. It returns every company
that matches, each with its full identifier set plus sector, industry and
listing data. One element of the array is one company, not one filing. The list
covers listed and delisted companies on US exchanges: ADRs, common and
preferred stock, warrants, ETFs, ETNs and ETDs. It also holds some
institutional filers. The database is updated daily. Most workflows start here,
because the other tools need a ticker or a CIK.

## When to use it

- What is the CIK of Apple?
- Which company has CUSIP 037833100?
- Is this ticker still listed, or was it delisted?
- What sector, industry and SIC code does this company belong to?

## When to use a different tool

| Situation                                    | Better tool                            | Why                                                                            |
| -------------------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------- |
| You need addresses, auditor or filer status  | [`edgar-entities`](./edgar-entities.md) | `mapping` returns identifiers and classification only.                          |
| You want to screen a whole exchange or sector | [`edgar-entities`](./edgar-entities.md) | `mapping` has no paging and returns megabytes for a broad `param`. See below.   |
| You want the filings of the company          | [`filing-search`](./filing-search.md)  | `mapping` never returns filings.                                                |

## Input

| Parameter | Type   | Required | Constraints                                                        | Notes                                                     |
| --------- | ------ | -------- | ------------------------------------------------------------------ | --------------------------------------------------------- |
| `param`   | string | yes      | enum: `cik`, `ticker`, `cusip`, `name`, `exchange`, `sector`, `industry` | The identifier type to look up by.                  |
| `value`   | string | yes      | max 400 characters                                                 | The value to match.                                       |

How matching works:

- `cik` matches exactly, after leading zeros are stripped. `0000320193` and
  `320193` both return Apple.
- Every other `param` treats `value` as a case-insensitive regular expression.
  `name=Apple` returned 12 companies, one of them `APPLE ORTHODONTIX INC`.
  An anchored value matches one name: `name=^APPLE INC$` returned 1.
- One CIK can return several rows, one per security: share classes, warrants
  and the like.
- One ticker can also return several rows, because a symbol can be reused after
  a delisting. Read `isDelisted` to pick the security that still trades.
- No match returns an empty array, `[]`, not an error.

The server also accepts `param: "sic"`. `sic=3571` returns 43 companies. The
schema leaves `sic` out of the enum, so a client that validates against the
schema may block the call.

## Output

The response is a **bare JSON array**. There is no `{total, data[]}` wrapper
here. The first record is `response[0]`, not `response.data[0]`. Seven other
tools do the same: [`compensation`](./compensation.md),
[`compensation-by-key`](./compensation-by-key.md) and five of the Form ADV
schedule tools. See [response format](../response-format.md).

| Field         | Type    | Meaning                                                                 |
| ------------- | ------- | ------------------------------------------------------------------------- |
| `name`        | string  | Name of the company, in upper case, `APPLE INC`.                        |
| `ticker`      | string  | Trading symbol of the company, `AAPL`.                                  |
| `cik`         | string  | CIK of the company, no leading zeros. The other tools take this value.  |
| `cusip`       | string  | One or more CUSIPs linked to the company, separated by a space. Empty for filers that have none. |
| `exchange`    | string  | The main exchange the company is listed on, `NASDAQ`, `NYSE`, `NYSEMKT`. |
| `isDelisted`  | boolean | `true` if the company is no longer listed, `false` if it still trades.  |
| `category`    | string  | The security category, `Domestic Common Stock`, `Institutional Investor` and similar. |
| `sector`      | string  | Market sector of the company, `Technology`.                             |
| `industry`    | string  | Market industry of the company, `Consumer Electronics`. It holds one of 175 values. |
| `sic`         | string  | Four-digit SEC Standard Industrial Classification code, `3571`.         |
| `sicSector`   | string  | Name of the SIC sector, `Manufacturing`.                                |
| `sicIndustry` | string  | Name of the SIC industry, `Electronic Computers`.                       |
| `famaSector`  | string  | Name of the Fama-French sector. An empty string on most records.        |
| `famaIndustry` | string | Name of the Fama-French industry, `Computers`, `Automobiles and Trucks`. |
| `currency`    | string  | Operating currency of the company, `USD`.                               |
| `location`    | string  | Where the company has its headquarters, `California; U.S.A`.            |
| `id`          | string  | Unique internal ID of the company record.                               |

Unknown values come back as empty strings. An investment manager, for example,
returns empty `cusip`, `exchange`, `sector`, `industry` and `sic`. `famaSector`
is optional. Some records leave it out, and the Apple example below is one of
them.

**This tool has no pagination.** It returns every match in one text block, and
you cannot ask for fewer. Measured sizes:

| Call                        | Records | Bytes     |
| --------------------------- | ------- | --------- |
| `ticker` = `AAPL`           | 1       | 396       |
| `name` = `Apple`            | 12      | 4,706     |
| `sic` = `3571`              | 43      | 17,550    |
| `sector` = `Technology`     | 3,950   | 1,701,747 |
| `exchange` = `NASDAQ`       | 16,234  | 6,913,084 |

The `exchange` call returns 6.9 MB in one response. `sector` and `industry`
return comparable volumes. An agent that keeps the result in its context holds
all of it.

## Example

Prompt: "What is Apple's CIK and CUSIP?"

```json
{ "name": "mapping", "arguments": { "param": "ticker", "value": "AAPL" } }
```

```json
[
  {
    "name": "APPLE INC",
    "ticker": "AAPL",
    "cik": "320193",
    "cusip": "037833100",
    "exchange": "NASDAQ",
    "isDelisted": false,
    "category": "Domestic Common Stock",
    "sector": "Technology",
    "industry": "Consumer Electronics",
    "sic": "3571",
    "sicSector": "Manufacturing",
    "sicIndustry": "Electronic Computers",
    "famaIndustry": "Computers",
    "currency": "USD",
    "location": "California; U.S.A",
    "id": "442d0fd86e9d78fe79d539ee1b7fc0e4"
  }
]
```

## Limits and errors

- An unsupported `param` returns
  `sec-api error: Invalid parameter. Valid parameters are: cik, cusip, ticker, name, exchange, sic, sector, industry.`
- A `value` longer than 400 characters is rejected.
- Short values are expensive, because the match is a regular expression.
  `name=In` hits every company with those two letters anywhere in its name.
- Coverage is company reference data. Individuals who file Forms 3, 4 and 5 are
  not in this list. [`edgar-entities`](./edgar-entities.md) covers those.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`edgar-entities`](./edgar-entities.md) for the full EDGAR filer profile.
- [`float`](./float.md) and [`subsidiaries`](./subsidiaries.md) both take the
  ticker or CIK this tool returns.
- Envelope comparison across tools: [response format](../response-format.md)
- REST documentation: [CUSIP/CIK/Ticker Mapping API](https://sec-api.io/docs/mapping-api)
