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
`indvlPK` returns exactly one row. There is no history of versions.

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
- Which representatives hold the Certified Financial Planner designation?

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
| `from`    | integer | No       | Minimum 0                | Offset of the first result. Default 0.             |
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

| Field                             | Type   | Meaning                                                           |
| --------------------------------- | ------ | ----------------------------------------------------------------- |
| `Info.indvlPK`                    | number | Individual CRD number. Equals `id`.                               |
| `Info.firstNm`, `midNm`, `lastNm` | string | Legal name parts. `midNm` is often absent.                        |
| `Info.actvAGReg`                  | string | `Y` or `N`. Active agent registration flag.                        |
| `Info.link`                       | string | IAPD summary page for the person.                                  |
| `OthrNms.OthrNm[]`                | array  | Other names used, same name parts.                                 |
| `CrntEmps.CrntEmp[]`              | array  | Current employers. `orgNm`, `orgPK`, and the firm address.        |
| `CrntEmps.CrntEmp[].CrntRgstns.CrntRgstn[]` | array | State registrations. `regAuth`, `regCat`, `st`, `stDt`.  |
| `CrntEmps.CrntEmp[].BrnchOfLocs.BrnchOfLoc[]` | array | Branch offices the person works from.                  |
| `Exms.Exm[]`                      | array  | Exams passed. `exmCd`, `exmNm`, `exmDt`, for example `S65`.        |
| `Dsgntns.Dsgntn[]`                | array  | Professional designations. `dsgntnNm`, for example `Certified Financial Planner`. |
| `PrevRgstns.PrevRgstn[]`          | array  | Former firms. `orgNm`, `orgPK`, `regBeginDt`, `regEndDt`.         |
| `EmpHss.EmpHs[]`                  | array  | Employment history. `fromDt` and `toDt` are `MM/YYYY`. An open `toDt` means current. |
| `OthrBuss.OthrBus`                | object | Outside business activities as one free-text `desc` string.        |
| `DRPs.DRP[]`                      | array  | Disclosure flags, `Y` or `N`. `hasRegAction`, `hasCriminal`, `hasBankrupt`, `hasCivilJudc`, `hasBond`, `hasJudgment`, `hasInvstgn`, `hasCustComp`, `hasTermination`. |

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
- A firm query on `orgPK` can match more than 10,000 people. Narrow it with a
  state, an exam code or a name before you page.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-firms`](./form-adv-firms.md),
  [`form-adv-brochures`](./form-adv-brochures.md)
- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
