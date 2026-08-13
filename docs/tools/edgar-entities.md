# edgar-entities

Search EDGAR's entity master, the profile record of every filer registered with
the SEC.

|                 |                                                                                               |
| --------------- | --------------------------------------------------------------------------------------------- |
| Category        | Company and entity                                                                            |
| Required input  | `query`                                                                                       |
| Returns         | `{total, data[]}`                                                                             |
| Pagination      | `from` (0 to 10000), `size` (1 to 50, default 50), `sort` (default `cikUpdatedAt` descending) |
| REST equivalent | `POST /edgar-entities`                                                                        |

## What it does

Every filer that EDGAR knows has one record here: operating companies, funds,
foreign issuers and individuals. One row is one CIK. The row holds the filer's
name, addresses, SIC code, auditor, filer category and the set of form types it
has filed. Almost every field is paired with an `<field>UpdatedAt` timestamp
that says when EDGAR last changed it. A query returns the profile of one filer,
or the list of filers that share an attribute.

The database holds more than 890,000 entities and starts in 1994. It keeps the
entities that stopped filing, so a search covers dead filers too. New filings
update a record in real time.

## When to use it

- What is the CIK, SIC code and state of incorporation of Apple?
- Who audits this company, and where is the audit office?
- Which filers are incorporated in Delaware?
- Which companies does the SEC classify as shell companies?
- Which filers has the Office of Technology assigned to itself?

## When to use a different tool

| Situation                               | Better tool                           | Why                                                                            |
| --------------------------------------- | ------------------------------------- | ------------------------------------------------------------------------------ |
| You only need to turn a name into a CIK | [`mapping`](./mapping.md)             | One call, no Lucene, and it also returns the ticker and CUSIP.                 |
| You want filings, not filer profiles    | [`filing-search`](./filing-search.md) | This tool tells you which form types a filer uses, not the individual filings. |
| You want subsidiaries of a company      | [`subsidiaries`](./subsidiaries.md)   | Most subsidiaries never register with the SEC, so they have no entity record.  |

## Input

| Parameter | Type    | Required | Constraints                                       | Notes                                                                  |
| --------- | ------- | -------- | ------------------------------------------------- | ---------------------------------------------------------------------- |
| `query`   | string  | yes      | Lucene, max 1000 characters, must contain a colon | `cik:320193`. A bare word is rejected.                                 |
| `from`    | integer | no       | 0 or greater                                      | Values above 10000 return an empty result, not an error.               |
| `size`    | integer | no       | 1 to 50, default 50                               | Above 50 the server returns an error.                                  |
| `sort`    | array   | no       | Sort clause                                       | Format `[{"<field>":"<order>"}]`. Default `[{"cikUpdatedAt":"desc"}]`. |

Every response field is searchable, and so is every `<field>UpdatedAt`
timestamp. Common ones:

| Field                                                                                 | Example                                                       |
| ------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `cik`                                                                                 | `cik:320193`                                                  |
| `name`                                                                                | `name:"Tesla"`                                                |
| `sic`, `sicLabel`                                                                     | `sic:3571`, `sicLabel:"ELECTRONIC COMPUTERS"`                 |
| `stateOfIncorporation`                                                                | `stateOfIncorporation:DE`                                     |
| `businessAddress.state`                                                               | `businessAddress.state:CA`                                    |
| `businessAddress.city`, `businessAddress.zip`                                         | `businessAddress.city:"SAN FRANCISCO"`                        |
| `phone`, `irsNo`, `fiscalYearEnd`                                                     | `fiscalYearEnd:1231`                                          |
| `cfOffice`, `auditorName`                                                             | `cfOffice:"06 Technology"`, `auditorName:"Ernst & Young LLP"` |
| `auditorFirmId`, `auditorLocation`                                                    | `auditorFirmId:238`                                           |
| `latestIcfrAuditFiledAt`                                                              | `latestIcfrAuditFiledAt:[2024-01-01 TO 2024-12-31]`           |
| `filerCategory`                                                                       | `filerCategory:"Large Accelerated Filer"`                     |
| `shellCompany`                                                                        | `shellCompany:true`                                           |
| `smallBusiness`, `emergingGrowthCompany`, `voluntaryFiler`, `wellKnownSeasonedIssuer` | `smallBusiness:true`                                          |
| `currentReportingStatus`, `interactiveDataCurrent`                                    | `currentReportingStatus:true`                                 |
| `formTypes.10-K`                                                                      | `formTypes.10-K:true`                                         |

