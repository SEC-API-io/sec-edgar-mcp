# form-8k

Search 8-K current reports and get the event details of three items as
structured data.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-8k`                              |

## What it does

A public company files an 8-K when something material happens. This tool
searches an index that reads the 8-K body and returns the event as fields. One
row is one 8-K filing.

Coverage starts 2004-08-23.

**Only three 8-K items are extracted.**

| Item object | 8-K item | Subject                                            | Rows          |
| ----------- | -------- | -------------------------------------------------- | ------------- |
| `item4_01`  | 4.01     | Change of the certifying accountant                 | over 25,000   |
| `item4_02`  | 4.02     | Non-reliance on previously issued financials        | over 8,000    |
| `item5_02`  | 5.02     | Director and officer departures and appointments    | over 250,000  |

Queries for `item1_01`, `item1_02`, `item2_01`, `item2_03`, `item3_01`,
`item5_01`, `item5_03`, `item7_01`, `item8_01` and `item9_01` all return zero
rows. The registry description names "acquisitions", "earnings releases" and
"restructurings". Those items are **not** in this index. Treat that part of the
description as wrong. Every row still carries the `items[]` array, which lists
the full item titles the filing reported, including items with no structured
object.

## When to use it

- Which companies dismissed or hired an auditor in the last month?
- Which companies restated their financial statements this quarter, and why?
- Who joined or left this company's board, and on what date?
- What sign-on package did this new CFO get?

## When to use a different tool

| Situation                                    | Better tool                                                 | Why                                              |
| -------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------ |
| You want every 8-K, or an item other than 4.01, 4.02, 5.02 | [filing-search](./filing-search.md)          | Indexes all 8-K filings by form type and date    |
| You want the 8-K text or its exhibits        | [extractor](./extractor.md), [get-edgar-file](./get-edgar-file.md) | Returns the document itself             |
| You want a phrase anywhere in an 8-K         | [full-text-search](./full-text-search.md)                   | Searches the filing body text                    |
| You want the current board roster            | [directors-and-board-members](./directors-and-board-members.md) | Proxy-based roster, not one-off changes       |
| You want annual pay tables                   | [compensation](./compensation.md)                           | Summary compensation table, not a single hire    |

## Input

| Parameter | Type    | Required | Constraints        | Notes                                        |
| --------- | ------- | -------- | ------------------ | -------------------------------------------- |
| `query`   | string  | yes      | must contain a `:` | Lucene syntax. Max 1000 characters.          |
| `from`    | integer | no       | 0 or more          | Offset. `from` above 10000 returns HTTP 400. |
| `size`    | integer | no       | 1 to 50            | Default 50. Above 50 returns HTTP 400.       |
| `sort`    | array   | no       | ES sort clause     | Default `[{"filedAt":{"order":"desc"}}]`.    |

The colon rule is strict. A bare word such as `apple` returns HTTP 400 with
`Invalid request parameter provided.`

Query fields:

Every field of the extracted data is searchable. These are the ones used most.

| Field            | Example                                     |
| ---------------- | ------------------------------------------- |
| `ticker`         | `ticker:AAPL`                               |
| `cik`            | `cik:320193`                                |
| `companyName`    | `companyName:Apple*`                        |
| `formType`       | `formType:"8-K"`                            |
| `accessionNo`    | `accessionNo:"0001140361-26-032653"`        |
| `filedAt`        | `filedAt:[2024-01-01 TO 2024-12-31]`        |
| `periodOfReport` | `periodOfReport:[2026-01-01 TO 2026-12-31]` |
| `items`          | `items:"Item 5.02"`                         |
| `item4_01`       | `item4_01:*`                                |
| `item4_02`       | `item4_02:*`                                |
| `item5_02`       | `item5_02:*`                                |

The `itemX_YY:*` form is the idiom for "give me the filings that carry this
structured block". Combine it with a date range, for example
`item4_01:* AND filedAt:[2024-01-01 TO 2024-12-31]`. Nested fields inside the
item objects, such as `item4_01.formerAccountantName`, follow the same dotted
syntax.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

### Envelope

| Field            | Type   | Meaning                                                        |
| ---------------- | ------ | --------------------------------------------------------------- |
| `total.value`    | number | Number of filings that match the query.                          |
| `total.relation` | string | `eq` means the count is exact. `gte` means at least that many.    |
| `data[]`         | array  | The matching filings. One item per filing, up to 50 per request.  |

### Filing

| Field                   | Type   | Meaning                                                 |
| ----------------------- | ------ | ------------------------------------------------------- |
| `data[].id`             | string | System-internal identifier of the record.                |
| `data[].accessionNo`    | string | EDGAR accession number of the 8-K filing.                |
| `data[].formType`       | string | EDGAR form type. `8-K`, or `8-K/A` for an amendment that adds or clarifies information. |
| `data[].filedAt`        | string | Date and time EDGAR accepted the filing for processing. ISO 8601 with an offset. |
| `data[].periodOfReport` | string | Date the event happened, `YYYY-MM-DD`. A company that finds an error on 1 May and reports it on 2 May gets `periodOfReport` 1 May. |
| `data[].cik`            | string | Central Index Key of the issuer. Leading zeros are removed. |
| `data[].ticker`         | string | Trading symbol of the issuer when the filing was indexed. Empty when there is none. |
| `data[].companyName`    | string | Name of the issuer.                                      |
| `data[].items[]`        | array  | Strings. The full titles of the items the filing reports. One 8-K can report several items. |
| `data[].item4_01`       | object | Change of the certifying accountant. Present only for that item. |
| `data[].item4_02`       | object | Non-reliance on financial statements. Present only for that item. |
| `data[].item5_02`       | object | Director and officer changes. Present only for that item. |

The three item objects follow. Each one opens with `keyComponents`, a
plain-language summary of the event. The paths below drop the `data[].` prefix.

### `data[].item4_01`. Change of the certifying accountant

| Field                              | Type    | Meaning                                       |
| ---------------------------------- | ------- | --------------------------------------------- |
| `item4_01.keyComponents`           | string  | The accountant change in one or two sentences. |
| `item4_01.formerAccountantName`    | string  | Name of the former accounting firm.            |
| `item4_01.formerAccountantDate`    | string  | Date the former accountant's engagement ended, `YYYY-MM-DD`. |
| `item4_01.engagementEndReason`     | string  | Why the former accountant's engagement ended. Values: `resignation`, `dismissal`, `dissolution`, `declination to stand for reappointment`. |
| `item4_01.reasonDetails`           | string  | Text that explains the end of the engagement.  |
| `item4_01.newAccountantName`       | string  | Name of the newly engaged accounting firm.     |
| `item4_01.newAccountantDate`       | string  | Date the new accountant was engaged, `YYYY-MM-DD`. |
| `item4_01.engagedNewAccountant`    | boolean | Whether the registrant engaged the new accountant. |
| `item4_01.consultedNewAccountant`  | boolean | Whether the registrant consulted the new accountant. |
| `item4_01.reportedDisagreements`   | boolean | Whether the filing mentions disagreements with the former accountant. |
| `item4_01.disagreementsList[]`     | array   | Strings. The single disagreements, for example `Use of non-independent counsel to lead investigation.` |
| `item4_01.resolvedDisagreements`   | boolean | Whether the disagreements were resolved. It is also `true` when the filing reports no disagreement. |
| `item4_01.reportableEventsExist`   | boolean | Whether reportable events exist.               |
| `item4_01.reportableEventsList[]`  | array   | Strings. The single reportable events, for example `The accountant advised of the need to expand significantly the scope of its audit.` |
| `item4_01.reportedIcfrWeakness`    | boolean | Whether the filing discloses a material weakness in internal control over financial reporting. |
| `item4_01.icfrWeaknessesDetails`   | string  | Text on the weaknesses in internal control over financial reporting. |
| `item4_01.remediatedIcfrWeakness`  | boolean | Whether those weaknesses were remediated.      |
| `item4_01.remediatedIcfrWeaknessDetails` | string | Text on the remediation.                 |
| `item4_01.goingConcern`            | boolean | Whether the former accountant raised a going concern. |
| `item4_01.goingConcernDetail`      | string  | Text on the going concern.                     |
| `item4_01.opinionType`             | string  | Type of opinion in the accountant's report. Values: `unqualified`, `qualified`, `adverse`. |
| `item4_01.auditDisclaimer`         | boolean | Whether the audit report held a disclaimer.    |
| `item4_01.disclaimerDetails`       | string  | Text on the disclaimer in the audit report.    |
| `item4_01.authorizedInquiry`       | boolean | Whether the registrant authorised the new accountant to make inquiries of the former accountant. |
| `item4_01.approvedChange`          | boolean | Whether the board or the audit committee approved the change. |
| `item4_01.attachments[]`           | array   | Strings. The attachments and exhibits the text names, for example `Exhibit 16.1`. |

### `data[].item4_02`. Non-reliance on previously issued financials

| Field                                 | Type    | Meaning                                    |
| ------------------------------------- | ------- | ------------------------------------------ |
| `item4_02.keyComponents`              | string  | The non-reliance disclosure in one or two sentences. |
| `item4_02.identifiedIssues[]`         | array   | Strings. The issues the company found, for example `Misstated inventory by a former employee`. |
| `item4_02.affectedReportingPeriods[]` | array   | Strings. The reporting periods that may need a restatement, quarters and financial years, for example `Q1 2023`. |
| `item4_02.identifiedBy[]`             | array   | Strings. Who found the issue. Values: `Company`, `Auditors`, `SEC`. The SEC can find an error in a routine review and ask the company to disclose it. |
| `item4_02.restatementIsNecessary`     | boolean | Whether a restatement of the financial statements already published is necessary. |
| `item4_02.reasonsForRestatement[]`    | array   | Strings. Why the restatement is needed, for example `Inaccurate and unsupported manual journal entries`. |
| `item4_02.impactYetToBeDetermined`    | boolean | Whether the company states that the impact of the error is still unknown. |
| `item4_02.impactOfError`              | string  | Text on the impact of the error, with the amounts the company gives. |
| `item4_02.impactIsMaterial`           | boolean | Whether the company states that the impact is material. |
| `item4_02.materialWeaknessIdentified` | boolean | Whether the company discloses a material weakness in its internal financial controls. |
| `item4_02.auditors[]`                 | array   | Strings. The auditors that take part in the restatement. |
| `item4_02.affectedLineItems[]`        | array   | Strings. The line items of the income statement, balance sheet or cash flow statement that the error hits, for example `Inventory`. |
| `item4_02.netIncomeDecreased`         | boolean | Whether the company states that net income falls after the restatement. |
| `item4_02.netIncomeIncreased`         | boolean | Whether the company states that net income rises after the restatement. Both flags can be `true` when the restatement hits different periods. |
| `item4_02.netIncomeAdjustment`        | string  | The size of the net income adjustment, when the company gives it, for example `$348.0 million`. |
| `item4_02.revenueDecreased`           | boolean | Whether the company states that revenue falls after the restatement. |
| `item4_02.revenueIncreased`           | boolean | Whether the company states that revenue rises after the restatement. Both flags can be `true` when the restatement hits different periods. |
| `item4_02.revenueAdjustment`          | string  | The size of the revenue adjustment, when the company gives it. |
| `item4_02.eventClassification`        | string  | A one-line class for the event, for example `Financial Restatement Due to Error in Recording Write-off of Acquired In-Process Research and Development`. |

### `data[].item5_02`. Director and officer changes

| Field                                 | Type   | Meaning                                     |
| ------------------------------------- | ------ | ------------------------------------------- |
| `item5_02.keyComponents`              | string | The departures and appointments in one or two sentences, and what they mean for the leadership. |
| `item5_02.personnelChanges[]`         | array  | One item per personnel change the filing reports. |
| `item5_02.bonusPlans[]`               | array  | One item per bonus plan the filing reports.  |
| `item5_02.organizationChanges`        | object | The organisational changes the filing reports. |
| `item5_02.additionalReportableEvents[]` | array | Strings. Other reportable events under Section 404 (a), (b) or (c) that the text states. They cover changes to executive roles, compensatory agreements or unforeseen events that hit key personnel, and they exclude departures, appointments, incentive plans and bonuses. |
| `item5_02.attachments[]`              | array  | Strings. The documents and sources the text names, for example `Company Press Release`. |

### `item5_02.personnelChanges[]`

| Field                                          | Type    | Meaning                            |
| ---------------------------------------------- | ------- | ---------------------------------- |
| `personnelChanges[].type`                      | string  | Type of the change. Values: `appointment`, `nomination`, `refusal`, `departure`, `amendment`, `bonus`. |
| `personnelChanges[].departureType`             | string  | Type of the departure. Values: `resignation`, `retirement`, `termination`, `other`. |
| `personnelChanges[].effectiveDate`             | string  | Date the change takes effect.       |
| `personnelChanges[].positions[]`               | array   | Strings. The positions the change hits. |
| `personnelChanges[].reason`                    | string  | The reason for the change, when the filing gives one. |
| `personnelChanges[].person`                    | object  | The person the change hits.         |
| `personnelChanges[].person.name`               | string  | Full name of the person.            |
| `personnelChanges[].person.age`                | number  | Age of the person, when disclosed.  |
| `personnelChanges[].person.continuedPositions[]` | array | Strings. Roles the person already holds at the company and keeps. |
| `personnelChanges[].person.positionsAtOtherCompanies[]` | array | Strings. Roles the person holds at other organisations. |
| `personnelChanges[].person.academicAffiliations[]` | array | Strings. Academic ties of the person. |
| `personnelChanges[].person.background`         | string  | The person's work history, qualifications and expertise for the new role. |
| `personnelChanges[].person.previousPositions[]` | array  | Strings. Roles the person held before, inside or outside the company. |
| `personnelChanges[].person.familyRelationships` | string | Family ties between the person and another officer or director, when the text states them. |
| `personnelChanges[].compensation`              | object  | The new pay terms on an appointment or an amendment, when the text states them. |
| `personnelChanges[].compensation.onetime`      | string  | One-time payment in US dollars.     |
| `personnelChanges[].compensation.annual`       | string  | Annual compensation in US dollars.  |
| `personnelChanges[].compensation.equity`       | string  | The equity award.                   |
| `personnelChanges[].compensation.equityVesting` | string | The vesting terms of the equity award. |
| `personnelChanges[].compensation.noCompensation` | boolean | Whether the text states that there is no pay. |
| `personnelChanges[].restitution`               | string  | Restitution the person owes, for example a return of unvested stock options. |
| `personnelChanges[].continuedConsultingRole`   | boolean | Whether the person stays on in a consulting role. |
| `personnelChanges[].consultingEndDate`         | string  | Date the consulting role ends.      |
| `personnelChanges[].amendmentSummary`          | string  | The main points of the amendment, for example new pay, termination terms and stock vesting terms. |
| `personnelChanges[].termExtended`              | boolean | Whether the term of office got longer. |
| `personnelChanges[].termShortened`             | boolean | Whether the term of office got shorter. |
| `personnelChanges[].oldTermEndDate`            | string  | End date of the previous term, `YYYY-MM-DD`. |
| `personnelChanges[].termEndDate`               | string  | End date of the new term, `YYYY-MM-DD`. |
| `personnelChanges[].compensationIncreased`     | boolean | Whether the pay went up.            |
| `personnelChanges[].compensationDecreased`     | boolean | Whether the pay went down.          |
| `personnelChanges[].disagreements`             | boolean | Whether the text states a disagreement that led to the change or came with it. |
| `personnelChanges[].interim`                   | boolean | Whether the role is, or was, an interim role. |

### `item5_02.bonusPlans[]`

| Field                                   | Type    | Meaning                                   |
| --------------------------------------- | ------- | ----------------------------------------- |
| `bonusPlans[].specificRoles`            | boolean | Whether the plan covers named roles rather than the people who hold them. |
| `bonusPlans[].eligibleRoles[]`          | array   | Strings. The roles the plan covers.        |
| `bonusPlans[].generalEmployee`          | boolean | Whether the plan covers employees in general. |
| `bonusPlans[].specificIndividuals`      | boolean | Whether the plan covers named people.      |
| `bonusPlans[].eligibleIndividuals[]`    | array   | Strings. The people the plan covers.       |
| `bonusPlans[].compensation`             | object  | What the plan pays.                        |
| `bonusPlans[].compensation.cash`        | string  | The cash bonus.                            |
| `bonusPlans[].compensation.equity`      | string  | The equity bonus.                          |
| `bonusPlans[].compensation.equityDetails` | string | The terms of the equity bonus, for example the vesting period. |
| `bonusPlans[].conditional`              | boolean | Whether the bonus depends on conditions.   |
| `bonusPlans[].conditions`               | string  | The conditions the person must meet to get the bonus. |

### `item5_02.organizationChanges`

| Field                                    | Type    | Meaning                                  |
| ---------------------------------------- | ------- | ---------------------------------------- |
| `organizationChanges.organ`              | string  | The organisational unit the change hits, for example `Board of Directors`. |
| `organizationChanges.details`            | string  | What the change is, for example `Expansion of the board`. |
| `organizationChanges.sizeIncrease`       | boolean | Whether the unit grew.                    |
| `organizationChanges.sizeDecrease`       | boolean | Whether the unit shrank.                  |
| `organizationChanges.created`            | boolean | Whether the unit was created.             |
| `organizationChanges.abolished`          | boolean | Whether the unit was abolished.           |
| `organizationChanges.affectedPersonnel[]` | array  | Strings. The people the change hits.      |

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.

## Example

Prompt: "Show me the newest 8-K filings with a structured event block."

```json
{ "name": "form-8k", "arguments": { "query": "cik:*", "size": 1 } }
```

Response, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "id": "f9c18b5e5b1e1b7471d6adb39fdd055e",
      "accessionNo": "0001140361-26-032653",
      "formType": "8-K",
      "filedAt": "2026-08-13T08:48:37-04:00",
      "periodOfReport": "2026-08-08",
      "cik": "1130713",
      "ticker": "BBBY",
      "companyName": "BED BATH & BEYOND, INC.",
      "items": ["Item 5.02: Departure of Directors or Certain Officers; Election of Directors; Appointment of Certain Officers: Compensatory Arrangements of Certain Officers"],
      "item5_02": {
        "keyComponents": "Jill Windrum was appointed as the Company's Chief Accounting Officer and Deputy Chief Financial Officer, effective August 31, 2026. She will replace Brian LaRose as the principal accounting officer.",
        "personnelChanges": [
          {
            "type": "appointment",
            "effectiveDate": "2026-08-31",
            "positions": ["Chief Accounting Officer", "Deputy Chief Financial Officer", "principal accounting officer"],
            "person": { "name": "Jill Windrum", "age": 46 },
            "compensation": { "annual": "400,000", "equity": "Sign-on equity awards with an aggregate target value of $400,000", "noCompensation": false },
            "interim": false
          }
        ]
      }
    }
  ]
}
```

## Limits and errors

- No colon in `query` gives HTTP 400 `Invalid request parameter provided.`
- `query` longer than 1000 characters gives the same 400.
- `from` above 10000 gives the same 400.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- A query for an item outside 4.01, 4.02 and 5.02 returns zero rows, not an
  error. Zero rows here means "not covered", not "no such event happened".
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [filing-search](./filing-search.md), [full-text-search](./full-text-search.md), [extractor](./extractor.md)
- [directors-and-board-members](./directors-and-board-members.md), [compensation](./compensation.md), [audit-fees](./audit-fees.md)
- REST docs: <https://sec-api.io/docs/form-8k-data-item4-1-search-api>,
  <https://sec-api.io/docs/form-8k-data-search-api>,
  <https://sec-api.io/docs/form-8k-data-item5-2-search-api>
