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
covers listed and delisted companies, and some institutional filers. Use it as
the first step of most workflows, because other tools need a ticker or a CIK.

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
  Anchor the value for one exact hit: `name=^APPLE INC$` returned 1.
- No match returns an empty array, `[]`, not an error.

The server also accepts `param: "sic"`. Verified live: `sic=3571` returned 43
companies. The schema leaves `sic` out of the enum, so a client that validates
against the schema may block the call.

## Output

The response is a **bare JSON array**. There is no `{total, data[]}` wrapper
here. Read `response[0]`, not `response.data[0]`. Seven other tools do the same:
[`compensation`](./compensation.md),
[`compensation-by-key`](./compensation-by-key.md) and five of the Form ADV
schedule tools. See [response format](../response-format.md).

| Field                       | Type    | Meaning                                                           |
| --------------------------- | ------- | ----------------------------------------------------------------- |
| `name`, `ticker`            | string  | Company name in upper case, `APPLE INC`, and its trading symbol.  |
| `cik`                       | string  | CIK, no leading zeros. Pass this to the other tools.              |
| `cusip`                     | string  | CUSIP of the main security. Empty for filers that have none.      |
| `exchange`                  | string  | `NASDAQ`, `NYSE`, `NYSEMKT` and similar.                          |
| `isDelisted`                | boolean | `true` if the security no longer trades.                          |
| `category`                  | string  | `Domestic Common Stock`, `Institutional Investor` and similar.    |
| `sector`, `industry`        | string  | Market classification, `Technology`, `Consumer Electronics`.      |
| `sic`, `sicSector`, `sicIndustry` | string | SEC classification, `3571`, `Manufacturing`, `Electronic Computers`. |
| `famaIndustry`              | string  | Fama-French industry, `Computers`.                                |
| `currency`, `location`      | string  | `USD`, `California; U.S.A`.                                       |
| `id`                        | string  | Internal record ID.                                               |

Unknown values come back as empty strings. An investment manager, for example,
returns empty `cusip`, `exchange`, `sector`, `industry` and `sic`. The canonical
REST response also has `famaSector`, which the Apple capture does not. Treat
that one field as optional.

**This tool has no pagination.** It returns every match in one text block, and
you cannot ask for fewer. Measured sizes:

| Call                        | Records | Bytes     |
| --------------------------- | ------- | --------- |
| `ticker` = `AAPL`           | 1       | 396       |
| `name` = `Apple`            | 12      | 4,706     |
| `sic` = `3571`              | 43      | 17,550    |
| `sector` = `Technology`     | 3,950   | 1,701,747 |
| `exchange` = `NASDAQ`       | 16,234  | 6,913,084 |

The `exchange` call returns 6.9 MB in one response, the largest payload of any
tool in this server. Never call `mapping` with `exchange`, `sector` or
`industry` from an agent that keeps the result in its context.

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
  not in this list. Use [`edgar-entities`](./edgar-entities.md) for those.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`edgar-entities`](./edgar-entities.md) for the full EDGAR filer profile.
- [`float`](./float.md) and [`subsidiaries`](./subsidiaries.md) both take the
  ticker or CIK this tool returns.
- Envelope comparison across tools: [response format](../response-format.md)
- REST documentation: [CUSIP/CIK/Ticker Mapping API](https://sec-api.io/docs/mapping-api)