`mailingAddress` mirrors `businessAddress`, so `mailingAddress.state` follows
the same pattern.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with value 10000 means "10,000 or more", the search-window ceiling.
One element of `data[]` is one filer, one CIK.

### Identity and contact

| Field                  | Type   | Meaning                                                                                                                 |
| ---------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------- |
| `id`                   | string | Internal record ID. It repeats the CIK.                                                                                 |
| `cik`                  | string | Central Index Key of the filer, leading zeros removed, `320193`.                                                        |
| `name`                 | string | Current entity name in EDGAR.                                                                                           |
| `irsNo`                | string | IRS tax identification number of the entity, nine digits, `942404110`.                                                  |
| `phone`                | string | Contact phone number of the entity.                                                                                     |
| `stateOfIncorporation` | string | Two-letter code of the state under whose law the entity is formed, `DE` for Delaware. It is not the headquarters state. |
| `fiscalYearEnd`        | string | Month and day that close the fiscal year, `MMDD`. Apple returns `0926`.                                                 |

### Addresses

| Field                       | Type   | Meaning                                                                    |
| --------------------------- | ------ | -------------------------------------------------------------------------- |
| `businessAddress`           | object | Address of the principal place of business.                                |
| `businessAddress.street1`   | string | First street line.                                                         |
| `businessAddress.street2`   | string | Second street line, such as a suite or a floor. `null` when there is none. |
| `businessAddress.city`      | string | City, upper case.                                                          |
| `businessAddress.state`     | string | Two-letter postal code of the state, `CA`.                                 |
| `businessAddress.stateName` | string | Full name of that state, upper case, `CALIFORNIA`.                         |
| `businessAddress.zip`       | string | Postal code.                                                               |
| `businessAddress.country`   | string | Country of the address. Often an empty string.                             |
| `mailingAddress`            | object | Address EDGAR uses for mail. It often repeats `businessAddress`.           |
| `mailingAddress.street1`    | string | First street line.                                                         |
| `mailingAddress.street2`    | string | Second street line. `null` when there is none.                             |
| `mailingAddress.city`       | string | City, upper case.                                                          |
| `mailingAddress.state`      | string | Two-letter postal code of the state.                                       |
| `mailingAddress.stateName`  | string | Full name of that state, upper case.                                       |
| `mailingAddress.zip`        | string | Postal code.                                                               |
| `mailingAddress.country`    | string | Country of the address. Often an empty string.                             |

### Industry classification

| Field      | Type   | Meaning                                                                                                                                                      |
| ---------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `sic`      | string | Standard Industrial Classification code, four digits, `3571`.                                                                                                |
| `sicLabel` | string | The code and its description in one string, `3571 ELECTRONIC COMPUTERS`.                                                                                     |
| `cfOffice` | string | Office of the Division of Corporation Finance that handles the filer, `06 Technology`. One office covers dozens of SIC codes, so it groups a whole industry. |

### Filer status

| Field                     | Type    | Meaning                                                                                                                                        |
| ------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `filerCategory`           | string  | `Large Accelerated Filer`, `Accelerated Filer` or `Non-accelerated Filer`.                                                                     |
| `wellKnownSeasonedIssuer` | boolean | `true` if the entity is a well-known seasoned issuer under Rule 405 of the Securities Act.                                                     |
| `voluntaryFiler`          | boolean | `true` if the entity is not required to file.                                                                                                  |
| `smallBusiness`           | boolean | `true` if the entity is a Smaller Reporting Company.                                                                                           |
| `emergingGrowthCompany`   | boolean | `true` if the entity is an emerging growth company under the JOBS Act.                                                                         |
| `shellCompany`            | boolean | `true` if the entity is a shell company under Rule 12b-2 of the Exchange Act.                                                                  |
| `currentReportingStatus`  | boolean | `true` if the filer sent every report that Section 13 or 15(d) asked for in the last 12 months, and has been subject to that duty for 90 days. |
| `interactiveDataCurrent`  | boolean | `true` if the filer sent every Interactive Data File that Rule 405 of Regulation S-T asked for in the last 12 months.                          |

### Auditor and internal control

