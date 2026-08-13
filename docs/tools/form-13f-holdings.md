# form-13f-holdings

Search the equity positions that institutional investment managers report on
Form 13F.

|                 |                                                                                   |
| --------------- | --------------------------------------------------------------------------------- |
| Category        | Ownership and insiders                                                            |
| Required input  | `query`                                                                           |
| Returns         | `{total, data[]}`. One item per 13F filing, with a nested `holdings[]` array.      |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                                                   |
| REST equivalent | `POST /form-13f/holdings`                                                         |

## What it does

Form 13F is the quarterly report of an institutional investment manager that
holds more than $100 million in US-listed equities. This tool searches the
holdings index built from those filings. One item in `data[]` is one filing, not
one position. The positions sit in the `holdings[]` array inside each item.

A request for `cik:1067983` (Berkshire Hathaway) with `size: 1` returned
`total.value: 210` and one 13F-HR filing for the period ending
2026-03-31. That single filing carried **90 holdings and 28,316 bytes**. The
registry description says the tool returns "individual holding rows". The
response contradicts this. Read `data[i].holdings` to reach the positions.

## When to use it

- What did Berkshire Hathaway hold at the end of the last quarter?
- How many shares of one CUSIP does a manager hold, and at what market value?
- Does the manager have sole or shared voting power over a position?
- How did one manager's position in one stock change over four quarters?

## When to use a different tool

| Situation                                        | Better tool                                            | Why                                                                             |
| ------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------- |
| You want one row per filing and the total value  | [`form-13f-cover-pages`](./form-13f-cover-pages.md)    | Cover pages carry `tableValueTotal` and the manager's address, with no position list. |
| You track stakes above 5% of a company           | [`form-13d-13g`](./form-13d-13g.md)                    | 13D and 13G report beneficial ownership as a percent of the class.               |
| You want mutual fund or ETF holdings             | [`form-nport`](./form-nport.md)                        | N-PORT covers registered funds and reports monthly. 13F covers managers, quarterly. |
| You only want the filing metadata or the raw XML | [`filing-search`](./filing-search.md)                  | `filing-search` returns links without the position payload.                      |

## Input

| Parameter | Type    | Required | Constraints                          | Notes                                                        |
| --------- | ------- | -------- | ------------------------------------ | ------------------------------------------------------------ |
| `query`   | string  | yes      | must contain `:`, max 1,000 characters | Lucene syntax. See [query language](../query-language.md).  |
| `from`    | integer | no       | 0 to 10000                           | Offset into the result set.                                   |
| `size`    | integer | no       | 1 to 50                              | Number of **filings**, not positions. **Defaults to 50** when you omit it. |
| `sort`    | array   | no       | array of sort objects                | Defaults to `[{"filedAt": {"order": "desc"}}]`.               |

**The server rewrites your query.** If the query string does not contain the
text `13F`, the server prepends `formType:"13F-HR" AND ` to it. The query
`cik:1067983` runs as `formType:"13F-HR" AND cik:1067983`. Put `13F` in your own
query to keep control, for example when you want amendments. This tool searches
the general filings index, the same one [`filing-search`](./filing-search.md)
uses.

Query fields: `cik`, `periodOfReport`, `filedAt`, `accessionNo`, `formType`,
`isAmendment`, `holdings.ticker`, `holdings.cusip`, `holdings.cik`,
`holdings.putCall` and `holdings.shrsOrPrnAmt.sshPrnamtType`. `cik` is the CIK
of the filing manager. `holdings.cik` is the CIK of the held company. The
example uses `cik:1067983`.

## Output

The envelope is `{total, data[]}`. `total` is an object, not a number. Read
`total.value`. `total.relation` is `"eq"` for an exact count, `"gte"` at 10000
for 10,000 or more.

### Envelope

| Field            | Type   | Meaning                                                    |
| ---------------- | ------ | ---------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                     |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many. |
| `data[]`         | array  | The matching 13F filings. One item per filing.               |

### Filing

