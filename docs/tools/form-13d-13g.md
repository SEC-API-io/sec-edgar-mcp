# form-13d-13g

Search Schedule 13D and Schedule 13G beneficial-ownership filings.

|                 |                                                              |
| --------------- | ------------------------------------------------------------ |
| Category        | Ownership and insiders                                       |
| Required input  | `query`                                                      |
| Returns         | `{total, filings[]}`. **Not `data[]`.** One item per filing. |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`                 |
| REST equivalent | `POST /form-13d-13g`                                         |

## What it does

An investor who crosses 5% of a class of a public company's shares must file a
Schedule 13D or 13G. 13D signals an active intent. 13G signals a passive
position. This tool searches both. One item in `filings[]` is one filing. Each
filing lists one or more reporting persons in `owners[]`, with voting power,
dispositive power and percent of class.

A request for `owners.name:Point72 AND owners.amountAsPercent:[10 TO *]`
with `size: 1` returned `total.value: 8` and one SC 13D from 2022 in 1,319
bytes. That filing named two reporting persons, Point72 Private Investments and
Steven A. Cohen, each at 20.3% of Tempo Automation.

## When to use it

- Which activist investors crossed 5% of a company this quarter?
- Who owns more than 10% of a named issuer?
- What is the exact share count and percent behind a reported stake?
- Who else is in the reporting group for this position?

## When to use a different tool

| Situation                                     | Better tool                                   | Why                                                                   |
| --------------------------------------------- | --------------------------------------------- | --------------------------------------------------------------------- |
| You want a manager's full quarterly portfolio | [`form-13f-holdings`](./form-13f-holdings.md) | 13F lists every position. 13D and 13G cover one issuer above 5% only. |
| You want officer and director trades          | [`insider-trading`](./insider-trading.md)     | Form 3, 4 and 5 report Section 16 insiders, at any ownership level.   |
| You want the filing document itself           | [`get-edgar-file`](./get-edgar-file.md)       | This tool returns parsed data, not the source text.                   |

## Input

| Parameter | Type    | Required | Constraints           | Notes                                                                                                                |
| --------- | ------- | -------- | --------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `query`   | string  | yes      | max 1,000 characters  | Lucene syntax. See [query language](../query-language.md). This tool does not require a `:`, unlike its siblings. |
| `from`    | integer | no       | 0 to 10000            | Offset into the result set.                                                                                          |
| `size`    | integer | no       | 1 to 50               | Filings per call. **Defaults to 50** when you omit it.                                                               |
| `sort`    | array   | no       | array of sort objects | Defaults to `[{"filedAt": {"order": "desc"}}]`.                                                                      |

The search runs across the 13D and 13G indices together, so one query
returns both form families. Query fields:

- `owners.name`, for example `owners.name:Point72`.
- `owners.amountAsPercent`, including range syntax, for example
  `owners.amountAsPercent:[10 TO *]`.
- `accessionNo`, for example `accessionNo:*`.
- `formType`, `nameOfIssuer`, `cusip`, `filers.cik`, `eventDate`, `filedAt`,
  `titleOfSecurities`. All present in the response body.

## Output

The envelope is `{total, filings[]}`. This is one of six tools that use
`filings[]` instead of `data[]`. Read `total.value` and `total.relation`.
`"gte"` at 10000 means 10,000 or more.

| Field                                                                                                | Type    | Meaning                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `accessionNo`                                                                                        | string  | EDGAR accession number.                                                                                                                                                                                                                                                  |
| `formType`                                                                                           | string  | `SC 13D` in the example response. `SC 13D/A` also appears.                                                                                                                                                                                                              |
| `filedAt`, `eventDate`                                                                               | string  | Filing timestamp in ISO 8601, and the `YYYY-MM-DD` date of the event that triggered the filing.                                                                                                                                                                          |
| `filers[]`                                                                                           | array   | `cik` and `name`. The name carries the role as a suffix, `(Subject)` for the issuer and `(Filed by)` for the investor. Parse that suffix to tell them apart.                                                                                                             |
| `nameOfIssuer`, `titleOfSecurities`                                                                  | string  | The subject company and the class covered, as filed.                                                                                                                                                                                                                     |
| `cusip`                                                                                              | array   | Array of strings. It can be empty.                                                                                                                                                                                                                                       |
| `schedule13GFiledPreviously`                                                                         | boolean | Whether a 13G was filed before this 13D. `amendmentNo` appears on amendments and is absent here.                                                                                                                                                                         |
| `owners[]`                                                                                           | array   | One item per reporting person.                                                                                                                                                                                                                                           |
| `owners[].name`                                                                                      | string  | **Type varies.** A string in the example response, an array of strings in other records. Handle both.                                                                                                                                                                   |
| `owners[].sourceOfFunds`                                                                             | array   | **Type varies.** `["OO"]` in the example response, `"OO"` in other records.                                                                                                                                                                                             |
| `owners[].soleVotingPower`, `.sharedVotingPower`, `.soleDispositivePower`, `.sharedDispositivePower` | number  | Shares the person can vote or sell, alone or with others.                                                                                                                                                                                                                |
| `owners[].aggregateAmountOwned`, `.amountAsPercent`                                                  | number  | Total beneficially owned shares, and percent of the class. `5351000` and `20.3` in the example response.                                                                                                                                                                 |
| `owners[].typeOfReportingPerson`                                                                     | array   | SEC codes, such as `IN` individual, `CO` corporation, `OO` other. Not the full code list.                                                                                                                                                                               |
| `owners[].place`                                                                                     | string  | Place of organisation or citizenship. `Delaware` and `United States` in the example response. Other records use codes such as `X0` and `D8`.                                                                                                                             |
| `owners[].amountExcludesCertainShares`                                                               | boolean | **Field name varies.** The example response uses this name. Other records use `isAggregateExcludeShares`. Check for both.                                                                                                                                                |
| `item1` to `item7`                                                                                   | object  | Narrative items: security and issuer, identity, source of funds, purpose, interest in securities, contracts, exhibits. **Absent in the example response.** Present on an `SC 13D/A` record. They do not appear on every record.                                          |

Add the `owners[]` numbers with care. In the example response both reporting
persons report the same 5,351,000 shares and the same 20.3%. That is one economic
position disclosed by two people. Summing across `owners[]` double counts.

Size behaviour: one filing was 1,319 bytes without the `item1` to `item7`
narrative blocks. Those blocks hold long free text and can be several kilobytes
each. Keep `size` low when a query returns 13D filings with items.

## Example

Prompt: "Show me Point72 positions above 10% of a company."

```json
{
  "name": "form-13d-13g",
  "arguments": {
    "query": "owners.name:Point72 AND owners.amountAsPercent:[10 TO *]",
    "size": 1
  }
}
```

```json
{
  "total": { "value": 8, "relation": "eq" },
  "filings": [
    {
      "accessionNo": "0000902664-22-005029",
      "formType": "SC 13D",
      "filedAt": "2022-12-05T16:00:20-05:00",
      "filers": [
        {
          "cik": "1813658",
          "name": "Tempo Automation Holdings, Inc. (Subject)"
        },
        {
          "cik": "1954961",
          "name": "Point72 Private Investments, LLC (Filed by)"
        }
      ],
      "nameOfIssuer": "Tempo Automation Holdings, Inc.",
      "owners": [
        {
          "name": "Point72 Private Investments, LLC",
          "sharedVotingPower": 5351000,
          "sharedDispositivePower": 5351000,
          "aggregateAmountOwned": 5351000,
          "amountAsPercent": 20.3,
          "typeOfReportingPerson": ["OO"]
        }
      ]
    }
  ]
}
```

Trimmed. The full response also holds `titleOfSecurities`, `cusip`, `eventDate`,
`schedule13GFiledPreviously` and a second owner, Steven A. Cohen.

## Limits and errors

- Field types are not stable across records. `owners[].name` and
  `owners[].sourceOfFunds` each appear as both a scalar and an array.
- The percent is as reported on the filing date. It is never restated.
- A missing `query`, a query over 1,000 characters, or `from` above 10000 all
  fail with HTTP 400 `Invalid request parameter provided.`
- `size` above 50 fails with `Maximum 'size' limit of 50 exceeded.`
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-13f-holdings`](./form-13f-holdings.md)
- [`insider-trading`](./insider-trading.md)
- [`form-144`](./form-144.md)
- REST docs: [Form 13D/13G API](https://sec-api.io/docs/form-13d-13g-search-api)
