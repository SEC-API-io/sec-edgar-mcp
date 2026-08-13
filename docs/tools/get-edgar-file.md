# get-edgar-file

Download any SEC EDGAR filing or exhibit in its original filed format.

|                 |                                                     |
| --------------- | --------------------------------------------------- |
| Category        | Filings and documents                               |
| Required input  | `url`                                               |
| Returns         | the file source in one text block, no JSON envelope |
| Pagination      | **None.** No `from`, `size` or `sort`.                                                |
| REST equivalent | none. The public route is `https://archive.sec-api.io/<cik>/<accession-no>/<file>` |

## What it does

The server maps the EDGAR URL to its own mirror and returns the file bytes
untouched. Coverage runs from 1993 to today and includes primary documents,
exhibits, XML, plain text, complete submission files, SGML headers and images.
Nothing is cleaned, converted or re-rendered. The `url` must start with
`https://www.sec.gov/`. iXBRL viewer wrappers are stripped, so
`.../ix?doc=/Archives/...` and `.../ix.xhtml?doc=/Archives/...` both resolve to
the underlying document.

The capture returned Apple's EX-21.1 exhibit complete, 11,807 bytes. The
response opened with the EDGAR SGML wrapper, `<DOCUMENT>`, `<TYPE>`,
`<SEQUENCE>`, `<FILENAME>`, `<DESCRIPTION>`, `<TEXT>`, then the HTML body, and
closed with `</TEXT></DOCUMENT>`. Expect that wrapper on document files.

## When to use it

- I want this exhibit exactly as it was filed, with no clean-up.
- I need the raw XML of a Form 4, an N-PORT or an XBRL instance.
- I want the complete submission `.txt` file for one accession number.
- I need the SGML header of a filing.

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| You only want one section | [`extractor`](./extractor.md) | Returns Risk Factors or MD&A as text. Far smaller than the whole file. |
| You want something printable | [`filing-to-pdf`](./filing-to-pdf.md) | Renders the same file to PDF. |
| You want financial figures | [`xbrl-to-json`](./xbrl-to-json.md) | Parses the XBRL instead of handing you the markup. |
| You do not know the file URL | [`filing-search`](./filing-search.md) | Returns the document URLs for a filing. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `url` | string | yes | `format: uri`. Must match `https://www.sec.gov/...` | Any EDGAR file URL. An iXBRL viewer URL is accepted and unwrapped. |

The schema sets `additionalProperties: true`. No other key is documented or
verified.

This tool takes no `query`, so there are no Lucene fields.

## Output

One MCP `text` content block that holds the file source. There is no JSON
envelope. There is no `total`, no `data[]`, no file name, no content type and no
byte count. You get the file body and nothing else.

| Detail | Observed or implemented behaviour |
| ------ | --------------------------------- |
| Text files | Returned as UTF-8. Applies to `.htm`, `.html`, `.xml`, `.txt`, `.json`, `.sgml`, `.css`, `.js`, and to any response whose content type is text, HTML, XML or JSON. Verified in the capture. |
| Binary files | Returned base64-encoded in the same text block. Taken from the server implementation. Not verified through MCP. |
| Size in the capture | 11,807 bytes, the complete EX-21.1 exhibit |

**This tool has no pagination.** There is no `from`, no `size`, no `sort` and no
byte-range parameter. The whole file lands in your context in one block. A
complete submission `.txt` file for a large 10-K can exceed 9 MB. Check the size
first with [`filing-search`](./filing-search.md), which reports each document's
size, before you fetch a primary document or a submission file.

## Example

Prompt: "Show me Apple's Exhibit 21.1 from the 2025 10-K exactly as filed."

```json
{
  "name": "get-edgar-file",
  "arguments": {
    "url": "https://www.sec.gov/Archives/edgar/data/320193/000032019325000079/a10-kexhibit21109272025.htm"
  }
}
```

```text
<DOCUMENT>
<TYPE>EX-21.1
<SEQUENCE>3
<FILENAME>a10-kexhibit21109272025.htm
<DESCRIPTION>EX-21.1
<TEXT>
<html><head>
<!-- Document created using Wdesk -->
<!-- Copyright 2025 Workiva -->
<title>Document</title></head><body><div id="ica0168a2a02e453f9577fbb3c09df39d_1">
```

The HTML body is one very long line in the real response. It is wrapped above.

## Limits and errors

- A `url` that is not on `www.sec.gov` returns HTTP 404 with `Invalid filing
  URL`.
- A URL that the mirror cannot resolve returns HTTP 404 with `File not found.`
  A trailing slash, a directory listing URL or a typo in the file name all land
  here.
- The mirror request times out after 30 seconds and is reported as
  `File not found.`
- Every call is billed on response size.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`extractor`](./extractor.md). One section as clean text.
- [`filing-to-pdf`](./filing-to-pdf.md). The same file rendered to PDF.
- [`filing-search`](./filing-search.md). Find the file URL and its size first.
- [Response format](../response-format.md). Why this is a text block.
- REST docs. [Filing Render & Download API](https://sec-api.io/docs/sec-filings-render-api)
