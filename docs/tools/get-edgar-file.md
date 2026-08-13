# get-edgar-file

Download any SEC EDGAR filing or exhibit in its original filed format.

|                 |                                                                                                        |
| --------------- | ------------------------------------------------------------------------------------------------------ |
| Category        | Filings and documents                                                                                  |
| Required input  | `url`                                                                                                  |
| Returns         | the file source in one text block, no JSON envelope                                                    |
| Pagination      | **None.** No `from`, `size` or `sort`.                                                                 |
| REST equivalent | none. The public route is `https://edgar-mirror.sec-api.io/<cik>/<accession-no-without-dashes>/<file>` |

## What it does

The server maps the EDGAR URL to its own mirror and returns the file bytes
untouched. Coverage runs from 1993 to today and includes primary documents,
exhibits, XML, XSD, XBRL, plain text, complete submission files, SGML headers,
images, PDF, Excel, Word and ZIP files. Nothing is cleaned, converted or
re-rendered. The `url` must start with `https://www.sec.gov/`. iXBRL viewer
wrappers are stripped, so `.../ix?doc=/Archives/...` and
`.../ix.xhtml?doc=/Archives/...` both resolve to the underlying document.

A request for Apple's EX-21.1 exhibit returned it complete, 11,807 bytes. The
response opened with the EDGAR SGML wrapper, `<DOCUMENT>`, `<TYPE>`,
`<SEQUENCE>`, `<FILENAME>`, `<DESCRIPTION>`, `<TEXT>`, then the HTML body, and
closed with `</TEXT></DOCUMENT>`. Expect that wrapper on document files.

## When to use it

- I want this exhibit exactly as it was filed, with no clean-up.
- I need the raw XML of a Form 4, an N-PORT or an XBRL instance.
- I want the complete submission `.txt` file for one accession number.
- I need the SGML header of a filing.

## When to use a different tool

| Situation                    | Better tool                           | Why                                                                    |
| ---------------------------- | ------------------------------------- | ---------------------------------------------------------------------- |
| You only want one section    | [`extractor`](./extractor.md)         | Returns Risk Factors or MD&A as text. Far smaller than the whole file. |
| You want something printable | [`filing-to-pdf`](./filing-to-pdf.md) | Renders the same file to PDF.                                          |
| You want financial figures   | [`xbrl-to-json`](./xbrl-to-json.md)   | Parses the XBRL instead of handing you the markup.                     |
| You do not know the file URL | [`filing-search`](./filing-search.md) | Returns the document URLs for a filing.                                |

## Input

| Parameter | Type   | Required | Constraints                                         | Notes                                                              |
| --------- | ------ | -------- | --------------------------------------------------- | ------------------------------------------------------------------ |
| `url`     | string | yes      | `format: uri`. Must match `https://www.sec.gov/...` | Any EDGAR file URL. An iXBRL viewer URL is accepted and unwrapped. |

The schema sets `additionalProperties: true`. No other key is documented.

This tool takes no `query`, so there are no Lucene fields.

## Output

One MCP `text` content block that holds the file source. There is no JSON
envelope. There is no `total`, no `data[]`, no file name, no content type and no
byte count. You get the file body and nothing else.

The block carries no named fields. Its content is the file. The tables below
list every element that can appear in it.

### Block

| Element             | Type   | Meaning                                                                                                                                                            |
| ------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Content block       | string | The file content unchanged. Nothing is cleaned, converted or re-rendered.                                                                                          |
| Text files          | string | Returned as UTF-8. Applies to `.htm`, `.html`, `.xml`, `.txt`, `.json`, `.sgml`, `.css`, `.js`, and to any response whose content type is text, HTML, XML or JSON. |
| Binary files        | string | Returned base64-encoded in the same text block.                                                                                                                    |
| Size in the example | bytes  | 11,807 bytes, the complete EX-21.1 exhibit                                                                                                                         |

### Document wrapper

Every filed document carries the EDGAR SGML wrapper. Asset files such as `.xsd`
schemas and images have no wrapper. You get their bytes only.

