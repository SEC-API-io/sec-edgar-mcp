# Response format

Every tool on this server answers in the same MCP envelope. Inside that envelope
the payload shape changes from tool to tool. This page describes both layers, and
gives you one parser that handles all 49 tools.

The most common mistake is to assume `data[]`. Only half the tools use it.

## The MCP envelope

One tool call returns **one** `text` content block. That is true for all 49
tools.

A successful call:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"date\":\"2026-08\",\"monthlyBandwidthUsedInMb\":39.12}"
      }
    ],
    "isError": false
  }
}
```

A failed call:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "sec-api error: Invalid date format. Use 'YYYY-MM' for the 'date' parameter."
      }
    ],
    "isError": true
  }
}
```

| Fact | Detail |
| ---- | ------ |
| Content blocks | Always one. The payload sits at `result.content[0].text`. |
| Block type | Always `text`. No `image`, no `resource`, no `audio`. |
| JSON payloads | **Stringified** inside `text`. The value is a JSON string, not an object. |
| `structuredContent` | **Never present.** No tool declares an `outputSchema`, so the server sends no parsed copy. |
| `isError` | Always present. `false` on success, `true` on failure. |
| Errors | Arrive as text, not as a JSON-RPC `error` object. The HTTP status stays 200. |
| Error text | Always starts with `sec-api error: `. |

Transport errors are different. A bad API key returns HTTP 401. It has no
`result`. See [limits and errors](./limits-and-errors.md).

The transport is stateless Streamable HTTP. There is no `Mcp-Session-Id` header
to keep. See [transport](./transport.md).

You can see the raw wire format with one command:

```bash
curl -s 'https://api.sec-api.io/mcp?apiKey=YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"mapping","arguments":{"param":"ticker","value":"AAPL"}}}'
```

## Ten payload envelopes

Inside `content[0].text` the server uses ten different shapes. The counts below
cover 48 tools. `compensation-by-key` is not in the table.

| Envelope | Count | Read rows from | Tools |
| -------- | ----- | -------------- | ----- |
| `{total, data[]}` | 24 | `body.data` | `aaers`, `audit-fees`, `directors-and-board-members`, `edgar-entities`, `edgar-ingestion-log`, `float`, `form-8k`, `form-13f-cover-pages`, `form-13f-holdings`, `form-144`, `form-c`, `form-ncen`, `form-npx`, `form-s1-424b4`, `form-x-17a-5`, `reg-a-form-1a`, `reg-a-form-1k`, `reg-a-form-1z`, `reg-a-search`, `sec-administrative-proceedings`, `sec-enforcement-actions`, `sec-litigation-releases`, `sro`, `subsidiaries` |
| `{total, filings[]}` | 6 | `body.filings` | `filing-search`, `full-text-search`, `form-13d-13g`, `form-adv-firms`, `form-adv-individuals`, `form-nport` |
| Bare array, no envelope | 7 | `body` | `compensation`, `mapping`, `form-adv-schedule-a-direct-owners`, `form-adv-schedule-b-indirect-owners`, `form-adv-schedule-d-1-b`, `form-adv-schedule-d-7-a`, `form-adv-schedule-d-7-b-1` |
| Raw document, not JSON | 4 | the text itself | `extractor`, `filing-to-pdf`, `get-edgar-file`, `aaer-file` |
| `{total, transactions[]}` | 1 | `body.transactions` | `insider-trading` |
| `{total, offerings[]}` | 1 | `body.offerings` | `form-d` |
| `{brochures[]}`, no `total` | 1 | `body.brochures` | `form-adv-brochures` |
| `{proxyVotingRecords[]}`, no `total` | 1 | `body.proxyVotingRecords` | `form-npx-file` |
| Named-section object | 2 | named keys | `xbrl-to-json`, `form-adv-schedule-d-5-k` |
| Scalar object | 1 | named keys | `api-key-usage` |

Notes on the rows that surprise people:

- **`edgar-ingestion-log`** adds `lastUpdatedAt` beside `total` and `data`.
  **`filing-search`** adds a `query` echo beside `total` and `filings`.
  `full-text-search` does not echo the query.
- **The four singleton envelopes** are `insider-trading`, `form-d`,
  `form-adv-brochures` and `form-npx-file`. They look like the common case, so a
  hard-coded `body.data` returns `undefined` instead of an error.
- **`form-npx-file`** returns a bare filing object. The votes sit in
  `proxyVotingRecords[]` next to `accessionNo`, `cik`, `headerData` and
  `formData`. There is no wrapper and no `total`.
- **`xbrl-to-json`** names its top-level keys after the filing's own statements,
  such as `CoverPage`, `BalanceSheets` and `StatementsOfCashFlows`. The names
  change between filers and between years. A hard-coded name does not carry
  from one filing to the next.
- **`form-adv-schedule-d-5-k`** returns a fixed three-key object:
  `1-separatelyManagedAccounts`, `2-borrowingsAndDerivatives` and
  `3-custodiansForSeparatelyManagedAccounts`. The other six Form ADV schedule
  tools return a bare array.
- **`api-key-usage`** returns two scalars and no array of any kind.

Each tool page names its own row in this table. Start at the
[tool reference](./tools/README.md).

## Tools that return HTML, text or PDF

Four tools return the document itself. A JSON parse fails on all four.

