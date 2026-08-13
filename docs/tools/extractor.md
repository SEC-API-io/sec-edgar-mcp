# extractor

Extract one named section from a 10-K, 10-Q or 8-K filing.

|                 |                                                        |
| --------------- | ------------------------------------------------------ |
| Category        | Filings and documents                                  |
| Required input  | `url`, `item`                                          |
| Returns         | the section as plain text or HTML in one text block, no JSON envelope |
| Pagination      | **None.** No `from`, `size` or `sort`.                                                   |
| REST equivalent | `GET /extractor?url=<filing URL>&item=<item>&type=<text\|html>` |

## What it does

The server locates the named item inside the filing and returns only that
section. It supports three form types: 10-K, 10-Q and 8-K, and their variants.
Any other form type is rejected. Coverage starts with filings from 1994. Each
form type has its own item vocabulary, and the vocabularies do not overlap. The
`url` must start with `https://www.sec.gov/` and must point at the primary
filing document, not at an exhibit. Output is cleaned text by default, or the
original HTML of the section.

One request pulled Item 1A, Risk Factors, from Apple's fiscal 2025 10-K. The
section came back as 69,877 bytes of text.

## When to use it

- What are the risk factors in this 10-K?
- What does MD&A say about margins this quarter?
- What did this 8-K report under Item 5.02?
- I need the Legal Proceedings section, not the whole filing.

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| The form is not 10-K, 10-Q or 8-K, or you want an exhibit | [`get-edgar-file`](./get-edgar-file.md) | `extractor` rejects every other form and cannot read exhibits. |
| You want financial figures | [`xbrl-to-json`](./xbrl-to-json.md) | Item 8 as text is not machine-readable. XBRL is. |
| You want a printable copy | [`filing-to-pdf`](./filing-to-pdf.md) | Renders the filing to PDF. |
| You want structured 8-K event data | [`form-8k`](./form-8k.md) | Returns parsed 8-K items as fields, searchable across companies. |

## Input

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `url` | string | yes | `format: uri`. Must match `https://www.sec.gov/...` | The primary 10-K, 10-Q or 8-K document. The `.htm` or the `.txt` version. |
| `item` | string | yes | enum, 68 values, listed below | Must belong to the form type at `url`. |
| `type` | string | no | enum: `text`, `html`. Default `text` | `html` keeps the original markup of the section. |

The schema sets `additionalProperties: true`. No other key is documented. This
tool takes no `query`, so there are no Lucene fields.

### `item` values

**10-K.** `1` Business, `1A` Risk Factors, `1B` Unresolved Staff Comments,
`1C` Cybersecurity, `2` Properties, `3` Legal Proceedings, `4` Mine Safety
Disclosures, `5` Market for Registrant's Common Equity, `6` Selected Financial
Data, `7` MD&A, `7A` Market Risk, `8` Financial Statements and Supplementary
Data, `9` Changes in and Disagreements with Accountants, `9A` Controls and
Procedures, `9B` Other Information, `9C`, `10` Directors, Executive Officers and
Corporate Governance, `11` Executive Compensation, `12` Security Ownership,
`13` Certain Relationships and Related Transactions, `14` Principal Accountant
Fees and Services, `15` Exhibits and Financial Statement Schedules. The server
accepts `9C`, but no section title is documented for it.

**10-Q.** `part1item1` Financial Statements, `part1item2` MD&A,
`part1item3` Market Risk, `part1item4` Controls and Procedures,
`part2item1` Legal Proceedings, `part2item1a` Risk Factors,
`part2item2` Unregistered Sales of Equity Securities, `part2item3` Defaults Upon
Senior Securities, `part2item4` Mine Safety Disclosures, `part2item5` Other
Information, `part2item6` Exhibits.

**8-K.** The value `X-Y` maps to SEC Item X.0Y. For example `2-2` is Item 2.02,
Results of Operations and Financial Condition, and `5-2` is Item 5.02, Departure
of Directors or Certain Officers. The one exception is `6-10`, which maps to
Item 6.10. Full set: `1-1`, `1-2`, `1-3`, `1-4`, `1-5`, `2-1`, `2-2`, `2-3`,
`2-4`, `2-5`, `2-6`, `3-1`, `3-2`, `3-3`, `4-1`, `4-2`, `5-1`, `5-2`, `5-3`,
`5-4`, `5-5`, `5-6`, `5-7`, `5-8`, `6-1`, `6-2`, `6-3`, `6-4`, `6-5`, `6-6`,
`6-10`, `7-1`, `8-1`, `9-1`, `signature`. Only 8-K supports `signature`.

