# form-adv-individuals

Search the register of investment adviser representatives, the people who give
advice for a Form ADV firm.

|                 |                                    |
| --------------- | ---------------------------------- |
| Category        | Investment advisers                |
| Required input  | `query`                            |
| Returns         | `{total, filings[]}`               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`    |
| REST equivalent | `POST /form-adv/individual`        |

## What it does

One item in `filings[]` is one person, not one filing. The row is the current
record for that individual CRD number, `Info.indvlPK`. A query for a single
`indvlPK` returns exactly one row. There is no history of versions. The set
holds more than 380,000 individual advisers. It updates once a day.

Each row carries the person's name, the firm that employs them now, their state
registrations, their exams, their professional designations, ten years of
employment history, their outside business activities, and a disclosure summary.

The disclosure block is a **summary only**. `DRPs.DRP[]` holds nine yes or no
flags, such as `hasCriminal` and `hasBankrupt`. The registry description says the
tool returns "disciplinary events". It returns the flags, not the events. Read
the full detail on the IAPD page in `Info.link`.

## When to use it

- Who works as an adviser representative at this firm?
- Which exams has this person passed, and when?
- Where did this person work before the current firm?
- Which representatives report a bankruptcy or a customer complaint?
- Which professional designations does this person hold?

## When to use a different tool

| Situation                                  | Better tool                                                       | Why                                                     |
| ------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------- |
| You want the firm, not its people          | [`form-adv-firms`](./form-adv-firms.md)                           | Firm identity, assets and registration status.          |
| You want the owners of the firm            | [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md) | Schedule A names owners and control persons. |
| You want corporate directors and officers  | [`directors-and-board-members`](./directors-and-board-members.md) | That tool covers public company boards.                  |
| You want insider trades by a person        | [`insider-trading`](./insider-trading.md)                         | Form 3, 4 and 5 report trades, Form ADV does not.        |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                              |
| --------- | ------- | -------- | ------------------------ | -------------------------------------------------- |
| `query`   | string  | Yes      | Lucene syntax            | For example `CrntEmps.CrntEmp.orgPK:149777`.       |
| `from`    | integer | No       | 0 to 10,000              | Offset of the first result. Default 0.             |
| `size`    | integer | No       | 1 to 50                  | Default 50.                                         |
| `sort`    | array   | No       | Elasticsearch sort array | Default `[{"Info.indvlPK": {"order": "desc"}}]`.   |

Query fields. The hit count on 2026-08-13 is in brackets.

| Field                                            | Example                                              |
| ------------------------------------------------ | ---------------------------------------------------- |
| `Info.indvlPK`                                   | `Info.indvlPK:8213636` (1)                           |
| `Info.lastNm`                                    | `Info.lastNm:Kim` (515)                              |
| `CrntEmps.CrntEmp.orgPK`                         | `CrntEmps.CrntEmp.orgPK:149777` (10,000 or more)     |
| `CrntEmps.CrntEmp.CrntRgstns.CrntRgstn.regAuth`  | `...regAuth:NY` (10,000 or more)                     |
| `Exms.Exm.exmCd`                                 | `Exms.Exm.exmCd:S65` (10,000 or more)                |

`orgPK` is the firm CRD number. Get it from
[`form-adv-firms`](./form-adv-firms.md) first, then list the staff.

`_exists_` works and is the way to find populated blocks, for example
`_exists_:DRPs.DRP`, `_exists_:Dsgntns.Dsgntn` and `_exists_:PrevRgstns.PrevRgstn`.
`Info.firstNm` is present in every row. An unknown field returns `total: 0`
with no error. See [query language](../query-language.md).

## Output

The envelope is `{total, filings[]}`, not `{total, data[]}`. `total` is an
object, `{value, relation}`. A `relation` of `gte` means the count stopped at
10,000.

Every row carries the same ten keys. Nested blocks use a plural container with a
singular member inside it, for example `Exms.Exm[]`. Query the full path.

### Envelope

| Field            | Type   | Meaning                                                             |
| ---------------- | ------ | ------------------------------------------------------------------- |
| `total`          | object | Hit count for the query.                                            |
| `total.value`    | number | Number of people that match. The count stops at 10,000.             |
| `total.relation` | string | `eq` when `value` is the exact count. `gte` when the count stopped at 10,000. |
| `filings[]`      | array  | One item is one person.                                             |
| `filings[].id`   | number | Record key. It repeats the individual CRD number in `Info.indvlPK`. |

### `Info`

Basic information that describes the person.

| Field               | Type   | Meaning                                                          |
| ------------------- | ------ | ---------------------------------------------------------------- |
| `Info.indvlPK`      | number | Individual CRD number.                                           |
| `Info.actvAGReg`    | string | `Y` when the person holds an active agent registration, `N` when not. |
| `Info.lastNm`       | string | Last name.                                                       |
| `Info.firstNm`      | string | First name.                                                      |
| `Info.midNm`        | string | Middle name. Often absent.                                       |
| `Info.sufNm`        | string | Name suffix, for example `JR`.                                   |
| `Info.link`         | string | IAPD composite page for the person.                              |

### `OthrNms`

| Field                        | Type   | Meaning                                                 |
| ---------------------------- | ------ | ------------------------------------------------------- |
| `OthrNms.OthrNm[]`           | array  | Names the person uses, or has used, since the age of 18, other than the legal name. It holds nicknames, aliases, and names used before or after marriage. |
| `OthrNms.OthrNm[].lastNm`    | string | Last name in the other name.                            |
| `OthrNms.OthrNm[].firstNm`   | string | First name in the other name.                           |
| `OthrNms.OthrNm[].midNm`     | string | Middle name in the other name.                          |
| `OthrNms.OthrNm[].sufNm`     | string | Suffix in the other name.                               |

### `CrntEmps`

The firms that employ the person now, and the registration the person holds at
each firm.

| Field                                                    | Type   | Meaning                                       |
| -------------------------------------------------------- | ------ | --------------------------------------------- |
| `CrntEmps.CrntEmp[]`                                      | array  | One item is one active employment.            |
| `CrntEmps.CrntEmp[].orgNm`                                | string | Firm business name from the IARD composite record. |
| `CrntEmps.CrntEmp[].orgPK`                                | number | Firm CRD number.                              |
| `CrntEmps.CrntEmp[].str1`                                 | string | Firm street address, first line.              |
| `CrntEmps.CrntEmp[].str2`                                 | string | Firm street address, second line.             |
| `CrntEmps.CrntEmp[].city`                                 | string | Firm city.                                    |
| `CrntEmps.CrntEmp[].state`                                | string | Firm state code.                              |
| `CrntEmps.CrntEmp[].cntry`                                | string | Firm country.                                 |
| `CrntEmps.CrntEmp[].postlCd`                              | string | Firm postal code.                             |
| `CrntEmps.CrntEmp[].CrntRgstns.CrntRgstn[]`               | array  | Registrations the person holds through this employment. |
| `CrntEmps.CrntEmp[].CrntRgstns.CrntRgstn[].regAuth`       | string | State code of the regulatory authority, for example `NY` for New York. |
| `CrntEmps.CrntEmp[].CrntRgstns.CrntRgstn[].regCat`        | string | Registration category with that regulator. `RA` is investment adviser representative. |
| `CrntEmps.CrntEmp[].CrntRgstns.CrntRgstn[].st`            | string | Current registration status, not a state. Values include `APPROVED`, `APPROVED_RES` restricted approval, `PENDING`, `DEFICIENT`, `TEMPREG` temporary registration, `CE_INACTIVE` inactive for continuing education, `REQUEST_TERM` termination requested, `TERMED`, `FTR` terminated for failure to renew, `SUSPENSION`, `REVOKED` and `BAR`. |
| `CrntEmps.CrntEmp[].CrntRgstns.CrntRgstn[].stDt`          | string | Date the system posted the status change. `YYYY-MM-DD`. |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[]`             | array  | Branch offices tied to this employment.       |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[].str1`        | string | Branch street address, first line.            |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[].str2`        | string | Branch street address, second line.           |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[].city`        | string | Branch city.                                  |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[].state`       | string | Branch state code.                            |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[].cntry`       | string | Branch country.                               |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[].postlCd`     | string | Branch postal code.                           |

