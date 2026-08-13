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
position. This tool searches both, from 1994 to present. One item in `filings[]`
is one filing. Each filing lists one or more reporting persons in `owners[]`,
with voting power, dispositive power and percent of class.

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
- `owners.typeOfReportingPerson`, `owners.sourceOfFunds`,
  `owners.memberOfGroup.a` and `owners.memberOfGroup.b`.
- `formType`, `nameOfIssuer`, `cusip`, `filers.cik`, `filers.name`, `eventDate`,
  `filedAt`, `titleOfSecurities`. All present in the response body.
- The narrative items are searchable too. `item4.transactionPurpose` and
  `item6.contractDescription` cover 13D. `item5.classOwnership5PercentOrLess`
  and `item8.identificationAndClassificationOfGroupMembers` cover 13G.

## Output

The envelope is `{total, filings[]}`. This is one of six tools that use
`filings[]` instead of `data[]`. Read `total.value` and `total.relation`.
`"gte"` at 10000 means 10,000 or more.

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | --------------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                          |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many.    |
| `filings[]`      | array  | The matching filings. One item per filing.                       |

### Filing

| Field                                   | Type    | Meaning                                                                                                                        |
| --------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `filings[].id`                          | string  | System-internal identifier of the filing record.                                                                                 |
| `filings[].accessionNo`                 | string  | EDGAR accession number of the filing.                                                                                            |
| `filings[].formType`                    | string  | EDGAR form type. `SC 13D` in the example response. `SC 13D/A`, `SC 13G` and `SC 13G/A` also appear. The `/A` suffix marks an amendment. |
| `filings[].amendmentNo`                 | string  | Sequence number of the amendment. Only an amended filing carries it.                                                              |
| `filings[].filedAt`                     | string  | Time EDGAR accepted the filing, ISO 8601 with an offset.                                                                         |
| `filings[].eventDate`                   | string  | Date of the event that made the filing obligatory, `YYYY-MM-DD`.                                                                 |
| `filings[].filers[]`                    | array   | The parties on the EDGAR header. It holds the issuer and the investor.                                                            |
| `filings[].filers[].cik`                | string  | CIK of the party.                                                                                                                |
| `filings[].filers[].name`               | string  | Name of the party, with the role as a suffix. `(Subject)` marks the issuer. `(Filed by)` marks the investor. Parse that suffix to tell them apart. |
| `filings[].nameOfIssuer`                | string  | Name of the issuer of the acquired security.                                                                                     |
| `filings[].titleOfSecurities`           | string  | Title of the class of securities acquired, as filed.                                                                             |
| `filings[].cusip[]`                     | array   | Strings. CUSIP numbers of the acquired securities. It can be empty.                                                              |
| `filings[].schedule13GFiledPreviously`  | boolean | 13D only. Whether the person filed a Schedule 13G on this position before.                                                        |
| `filings[].applicableRule`              | object  | 13G only. The rule the filer relies on.                                                                                          |
| `filings[].applicableRule.13d-1b`       | boolean | 13G only. `true` when the filer files under Rule 13d-1(b).                                                                        |
| `filings[].applicableRule.13d-1c`       | boolean | 13G only. `true` when the filer files under Rule 13d-1(c).                                                                        |
| `filings[].applicableRule.13d-1d`       | boolean | 13G only. `true` when the filer files under Rule 13d-1(d).                                                                        |
| `filings[].owners[]`                    | array   | The reporting persons. One item per person.                                                                                      |
| `filings[].signatures[]`                | array   | The signature blocks of the filing.                                                                                              |
| `filings[].exhibits[]`                  | array   | The exhibits attached to the filing.                                                                                             |

### `filings[].owners[]`

