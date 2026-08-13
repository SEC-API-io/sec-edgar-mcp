# sro

Search SRO rule filings, the rule changes that exchanges and self-regulatory
organizations file with the SEC.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Enforcement                                  |
| Required input  | `query`                                      |
| Returns         | `{total: {value, relation}, data[]}`         |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST https://api.sec-api.io/sro`            |

## What it does

`sro` searches the notices the SEC publishes for proposed rule changes from
self-regulatory organizations. One row is one notice. Each row carries the
release number, the issue date, the filing SRO, a one-line `details`
description, the comment deadline and links to the notice PDF and its exhibits.
The capture returned `total.value: 7743` for `sro:NASDAQ`, so Nasdaq alone had
7,743 notices on 2026-08-13.

This tool sits in the Enforcement category, but the content is regulatory, not
punitive. A row is a proposed rule or a fee change, never a charge against a
firm. The registry description names NYSE, Nasdaq, FINRA and MSRB as filers.
Only the Nasdaq value is verified by the capture.

## When to use it

- Which fee changes did Nasdaq file this month?
- What rule changes are open for comment right now?
- When did an exchange propose this rule, and where is the notice PDF?
- Which SROs filed rule changes about options pricing this year?
- What is the SEC release number for a given rule filing?

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| You want charges against a firm | [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | SRO filings are rule proposals. They contain no allegations. |
| You want SEC lawsuits | [`sec-litigation-releases`](./sec-litigation-releases.md) | Court actions are a different data set. |
| You want a company's own EDGAR filings | [`filing-search`](./filing-search.md) | SRO notices are SEC publications, not EDGAR filings. |
| You want broker-dealer financial reports | [`form-x-17a-5`](./form-x-17a-5.md) | FOCUS reports come from the broker-dealer, not from the SRO. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `query` | string | yes | Lucene syntax, max 1,000 characters, must contain a `:` | Example: `sro:NASDAQ`. |
| `from` | integer | no | 0 to 10000 | Offset of the first row. Default 0. Over 10000 returns an error. |
| `size` | integer | no | 1 to 50 | Rows per call. Default 50. Over 50 returns an error. |
| `sort` | array | no | Elasticsearch sort array | Default `[{"issueDate": {"order": "desc"}}]`. |

The schema sets `additionalProperties: true`, but this handler reads no extra
key. Unlike the other enforcement tools, it ignores `time_zone`.

Query field verified by a live call: `sro`. The field holds the full name, for
example `The Nasdaq Stock Market LLC (NASDAQ)`. A single token such as `NASDAQ`
matches it, so the field is analysed text.

Every other name in the Output table below is a response field. The sec-api REST
docs treat those as searchable too, but none was verified through MCP. Treat
`releaseNumber`, `issueDate`, `fileNumber`, `details` and `commentsDue` as
unverified. The sec-api Node SDK uses `sro:NYSE`.

## Output

The envelope has two keys. `total` is an object, `{value, relation}`, not a
number. `data` is the array of rows. This is the `{total, data[]}` family in
[response format](../response-format.md).

Rows are flat and small. There are seven fields, and no nested party or
allegation data.

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `total.value` | number | Number of matching notices. |
| `total.relation` | string | `eq` means exact. `gte` means at least that many. |
| `data[].id` | string | Internal record hash. |
| `data[].releaseNumber` | string | SEC release number, for example `34-106072`. Note the name. It is `releaseNo` on the other enforcement tools. |
| `data[].issueDate` | string | Publication date, `YYYY-MM-DD`. The sort field for this tool. |
| `data[].fileNumber` | string | SRO file number, for example `SR-NYSEAMER-2026-25`. Can be an empty string. |
| `data[].sro` | string | Full name of the filing organisation, with its short code in brackets. |
| `data[].details` | string | One-line description of the rule change. Long titles are truncated with `...`. |
| `data[].commentsDue` | string | Free text comment deadline, for example `21 days after date of publication in the Federal Register.` Not a date. |
| `data[].urls[]` | array | Documents. Each has `type` and `url`. The notice itself uses the release number as its `type`. Exhibits use `Exhibit 5`. A comment link can also appear. |

Size behaviour. `size` defaults to 50 and cannot exceed 50. Page with `from`,
which stops at 10000. The default of 50 comes from the server handler. The
capture set `size: 1` and returned 623 bytes for one row, so a `size: 50` call
stays near 30 KB. This is the lightest tool in the category.

## Example

Prompt: "What is the newest Nasdaq rule filing?"

```json
{ "name": "sro", "arguments": { "query": "sro:NASDAQ", "size": 1 } }
```

```json
{
  "total": { "value": 7743, "relation": "eq" },
  "data": [
    {
      "releaseNumber": "34-106072",
      "issueDate": "2026-08-11",
      "fileNumber": "",
      "sro": "The Nasdaq Stock Market LLC (NASDAQ)",
      "details": "Notice of Filing and Immediate Effectiveness of Proposed Rule Change to Amend the Exchange’s Transaction Fees at Options 7, Section 2",
      "commentsDue": "21 days after date of publication in the Federal Register.",
      "urls": [
        { "type": "34-106072", "url": "https://www.sec.gov/files/rules/sro/nasdaq/2026/34-106072.pdf" },
        { "type": "Exhibit 5", "url": "https://www.sec.gov/files/rules/sro/nasdaq/2026/34-106072-ex5.pdf" }
      ]
    }
  ]
}
```

## Limits and errors

- This handler returns one error for several causes. A query without a `:`, a
  query over 1,000 characters, or `from` above 10000 all fail with HTTP 400
  `Invalid request parameter provided.` The sibling tools give more precise
  messages. Check your query yourself.
- A missing `query` fails with `"query" parameter not provided.`
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- `fileNumber` is not reliable. It was an empty string in the capture, while the
  canonical SDK response has `SR-NYSEAMER-2026-25`. Do not use it as a key.
- `commentsDue` is free text, not a date. You cannot sort or range query it as a
  date.
- The error texts above come from the server handler. The capture did not
  trigger them.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). SEC actions against firms.
- [`sec-enforcement-actions`](./sec-enforcement-actions.md). Announced enforcement actions.
- [Query language](../query-language.md). Lucene syntax and field names.
- [sec-api.io SRO Filings API docs](https://sec-api.io/docs/sro-filings-database-api)
