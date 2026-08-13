# xbrl-to-json

Convert the XBRL data in a filing into normalized JSON financial statements.

|                 |                                                          |
| --------------- | -------------------------------------------------------- |
| Category        | Filings and documents                                    |
| Required input  | none in the schema. In practice one of `htm-url`, `xbrl-url`, `accession-no` |
| Returns         | a bare JSON object keyed by statement and disclosure name. No `total`, no `data[]` |
| Pagination      | **None.** No `from`, `size` or `sort`.                                                     |
| REST equivalent | `GET /xbrl-to-json?accession-no=<accession number>`      |

## What it does

The server reads the XBRL instance attached to a filing and returns every tagged
fact, grouped under the filing's own statement and disclosure names. One
top-level key is one statement or one note. One value is either a plain string or
a list of fact objects, one per period and per reporting dimension.

A request for Apple's fiscal 2025 10-K, accession
`0000320193-25-000079`, returned **84 top-level keys** and **1,313,231 bytes**
of JSON. It covers 10-K and 10-Q filings, and also 20-F, 40-F, S-1, POS AM,
485BPOS and 8-K.

## When to use it

- What were this company's revenue, net income and total assets last year?
- I need the balance sheet as numbers, not as a table in HTML.
- Who audited this filer, and where is the audit firm based?
- What did the cash flow statement report for each of the last three years?

## When to use a different tool

| Situation | Better tool | Why |
| --------- | ----------- | --- |
| You want narrative, not numbers | [`extractor`](./extractor.md) | Returns Risk Factors or MD&A as text. |
| You want the raw XBRL instance | [`get-edgar-file`](./get-edgar-file.md) | Returns the `_htm.xml` file untouched. |
| You want one metric across many companies | [`filing-search`](./filing-search.md) then this tool | This tool handles one filing per call. |
| You want share counts and public float only | [`float`](./float.md) | Far smaller response. |

## Input

One of the three inputs identifies the filing. The server uses the first one
present, in this order: `htm-url`, `xbrl-url`, `accession-no`.

| Parameter | Type | Required | Constraints | Notes |
| --------- | ---- | -------- | ----------- | ----- |
| `htm-url` | string | no | none in the schema | URL of the primary filing document. |
| `xbrl-url` | string | no | none in the schema | URL of the XBRL instance file. |
| `accession-no` | string | no | none in the schema | Dashed accession number, for example `0000320193-25-000079`. |

The schema marks nothing as required. That contradicts the server, which returns
HTTP 400 `Invalid request parameter` when all three are missing. One of the
three is mandatory in practice. The schema sets `additionalProperties: true`.
This tool takes no `query`, so there are no Lucene fields.

## Output

A JSON object. The top-level keys are the filing's own statement and disclosure
names, so **they differ between filers and between years**. A key in one filing
can be absent in the next.

| Key seen in the response | Type | Meaning |
| ----------------------- | ---- | ------- |
| `CoverPage` | object | Filer identity and document metadata. Holds `DocumentType`, `DocumentPeriodEndDate`, `EntityRegistrantName`, `EntityCentralIndexKey`, `EntityFilerCategory`, `TradingSymbol` and 30 more. |
| `AuditorInformation` | object | `AuditorName`, `AuditorLocation`, `AuditorFirmId`. All plain strings. |
| `StatementsOfIncome` | object | 15 line items, from `RevenueFromContractWithCustomerExcludingAssessedTax` to `EarningsPerShareDiluted`. |
| `BalanceSheets` | object | 30 line items, from `CashAndCashEquivalentsAtCarryingValue` to `LiabilitiesAndStockholdersEquity`. |
| `StatementsOfCashFlows` | object | 28 line items. `StatementsOfComprehensiveIncome` has 13 and `StatementsOfShareholdersEquity` has 9. |
| `InsiderTradingArrangements` | object | Rule 10b5-1 adoption and termination flags. |
| `Cybersecurity*` | string | Item 1C disclosures. 15 keys in this response, 9 ending in `TextBlock` and 6 in `Flag`. |

Line-item values are lists of fact objects. A fact object has these fields.

| Field | Type | Meaning |
| ----- | ---- | ------- |
| `value` | string | The reported figure. **Always a string**, never a number. |
| `unitRef`, `decimals` | string | The unit, for example `usd` or `shares`, and the XBRL rounding, for example `-6` for millions. |
| `period.startDate`, `period.endDate` | string | The range, for flow items such as revenue. |
| `period.instant` | string | The date, for stock items such as cash. |
| `segment.explicitMember` | object | `dimension` is the XBRL axis, for example `srt:ProductOrServiceAxis`. `$t` is the member, for example `us-gaap:ProductMember`. |

A fact with no `segment` is the consolidated total. A fact with a `segment` is a
breakdown. You may see `segment` described as `{dimension, value}` elsewhere.
The correct shape is `{explicitMember: {dimension, $t}}`.

**This tool has no pagination.** There is no `from`, no `size`, no `sort` and no
way to ask for one statement only. The response holds all 84 keys or nothing. A
large 10-K returns 1 MB or more of context per call.

## Example

Prompt: "Pull Apple's fiscal 2025 financial statements from the 10-K."

```json
{ "name": "xbrl-to-json", "arguments": { "accession-no": "0000320193-25-000079" } }
```

```json
{
  "CoverPage": {
    "DocumentType": "10-K",
    "DocumentPeriodEndDate": "2025-09-27",
    "EntityRegistrantName": "Apple Inc."
  },
  "AuditorInformation": { "AuditorName": "Ernst & Young LLP" },
  "StatementsOfIncome": {
    "NetIncomeLoss": [
      {
        "decimals": "-6",
        "unitRef": "usd",
        "period": { "startDate": "2024-09-29", "endDate": "2025-09-27" },
        "value": "112010000000"
      }
    ]
  },
  "BalanceSheets": {
    "AssetsCurrent": [{ "period": { "instant": "2025-09-27" }, "value": "147957000000" }]
  }
}
```

Four of 84 keys are shown. Values are verbatim from the response.

## Limits and errors

- Sending none of the three inputs returns HTTP 400 with `Invalid request
  parameter`. A filing with no XBRL data cannot be converted.
- If the conversion is not cached, the server answers HTTP 202 with `XBRL
  conversion started, but the processing has not been completed. Please try
  again after 60 seconds.`
- Every call is billed on response size. At 1.31 MB per 10-K, a loop over 100
  filings costs about 130 MB.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`extractor`](./extractor.md). The narrative sections of the same filing.
- [`get-edgar-file`](./get-edgar-file.md). The raw XBRL instance file.
- [`float`](./float.md) and [`filing-search`](./filing-search.md). Share counts, and the accession number you need here.
- REST docs. [XBRL-to-JSON Converter API](https://sec-api.io/docs/xbrl-to-json-converter-api)