| Field                                          | Type            | Meaning                                                                                                                          |
| ---------------------------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `owners[].name`                                | string or array | Full legal name of the reporting person. **Type varies.** A string in the example response, an array of strings in other records. Handle both. |
| `owners[].memberOfGroup`                       | object          | Group membership of the person. `a` affirms it, `b` disclaims it.                                                                  |
| `owners[].memberOfGroup.a`                     | boolean         | `true` when the person holds the shares as an affirmed member of a group.                                                          |
| `owners[].memberOfGroup.b`                     | boolean         | `true` when the person disclaims membership in the group.                                                                          |
| `owners[].sourceOfFunds`                       | array or string | 13D only. Codes for the money used to buy the shares: `SC` securities of the issuer, `BK` bank loan, `AF` affiliate funds, `WC` working capital, `PF` personal funds, `OO` other. **Type varies.** `["OO"]` in the example response, `"OO"` in other records. |
| `owners[].legalProceedingsDisclosureRequired`  | boolean         | 13D only. Whether the person must disclose legal proceedings.                                                                      |
| `owners[].place`                               | string          | Citizenship of a person, or place of organisation of an entity. `Delaware` and `United States` in the example response. Other records use codes such as `X0` and `D8`. |
| `owners[].soleVotingPower`                     | number          | Shares the person alone can vote.                                                                                                  |
| `owners[].sharedVotingPower`                   | number          | Shares the person votes together with others.                                                                                      |
| `owners[].soleDispositivePower`                | number          | Shares the person alone can sell.                                                                                                  |
| `owners[].sharedDispositivePower`              | number          | Shares the person can sell together with others.                                                                                   |
| `owners[].aggregateAmountOwned`                | number          | Total shares the person owns beneficially. `5351000` in the example response.                                                      |
| `owners[].amountExcludesCertainShares`         | boolean         | `true` when the aggregate amount leaves out shares the person disclaims. **Field name varies.** Other records use `isAggregateExcludeShares`. Check for both. |
| `owners[].isAggregateExcludeShares`            | boolean         | The same flag under a second name. A record carries one name or the other.                                                         |
| `owners[].amountAsPercent`                     | number          | Beneficial ownership as a percent of the class, rounded to one decimal. `20.3` in the example response.                            |
| `owners[].typeOfReportingPerson[]`             | array           | Strings. Classification codes of the person, such as `IN` individual, `CO` corporation and `OO` other. The full set is `BD`, `BK`, `CO`, `CP`, `EP`, `FI`, `HC`, `IA`, `IC`, `IN`, `IV`, `PN`, `SA` and `OO`. |
| `owners[].footnotes[]`                         | array           | Footnotes attached to the row of the person, if any.                                                                               |

### Schedule 13D items

A 13D carries seven narrative items. They hold long free text. They are absent
from the example response and do not appear on every record.

| Field                                          | Type   | Meaning                                                                     |
| ---------------------------------------------- | ------ | ----------------------------------------------------------------------------- |
| `item1.securityTitle`                          | string | Title of the class of security the schedule covers.                           |
| `item1.issuerName`                             | string | Name of the issuer.                                                           |
| `item1.issuerPrincipalAddress.street1`         | string | First street line of the principal office of the issuer.                      |
| `item1.issuerPrincipalAddress.street2`         | string | Second street line of that office.                                            |
| `item1.issuerPrincipalAddress.city`            | string | City of that office.                                                          |
| `item1.issuerPrincipalAddress.stateOrCountry`  | string | State or country of that office.                                              |
| `item1.issuerPrincipalAddress.zipCode`         | string | Postal code of that office.                                                   |
| `item1.commentText`                            | string | Context the filer added to item 1.                                            |
| `item2.filingPersonName`                       | string | Name of the person that files the schedule.                                   |
| `item2.principalBusinessAddress`               | string | Residence or business address of that person.                                 |
| `item2.principalJob`                           | string | Principal occupation or employment of that person.                            |
| `item2.hasBeenConvicted`                       | string | Whether a criminal proceeding convicted that person.                          |
| `item2.convictionDescription`                  | string | Civil proceedings and securities law violations of that person.               |
| `item2.citizenship`                            | string | Citizenship of that person.                                                   |
| `item3.fundsSource`                            | string | Source and amount of the money used to buy the shares.                        |
| `item4.transactionPurpose`                     | string | Purpose of the purchase, and the actions the person plans.                    |
| `item5.percentageOfClassSecurities`            | string | Aggregate percent of the class the person owns.                               |
| `item5.numberOfShares`                         | string | Breakdown of voting power and dispositive power.                              |
| `item5.transactionDescription`                 | string | Transactions in the class in the past 60 days.                                |
| `item5.listOfShareholders`                     | string | Other persons with a right to the dividends or the sale proceeds.             |
| `item5.date5PercentOwnership`                  | string | Date the person stopped owning more than 5% of the class.                     |
| `item6.contractDescription`                    | string | Contracts, arrangements and understandings about the securities.              |
| `item7.filedExhibits`                          | string | Description of the exhibits filed with the schedule.                          |

