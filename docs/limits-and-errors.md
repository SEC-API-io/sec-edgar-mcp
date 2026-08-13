# Limits and errors

This page explains the limits and error messages. Per-tool quirks live on the [tool pages](./tools/README.md).

## Quick reference

| Limit               | Value                                                                   |
| ------------------- | ----------------------------------------------------------------------- |
| Max rows per call   | 50 on most tools. 10 on `form-nport`. 100 on `full-text-search`         |
| Paging ceiling      | `from` plus `size` must stay at or below 10,000                         |
| Max `total.value`   | 10000. It means "10,000 or more"                                        |
| Query string length | 1,000 characters on most tools. 2,000 on some. 3,500 on `filing-search` |
| Largest response    | +100 MB, from `form-nport`                                              |
| Response size cap   | None. The server sends what the data weighs                             |
| Session             | None. The transport is stateless                                        |

## Payload size

There is no cap on the response size. One tool call can return megabytes in one
text block. Read [response format](./response-format.md) for the block shape.

Estimate tokens at about four characters each. The 1.94 MB `form-npx-file`
response is therefore near 500,000 tokens. That is more than most agents hold.

Three rules keep an agent alive:

1. Call an unfamiliar search tool with `size: 1` first. Multiply the row size by
   the page size you actually want.
2. Never put `form-npx-file` or `xbrl-to-json` in a loop without a size check.
3. Ask [`extractor`](./tools/extractor.md) for one section instead of pulling a
   whole filing with [`get-edgar-file`](./tools/get-edgar-file.md).

Row size varies a lot. A `filing-search` row is about 4.2 KB, so a full page of
50 reaches 200 KB. An `aaers` row is 1.1 KB, so the same page is 55 KB.

## Pagination

Paging is not uniform. There are four families.

| Family           | Parameters                       | Tools                                                     |
| ---------------- | -------------------------------- | --------------------------------------------------------- |
| Standard         | `from`, `size` (1 to 50), `sort` | Most search tools, including every `{total, data[]}` tool |
| Capped at 10     | `from`, `size` (1 to 10), `sort` | [`form-nport`](./tools/form-nport.md)                     |
| Page number only | `page`, 100 rows per page        | [`full-text-search`](./tools/full-text-search.md)         |
| None             | No paging parameter at all       | See the table below                                       |

`size` defaults to 50 on the standard family, not to 10. A call that omits
`size` returns a full page.

`form-nport` advertises `size` up to 50 in its input schema, but the server caps
it at 10. Values from 11 to 50 pass the schema and then fail.

### Tools with no pagination

These tools return everything they hold for the input. You cannot ask for less.

| Tool                                                                                                                                                                 | What one call returns                                                       |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| [`form-npx-file`](./tools/form-npx-file.md)                                                                                                                          | Every proxy vote in the filing. 4,372 records and 1.94 MB in the probe      |
| [`float`](./tools/float.md)                                                                                                                                          | Every reporting period on record. 61 rows and 19 KB for Apple, back to 2011 |
| [`xbrl-to-json`](./tools/xbrl-to-json.md)                                                                                                                            | Every statement in the filing                                               |
| [`form-adv-brochures`](./tools/form-adv-brochures.md)                                                                                                                | Every brochure for the CRD                                                  |
| The six `form-adv-schedule-*` tools                                                                                                                                  | Every row for the CRD                                                       |
| [`mapping`](./tools/mapping.md)                                                                                                                                      | Every match for the value                                                   |
| [`api-key-usage`](./tools/api-key-usage.md)                                                                                                                          | One scalar object                                                           |
| [`extractor`](./tools/extractor.md), [`filing-to-pdf`](./tools/filing-to-pdf.md), [`get-edgar-file`](./tools/get-edgar-file.md), [`aaer-file`](./tools/aaer-file.md) | One document                                                                |