| Field                                        | Type   | Meaning                                                                                             |
| -------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| `data[].id`                                  | string | Internal ID of the filing object. Two objects can share an accession number when a filing names several entities. |
| `data[].accessionNo`                         | string | EDGAR accession number of the filing. Joins to `form-13f-cover-pages`.                               |
| `data[].cik`                                 | string | CIK of the filing manager, leading zeros removed.                                                     |
| `data[].ticker`                              | string | Ticker of the filing manager. Absent when the manager is not publicly traded.                         |
| `data[].companyName`                         | string | Name of the filing manager.                                                                           |
| `data[].companyNameLong`                     | string | Manager name with the EDGAR role, such as `BERKSHIRE HATHAWAY INC (Filer)`.                          |
| `data[].formType`                            | string | EDGAR form type, such as `13F-HR` or `13F-HR/A`.                                                      |
| `data[].description`                         | string | Text description of the form type.                                                                    |
| `data[].filedAt`                             | string | Time EDGAR accepted the filing, ISO 8601 with an offset.                                              |
| `data[].periodOfReport`                      | string | End of the quarter the holdings cover, `YYYY-MM-DD`.                                                  |
| `data[].effectivenessDate`                   | string | Date the filing became effective, `YYYY-MM-DD`.                                                       |
| `data[].linkToTxt`                           | string | URL of the complete submission text file.                                                             |
| `data[].linkToHtml`                          | string | URL of the EDGAR filing index page.                                                                   |
| `data[].linkToXbrl`                          | string | URL of the XBRL instance file. It is an empty string for 13F.                                         |
| `data[].linkToFilingDetails`                 | string | URL of the cover page document on sec.gov.                                                            |
| `data[].entities[]`                          | array  | Every entity named in the filing. The first item is the filer.                                        |
| `data[].documentFormatFiles[]`               | array  | Every primary file of the submission, exhibits included.                                              |
| `data[].dataFiles[]`                         | array  | Data files of the submission, mostly XBRL. Empty for 13F.                                             |
| `data[].seriesAndClassesContractsInformation[]` | array | Fund series and class data. Empty for 13F.                                                        |
| `data[].holdings[]`                          | array  | The positions. One item per line of the information table.                                            |

### `data[].entities[]`

| Field                                  | Type   | Meaning                                                                  |
| -------------------------------------- | ------ | ------------------------------------------------------------------------- |
| `entities[].companyName`               | string | Name of the entity with its EDGAR role.                                    |
| `entities[].cik`                       | string | CIK of the entity. Leading zeros stay here.                                |
| `entities[].irsNo`                     | string | IRS employer number of the entity.                                         |
| `entities[].stateOfIncorporation`      | string | Two-letter code of the state the entity is incorporated in.                |
| `entities[].fiscalYearEnd`             | string | Fiscal year end as `MMDD`, such as `1231`.                                 |
| `entities[].type`                      | string | Form type the entity filed. Same value as `formType`.                      |
| `entities[].act`                       | string | SEC act the filing was made under, such as `34`.                           |
| `entities[].fileNo`                    | string | Filer number of the entity, such as `028-04545`.                           |
| `entities[].filmNo`                    | string | Film number EDGAR assigned to the entity.                                  |
| `entities[].sic`                       | string | SIC code and industry name. It holds HTML entities such as `&amp;`.        |
| `entities[].undefined`                 | string | Overflow from the SIC line of the EDGAR header. Holds the tail of the office label, such as `02 Finance)`. |

### `data[].documentFormatFiles[]` and `data[].dataFiles[]`

Both arrays hold the same five keys.

| Field           | Type   | Meaning                                                     |
| --------------- | ------ | ------------------------------------------------------------ |
| `[].sequence`   | string | Order of the file in the submission. A space marks the complete text file. |
| `[].description`| string | Text description of the file, such as `INFORMATION TABLE FOR FORM 13F`. Absent on some rows. |
| `[].documentUrl`| string | URL of the file on sec.gov.                                   |
| `[].type`       | string | Type of the file, such as `13F-HR` or `INFORMATION TABLE`.    |
| `[].size`       | string | Size of the file in bytes. Rendered copies hold a space.      |

### `data[].seriesAndClassesContractsInformation[]`

Empty for 13F filings. Other form types fill it.

| Field                                        | Type   | Meaning                                      |
| -------------------------------------------- | ------ | --------------------------------------------- |
| `[].series`                                  | string | Series ID, such as `S000001297`.               |
| `[].name`                                    | string | Name of the entity that holds the series.      |
| `[].classesContracts[]`                      | array  | The classes or contracts of the series.        |
| `[].classesContracts[].classContract`        | string | Class or contract ID, such as `C000011787`.    |
| `[].classesContracts[].name`                 | string | Name of the class or contract.                 |
| `[].classesContracts[].ticker`               | string | Ticker of the class or contract.               |

### `data[].holdings[]`

