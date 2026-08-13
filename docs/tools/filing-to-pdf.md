# filing-to-pdf

Fetch any SEC EDGAR filing or exhibit and return it as a rendered PDF.

|                 |                                                    |
| --------------- | -------------------------------------------------- |
| Category        | Filings and documents                              |
| Required input  | `url`                                              |
| Returns         | raw PDF bytes in one text block, no JSON envelope  |
| Pagination      | **None.** No `from`, `size` or `sort`.             |
| REST equivalent | `GET https://api.sec-api.io/filing-reader?url=<url>` |

> **Warning.** The PDF comes back inside an MCP `text` content block, not as a
> file, a link, or base64. The bytes are decoded as UTF-8 first, so the block is
> mostly unreadable binary. Most clients cannot turn it back into a valid PDF.
> Read [Output](#output) before you call this tool.

## What it does

The server renders the EDGAR file at `url` with headless Chrome and returns the
generated PDF. The output carries `/Producer (Skia/PDF m142)` and a
`HeadlessChrome/142.0.0.0` creator string, so the PDF is generated, not stored
by the SEC. Rendered PDFs are cached. The `url` must start with
`https://www.sec.gov/`. Any EDGAR document works: a primary 10-K, 10-Q or 8-K,
or an exhibit such as EX-21 or EX-99. The source file can be HTML, XML or text.

The PDF keeps the layout of the source file, with its images and its tables.
The renderer removes the invisible inline XBRL tags. It scales the images for
print quality.

## When to use it

- I want a page-faithful copy of this filing to archive or to print.
- I need to hand a human a single PDF of one exhibit.
- I want the filing with its tables and layout intact, not as flat text.

## When to use a different tool

| Situation                             | Better tool                             | Why                                                                   |
| ------------------------------------- | --------------------------------------- | --------------------------------------------------------------------- |
| An agent has to read the content      | [`extractor`](./extractor.md)           | Returns clean text. A PDF byte stream is useless to a language model. |
| You want the file as the SEC holds it | [`get-edgar-file`](./get-edgar-file.md) | Returns the original HTML, XML or text source.                        |
| You want the numbers                  | [`xbrl-to-json`](./xbrl-to-json.md)     | Returns parsed financial statements.                                  |
| You do not know the file URL yet      | [`filing-search`](./filing-search.md)   | Returns `linkToFilingDetails` and the document URLs.                  |

## Input

| Parameter | Type   | Required | Constraints                                         | Notes                                        |
| --------- | ------ | -------- | --------------------------------------------------- | -------------------------------------------- |
| `url`     | string | yes      | `format: uri`. Must match `https://www.sec.gov/...` | The EDGAR file URL of the filing or exhibit. |

The schema sets `additionalProperties: true`, so other keys are accepted and
passed through. No other key is documented.

This tool takes no `query`, so there are no Lucene fields.

## Output

One MCP `text` content block. There is no JSON envelope. There is no `total`,
no `data[]`, and no metadata of any kind. The block is the PDF file itself.

The block carries no JSON fields. Its fields are the objects inside the PDF.
The tables below list every one of them.

### Block

| Element               | Type   | Meaning                                                  |
| --------------------- | ------ | -------------------------------------------------------- |
| Content block         | string | The generated PDF file. The REST route declares the content type `application/pdf`. |
| First line            | string | `%PDF-1.4`, the PDF version header                       |
| Second line           | string | A comment of four high bytes. It tells a reader that the file holds binary data. |
| Block size            | bytes  | 68,147 bytes for Apple's EX-21.1 exhibit                 |
| Encoding              | string | PDF bytes decoded as UTF-8, so binary sections are lossy |
| Content type declared | none   | The MCP block is typed `text`                            |

### Document information

| Element         | Type   | Meaning                                                        |
| --------------- | ------ | -------------------------------------------------------------- |
| `/Title`        | string | Title of the PDF. `Document` for every render.                 |
| `/Creator`      | string | The browser that laid out the source file.                     |
| `/Producer`     | string | The library that wrote the PDF file, `Skia/PDF m142`.          |
| `/CreationDate` | string | When the PDF was generated, in PDF date form `D:YYYYMMDDhhmmss+00'00'`. It is not the filing date. |
| `/ModDate`      | string | Last change of the PDF. It matches `/CreationDate`.            |

### Catalog and pages

| Element                              | Type      | Meaning                                                    |
| ------------------------------------ | --------- | ---------------------------------------------------------- |
| `/Type /Catalog`                     | object    | Root object of the file. It points to the page tree, the tag tree, the viewer settings and the language. |
| `/Pages`                             | reference | The page tree node of the catalog.                         |
| `/Lang`                              | string    | Language of the document, for example `en-US`.             |
| `/MarkInfo /Marked`                  | boolean   | `true` when the PDF carries tags for screen readers.       |
| `/ViewerPreferences /DisplayDocTitle` | boolean  | `true` tells the reader to show `/Title` in its title bar. |
| `/Type /Pages`                       | object    | The page tree node.                                        |
| `/Count`                             | number    | Number of pages in the PDF.                                |
| `/Kids`                              | array     | One reference per page.                                    |
| `/Type /Page`                        | object    | One rendered page.                                         |
| `/MediaBox`                          | number[]  | Page box in points. `[0 0 595.91998 842.88]` is A4.        |
| `/Contents`                          | reference | The drawing commands of the page.                          |
| `/Parent`                            | reference | The page tree node above the page.                         |
| `/StructParents`                     | number    | Key of the page in `/ParentTree`.                          |
| `/Tabs`                              | name      | Tab order of the page. `/S` means follow the tag tree.     |
| `/Resources`                         | object    | Everything the page draws with.                            |
| `/Resources /ProcSet`                | name[]    | `[/PDF /Text /ImageB /ImageC /ImageI]`. The operator sets the page needs. |
| `/Resources /Font`                   | object    | The fonts of the page, keyed `/F4`, `/F5`, `/F6`.          |
| `/Resources /ExtGState`              | object    | The graphics states of the page, keyed `/G3`.              |
| `/ca`                                | number    | Fill opacity of a graphics state. `1` is solid.            |
| `/BM`                                | name      | Blend mode of a graphics state, `/Normal`.                 |

### Streams

| Element    | Type   | Meaning                                                            |
| ---------- | ------ | ------------------------------------------------------------------ |
| `/Filter`  | name   | Compression of the stream. `/FlateDecode` is deflate.              |
| `/Length`  | number | Byte length of the compressed stream.                              |
| `/Length1` | number | Byte length of the font program before compression. Only embedded font streams carry it. |

### Tag tree

| Element                | Type                       | Meaning                                                |
| ---------------------- | -------------------------- | ------------------------------------------------------ |
| `/Type /StructTreeRoot` | object                    | Root of the tag tree. It holds the reading order of the document. |
| `/ParentTreeNextKey`   | number                     | Next free key in `/ParentTree`.                        |
| `/Type /ParentTree`    | object                     | Map from the marked content of a page to its tags.     |
| `/Nums`                | array                      | The entries of that map.                               |
| `/Type /StructElem`    | object                     | One tagged element.                                    |
| `/S`                   | name                       | Role of the element. `/Document` for the whole file, `/Div` for a block, `/Table` for a table, `/TR` for a row, `/TD` for a cell, `/NonStruct` for a wrapper with no role. |
| `/P`                   | reference                  | The parent element.                                    |
| `/Pg`                  | reference                  | The page the element is drawn on.                      |
| `/K`                   | reference, array or number | The children of the element, or the id of its marked content on the page. |
| `/A`                   | object[]                   | Attributes of a table cell. Each entry names `/O /Table` as its owner. |
| `/Headers`             | array                      | The header cells that describe this cell. Empty when the source table has no header row. |
| `/RowSpan`             | number                     | Rows the cell spans.                                   |
| `/ColSpan`             | number                     | Columns the cell spans.                                |

**This tool has no pagination.** There is no `from`, no `size`, and no `sort`.
You get the whole PDF or nothing. Size follows the source document. The Apple
EX-21.1 exhibit gave 68 KB. The Apple 10-K primary document is 1.5 MB and the
complete submission file is 9.4 MB, per the `filing-search` metadata for the
same filing. A long 10-K can pass 10 MB.

## Example

Prompt: "Get Apple's Exhibit 21.1 subsidiary list from the 2025 10-K as a PDF."

```json
{
  "name": "filing-to-pdf",
  "arguments": {
    "url": "https://www.sec.gov/Archives/edgar/data/320193/000032019325000079/a10-kexhibit21109272025.htm"
  }
}
```

```text
%PDF-1.4
1 0 obj
<</Title (Document)
/Creator (Mozilla/5.0 \(X11; Linux x86_64\) AppleWebKit/537.36 \(KHTML, like Gecko\) HeadlessChrome/142.0.0.0 Safari/537.36)
/Producer (Skia/PDF m142)
/CreationDate (D:20251101180547+00'00')
/ModDate (D:20251101180547+00'00')>>
endobj
3 0 obj
<</ca 1
/BM /Normal>>
endobj
```

The second line of the real response is a binary marker and is omitted above.
Everything else is verbatim from the response.

## Limits and errors

- A `url` that is not on `www.sec.gov` returns HTTP 404 with `Invalid filing
URL`.
- If the PDF is not in the cache, the REST route answers HTTP 202 with
  `Cache miss: PDF generation started. Retry in 5 seconds.`
- An old text filing, `.txt`, can be wider than the A4 page. Its lines then
  break. Use [`get-edgar-file`](./get-edgar-file.md) for those documents.
- Every call is billed on response size. A 10 MB PDF costs 10 MB of bandwidth.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`get-edgar-file`](./get-edgar-file.md). The same file in its original format.
- [`extractor`](./extractor.md). One section as clean text.
- [`xbrl-to-json`](./xbrl-to-json.md). The financial statements as JSON.
- [Response format](../response-format.md). Why this is a text block.
- REST docs. [Filing Render & Download API](https://sec-api.io/docs/sec-filings-render-api)