`form-npx-file` and `float` are the two to watch. They accept no `size`, no
`from` and no `sort`, and a large fund makes `form-npx-file` the worst context
trap in the tool set.

### The 10,000 ceiling

Two separate ceilings both sit at 10,000.

**The paging ceiling.** `from` plus `size` must stay at or below 10,000. You
cannot read row 10,001 with any combination of parameters. Narrow the query and
page again, usually with a date range.

Tools disagree on what happens above the ceiling:

- Some return HTTP 400 `Invalid request parameter provided.` for `from` above 10000. [`form-8k`](./tools/form-8k.md), [`form-s1-424b4`](./tools/form-s1-424b4.md),
  [`sro`](./tools/sro.md) and [`subsidiaries`](./tools/subsidiaries.md) do this.
- Some return an empty result with no error at all. `{"total":{"value":0},"data":[]}`
  is what [`edgar-entities`](./tools/edgar-entities.md) gives for `from` above
  10000, and what most tools give when `from` plus `size` crosses 10,000.

An empty page above the ceiling does not mean the data ended. Never treat it as
the end of a result set.

**The counting ceiling.** `total` is an object, not a number:

```json
{ "total": { "value": 10000, "relation": "gte" }, "data": [] }
```

`relation` is `eq` for an exact count. It is `gte` when the count hit the
result-window ceiling. A `value` of exactly `10000` with `relation: "gte"`
means **10,000 or more**. It is never a real count.

Never print `10000` as a number of filings. To get a true count, split the query
by date range until every part reports `relation: "eq"`, then add the parts.

## Error shape

Errors arrive in two layers. Tell them apart before you write a handler.

**Transport errors** never reach a tool. The HTTP request fails and the body is
a small JSON object.

**Tool errors** arrive as **HTTP 200**. The JSON-RPC result carries
`isError: true` and one text content block:

```json
{
  "content": [
    { "type": "text", "text": "sec-api error: Invalid Lucene query string" }
  ],
  "isError": true
}
```

Check `isError`, or the `sec-api error: ` prefix. Do not check the HTTP status.
A failed tool call and a good one both return 200.

## Transport errors

| HTTP | Body                                     | Cause                                    | Fix                                                                 |
| ---- | ---------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------- |
| 401  | `{"error":"invalid or missing apiKey"}`  | No key in the URL or header              | Send `?apiKey=YOUR_API_KEY` or `Authorization: Bearer YOUR_API_KEY` |
| 503  | `{"error":"mcp server not initialized"}` | The process is still loading the MCP SDK | Retry after a few seconds                                           |
| 500  | `{"error":"mcp handler failed"}`         | The transport threw before a tool ran    | Retry once. Report it if it repeats                                 |

A client that hangs on connect is a different problem, not an error. The
transport is stateless, so the server issues no `Mcp-Session-Id`. Clients that
demand an SSE session need the stdio bridge. See [transport](./transport.md).

## Auth errors

The error arrives as a tool error, with HTTP 200 and `isError: true`.

| Message                                                       | Cause                                                                                              | Fix                                |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------- |
| `API token invalid. Please get a valid token from sec-api.io` | The key has the right shape but is not a real key. The URL gate passed it, the handler rejected it | Copy the key again from sec-api.io |

A malformed key fails earlier, with HTTP 401. A well-formed wrong key gets this
far. The two failures look different, so read the layer before you debug.

## Query and paging errors

These come from the search handlers. Every one is HTTP 400 under the tool error
envelope.