| Field                                     | Type   | Meaning                                                                                     |
| ----------------------------------------- | ------ | --------------------------------------------------------------------------------------------- |
| `holdings[].nameOfIssuer`                 | string | Name of the issuer of the security, as it appears in the Official List of Section 13F Securities. |
| `holdings[].titleOfClass`                 | string | Title of the class, such as `COM` or `CAP STK CL A`.                                           |
| `holdings[].cusip`                        | string | Nine-digit CUSIP number of the security.                                                       |
| `holdings[].value`                        | number | Market value of the position in US dollars. Divide by `sshPrnamt` for the price per share.     |
| `holdings[].shrsOrPrnAmt`                 | object | Amount and type of the security held.                                                          |
| `holdings[].shrsOrPrnAmt.sshPrnamt`       | number | Number of shares of the class, or the principal amount of the class.                           |
| `holdings[].shrsOrPrnAmt.sshPrnamtType`   | string | `SH` for shares. `PRN` for a principal amount.                                                 |
| `holdings[].putCall`                      | string | `Put` or `Call`. Present only when the position is an option.                                  |
| `holdings[].investmentDiscretion`         | string | Nature of the investment discretion the manager holds over the position, such as `DFND`. The form defines the other codes. |
| `holdings[].otherManager`                 | string | Sequence numbers of the other managers that share the discretion, comma separated, such as `2,4,11`. Resolve them through `form-13f-cover-pages`. |
| `holdings[].votingAuthority`              | object | Voting authority over the position. Note the capital keys.                                     |
| `holdings[].votingAuthority.Sole`         | number | Shares over which the manager votes alone.                                                     |
| `holdings[].votingAuthority.Shared`       | number | Shares over which the manager shares the vote.                                                 |
| `holdings[].votingAuthority.None`         | number | Shares over which the manager has no vote.                                                     |
| `holdings[].ticker`                       | string | Ticker of the held security. Not in the raw filing.                                            |
| `holdings[].cik`                          | string | CIK of the held company, leading zeros removed. Not in the raw filing.                         |

The same issuer can appear several times in `holdings[]`. Berkshire reports
`ALLY FINL INC` six times, once per `otherManager` group. Sum the lines before
you report a position.

Size behaviour: `size` counts filings, and **it defaults to 50**. One filing can
hold hundreds of positions. The Berkshire filing returned 90 positions in 28 KB.
A Bridgewater cover page reports 1,040 entries, so that filing returns far more.
**Always set `size` on this tool. Start with `size: 1`.** A default call can
fill a context window in one shot.

## Example

Prompt: "What were Berkshire Hathaway's largest holdings in its most recent 13F?"

```json
{ "name": "form-13f-holdings", "arguments": { "query": "cik:1067983", "size": 1 } }
```

```json
{
  "total": { "value": 210, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001193125-26-226661",
      "cik": "1067983",
      "formType": "13F-HR",
      "filedAt": "2026-05-15T16:06:05-04:00",
      "periodOfReport": "2026-03-31",
      "holdings": [
        {
          "nameOfIssuer": "ALLY FINL INC",
          "cusip": "02005N100",
          "value": 498992850,
          "shrsOrPrnAmt": { "sshPrnamt": 12719675, "sshPrnamtType": "SH" },
          "investmentDiscretion": "DFND",
          "votingAuthority": { "Sole": 12719675, "Shared": 0, "None": 0 },
          "otherManager": "4",
          "ticker": "ALLY",
          "cik": "40729"
        }
      ]
    }
  ]
}
```

Trimmed. The full response holds 89 more `holdings[]` entries, plus `entities[]`,
`documentFormatFiles[]` and the link fields.

## Limits and errors

- Response size is the real risk here, not errors. See the size warning above.
- A missing `query`, a query without `:`, a query over 1,000 characters, or
  `from` above 10000 all fail with HTTP 400 `Invalid request parameter
  provided.` One message covers four causes, so check all four.
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- Holdings are reported quarterly, up to 45 days after the quarter ends. The
  filing above covers the quarter ending 2026-03-31 and was filed 2026-05-15.
- Position tables start with the 2013 filings. Filing metadata goes back to
  1998 for 13F-HR, and from 1994 to 1998 for 13F-E.
- One query returns at most 10,000 filings. Narrow the query to reach more.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-13f-cover-pages`](./form-13f-cover-pages.md)
- [`form-13d-13g`](./form-13d-13g.md)
- [`form-nport`](./form-nport.md)
- REST docs: [Form 13F API](https://sec-api.io/docs/form-13-f-filings-institutional-holdings-api)