| Field                    | Type   | Meaning                                                                                                           |
| ------------------------ | ------ | ----------------------------------------------------------------------------------------------------------------- |
| `auditorName`            | string | Name of the audit firm, `Ernst & Young LLP`.                                                                      |
| `auditorFirmId`          | string | ID of that audit firm, `42`. One firm keeps one ID across every entity it audits, so you can query all of them.   |
| `auditorLocation`        | string | Office of the audit firm, city and state, `San Jose, California`.                                                 |
| `latestIcfrAuditFiledAt` | string | Date of the latest auditor attestation of internal control over financial reporting (ICFR). ISO 8601 with offset. |
| `latestIcfrAuditSource`  | string | Accession number of the filing that carried that attestation.                                                     |

### Filing history

| Field                  | Type    | Meaning                                                                                                            |
| ---------------------- | ------- | ------------------------------------------------------------------------------------------------------------------ |
| `formTypes`            | object  | Every form type the entity has filed since it registered with the SEC.                                             |
| `formTypes.<formType>` | boolean | One key per form type, `10-K`, `8-K`, `SCHEDULE 13G/A`. The value is `true`. The map holds no counts and no dates. |

### Update timestamps

Almost every field above has a twin. It records when EDGAR last changed that
one field. All are strings, ISO 8601 with an offset. You can sort and filter on
them. `id` has no twin.

| Field                              | Type   | Meaning                                                                                            |
| ---------------------------------- | ------ | -------------------------------------------------------------------------------------------------- |
| `cikUpdatedAt`                     | string | When EDGAR last touched the record. Sort on it for record freshness. It is the default sort field. |
| `nameUpdatedAt`                    | string | When the entity name last changed.                                                                 |
| `businessAddressUpdatedAt`         | string | When the business address last changed.                                                            |
| `mailingAddressUpdatedAt`          | string | When the mailing address last changed.                                                             |
| `stateOfIncorporationUpdatedAt`    | string | When the state of incorporation last changed.                                                      |
| `phoneUpdatedAt`                   | string | When the phone number last changed.                                                                |
| `irsNoUpdatedAt`                   | string | When the IRS number last changed.                                                                  |
| `fiscalYearEndUpdatedAt`           | string | When the fiscal year end last changed.                                                             |
| `sicUpdatedAt`                     | string | When the SIC code last changed.                                                                    |
| `sicLabelUpdatedAt`                | string | When the SIC label last changed.                                                                   |
| `cfOfficeUpdatedAt`                | string | When the Corporation Finance office last changed.                                                  |
| `formTypesUpdatedAt`               | string | When the form type map last changed. A form type new to the filer moves it.                        |
| `filerCategoryUpdatedAt`           | string | When the filer category last changed.                                                              |
| `wellKnownSeasonedIssuerUpdatedAt` | string | When the well-known seasoned issuer flag last changed.                                             |
| `voluntaryFilerUpdatedAt`          | string | When the voluntary filer flag last changed.                                                        |
| `smallBusinessUpdatedAt`           | string | When the smaller reporting company flag last changed.                                              |
| `emergingGrowthCompanyUpdatedAt`   | string | When the emerging growth company flag last changed.                                                |
| `shellCompanyUpdatedAt`            | string | When the shell company flag last changed.                                                          |
| `currentReportingStatusUpdatedAt`  | string | When the current reporting status last changed.                                                    |
| `interactiveDataCurrentUpdatedAt`  | string | When the interactive data flag last changed.                                                       |
| `auditorNameUpdatedAt`             | string | When the auditor name last changed.                                                                |
| `auditorFirmIdUpdatedAt`           | string | When the auditor firm ID last changed.                                                             |
| `auditorLocationUpdatedAt`         | string | When the auditor office last changed.                                                              |
| `latestIcfrAuditFiledAtUpdatedAt`  | string | When the ICFR attestation date last changed.                                                       |
| `latestIcfrAuditSourceUpdatedAt`   | string | When the ICFR accession number last changed.                                                       |

Size behaviour: `from` and `size` page the result set, 50 rows per call. Rows
are small. The single Apple row is 2,432 bytes, so a full page of 50 is roughly
100 KB.

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
      "businessAddress": {
        "street1": "ONE APPLE PARK WAY",
        "city": "CUPERTINO",
        "state": "CA",
        "stateName": "CALIFORNIA",
        "zip": "95014"
      },
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
- `nameUpdatedAt` tells you when the name last changed, but it does not
  give you the old name.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`mapping`](./mapping.md) for a one-shot identifier lookup.
- [`subsidiaries`](./subsidiaries.md) for owned entities that do not file.
- [`float`](./float.md) for the share counts of a filer.
- Lucene syntax rules: [query language](../query-language.md)
- REST documentation: [EDGAR Entities Database API](https://sec-api.io/docs/edgar-entities-database-api)