### `Exms`

| Field                | Type   | Meaning                                                          |
| -------------------- | ------ | ---------------------------------------------------------------- |
| `Exms.Exm[]`         | array  | State exams the person passed.                                   |
| `Exms.Exm[].exmCd`   | string | Exam code. `S63` Uniform Securities Agent State Law Examination, `S64` NASAA Real Estate Securities Exam, `S65` Uniform Investment Adviser Law Examination, `S66` Uniform Combined State Law Examination. |
| `Exms.Exm[].exmNm`   | string | Exam name.                                                       |
| `Exms.Exm[].exmDt`   | string | Date the person took the exam. `YYYY-MM-DD`.                     |

### `Dsgntns`

| Field                        | Type   | Meaning                                            |
| ---------------------------- | ------ | -------------------------------------------------- |
| `Dsgntns.Dsgntn[]`           | array  | Professional designations the person holds.        |
| `Dsgntns.Dsgntn[].dsgntnNm`  | string | Designation code.                                  |

### `PrevRgstns`

| Field                                                | Type   | Meaning                                           |
| ---------------------------------------------------- | ------ | ------------------------------------------------- |
| `PrevRgstns.PrevRgstn[]`                              | array  | Registrations the person held before.             |
| `PrevRgstns.PrevRgstn[].orgNm`                        | string | Firm business name from the IARD composite record. |
| `PrevRgstns.PrevRgstn[].orgPK`                        | number | Firm CRD number.                                  |
| `PrevRgstns.PrevRgstn[].regBeginDt`                   | string | Date the registration began. `YYYY-MM-DD`.        |
| `PrevRgstns.PrevRgstn[].regEndDt`                     | string | Date the registration ended. `YYYY-MM-DD`.        |
| `PrevRgstns.PrevRgstn[].BrnchOfLocs[]`                | array  | Branch offices tied to the former registration.   |
| `PrevRgstns.PrevRgstn[].BrnchOfLocs[].BrnchOfLoc[]`   | array  | One item is one branch office.                    |
| `PrevRgstns.PrevRgstn[].BrnchOfLocs[].BrnchOfLoc[].city`  | string | Branch city.                                  |
| `PrevRgstns.PrevRgstn[].BrnchOfLocs[].BrnchOfLoc[].state` | string | Branch state code.                            |