### Schedule 13G items

A 13G carries ten items, not seven. The keys differ from the 13D item that
carries the same number.

| Field                                                          | Type    | Meaning                                                                    |
| -------------------------------------------------------------- | ------- | ---------------------------------------------------------------------------- |
| `item1.issuerName`                                             | string  | Name of the issuer.                                                          |
| `item1.issuerPrincipalExecutiveOfficeAddress`                  | string  | Address of the principal executive office of the issuer.                     |
| `item2.filingPersonName`                                       | string  | Name of the reporting person.                                                |
| `item2.principalBusinessOfficeOrResidenceAddress`              | string  | Business office or residence address of that person.                         |
| `item2.citizenship`                                            | string  | Citizenship of that person.                                                  |
| `item3.notApplicable`                                          | boolean | `true` when item 3 does not apply to the filer.                              |
| `item3.typesOfPersons[]`                                       | array   | Classification of the filer under sub-paragraphs (a) to (k) of the rule.     |
| `item3.otherTypeOfPersonFiling`                                | string  | Context for a filer type outside that list.                                  |
| `item4[]`                                                      | array   | Ownership of each reporting person. One item per person.                     |
| `item4[].amountBeneficiallyOwned`                              | number  | Shares the person owns beneficially.                                         |
| `item4[].classPercent`                                         | number  | Percent of the class the person owns.                                        |
| `item4[].numberOfSharesPersonHas.solePowerOrDirectToVote`      | number  | Shares the person alone can vote or direct the vote of.                      |
| `item4[].numberOfSharesPersonHas.sharedPowerOrDirectToVote`    | number  | Shares the person votes or directs the vote of with others.                  |
| `item4[].numberOfSharesPersonHas.solePowerOrDirectToDispose`   | number  | Shares the person alone can sell or direct the sale of.                      |
| `item4[].numberOfSharesPersonHas.sharedPowerOrDirectToDispose` | number  | Shares the person can sell or direct the sale of with others.                |
| `item5.classOwnership5PercentOrLess`                           | boolean | `true` when the person stopped owning more than 5% of the class.             |
| `item6.notApplicable`                                          | boolean | `true` when item 6 does not apply.                                           |
| `item6.ownershipMoreThan5PercentOnBehalfOfAnotherPerson`       | string  | Disclosure of shares the person holds for someone else.                      |
| `item7.notApplicable`                                          | boolean | `true` when item 7 does not apply.                                           |
| `item7.subsidiaryIdentificationAndClassification`              | string  | Identity and classification of the subsidiary that bought the shares.        |
| `item8.notApplicable`                                          | boolean | `true` when item 8 does not apply.                                           |
| `item8.identificationAndClassificationOfGroupMembers`          | string  | Identity and classification of each member of the group.                     |
| `item9.notApplicable`                                          | boolean | `true` when item 9 does not apply.                                           |
| `item9.groupDissolutionNotice`                                 | string  | Notice that the group has dissolved.                                         |
| `item10.notApplicable`                                         | boolean | `true` when item 10 does not apply.                                          |
| `item10.certifications`                                        | string  | Certification text of the filer.                                             |

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
