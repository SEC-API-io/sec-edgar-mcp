# compensation-by-key

Get the executive pay history of one company by its CIK or ticker.

|                 |                                                    |
| --------------- | -------------------------------------------------- |
| Category        | Governance and compensation                        |
| Required input  | `cikOrTicker`                                      |
| Returns         | a bare JSON array. No envelope, no `total`.        |
| Pagination      | **None.** No `from`, `size` or `sort`. 200 rows max. |
| REST equivalent | `GET /compensation/{cikOrTicker}`                  |

## What it does

Public companies publish a summary compensation table in the proxy statement.
This tool returns every row sec-api holds for one company, newest fiscal year
first. One element of the array is one person in one fiscal year. The data comes
from DEF 14A filings from 2005 to today. Records remain after a company
delists or stops reporting to the SEC.

A request for NVIDIA returns 110 rows in 38,842 bytes. They cover nine
people across fiscal years 2007 to 2026, about five named executive officers per
year. The server caps the answer at 200 rows.

Use this tool when you know the company. Use
[`compensation`](./compensation.md) when you want to search across companies.

## When to use it

- What has NVIDIA paid its executives over the years?
- How has this CEO's pay changed since 2007?
- Which executives does this company name in its proxy statement?
- What share of pay is salary, and what share is stock?
- Who replaced the former CFO, and what did each of them earn?

## When to use a different tool

| Situation                                  | Better tool                                                       | Why                                                          |
| ------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------ |
| You want to rank or filter across companies | [`compensation`](./compensation.md)                              | It takes a Lucene query and pages with `from` and `size`.     |
| You have a company name, not an identifier | [`mapping`](./mapping.md)                                         | This tool consumes an identifier. It does not resolve one.    |
| You want the board and its committees      | [`directors-and-board-members`](./directors-and-board-members.md) | Seats and independence live there, not in the pay table.      |
| You want share sales by the same people    | [`insider-trading`](./insider-trading.md)                         | Forms 3, 4 and 5 report trades, not pay.                      |

## Input

| Parameter     | Type   | Required | Constraints                    | Notes                                                          |
| ------------- | ------ | -------- | ------------------------------ | -------------------------------------------------------------- |
| `cikOrTicker` | string | Yes      | a numeric CIK, or a ticker     | `1045810` and `NVDA` both work. The example uses `NVDA`.       |

The server reads an all-digit value as a CIK and anything else as a ticker. It
upper-cases the ticker for you, so `nvda` works. Send a CIK without leading
zeros. The stored `cik` has none, so `0001045810` matches nothing.

Some clients still show an older description for this tool. It names the input
`key` and calls it "an executive ID, CIK, or company slug". That text is out of
date. The parameter is `cikOrTicker`, and it takes a company, not a person.

There is no `query` here. There is no Lucene syntax.

## Output

The response is a **bare JSON array**. There is no `{total, data[]}` wrapper.
Read `response[0]`, not `response.data[0]`. See
[response format](../response-format.md).

There is no result count anywhere in the response. Count the array yourself.

Every field of a row is listed below. A row has 14 fields and no nested object.

| Field                                     | Type   | Meaning                                                   |
| ----------------------------------------- | ------ | --------------------------------------------------------- |
| `id`                                      | string | Record hash of 32 hexadecimal characters. It identifies one person-year row. |
| `cik`                                     | string | CIK of the reporting company, no leading zeros.           |
| `ticker`                                  | string | Ticker of the reporting company.                          |
| `name`                                    | string | Executive name as filed. Spelling follows the proxy, so `Jen-Hsun Huang` appears rather than `Jensen Huang`. |
| `position`                                | string | Position of the executive, for example `EVP and CFO`. Free text, not normalized. It changes as the person is promoted. |
| `year`                                    | number | Year of the reporting period the pay belongs to.          |
| `salary`                                  | number | Salary of the executive for the reporting year, in dollars. |
| `bonus`                                   | number | Cash bonus. It is the discretionary part of the cash pay. |
| `stockAwards`                             | number | Stock awards, at grant-date value.                        |
| `optionAwards`                            | number | Option awards, at grant-date value.                       |
| `nonEquityIncentiveCompensation`          | number | Non-equity incentive plan compensation. A formula sets it, and the company pays it in cash. |
| `changeInPensionValueAndDeferredEarnings` | number | Change in pension value and nonqualified deferred compensation earnings. |
| `otherCompensation`                       | number | All other compensation. Perquisites, severance and benefits sit here. |
| `total`                                   | number | Total compensation for the year. It sums the pay components above. |

`total` is a field on every row, not a result count. Zero is a real value here,
not missing data.

**This tool has no pagination.** The server pins the query to the first 200
rows, sorted by `year` descending, and ignores anything else you send. NVIDIA
returns 110 rows, below the cap. A company with a long history and many
officers can hit it. Switch to [`compensation`](./compensation.md) and page
with `from` when it does.

Rows are small. One row is about 350 bytes, so 110 rows cost 39 KB.

## Example

Prompt: "Show me NVIDIA's executive pay history."

```json
{ "name": "compensation-by-key", "arguments": { "cikOrTicker": "NVDA" } }
```

Two of the 110 rows:

```json
[
  {
    "id": "0455e2636dcce08c81a373e6b3a321c7",
    "cik": "1045810",
    "ticker": "NVDA",
    "name": "Jen-Hsun Huang",
    "position": "President and CEO",
    "year": 2026,
    "salary": 1497627,
    "bonus": 0,
    "stockAwards": 24800511,
    "optionAwards": 0,
    "nonEquityIncentiveCompensation": 6000000,
    "changeInPensionValueAndDeferredEarnings": 0,
    "otherCompensation": 4045691,
    "total": 36343830
  },
  { "id": "f3fa7813fde10d7ebd774084b009a7af", "ticker": "NVDA", "name": "Colette M. Kress", "position": "EVP and CFO", "year": 2026, "total": 14340850 }
]
```

## Limits and errors

- The rows follow the fiscal year, not the calendar year. NVIDIA's fiscal 2026
  ended in January 2026.
- A person's `position` is free text and changes over the years. Group a career
  by `name`, not by `position`.
- An identifier the server does not hold returns an empty array, not an error.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`compensation`](./compensation.md). The search version of this dataset.
- [`directors-and-board-members`](./directors-and-board-members.md),
  [`audit-fees`](./audit-fees.md). The other proxy-statement datasets.
- [`mapping`](./mapping.md). Turn a company name into the ticker used here.
- REST documentation:
  [Executive Compensation API](https://sec-api.io/docs/executive-compensation-api)
