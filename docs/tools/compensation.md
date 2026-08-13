# compensation

Search the annual pay of named executive officers and directors of US public
companies.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Governance and compensation                  |
| Required input  | `query`                                      |
| Returns         | a bare JSON array. No envelope, no `total`.  |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /compensation`                         |

## What it does

Public companies publish a summary compensation table in the proxy statement.
This tool searches the rows of those tables across all issuers.

One element of the array is one person in one fiscal year. A five-year history
for one executive is five elements. The row holds salary, bonus, stock and
option awards, non-equity incentive pay, pension change, other pay, and the
company total. Director pay appears where the company reports it.

## When to use it

- Who were the highest paid CEOs last year?
- How has this CFO's pay changed over five years?
- Which executives earned more than $50 million in 2025?
- What is the mix of salary and stock awards at this company?

## When to use a different tool

| Situation                                     | Better tool                                                       | Why                                                     |
| --------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------- |
| You want every year for one company           | [`compensation-by-key`](./compensation-by-key.md)                 | It returns up to 200 rows for one CIK or ticker.        |
| You want the board, not the pay               | [`directors-and-board-members`](./directors-and-board-members.md) | Committee seats and independence live there.            |
| You want what the auditor was paid            | [`audit-fees`](./audit-fees.md)                                   | Audit fees are a separate table in the same proxy.       |
| You want insider share sales                  | [`insider-trading`](./insider-trading.md)                         | Forms 3, 4 and 5 report trades, not pay.                |

## Input

| Parameter | Type    | Required | Constraints        | Notes                                                     |
| --------- | ------- | -------- | ------------------ | --------------------------------------------------------- |
| `query`   | string  | Yes      | Lucene syntax      | 3 to 1,000 characters. `AND`, `OR` and ranges work.       |
| `from`    | integer | No       | minimum 0          | Offset. Default 0.                                        |
| `size`    | integer | No       | 1 to 50            | Default 50. Above 50 the server returns HTTP 400.         |
| `sort`    | array   | No       | Elasticsearch sort | Default `[{"year": {"order": "desc"}}]`. Newest year first. |

Query fields verified live on 2026-08-13: `ticker`, `cik`, `name`, `position`,
`year`, `total`.

The company identifier is bare `ticker` here. On [`audit-fees`](./audit-fees.md)
the same idea is `entities.ticker`. See [query language](../query-language.md).

Ranges work on the money fields. `year:2025 AND total:[50000000 TO *]` returned
the highest paid executives of that year.

## Output

The response is a **bare JSON array**. There is no `{total, data[]}` wrapper.
Read `response[0]`, not `response.data[0]`. Eight tools behave this way:
`compensation`, [`compensation-by-key`](./compensation-by-key.md),
[`mapping`](./mapping.md) and five of the Form ADV schedule tools. See
[response format](../response-format.md).

There is no result count anywhere in the response. If you need one, page until
you get fewer rows than you asked for.

| Field                                     | Type   | Meaning                                                     |
| ----------------------------------------- | ------ | ----------------------------------------------------------- |
| `id`                                      | string | Record hash. It identifies one person-year row.             |
| `cik`                                     | string | Company CIK, no leading zeros.                              |
| `ticker`                                  | string | Company ticker.                                             |
| `name`                                    | string | Executive name as filed.                                    |
| `position`                                | string | Title as filed, for example `Senior Vice President, Chief Financial Officer`. Free text, not normalized. |
| `year`                                    | number | Fiscal year the pay belongs to.                             |
| `salary`                                  | number | Base salary in dollars.                                     |
| `bonus`                                   | number | Cash bonus.                                                 |
| `stockAwards`                             | number | Grant-date value of stock awards.                           |
| `optionAwards`                            | number | Grant-date value of option awards.                          |
| `nonEquityIncentiveCompensation`          | number | Non-equity incentive plan pay.                              |
| `changeInPensionValueAndDeferredEarnings` | number | Pension value change and deferred earnings.                 |
| `otherCompensation`                       | number | All other compensation.                                     |
| `total`                                   | number | Total pay for the year, as the company reported it.         |

Careful with `total`. It is a field on every row, not a result count. Zero is a
real value here, not missing data.

`size` defaults to 50 and caps at 50. Page with `from`. One row was 362 bytes,
so a `size: 50` call stays near 18 KB.

## Example

Prompt: "What did Apple's CFO earn?"

```json
{ "name": "compensation", "arguments": { "query": "ticker:AAPL", "size": 1 } }
```

The full response from the capture:

```json
[
  {
    "id": "ee1e721445cba2376db6912e7c52387d",
    "cik": "320193",
    "ticker": "AAPL",
    "name": "Kevan Parekh",
    "position": "Senior Vice President, Chief Financial Officer",
    "year": 2025,
    "salary": 891519,
    "bonus": 0,
    "stockAwards": 18433135,
    "optionAwards": 0,
    "nonEquityIncentiveCompensation": 3120317,
    "changeInPensionValueAndDeferredEarnings": 0,
    "otherCompensation": 22338,
    "total": 22467309
  }
]
```

## Limits and errors

- This tool needs an active sec-api.io subscription. Without one the server
  returns HTTP 403 and
  `Access denied. Endpoint requires an active subscription.`
- An empty `query` returns HTTP 400 and
  `Query too short. Minimum length: 3 characters`.
- A `query` longer than 1,000 characters returns
  `Query too long. Maximum length: 1000 characters`.
- `size` above 50 returns HTTP 400 with `Maximum 'size' limit of 50 exceeded.`
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`compensation-by-key`](./compensation-by-key.md). One company, up to 200 rows.
- [`directors-and-board-members`](./directors-and-board-members.md),
  [`audit-fees`](./audit-fees.md). The other proxy-statement datasets.
- [Query language](../query-language.md). Lucene syntax and field names.
- REST documentation:
  [Executive Compensation API](https://sec-api.io/docs/executive-compensation-api)
