# aaer-file

Fetch the full document attached to an Accounting and Auditing Enforcement
Release.

|                 |                                                        |
| --------------- | ------------------------------------------------------ |
| Category        | Enforcement                                            |
| Required input  | `aaerNo`, `fileTypeAndName`                            |
| Returns         | the file content as one text block, no JSON envelope   |
| Pagination      | **None.** No `from`, `size` or `sort`.                                               |
| REST equivalent | `GET https://api.sec-api.io/aaers/{aaerNo}/{fileTypeAndName}` |

## What it does

`aaer-file` returns one document that belongs to one AAER. The document is the
order, the opinion or the complaint. The server looks the file up by AAER number
and file name and streams it back. It is a fetch tool, not a search tool. Find
the AAER first with [`aaers`](./aaers.md).

You build `fileTypeAndName` from the `urls` entry in the `aaers` result. No
single field holds it. Join the lower-cased `urls[].type` and the last segment of
`urls[].url` with a hyphen. Add `.html` when that segment has no extension.

## When to use it

- I have an AAER number. Give me the order text.
- I want to read the full opinion, not the summary.
- I need the complaint PDF that belongs to this AAER.

Add `.txt` to the end of `fileTypeAndName` to get a plain-text rendering. Always
do this in an agent. The text form of the AAER-4452 summary is 3 KB. The HTML
form of the same document is 70 KB.

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| You want to find AAERs, not read one | [`aaers`](./aaers.md) | `aaers` is the search tool. It also returns the sec.gov document URL. |
| You want an EDGAR filing, not an enforcement document | [`get-edgar-file`](./get-edgar-file.md) | AAER documents live outside the EDGAR archive. |
| You want the case in structured form | [`sec-administrative-proceedings`](./sec-administrative-proceedings.md) | The same order appears there with `orders`, `complaints` and `violatedSections` already extracted. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `aaerNo` | string | yes | Pattern `aaer-<1 to 5 digits>`, case insensitive, max 15 characters | Example: `AAER-4597`. Comes from `aaers` → `data[].aaerNo`. |
| `fileTypeAndName` | string | yes | Must contain a hyphen, max 500 characters | `<type>-<file name>`. Build it from `urls[]`. Example: `primary-33-11432.pdf`. |

The constraints come from the server handler, not from the input schema. The
schema declares both parameters as plain strings.

Build `fileTypeAndName` like this, from one entry of `urls[]` in the `aaers`
result:

1. Take `type`, lower case. For example `primary` or `administrative summary`.
   A space is sent literally through MCP.
2. Take the last segment of `url`, with its extension. For example
   `33-11432.pdf`.
3. Join the two with a hyphen.
4. Add `.html` when the segment has no extension. SEC `/enforce/` pages carry no
   extension in the URL but are stored as HTML.
5. Add `.txt` to the whole value to get plain text instead of the raw file.

Verified on AAER-4452: `primary-34-98243.pdf` returns a 199 KB PDF,
`primary-34-98243.pdf.txt` returns 29 KB of text, and
`administrative summary-34-98243-s.html.txt` returns 3 KB of text.

There is **no** `query`, `from`, `size` or `sort` parameter.

## Output

The server streams the file and sets the content type from the extension:
`application/pdf` for `.pdf`, `text/html` for `.html`, `text/plain` for `.txt`.
MCP carries the bytes in one text block. A PDF decoded as UTF-8 is unreadable,
so ask for the `.txt` form unless you need the raw file. See
[response format](../response-format.md).

There is **no** envelope, no `total` and no `data[]`. There is also **no
pagination**. The whole file arrives in one response. The captured PDF was
159 KB.

## Example

Prompt: "Get the full order for AAER-4597."

The `aaers` result for AAER-4597 holds one document:

```json
{ "type": "primary", "url": "https://www.sec.gov/files/litigation/opinions/2026/33-11432.pdf" }
```

That gives `primary` plus `33-11432.pdf`, so the call is:

```json
{ "name": "aaer-file", "arguments": { "aaerNo": "AAER-4597", "fileTypeAndName": "primary-33-11432.pdf" } }
```

The captured response is the raw PDF, 159 KB, in one text block. It starts:

```text
%PDF-1.6
74 0 obj
<</Linearized 1/L 88305/O 76/E 81005/N 2/T 87985/H [ 488 198]>>
endobj
```

Add `.txt` to `fileTypeAndName` to read the same document as text.

These input shapes fail, because the type prefix or the extension is missing:

| `fileTypeAndName`      | Result             |
| ---------------------- | ------------------ |
| `33-11432.pdf`         | File not found.    |
| `primary/33-11432.pdf` | File not found.    |
| `primary.pdf`          | Invalid file name. |
| `primary`              | Invalid file name. |

## Limits and errors

- `Invalid AAER number.` means `aaerNo` does not match `aaer-<digits>`.
- `Invalid file name.` means `fileTypeAndName` is missing, over 500 characters,
  or has no hyphen.
- `File not found.` means the lookup key was well formed but the document store
  holds nothing under it.
- Errors arrive with HTTP 200 and the message inside the text block. Check the
  text, not the status code.
- `aaerNo` is case insensitive. `AAER-4452` and `aaer-4452` both work.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`aaers`](./aaers.md). Search AAERs and get the `urls` entry.
- [`sec-administrative-proceedings`](./sec-administrative-proceedings.md). The same orders, already parsed.
- [sec-api.io AAER Database API docs](https://sec-api.io/docs/aaer-database-api)
