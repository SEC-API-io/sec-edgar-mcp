# edgar-entities

Search EDGAR's entity master, the profile record of every filer registered with
the SEC.

|                 |                                                              |
| --------------- | ------------------------------------------------------------ |
| Category        | Company and entity                                           |
| Required input  | `query`                                                      |
| Returns         | `{total, data[]}`                                            |
| Pagination      | `from` (0 to 10000), `size` (1 to 50, default 50), `sort` (default `cikUpdatedAt` descending) |
| REST equivalent | `POST /edgar-entities`                                       |

## What it does

Every filer that EDGAR knows has one record here: operating companies, funds,
foreign issuers and individuals. One row is one CIK. The row holds the filer's
name, addresses, SIC code, auditor, filer category and the set of form types it
has filed. Almost every field is paired with an `<field>UpdatedAt` timestamp
that says when EDGAR last changed it. Use the tool to profile a filer, or to
list filers that share an attribute.

## When to use it

- What is the CIK, SIC code and state of incorporation of Apple?
- Who audits this company, and where is the audit office?
- Which filers are incorporated in Delaware?
- Which companies does the SEC classify as shell companies?
- Which filers has the Office of Technology assigned to itself?

## When to use a different tool

| Situation                                      | Better tool                          | Why                                                                            |
| ---------------------------------------------- | ------------------------------------ | -------------------------------------------------------------------------------- |
| You only need to turn a name into a CIK        | [`mapping`](./mapping.md)            | One call, no Lucene, and it also returns the ticker and CUSIP.                  |
| You want filings, not filer profiles           | [`filing-search`](./filing-search.md) | This tool tells you which form types a filer uses, not the individual filings.  |
| You want subsidiaries of a company             | [`subsidiaries`](./subsidiaries.md)  | Most subsidiaries never register with the SEC, so they have no entity record.   |

## Input

| Parameter | Type    | Required | Constraints                       | Notes                                                     |
| --------- | ------- | -------- | --------------------------------- | --------------------------------------------------------- |
| `query`   | string  | yes      | Lucene, max 1000 characters, must contain a colon | `cik:320193`. A bare word is rejected.    |
| `from`    | integer | no       | 0 or greater                      | Values above 10000 return an empty result, not an error.  |
| `size`    | integer | no       | 1 to 50, default 50               | Above 50 the server returns an error.                     |
| `sort`    | array   | no       | Elasticsearch sort clause         | Default `[{"cikUpdatedAt":{"order":"desc"}}]`.            |

Query fields confirmed to return rows:

| Field                     | Example                                      |
| ------------------------- | -------------------------------------------- |
| `cik`                     | `cik:320193`                                 |
| `name`                    | `name:"Tesla"`                               |
| `sic`, `sicLabel`         | `sic:3571`, `sicLabel:"ELECTRONIC COMPUTERS"` |
| `stateOfIncorporation`    | `stateOfIncorporation:DE`                    |
| `businessAddress.state`   | `businessAddress.state:CA`                   |
| `cfOffice`, `auditorName` | `cfOffice:"06 Technology"`, `auditorName:"Ernst & Young LLP"` |
| `filerCategory`           | `filerCategory:"Large Accelerated Filer"`    |
| `shellCompany`            | `shellCompany:true`                          |
| `formTypes.10-K`          | `formTypes.10-K:true`                        |

`mailingAddress` mirrors `businessAddress`, so `mailingAddress.state` should
work the same way. That one is unverified.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with value 10000 means "10,000 or more", the search-window ceiling.

| Field                     | Type    | Meaning                                                                  |
| ------------------------- | ------- | ------------------------------------------------------------------------ |
| `id`, `cik`               | string  | The CIK, without leading zeros. `id` repeats it.                         |
| `name`                    | string  | Current entity name in EDGAR.                                            |
| `businessAddress`, `mailingAddress` | object | `street1`, `city`, `state`, `stateName`, `zip`.                |
| `stateOfIncorporation`    | string  | Two-letter code, `CA`.                                                   |
| `phone`, `irsNo`          | string  | Contact phone and IRS employer number as registered.                     |
| `fiscalYearEnd`           | string  | Four digits, `MMDD`. Apple returns `0926`.                               |
| `sic`, `sicLabel`         | string  | Industry code and its label, `3571 ELECTRONIC COMPUTERS`.                |
| `cfOffice`                | string  | The SEC review office, `06 Technology`.                                  |
| `formTypes`               | object  | A map of form type to `true`. Which forms the filer has filed, not counts. |
| `filerCategory`           | string  | `Large Accelerated Filer` and similar.                                   |
| `currentReportingStatus`, `interactiveDataCurrent` | boolean | Reporting and XBRL currency.                    |
| `emergingGrowthCompany`, `shellCompany`, `smallBusiness`, `voluntaryFiler`, `wellKnownSeasonedIssuer` | boolean | Cover-page status flags. |
| `auditorName`, `auditorFirmId`, `auditorLocation` | string | The audit firm, its PCAOB ID and its office. |
| `latestIcfrAuditSource`, `latestIcfrAuditFiledAt` | string | The filing that carried the latest ICFR audit. |
| `<field>UpdatedAt`        | string  | When EDGAR last changed that one field. Present for most fields above.   |

Size behaviour: page with `from` and `size`, 50 rows per call. Rows are small.
The single Apple row is 2,432 bytes, so a full page of 50 is roughly 100 KB.

## Example

Prompt: "Give me the EDGAR profile for CIK 320193."

```json
{ "name": "edgar-entities", "arguments": { "query": "cik:320193", "size": 1 } }
```

```json
{
  "total": { "value": 1, "relation": "eq" },
  "data": [
    {
      "id": "320193",
      "cik": "320193",
      "name": "Apple Inc.",
      "businessAddress": { "street1": "ONE APPLE PARK WAY", "city": "CUPERTINO", "state": "CA", "stateName": "CALIFORNIA", "zip": "95014" },
      "stateOfIncorporation": "CA",
      "fiscalYearEnd": "0926",
      "sic": "3571",
      "sicLabel": "3571 ELECTRONIC COMPUTERS",
      "cfOffice": "06 Technology",
      "filerCategory": "Large Accelerated Filer",
      "auditorName": "Ernst & Young LLP",
      "auditorLocation": "San Jose, California"
    }
  ]
}
```

The real row also carries `phone`, `irsNo`, `formTypes`, the status flags and an
`UpdatedAt` timestamp beside each field. They are cut here for length.

## Limits and errors

- A query without a colon returns `sec-api error: Invalid Lucene query string`.
- `size` above 50 returns
  `sec-api error: Maximum 'size' limit of 50 exceeded. ...`
- `from` above 10000 returns `{"total":{"value":0},"data":[]}` with no error.
  That empty set is a paging ceiling, not an end of data.
- The tool description promises former names. No `formerNames` field exists in
  the capture or in the canonical REST response, and `formerNames.name:*`
  matched nothing. Do not rely on it.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`mapping`](./mapping.md) for a one-shot identifier lookup.
- [`subsidiaries`](./subsidiaries.md) for owned entities that do not file.
- [`float`](./float.md) for the share counts of a filer.
- Lucene syntax rules: [query language](../query-language.md)
- REST documentation: [EDGAR Entities Database API](https://sec-api.io/docs/edgar-entities-database-api)