## Output

One MCP `text` content block that holds the section body. There is no JSON
envelope, no `total`, no `data[]`, no item title field and no length field.

The block carries no named fields. The `type` input decides its format. The
tables below list every element that can appear in it.

### Block

| Element | Type | Meaning |
| ------- | ---- | ------- |
| Content block | string | The one named section, from its heading to the end of the item. One call returns one item. |
| Format | text or HTML | Follows the `type` input. `text` is the default. |
| Size | bytes | Length of the section. Apple's Item 1A gave 69,877 bytes. |

### Text form, `type: text`

| Element | Type | Meaning |
| ------- | ---- | ------- |
| First line | string | The item heading as the filer wrote it, for example ` Item 1A. Risk Factors `. It keeps one leading and one trailing space. |
| Body | string | The section as clear text. Every XBRL, XML and HTML tag is stripped. |
| Paragraph | string | One paragraph is one line, however long the paragraph is. |
| Paragraph separator | string | A blank line between two paragraphs. |
| Character entities | string | **Not decoded.** The text still contains numeric HTML entities such as `&#160;` for a non-breaking space, `&#8217;` for an apostrophe and `&#8220;` for a quotation mark. Decode them yourself. |
| `##TABLE_START` | marker | The line before the first row of a table in the source filing. |
| `##TABLE_END` | marker | The line after the last row of that table. The lines between the two markers are the table. |

### HTML form, `type: html`

| Element | Type | Meaning |
| ------- | ---- | ------- |
| Body | string | The original HTML of the item, cleaned. It keeps the tables of the section. |
| Heading element | string | The item heading in its own element, for example `<p>RISK FACTORS</p>`. |
| `##TABLE_START`, `##TABLE_END` | none | The table markers belong to the text form. HTML output marks a table with `<table>` and its rows and cells. |

### Status value

| Element | Type | Meaning |
| ------- | ---- | ------- |
| `processing` | string | The whole block is this one word. The server is still extracting the sections of a new filing, or the item may not exist in the filing. Wait 500 to 1000 ms and call again. After three tries the section is not extractable. |

**This tool has no pagination.** There is no `from`, no `size` and no `sort`. The
whole section arrives in one block. Risk Factors and MD&A are the longest items
in most filings, and 50 KB to 150 KB is normal for a large filer.

## Example

Prompt: "What risk factors did Apple list in its 2025 10-K?"

```json
{
  "name": "extractor",
  "arguments": {
    "url": "https://www.sec.gov/Archives/edgar/data/320193/000032019325000079/aapl-20250927.htm",
    "item": "1A",
    "type": "text"
  }
}
```

```text
 Item 1A. Risk Factors

The following summarizes factors that could have a material adverse effect on
the Company&#8217;s business, reputation, results of operations, financial
condition and stock price. The Company may not be able to accurately predict,
control or mitigate these risks.

Macroeconomic and Industry Risks
```

Paragraphs are single long lines in the real response. They are wrapped above.

## Limits and errors

- A `url` that is not on `www.sec.gov` returns HTTP 404 with `not a valid SEC
  filing URL`. A filing that is not a 10-K, 10-Q or 8-K returns HTTP 404 with
  `filing type not supported`.
- An `item` that belongs to another form type returns HTTP 404 with, for
  example, `10-K item type not supported. Supported items are: 1, 1A, 1B, ...`.
  Sending `part2item1a` to a 10-K URL fails this way.
- The server defaults `item` to `1A` when it is missing, but the MCP schema
  marks `item` as required, so always send it.
- One call returns one item. Ask for each item separately.
- Filings from before 2002 have no standard structure, so extraction of these
  filings can fail. In about 1 filing in 1,000 the filer merges two or more
  sections into one section. These merged sections are not supported.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`get-edgar-file`](./get-edgar-file.md). The full document as filed.
- [`filing-to-pdf`](./filing-to-pdf.md). The full document as PDF.
- [`xbrl-to-json`](./xbrl-to-json.md). The numbers behind Item 8.
- [`filing-search`](./filing-search.md). Find the filing URL first.
- REST docs. [10-K/10-Q/8-K Section Extractor API](https://sec-api.io/docs/sec-filings-item-extraction-api)
