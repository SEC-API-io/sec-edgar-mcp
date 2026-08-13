# edgar-ingestion-log

List every SEC filing that sec-api ingested on one day.

|                 |                                                    |
| --------------- | -------------------------------------------------- |
| Category        | Ingestion logs                                     |
| Required input  | `date`                                             |
| Returns         | `{lastUpdatedAt, total, data[]}`                   |
| Pagination      | **None.** No `from`, `size` or `sort`.             |
| REST equivalent | `GET /edgar-index/ingestion-log/{date}`            |

## What it does

This tool reports the ingestion pipeline, not EDGAR itself. One row is one
filing that sec-api pulled in on the day you asked for. Each row holds the
accession number, the form type, and the time EDGAR accepted the filing. The
list carries one row per accession number. A filing with several reporting
entities still gets one row.

Coverage starts on 2025-12-01. Earlier dates return an error. A request for
2025-12-15 returned 2,818 rows in 265,912 bytes.

The day is the ingestion day, not the filing day. 169 of those 2,818 rows carry
a `filedAt` of 2025-12-12, a Friday. The pipeline picked them up on the Monday.

## When to use it

- Did sec-api ingest this accession number, and on which day?
- How many filings arrived on 15 December 2025?
- Which form types dominated a day's flow?
- How fresh is the data right now?
- Which filings landed while my own job was down?

## When to use a different tool

| Situation                                     | Better tool                             | Why                                                                  |
| --------------------------------------------- | --------------------------------------- | -------------------------------------------------------------------- |
| You want filings by company, form or date     | [`filing-search`](./filing-search.md)   | It filters on `ticker`, `cik`, `formType` and `filedAt`. This tool takes one date and nothing else. |
| You want the filing document                  | [`get-edgar-file`](./get-edgar-file.md) | The log holds metadata only. No URLs.                                |
| You want your own API key traffic             | [`api-key-usage`](./api-key-usage.md)   | The other `date` tool. It reports bandwidth, not filings.            |
| You want the text of what was filed that day  | [`full-text-search`](./full-text-search.md) | The log carries no content to search.                            |

## Input

| Parameter | Type   | Required | Constraints                              | Notes                                                    |
| --------- | ------ | -------- | ---------------------------------------- | -------------------------------------------------------- |
| `date`    | string | Yes      | `YYYY-MM-DD`. 2025-12-01 or later.       | One day per call. `2025-12-15`.                          |

The schema declares `date` as a plain string. The server applies the format
rule and the coverage rule. There are no other parameters, and there is no
Lucene query here.

Ask for today's date and you get the live log for the day so far. It grows
through the day, as the pipeline indexes each new filing.

## Output

The envelope is `{lastUpdatedAt, total, data[]}`. It is the only envelope in the
server that carries `lastUpdatedAt`. `total` is an object, `{value, relation}`,
not a number.

### Envelope

| Field            | Type   | Meaning                                                          |
| ---------------- | ------ | ---------------------------------------------------------------- |
| `lastUpdatedAt`  | string | When the log last changed. It is the indexing time of the newest filing processed for that date. ISO 8601. Use it to judge freshness. |
| `total.value`    | number | Number of filings indexed on that date. The count treats one accession number as one filing. |
| `total.relation` | string | `eq` for this tool. The count is exact.                           |
| `data[]`         | array  | The filings indexed that date, one item per accession number. A filing that names several reporting entities gives one item. |

### Filing

| Field                | Type   | Meaning                                                                 |
| -------------------- | ------ | ----------------------------------------------------------------------- |
| `data[].accessionNo` | string | Accession number of the filing, in the dashed form `0001213900-25-121864`. |
| `data[].formType`    | string | Form as EDGAR labels it, for example `4`, `8-K`, `424B2`, `SCHEDULE 13G`, `LETTER`. |
| `data[].filedAt`     | string | The `Accepted` value of the filing. It is when EDGAR accepted the submission. ISO 8601. |

Rows come back newest ingestion first. They are **not** sorted by `filedAt`. Two
neighbouring rows in this response read 21:55:06 and then 21:55:07. Sort the array
yourself if you need filing order.

Some rows carry a much older `filedAt`. The SEC publishes comment letters weeks
or months after it accepts them. They enter the log on the publication day.

**This tool has no pagination.** One call returns the whole day. A busy weekday
runs to about 2,800 rows and 260 KB. Budget the context before you call it,
because you cannot ask for less.

## Example

Prompt: "Which filings did sec-api ingest on 15 December 2025?"

```json
{ "name": "edgar-ingestion-log", "arguments": { "date": "2025-12-15" } }
```

Trimmed response:

```json
{
  "lastUpdatedAt": "2025-12-15T21:56:54-05:00",
  "total": { "value": 2818, "relation": "eq" },
  "data": [
    { "accessionNo": "0001213900-25-121864", "formType": "S-1MEF", "filedAt": "2025-12-15T21:56:31-05:00" },
    { "accessionNo": "0001031530-25-000015", "formType": "4", "filedAt": "2025-12-15T21:55:06-05:00" },
    { "accessionNo": "0002096472-25-000004", "formType": "4", "filedAt": "2025-12-15T21:55:07-05:00" },
    { "accessionNo": "0001193125-25-319730", "formType": "SCHEDULE 13G", "filedAt": "2025-12-15T21:53:07-05:00" },
    { "accessionNo": "0001628280-25-057223", "formType": "4", "filedAt": "2025-12-15T21:50:34-05:00" },
    { "accessionNo": "0001104659-25-121337", "formType": "144", "filedAt": "2025-12-15T21:47:40-05:00" }
  ]
}
```

Six of 2,818 rows are shown. That day's mix was 741 Form 4 filings, 301 424B2,
267 Form 144, 239 8-K, 138 Form D and 113 6-K.

## Limits and errors

- A date before 2025-12-01 returns
  `Data for the requested date is only available from December 1, 2025 onwards.`
- A date in any other shape returns
  `Invalid date provided. Use YYYY-MM-DD format.`
- A date with no log file returns `Log file not found for the specified date.`
- The log lists filings, not documents. To open one, take `accessionNo` to
  [`filing-search`](./filing-search.md).
- Use the log for a daily check. It is not built for historical backfills.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`filing-search`](./filing-search.md). Filings by company, form and date.
- [`api-key-usage`](./api-key-usage.md). The other tool that takes a `date`.
- [Response format](../response-format.md). This tool owns the only
  `lastUpdatedAt` envelope.
- [sec-api.io EDGAR Index APIs docs](https://sec-api.io/docs/edgar-index-apis)