| Element         | Type   | Meaning                                                                          |
| --------------- | ------ | -------------------------------------------------------------------------------- |
| `<DOCUMENT>`    | tag    | Start of the document. `</DOCUMENT>` closes it on the last line.                 |
| `<TYPE>`        | string | Document type. The form type for the primary document, for example `10-K`, or the exhibit type, for example `EX-21.1`. |
| `<SEQUENCE>`    | number | Position of the document inside the submission. `1` is the primary document.     |
| `<FILENAME>`    | string | File name of the document on EDGAR.                                              |
| `<DESCRIPTION>` | string | Text the filer gave to describe the document.                                    |
| `<TEXT>`        | tag    | Start of the document body. `</TEXT>` closes it.                                 |
| `<XML>`         | tag    | Inside `<TEXT>`. Wraps an XML body, for example a Form 4 ownership document.     |
| `<XBRL>`        | tag    | Inside `<TEXT>`. Wraps an XBRL or inline XBRL body.                              |
| Body            | string | The file itself: HTML, XML, XBRL or plain text.                                  |

### Complete submission file

A `.txt` submission file holds the SGML header and every document of one filing.
The header repeats a filer block for each entity the filing names.

| Element                              | Type   | Meaning                                                                     |
| ------------------------------------ | ------ | --------------------------------------------------------------------------- |
| `<SEC-DOCUMENT>`                     | string | First line. The submission file name and the acceptance date.               |
| `<SEC-HEADER>`                       | string | The header file name and its date. `</SEC-HEADER>` closes the header.       |
| `<ACCEPTANCE-DATETIME>`              | string | When EDGAR accepted the submission, as `YYYYMMDDhhmmss`.                    |
| `ACCESSION NUMBER`                   | string | Accession number of the filing, with dashes.                                |
| `CONFORMED SUBMISSION TYPE`          | string | Form type of the submission, for example `10-K` or `4`.                     |
| `PUBLIC DOCUMENT COUNT`              | number | Number of documents in the submission.                                      |
| `CONFORMED PERIOD OF REPORT`         | string | Period the filing reports on, as `YYYYMMDD`.                                |
| `FILED AS OF DATE`                   | string | Date the filing was made, as `YYYYMMDD`.                                    |
| `DATE AS OF CHANGE`                  | string | Date EDGAR last changed the record, as `YYYYMMDD`.                          |
| `FILER`                              | block  | One entity that filed the submission.                                       |
| `REPORTING-OWNER`                    | block  | On Forms 3, 4 and 5, the insider who reports the trade.                     |
| `ISSUER`                             | block  | On Forms 3, 4 and 5, the company whose shares the insider reports.          |
| `COMPANY DATA`                       | block  | Identity of a company inside a `FILER` or `ISSUER` block.                   |
| `OWNER DATA`                         | block  | Identity of a person inside a `REPORTING-OWNER` block.                      |
| `COMPANY CONFORMED NAME`             | string | Name of the entity as EDGAR holds it.                                       |
| `CENTRAL INDEX KEY`                  | string | CIK of the entity, ten digits with leading zeros.                           |
| `STANDARD INDUSTRIAL CLASSIFICATION` | string | Industry of the entity, name and SIC code, for example `ELECTRONIC COMPUTERS [3571]`. |
| `ORGANIZATION NAME`                  | string | SEC review office assigned to the entity, for example `06 Technology`. Empty for a person. |
| `EIN`                                | string | Employer identification number of the entity.                               |
| `STATE OF INCORPORATION`             | string | State or country code the entity is incorporated in.                        |
| `FISCAL YEAR END`                    | string | Fiscal year end of the entity, as `MMDD`.                                   |
| `FILING VALUES`                      | block  | How the entity filed this submission.                                       |
| `FORM TYPE`                          | string | Form type as filed.                                                         |
| `SEC ACT`                            | string | Act the filing is made under, for example `1934 Act`.                       |
| `SEC FILE NUMBER`                    | string | File number of the entity under that act.                                   |
| `FILM NUMBER`                        | string | Number EDGAR gave the filing when it accepted it.                           |
| `BUSINESS ADDRESS`                   | block  | Address of the entity's place of business.                                  |
| `MAIL ADDRESS`                       | block  | Address EDGAR sends mail to.                                                |
| `STREET 1`, `STREET 2`               | string | Street lines of an address block.                                           |
| `CITY`                               | string | City of an address block.                                                   |
| `STATE`                              | string | State or country code of an address block.                                  |
| `ZIP`                                | string | Postal code of an address block.                                            |
| `BUSINESS PHONE`                     | string | Telephone number in a `BUSINESS ADDRESS` block.                             |
| `FORMER COMPANY`                     | block  | An earlier name of the entity. It repeats once per name change.             |
| `FORMER CONFORMED NAME`              | string | The earlier name.                                                           |
| `DATE OF NAME CHANGE`                | string | Date the name changed, as `YYYYMMDD`.                                       |
| `<DOCUMENT>` blocks                  | block  | Every document of the submission follows the header, each in the wrapper above. |

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