| Message                                                                                                                                                                                               | Tools                                                                                       | Cause                                                                                                     | Fix                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `Invalid Lucene query string`                                                                                                                                                                         | `aaers`, `edgar-entities`, `form-npx`, the three enforcement tools                          | `query` has no colon                                                                                      | Use `field:value`. See [query language](./query-language.md) |
| `Invalid query`                                                                                                                                                                                       | `insider-trading`, `form-nport`                                                             | `query` is missing, or has no colon                                                                       | Same                                                         |
| `Invalid request parameter provided.`                                                                                                                                                                 | `form-8k`, `form-s1-424b4`, `form-13d-13g`, the two `form-13f` tools, `sro`, `subsidiaries` | One message, four causes. Missing `query`, no colon, `query` over 1,000 characters, or `from` above 10000 | Check all four                                               |
| `"query" parameter not provided.`                                                                                                                                                                     | `sro`                                                                                       | No `query` at all                                                                                         | Send a `query`                                               |
| `Query too long. Maximum length: 1000 characters`                                                                                                                                                     | `aaers`, `form-npx`, the three enforcement tools                                            | `query` over 1,000 characters                                                                             | Shorten it, or split the call                                |
| `Query too long. Maximum length: 2000 characters`                                                                                                                                                     | `full-text-search`, `insider-trading`, `form-nport`                                         | `query` over 2,000 characters                                                                             | Same                                                         |
| `Query too short. Minimum length: 3 characters`                                                                                                                                                       | `compensation`, `compensation-by-key`                                                       | The query is shorter than 3 characters                                                                    | Send 3 characters or more                                    |
| `Maximum 'size' limit of 50 exceeded. Please adjust 'size' to 50 or less. To retrieve more than 50 results, increase the 'from' parameter incrementally by 50 to access subsequent pages of results.` | Every standard search tool                                                                  | `size` above 50                                                                                           | Use `size: 50` and page with `from`                          |
| `Maximum 'size' limit of 10 exceeded. ...`                                                                                                                                                            | `form-nport`                                                                                | `size` above 10. The schema allows 50, the server does not                                                | Keep `size` at 10                                            |

`filing-search` allows 3,500 characters of query, the widest limit in the set.

## Input validation errors

| Message                                                                                               | Tool                                           | Cause                                                                                                             | Fix                                                                                      |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `Parameter <ticker> or <cik> invalid.`                                                                | `float`                                        | Neither `ticker` nor `cik` was sent. The schema marks neither required                                            | Send one of them                                                                         |
| `Invalid parameter. Valid parameters are: cik, cusip, ticker, name, exchange, sic, sector, industry.` | `mapping`                                      | `param` is not in that list                                                                                       | Use one of the eight names                                                               |
| `Invalid date format. Use 'YYYY-MM' for the 'date' parameter.`                                        | `api-key-usage`                                | `date` is not `YYYY-MM`                                                                                           | Send `2026-08`                                                                           |
| `Data for the requested date is only available from December 1, 2025 onwards.`                        | `edgar-ingestion-log`                          | `date` is before the coverage start                                                                               | Use a later date. There is no older data                                                 |
| `Invalid accession number. Must be in format "XXXXXXXXXX-YY-NNNNNN"`                                  | `form-npx-file`                                | Malformed `accessionNo`                                                                                           | Copy the value from a `form-npx` hit                                                     |
| `Invalid AAER number.`                                                                                | `aaer-file`                                    | `aaerNo` does not match `aaer-<digits>`                                                                           | Use `AAER-4452`. The case does not matter                                                |
| `Invalid file name.`                                                                                  | `aaer-file`                                    | `fileTypeAndName` is missing, over 500 characters, or has no hyphen                                               | Build it as `<type>-<file name>` from the `urls[]` entry                                 |
| `CRD invalid.`                                                                                        | `form-adv-brochures`                           | The CRD has a non-digit, or is over 10 characters                                                                 | Send digits only                                                                         |
| `Invalid CRD provided.`                                                                               | The six `form-adv-schedule-*` tools            | The CRD is one character, or has a non-digit                                                                      | Send a real CRD                                                                          |
| `Invalid request parameter`                                                                           | `xbrl-to-json`                                 | None of the three inputs was sent                                                                                 | Send `accession-no`, `htm-url` or `xbrl-url`                                             |
| `Invalid filing URL`                                                                                  | `filing-to-pdf`, `get-edgar-file`              | The URL is not on `www.sec.gov`                                                                                   | Use the `www.sec.gov` form of the URL                                                    |
| `not a valid SEC filing URL`                                                                          | `extractor`                                    | Same cause, different wording                                                                                     | Same                                                                                     |
| `filing type not supported`                                                                           | `extractor`                                    | The filing is not a 10-K, 10-Q or 8-K                                                                             | Use `get-edgar-file` for other forms                                                     |
| `10-K item type not supported. Supported items are: 1, 1A, 1B, ...`                                   | `extractor`                                    | The `item` belongs to another form type, for example `part2item1a` on a 10-K                                      | Match `item` to the form type                                                            |
| `File not found.`                                                                                     | `aaer-file`, `form-npx-file`, `get-edgar-file` | The key was well formed but nothing is stored under it. A trailing slash, a directory URL or a typo all land here | Check the exact file name. For `form-npx-file`, check `proxyVotingRecordsAttached` first |

