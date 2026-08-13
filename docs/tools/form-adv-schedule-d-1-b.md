# form-adv-schedule-d-1-b

Return the other business names an investment adviser trades under, from
Schedule D Section 1.B.

|                 |                                            |
| --------------- | ------------------------------------------ |
| Category        | Investment advisers                         |
| Required input  | `crd`                                       |
| Returns         | a bare JSON array. No envelope, no `total`. |
| Pagination      | **None.** No `from`, `size` or `sort`.      |
| REST equivalent | `GET /form-adv/schedule-d-1-b/{crd}`        |

## What it does

Item 1.B of Form ADV asks the adviser to list every name it does business under
that is not its legal name. Schedule D Section 1.B holds the answers. One item
in the array is one trade name, with the list of states and territories where
the adviser uses it.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date.

This is the tool that resolves a brand name to a legal entity. Morgan Stanley
Smith Barney LLC, CRD 149777, files under seven names, including
`GRAYSTONE CONSULTING` and `MORGAN STANLEY PRIVATE WEALTH MANAGEMENT`.

## When to use it

- Which brand names does this advisory firm operate under?
- The client knows the firm by a marketing name. Which legal entity is it?
- In which states does the adviser use this trade name?
- Does this adviser do business under a name that does not match its filings?

## When to use a different tool

| Situation                              | Better tool                                     | Why                                                     |
| -------------------------------------- | ----------------------------------------------- | ------------------------------------------------------- |
| You want the legal and business name    | [`form-adv-firms`](./form-adv-firms.md)         | `Info.LegalNm` and `Info.BusNm` are in the Part 1 record. |
| You want former names of an EDGAR filer | [`edgar-entities`](./edgar-entities.md)         | EDGAR tracks its own name history.                       |
| You want a ticker or CIK for a name     | [`mapping`](./mapping.md)                       | Advisers are keyed by CRD, not by CIK.                   |
| You want affiliated firms               | [`form-adv-schedule-d-7-a`](./form-adv-schedule-d-7-a.md) | Related persons are a different disclosure.   |

## Input

| Parameter | Type   | Required | Constraints                      | Notes                                             |
| --------- | ------ | -------- | -------------------------------- | ------------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 2 to 20 characters  | The firm CRD number, for example `"149777"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns a **bare JSON array**. There is no `total` and no wrapper
object. An adviser that uses only its legal name returns `[]`.

| Field           | Type   | Meaning                                                                 |
| --------------- | ------ | ----------------------------------------------------------------------- |
| `name`          | string | The other business name, upper case as filed.                            |
| `jurisdictions` | array  | Two-letter codes for the states and territories where the name is used. `GU`, `PR` and `VI` appear alongside the 50 states. |

Only these two fields are returned. There is no start date and no status.

**There is no pagination.** Every name arrives in one call. The response is
small. Seven names with full state lists came to about 2 KB.

## Example

Prompt: "Which other business names does adviser CRD 149777 use?"

```json
{ "name": "form-adv-schedule-d-1-b", "arguments": { "crd": "149777" } }
```

CRD 149777 uses seven other names. The first three:

```json
[
  {
    "name": "MORGAN STANLEY SMITH BARNEY",
    "jurisdictions": ["AL", "AK", "AZ", "AR", "CA", "CO", "CT", "DE", "DC", "FL"]
  },
  {
    "name": "MORGAN STANLEY WEALTH MANAGEMENT",
    "jurisdictions": ["AL", "AK", "AZ", "AR", "CA", "CO", "CT", "DE", "DC", "FL"]
  },
  {
    "name": "MORGAN STANLEY CONSULTING GROUP",
    "jurisdictions": ["AL", "AK", "AZ", "AR", "CA", "CO", "CT", "DE", "DC", "FL"]
  }
]
```

Four names were removed and each `jurisdictions` list was cut to its first ten
entries. The values shown are unchanged. The full lists run to 54 entries. Six
of the seven names cover every state plus `GU`, `PR` and `VI`. `GRAYSTONE
CONSULTING` omits `GU` and `PR`.

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD, and an adviser with no other names, both return HTTP 200 and
  `[]`. The response does not tell them apart.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-firms`](./form-adv-firms.md)
- [`form-adv-schedule-d-7-a`](./form-adv-schedule-d-7-a.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