### `EmpHss`

| Field                    | Type   | Meaning                                                      |
| ------------------------ | ------ | ------------------------------------------------------------ |
| `EmpHss.EmpHs[]`         | array  | Employment history. It covers work outside the industry too.  |
| `EmpHss.EmpHs[].fromDt`  | string | Employment begin date. `MM/YYYY`.                            |
| `EmpHss.EmpHs[].toDt`    | string | Employment end date. `MM/YYYY`. Absent while the job runs.    |
| `EmpHss.EmpHs[].orgNm`   | string | Employer name.                                                |
| `EmpHss.EmpHs[].city`    | string | City of employment.                                           |
| `EmpHss.EmpHs[].state`   | string | State of employment.                                          |

### `OthrBuss`

| Field                     | Type   | Meaning                                                     |
| ------------------------- | ------ | ----------------------------------------------------------- |
| `OthrBuss.OthrBus`        | object | Other businesses the person runs.                           |
| `OthrBuss.OthrBus.desc`   | string | All of those businesses in one free-text string. It packs the business name, whether the business is investment related, the address, the nature of the business, the position and title held, the start date, and the hours worked. |

### `DRPs`

Disclosure reporting page flags. Each flag is `Y` or `N`.

| Field                          | Type   | Meaning                                              |
| ------------------------------ | ------ | ---------------------------------------------------- |
| `DRPs.DRP[]`                   | array  | Reportable and disclosable events for the person.    |
| `DRPs.DRP[].hasRegAction`      | string | The person has a regulatory action disclosure.       |
| `DRPs.DRP[].hasCriminal`       | string | The person has a criminal disclosure.                |
| `DRPs.DRP[].hasBankrupt`       | string | The person has a bankruptcy disclosure.              |
| `DRPs.DRP[].hasCivilJudc`      | string | The person has a civil judicial disclosure.          |
| `DRPs.DRP[].hasBond`           | string | The person has a bond disclosure.                    |
| `DRPs.DRP[].hasJudgment`       | string | The person has a judgment disclosure.                |
| `DRPs.DRP[].hasInvstgn`        | string | The person has an investigation disclosure.          |
| `DRPs.DRP[].hasCustComp`       | string | The person has a customer complaint disclosure.      |
| `DRPs.DRP[].hasTermination`    | string | The person has a termination disclosure.             |

`OthrNms`, `Dsgntns`, `PrevRgstns` and `DRPs` are empty objects when the person
has nothing to report. Test for content, do not index blindly.

`OthrBuss.OthrBus` is an object, not an array, and `desc` is unparsed text with
pipe or asterisk separators. It can contain the U+FFFD replacement character
where the source encoding failed.

`size` counts people and caps at 50. Page with `from`. The JSON arrives as one
text block. See [response format](../response-format.md).

## Example

Prompt: "Show me the most recently added investment adviser representative."

```json
{ "name": "form-adv-individuals", "arguments": { "query": "CrntEmps.CrntEmp.orgPK:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "filings": [
    {
      "Info": {
        "lastNm": "Kim",
        "firstNm": "Aaron",
        "indvlPK": 8328529,
        "link": "https://adviserinfo.sec.gov/individual/summary/8328529"
      },
      "CrntEmps": {
        "CrntEmp": [
          {
            "CrntRgstns": { "CrntRgstn": [{ "regAuth": "HI", "regCat": "RA", "st": "APPROVED" }] },
            "orgNm": "FIRST ADVISORS NATIONAL, LLC",
            "orgPK": 166212,
            "city": "ATLANTA"
          }
        ]
      },
      "Exms": { "Exm": [{ "exmCd": "S65", "exmDt": "2026-07-20" }] },
      "id": 8328529
    }
  ]
}
```

Keys were removed to fit. The values are unchanged. The default sort is
`Info.indvlPK` descending, so the first row is the newest record.

## Limits and errors

- `size` above 50 returns HTTP 400 with
  `Maximum 'size' limit of 50 exceeded.`
- Omitting `size` returns 50 people, not 10.
- One query returns at most 10,000 people. `from` stops at 10,000.
- A firm query on `orgPK` can match more than 10,000 people. Narrow it with a
  state, an exam code or a name before you page.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-firms`](./form-adv-firms.md),
  [`form-adv-brochures`](./form-adv-brochures.md)
- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
