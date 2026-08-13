# form-adv-firms

Search the Form ADV firm register, the registration record of every
SEC-registered and state-registered investment adviser.

|                 |                                    |
| --------------- | ---------------------------------- |
| Category        | Investment advisers                |
| Required input  | `query`                            |
| Returns         | `{total, filings[]}`               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`    |
| REST equivalent | `POST /form-adv/firm`              |

## What it does

One item in `filings[]` is one adviser firm, not one filing. The row is the
current Form ADV Part 1 record for that CRD number. A query for a single CRD
returns exactly one row, for example `Info.FirmCrdNb:149777`. The tool holds
no filing history. `Filing[].Dt` is the date the record was last updated.

The index merges two sources, SEC-registered advisers and state-registered
advisers. The two produce **different row shapes**. See Output.

Part 1 answers arrive as raw item codes under `FormInfo.Part1A`, for example
`Item5F.Q5F2C`. The registry description promises assets under management,
client types and a disciplinary summary. They are present, but unlabelled. The
description also names an IARD number. No IARD field exists. The identifier is
the CRD number, `Info.FirmCrdNb`.

## When to use it

- What is the CRD number of the adviser called Bridgewater Associates?
- Which advisers report more than $100 billion of regulatory assets?
- Where is this adviser's main office, and what is its SEC file number?
- Which firms are exempt reporting advisers rather than registered advisers?
- Did this adviser answer yes to any Item 11 disciplinary question?

## When to use a different tool

| Situation                                    | Better tool                                                                 | Why                                                        |
| -------------------------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------- |
| You want the adviser's staff, not the firm   | [`form-adv-individuals`](./form-adv-individuals.md)                         | That tool indexes adviser representatives.                 |
| You want the plain-English disclosure        | [`form-adv-brochures`](./form-adv-brochures.md)                             | Part 2 brochures are separate documents.                   |
| You want who owns the firm                   | [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md) | Ownership sits on Schedule A and Schedule B.               |
| You want the private funds it advises        | [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md)               | Fund detail sits on Schedule D, Section 7.B.1.             |
| You want the positions the adviser holds     | [`form-13f-holdings`](./form-13f-holdings.md)                               | Form ADV carries no securities holdings.                    |
| The firm is a broker-dealer                  | [`form-x-17a-5`](./form-x-17a-5.md)                                         | Broker-dealers file FOCUS reports, not Form ADV.           |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                            |
| --------- | ------- | -------- | ------------------------ | ------------------------------------------------ |
| `query`   | string  | Yes      | Lucene syntax            | For example `Info.FirmCrdNb:149777`.             |
| `from`    | integer | No       | Minimum 0                | Offset of the first result. Default 0.           |
| `size`    | integer | No       | 1 to 50                  | Default 50.                                       |
| `sort`    | array   | No       | Elasticsearch sort array | Default `[{"Info.FirmCrdNb": {"order": "desc"}}]`. |

Query fields. The hit count on 2026-08-13 is in brackets.

| Field                             | Example                                            |
| --------------------------------- | -------------------------------------------------- |
| `Info.FirmCrdNb`                  | `Info.FirmCrdNb:149777` (1)                        |
| `Info.BusNm`                      | `Info.BusNm:"BRIDGEWATER ASSOCIATES"` (1)          |
| `Info.SECNb`                      | `Info.SECNb:"801-16048"` (1)                       |
| `MainAddr.State`                  | `MainAddr.State:NY` (4,574)                        |
| `Rgstn.FirmType`                  | `Rgstn.FirmType:ERA` (8,893)                       |
| `FormInfo.Part1A.Item5F.Q5F2C`    | `FormInfo.Part1A.Item5F.Q5F2C:[100000000000 TO *]` (270) |

`Info.LegalNm` is also present in every row. Ranges work on numeric item codes,
as the `Q5F2C` example shows. Sorting on `FormInfo.Part1A.Item5F.Q5F2C` works
too. Item 5 values are what the adviser typed into the form. The top firm by
that sort reports $11.5 trillion, which is not credible. An unknown field
returns `total: 0` with no error. See
[query language](../query-language.md).

## Output

The envelope is `{total, filings[]}`, not `{total, data[]}`. `total` is an
object, `{value, relation}`. A `relation` of `gte` means the count stopped at
10,000 and the true count is higher.

Rows come in two shapes. Both always carry `Info`, `MainAddr`, `MailingAddr`,
`Filing`, `FormInfo` and `id`. In a sample of 50 rows, 28 had the SEC shape and
22 had the state shape.

| Field                        | Type   | Meaning                                                          |
| ---------------------------- | ------ | ---------------------------------------------------------------- |
| `Info.FirmCrdNb`             | number | CRD number. The key for every other Form ADV tool. Equals `id`.  |
| `Info.BusNm`, `Info.LegalNm` | string | Business name and legal name, upper case.                        |
| `Info.SECNb`                 | string | SEC file number. `801-` for registered, `802-` for exempt.       |
| `Info.SECRgnCD`              | string | SEC regional office code, for example `NYRO`. SEC shape only.    |
| `Info.UmbrRgstn`             | string | `Y` or `N` umbrella registration flag. Rare, 4 rows in 50.       |
| `MainAddr`                   | object | `Strt1`, `Strt2`, `City`, `State`, `Cntry`, `PostlCd`, `PhNb`.   |
| `MailingAddr`                | object | Same shape. Often empty.                                          |
| `Rgstn[]`                    | array  | SEC shape. `FirmType`, `St`, `Dt`. `FirmType` is `Registered` or `ERA`. |
| `NoticeFiled.States[]`       | array  | SEC shape. State notice filings, `RgltrCd`, `St`, `Dt`.          |
| `StateRgstn.Rgltrs`          | object | State shape. State registrations.                                 |
| `ERA.Rgltrs.Rgltr[]`         | array  | State shape. Exempt reporting status per state, `Cd`, `St`, `Dt`. |
| `Filing[]`                   | array  | `Dt` and `FormVrsn`, for example `10/2021`.                      |
| `FormInfo.Part1A.Item1`      | object | Web addresses, LEI in `Q1P`, office counts. `Item3A.OrgFormNm` is the legal form, `Item5A.TtlEmp` the headcount. |
| `FormInfo.Part1A.Item5D`     | object | Client counts and assets by client type, codes `Q5DA1` to `Q5DN3`. |
| `FormInfo.Part1A.Item5F`     | object | Regulatory assets under management. `Q5F2A` plus `Q5F2B` equals `Q5F2C`, the total. |
| `FormInfo.Part1A.Item7B`     | object | `Q7B` is `Y` when the adviser advises private funds.             |
| `FormInfo.Part1A.Item11*`    | object | Disciplinary questions, `Y` or `N` only. No event detail.        |

The `QxxNN` codes are the question numbers on Form ADV Part 1A. sec-api does not
label them. An item object is empty when the adviser did not answer it. The
exempt reporting adviser in the example response has empty `Item5A` to `Item5L`.

`size` counts firms and caps at 50. `from` moves the window. A `from` plus
`size` above 10,000 returns `{"total":{"value":0},"filings":[]}` with no error
and no `relation` key. The JSON arrives as one text block. See
[response format](../response-format.md).

## Example

Prompt: "Show me the most recently registered adviser in the Form ADV register."

```json
{ "name": "form-adv-firms", "arguments": { "query": "Info.FirmCrdNb:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "filings": [
    {
      "Info": {
        "SECRgnCD": "NYRO",
        "FirmCrdNb": 344073,
        "SECNb": "802-137266",
        "BusNm": "LIBERTY PARTNERS WEALTH MANAGEMENT INC.",
        "LegalNm": "LIBERTY PARTNERS WEALTH MANAGEMENT INC."
      },
      "MainAddr": {
        "Strt1": "244 MADISON AVENUE",
        "City": "NEW YORK",
        "State": "NY",
        "Cntry": "United States",
        "PostlCd": "10016"
      },
      "Rgstn": [{ "FirmType": "ERA", "St": "ACTIVE", "Dt": "2026-08-11" }],
      "Filing": [{ "Dt": "2026-08-11", "FormVrsn": "10/2021" }],
      "id": 344073
    }
  ]
}
```

Keys were removed to fit. The values are unchanged. The default sort is CRD
descending, so the first row is the newest registrant.

## Limits and errors

- `size` above 50 returns HTTP 400 with
  `Maximum 'size' limit of 50 exceeded.`
- Omitting `size` returns 50 firms, not 10.
- `Rgstn` is absent from state-shaped rows. Those rows carry `StateRgstn` and
  `ERA` instead.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-individuals`](./form-adv-individuals.md),
  [`form-adv-brochures`](./form-adv-brochures.md)
- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md),
  [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
