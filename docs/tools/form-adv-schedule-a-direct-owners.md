# form-adv-schedule-a-direct-owners

Return the Schedule A direct owners and control persons of an investment
adviser.

|                 |                                                    |
| --------------- | -------------------------------------------------- |
| Category        | Investment advisers                                 |
| Required input  | `crd`                                               |
| Returns         | a bare JSON array. No envelope, no `total`.         |
| Pagination      | **None.** No `from`, `size` or `sort`.              |
| REST equivalent | `GET /form-adv/schedule-a-direct-owners/{crd}`      |

## What it does

Schedule A of Form ADV names every person or entity that directly owns 5% or
more of the adviser's voting securities or capital. It also names the executive
officers and the directors. One item in the array is one such person or entity.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date. You get the current picture. The data updates once a
day.

Direct means one step. The owner of the direct owner is on
[Schedule B](./form-adv-schedule-b-indirect-owners.md).

## When to use it

- Who owns this advisory firm?
- Who are the control persons, and what titles do they hold?
- Is the parent of this adviser a public reporting company?
- Since when has this shareholder held its stake?
- Which owners have their own CRD number?

## When to use a different tool

| Situation                                | Better tool                                                                     | Why                                                     |
| ---------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------- |
| You want the owner of the owner          | [`form-adv-schedule-b-indirect-owners`](./form-adv-schedule-b-indirect-owners.md) | Schedule B walks the chain upward.                    |
| You do not know the CRD number           | [`form-adv-firms`](./form-adv-firms.md)                                         | Search `Info.BusNm`, then read `Info.FirmCrdNb`.        |
| The company is a public issuer           | [`form-13d-13g`](./form-13d-13g.md)                                             | 5% holders of listed shares file 13D or 13G.            |
| You want subsidiaries, not owners        | [`subsidiaries`](./subsidiaries.md)                                             | Exhibit 21 lists what a company owns.                   |

## Input

| Parameter | Type   | Required | Constraints                       | Notes                                          |
| --------- | ------ | -------- | --------------------------------- | ---------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 2 to 20 characters   | The firm CRD number, for example `"793"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns a **bare JSON array**. There is no `total` and no wrapper
object. An adviser with nothing to report returns `[]`.

| Field                        | Type    | Meaning                                                       |
| ---------------------------- | ------- | ------------------------------------------------------------- |
| `[]`                         | array   | Direct owners and executive officers from Schedule A. Schedule C amends this information. |
| `[].name`                    | string  | Full legal name of the direct owner or executive officer. `LAST, FIRST MIDDLE` for a person, legal name for an entity. The schedule names each chief executive officer, chief financial officer, chief operations officer, chief legal officer and chief compliance officer, each director, and anyone with a similar function. It also names each 5% owner: a shareholder of a corporation, a general partner plus any limited or special partner with 5% of capital, a trust and its trustees, and, for a limited liability company, a member with 5% of capital plus every elected manager. |
| `[].ownerType`               | string  | `I` individual, `DE` domestic entity, `FE` entity incorporated or domiciled in a foreign country. |
| `[].titleStatus`             | string  | Title or status held. A board or management title, or a status such as partner, trustee, sole proprietor, elected manager, shareholder or member. For a shareholder or member it can also name the class of securities owned. |
| `[].dateTitleStatusAcquired` | string  | Date the owner acquired the title or status. `YYYY-MM`.       |
| `[].ownershipCode`           | string  | Ownership band as a letter. `NA` under 5%, `A` 5% to under 10%, `B` 10% to under 25%, `C` 25% to under 50%, `D` 50% to under 75%, `E` 75% or more. |
| `[].isControlPerson`         | boolean | True when the person has control as the Form ADV glossary defines it. Most executive officers, and all 25% owners, general partners, elected managers and trustees, are control persons. |
| `[].isPublicReporting`       | boolean | True when the owner is a public reporting company under Exchange Act Section 12 or 15(d). |
| `[].crd`                     | string  | The owner's own CRD number. Empty string when the owner has none. An owner without a CRD number gives a social security number and date of birth, an IRS tax number, or an employer ID number instead. |

**There is no pagination.** The whole schedule arrives in one call. It is small.
Stifel Nicolaus, CRD 793, returned 14 rows and about 3 KB on 2026-08-13.

## Example

Prompt: "Who directly owns adviser CRD 149777?"

```json
{ "name": "form-adv-schedule-a-direct-owners", "arguments": { "crd": "149777" } }
```

Morgan Stanley Smith Barney, CRD 149777, returned 10 rows. The first two rows:

```json
[
  {
    "name": "HANSEN, TIMOTHY GERARD",
    "ownerType": "I",
    "titleStatus": "CHIEF COMPLIANCE OFFICER (IA ONLY )",
    "dateTitleStatusAcquired": "2011-03",
    "ownershipCode": "NA",
    "isControlPerson": true,
    "isPublicReporting": false,
    "crd": "4956475"
  },
  {
    "name": "FINN, JED",
    "ownerType": "I",
    "titleStatus": "DIRECTOR, CHAIRMAN, PRESIDENT AND CHIEF EXECUTIVE OFFICER",
    "dateTitleStatusAcquired": "2024-01",
    "ownershipCode": "NA",
    "isControlPerson": true,
    "isPublicReporting": false,
    "crd": "5658048"
  }
]
```

Eight rows were removed to fit. The values shown are unchanged. The only entity
owner in the full list is `MORGAN STANLEY CAPITAL MANAGEMENT, LLC`, with
ownership code `E`.

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD returns HTTP 200 and `[]`. It is not an error.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-schedule-b-indirect-owners`](./form-adv-schedule-b-indirect-owners.md)
- [`form-adv-firms`](./form-adv-firms.md),
  [`form-adv-individuals`](./form-adv-individuals.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
