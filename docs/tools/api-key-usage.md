# api-key-usage

Get the monthly bandwidth used by your sec-api API key.

|                 |                                                                    |
| --------------- | ------------------------------------------------------------------ |
| Category        | Account                                                            |
| Required input  | `date`                                                             |
| Returns         | `{date, monthlyBandwidthUsedInMb}`. A scalar object, no array.    |
| Pagination      | **None.** No `from`, `size` or `sort`.                                                               |
| REST equivalent | `GET /api-key-usage?date=YYYY-MM`                                  |

## What it does

This tool reports how much data your API key downloaded in one calendar month.
It is the only account tool in the server. It reads sec-api's own usage counter,
not SEC EDGAR data, so no company, filing, or form is involved.

One call covers exactly one month. The response holds two values: the month you
asked for, and the bandwidth used in megabytes. The value counts bytes the API
served to your key. It does not count requests, and it does not report a plan
limit, a quota, or a price.

## When to use it

- How much bandwidth did my API key use this month?
- Am I close to the data volume in my plan?
- Did last month cost more traffic than the month before?
- Which month drove the jump in my bill?
- Did that batch job I ran yesterday move the monthly number?

## When to use a different tool

| Situation                                       | Better tool                                            | Why                                                                    |
| ----------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------- |
| You want the filings ingested on a given day     | [`edgar-ingestion-log`](./edgar-ingestion-log.md)      | Both take a `date`, but that tool returns EDGAR filings, not usage.    |
| You want your plan, quota, invoice, or API key   | [sec-api.io/profile](https://sec-api.io/profile)       | The MCP server exposes usage only. Billing lives in the web dashboard. |

## Input

| Parameter | Type   | Required | Constraints             | Notes                                              |
| --------- | ------ | -------- | ----------------------- | -------------------------------------------------- |
| `date`    | string | yes      | pattern `^20\d\d-[0-3]\d$` | The calendar month, format `YYYY-MM`, for example `2026-08`. |

The tool takes no `query`, so there is no Lucene syntax here.

Write the month with two digits. `2026-08` is valid. `2026-8` fails the pattern,
because the pattern needs two characters after the hyphen. A full date such as
`2026-08-13` also fails.

The pattern is looser than the format it describes. It accepts any two-digit
value from `00` to `39` in the month position, so `2026-13` passes validation.
Such a month has no counter behind it. Stay inside `01` to `12`.

## Output

The envelope is a plain scalar object. There is no `total`, no `data[]`, and no
array of any kind. This is one of the four singleton envelopes in the tool set.
See [response format](../response-format.md).

| Field                      | Type   | Meaning                                                       |
| -------------------------- | ------ | ------------------------------------------------------------- |
| `date`                     | string | The month you asked for, echoed back, format `YYYY-MM`.       |
| `monthlyBandwidthUsedInMb` | number | Bandwidth used by your API key in that month, in megabytes.   |

The value is rounded to two decimals. The server counts in kilobytes and divides
by 1024, so one megabyte is 1024 kilobytes here.

**This tool has no pagination.** It accepts no `from`, no `size`, and no `sort`.
There is nothing to page through. The response is a single, tiny object. The
capture came back in 51 bytes and 142 ms, which makes this the cheapest call in
the server.

The result arrives as one text block that holds stringified JSON, like every
other tool. The exact text of the capture was:

```text
{"date":"2026-08","monthlyBandwidthUsedInMb":37.58}
```

## Example

Prompt: "How much sec-api bandwidth did I use in August 2026?"

```json
{ "name": "api-key-usage", "arguments": { "date": "2026-08" } }
```

```json
{
  "date": "2026-08",
  "monthlyBandwidthUsedInMb": 37.58
}
```

To compare two months, call the tool twice, once per month. There is no range
input and no way to ask for several months in one call.

## Limits and errors

A bad `date` returns HTTP 400 with this message:

```text
Invalid date format. Use 'YYYY-MM' for the 'date' parameter.
```

The server applies the same pattern as the input schema, so a value that your
MCP client rejects locally would also be rejected by the API.

Facts not verified by a live call, taken from the server implementation:

- A month with no recorded traffic returns `0`, not an error and not an empty
  response.
- How far back the monthly counters are kept is unknown. Old months may return
  `0` once their counter is gone.
- The month boundary follows the server clock. The exact timezone is unverified.

The tool reports usage. It does not report whether MCP tool calls add to that
usage, and this page does not claim a number for that. The capture shows 37.58
MB for August 2026 on the probe key. That figure mixes every source of traffic
on the key, so do not read it as the cost of the probe run.

- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`edgar-ingestion-log`](./edgar-ingestion-log.md). The other `date` tool.
  It reports filings, not usage.
- [Tool reference index](./README.md)
- [Response format](../response-format.md)
- [sec-api.io MCP server docs](https://sec-api.io/docs/mcp-server)
- [Your sec-api profile and plan](https://sec-api.io/profile)