## Server defects

| Message                                                    | Tool                                | State                                                                                |
| ---------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------ |
| `Cannot read properties of undefined (reading 'includes')` | The six `form-adv-schedule-*` tools | Fixed on 2026-08-13. If you still see it, the deploy is stale. Retry, then report it |
| `Cannot read properties of undefined (reading 'split')`    | `form-x-17a-5`                      | Fixed on 2026-08-13. Same advice                                                     |

Two more messages come from the wrapper itself. They mean a bug, not bad input.
Report both.

| Message                               | Cause                                                                   |
| ------------------------------------- | ----------------------------------------------------------------------- |
| `sec-api error: no handler responded` | The handler chain ended without writing a response                      |
| `sec-api error: status 500`           | The handler failed and sent no `error` or `message` field to explain it |

One message has no `sec-api error: ` prefix, because it never reaches a tool:

```text
unknown tool: form-13f
```

The client asked for a tool name the server does not have. Refresh the tool
list. All 49 names are in [the tool reference](./tools/README.md).

## Failures that do not look like failures

The dangerous cases return `isError: false`. An agent reads them as success.

| What you get                                                                                                           | What it really means                                                                                                           |
| ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `{"total":{"value":10000,"relation":"gte"},...}`                                                                       | 10,000 or more. It is a ceiling, not a count                                                                                   |
| `{"total":{"value":0},"data":[]}` after a high `from`                                                                  | You crossed the 10,000 paging ceiling. There is more data you cannot reach this way                                            |
| `{"message":"Cache miss: PDF generation started. Retry in 5 seconds."}`                                                | HTTP 202 from `filing-to-pdf`. 202 sits inside the 2xx range, so the wrapper calls it a success. Wait 5 seconds and call again |
| `{"message":"XBRL conversion started, but the processing has not been completed. Please try again after 60 seconds."}` | HTTP 202 from `xbrl-to-json`. Wait 60 seconds and call again                                                                   |
| `[]` from a `form-adv-schedule-*` tool                                                                                 | Either the CRD is unknown, or the adviser has no rows of that kind. The response cannot tell them apart                        |
| `{"total":{"value":0},"data":[]}` from `float`                                                                         | The CIK was zero padded. Strip the leading zeros                                                                               |
| Zero rows from `form-8k` on an item other than 4.01, 4.02 or 5.02                                                      | That item is not covered. It does not mean the event never happened                                                            |
| Zero rows from `reg-a-search` on a bare word                                                                           | The analyzer is strict. Use a field query                                                                                      |
| `0` from `api-key-usage`                                                                                               | No traffic that month, or the counter has aged out                                                                             |

## Related

- [Response format](./response-format.md). The envelopes, and why there is no
  `structuredContent`.
- [Transport](./transport.md). Streamable HTTP, the stateless design, and the
  stdio bridge.
- [Query language](./query-language.md). Lucene syntax and the field names.
- [Tool reference](./tools/README.md). Per-tool limits, one page each.