| Tool | What arrives in the text block | Notes |
| ---- | ------------------------------ | ----- |
| [`extractor`](./tools/extractor.md) | Plain text, or HTML when you set `type: "html"` | Numeric HTML entities stay encoded. `&#8217;` is an apostrophe. Decode them yourself. Apple's Item 1A gave 69,877 bytes. |
| [`get-edgar-file`](./tools/get-edgar-file.md) | The file source, exactly as filed | Text files arrive as UTF-8. Binary files arrive base64-encoded in the same text block. |
| [`filing-to-pdf`](./tools/filing-to-pdf.md) | PDF bytes. The block starts with `%PDF-1.4` | The bytes are decoded as UTF-8, so binary sections are lossy. The block is a preview, not a file you can save. |
| [`aaer-file`](./tools/aaer-file.md) | PDF, HTML or plain text | The content type follows the file extension you ask for. A `.txt` suffix on `fileTypeAndName` gives a text rendering. One AAER summary gave 3 KB as text against 70 KB as HTML. |

None of the four sends a file name, a content type or a byte count. You get the
body and nothing else. None of the four supports paging.

## The `total` object

`total` is an object, not a number. Every tool that sends it uses the same shape.

```json
{ "total": { "value": 10000, "relation": "gte" } }
```

| Field | Meaning |
| ----- | ------- |
| `value` | The match count. |
| `relation` | `eq` for an exact count. `gte` for "at least this many". |

A `relation` of `gte` with a `value` of `10000` means the search engine stopped
counting. It means "10,000 or more". It is not an exact total.

`total` counts matches, not returned rows. The `length` of the array is the
count you received.

## How to parse reliably

A parser reads the response in six steps.

1. The payload is `result.content[0].text`. There is never a second block.
2. `result.isError` says what the text holds. On `true`, the text is an error
   message, not data, and it carries the `sec-api error: ` prefix.
3. `structuredContent` does not exist on this server.
4. A JSON parse of the text fails when the tool returns a raw document. The text
   is then the document itself.
5. The payload array sits under one of six keys. New tools reuse the same six
   keys, so a key probe outlives a switch on the tool name.
6. `total` is an object. The count is `total.value`, and `total.relation` says
   whether that count is exact.

An empty result is not an error. Searches return the envelope with an empty
array and `total.value` of `0`. `mapping` and the Form ADV schedules return a
bare `[]`. `form-adv-brochures` returns `{"brochures":[]}`.

### JavaScript

```javascript
const PAYLOAD_KEYS = [
  "data",
  "filings",
  "transactions",
  "offerings",
  "brochures",
  "proxyVotingRecords",
];

function readResult(result) {
  const text = result.content[0].text;

  if (result.isError) {
    throw new Error(text.replace(/^sec-api error: /, ""));
  }

  let body;
  try {
    body = JSON.parse(text);
  } catch {
    // extractor, get-edgar-file, filing-to-pdf, aaer-file
    return { kind: "document", text, rows: [], total: null };
  }

  if (Array.isArray(body)) {
    return { kind: "array", body, rows: body, total: body.length };
  }

  for (const key of PAYLOAD_KEYS) {
    if (Array.isArray(body[key])) {
      return {
        kind: key,
        body,
        rows: body[key],
        total: body.total ? body.total.value : body[key].length,
        exact: body.total ? body.total.relation === "eq" : true,
      };
    }
  }

  // xbrl-to-json, form-adv-schedule-d-5-k, api-key-usage
  return { kind: "object", body, rows: [], total: null };
}
```

### Python

```python
import json

PAYLOAD_KEYS = (
    "data",
    "filings",
    "transactions",
    "offerings",
    "brochures",
    "proxyVotingRecords",
)


def read_result(result):
    text = result["content"][0]["text"]

    if result.get("isError"):
        raise RuntimeError(text.removeprefix("sec-api error: "))

    try:
        body = json.loads(text)
    except json.JSONDecodeError:
        # extractor, get-edgar-file, filing-to-pdf, aaer-file
        return {"kind": "document", "text": text, "rows": [], "total": None}

    if isinstance(body, list):
        return {"kind": "array", "body": body, "rows": body, "total": len(body)}

    for key in PAYLOAD_KEYS:
        rows = body.get(key)
        if isinstance(rows, list):
            total = body.get("total")
            return {
                "kind": key,
                "body": body,
                "rows": rows,
                "total": total["value"] if total else len(rows),
                "exact": total["relation"] == "eq" if total else True,
            }

    # xbrl-to-json, form-adv-schedule-d-5-k, api-key-usage
    return {"kind": "object", "body": body, "rows": [], "total": None}
```

## Size

The whole payload arrives in one block. There is no streaming and no chunking,
so a large response lands in your context in one step.

| Tool | Largest response | Why it grows |
| ---- | ---------------- | ------------ |
| `form-npx-file` | 1.94 MB, 4,372 vote records | Takes no paging. One accession number returns every vote. |
| `xbrl-to-json` | 1.31 MB | Returns all statements of one filing. You cannot ask for one. |
| `filing-to-pdf` | 68 KB for a small exhibit | A full 10-K source is 1.5 MB, and a complete submission file is 9.4 MB. |
| `extractor` | 70 KB for Apple's Item 1A | Risk Factors and MD&A are the longest items. |
| `float` | 19 KB, 61 records | Takes no `size`. Returns every reporting period held. |

[`filing-search`](./tools/filing-search.md) reports the byte size of each file
in the filing. That size is available before you fetch a primary document.

## See also

- [Tool reference](./tools/README.md). Each page names its envelope.
- [Query language](./query-language.md). The field vocabulary per tool.
- [Limits and errors](./limits-and-errors.md). HTTP status codes and paging.
- [Transport](./transport.md). Streamable HTTP, and the stdio bridge.
