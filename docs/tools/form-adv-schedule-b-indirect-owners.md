# form-adv-schedule-b-indirect-owners

Return the Schedule B indirect owners of an investment adviser, the chain of
control above the direct owners.

|                 |                                                     |
| --------------- | --------------------------------------------------- |
| Category        | Investment advisers                                  |
| Required input  | `crd`                                                |
| Returns         | a bare JSON array. No envelope, no `total`.          |
| Pagination      | **None.** No `from`, `size` or `sort`.               |
| REST equivalent | `GET /form-adv/schedule-b-indirect-owners/{crd}`     |

## What it does

Schedule B names the 25% owners of the entities disclosed on
[Schedule A](./form-adv-schedule-a-direct-owners.md), then the owners of those
owners, up to the top of the chain. It also names every general partner, every
trustee and every elected manager of those entities. One item in the array is
one link in the chain.

Each row names the owner and, in `entityOwned`, the entity it owns. Follow
`entityOwned` from row to row to rebuild the ownership tree. Corient Private
Wealth, CRD 326262, returns 18 rows, from the immediate parent up to the
Government of Abu Dhabi.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date. The data updates once a day.

## When to use it

- Who ultimately controls this advisory firm?
- Which foreign parent sits at the top of this ownership chain?
- Did a new owner enter the chain this year?
- Is any entity in the chain a public reporting company?

## When to use a different tool

| Situation                                | Better tool                                                                 | Why                                                    |
| ---------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------ |
| You want the first level of ownership    | [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md) | Schedule A holds the direct owners and control persons. |
| You do not know the CRD number           | [`form-adv-firms`](./form-adv-firms.md)                                     | Search `Info.BusNm`, then read `Info.FirmCrdNb`.       |
| You want subsidiaries of a public company| [`subsidiaries`](./subsidiaries.md)                                         | Exhibit 21 lists what a company owns.                  |
| You want 5% holders of listed shares     | [`form-13d-13g`](./form-13d-13g.md)                                         | Beneficial ownership of an issuer is a separate filing. |

## Input

| Parameter | Type   | Required | Constraints                      | Notes                                             |
| --------- | ------ | -------- | -------------------------------- | ------------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 2 to 20 characters  | The firm CRD number, for example `"326262"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns a **bare JSON array**. There is no `total` and no wrapper
object. Most advisers report nothing here and return `[]`.

Each item has these nine fields. Schedule C amends the Schedule B information.

| Field                | Type    | Meaning                                                             |
| -------------------- | ------- | ------------------------------------------------------------------- |
| `name`               | string  | Full legal name of the indirect owner, person or entity. The list below says who must appear. |
| `ownerType`          | string  | `I` individual, `DE` domestic entity, `FE` entity incorporated or domiciled in a foreign country. |
| `entityOwned`        | string  | The entity in which the owner holds the interest. This is the link to the row below it. |
| `status`             | string  | Status held as partner, trustee, elected manager, shareholder or member. For a shareholder or member it also names the class of securities owned, when the entity issues more than one class. |
| `dateStatusAcquired` | string  | `YYYY-MM`. Month the owner acquired the status.                      |
| `ownershipCode`      | string  | Ownership band as a letter. `C` 25% to under 50%, `D` 50% to under 75%, `E` 75% or more, `F` other, meaning a general partner, a trustee or an elected manager. |
| `isControlPerson`    | boolean | True when the owner has control as the Form ADV glossary defines it. Most executive officers, and all 25% owners, general partners, elected managers and trustees, are control persons. |
| `isPublicReporting`  | boolean | True when the owner is a public reporting company under Exchange Act Section 12 or 15(d). |
| `crd`                | string  | CRD number of the owner. When the owner has no CRD number, the filer gives a social security number and date of birth, an IRS tax number or an employer ID number. The field is usually empty here. |

Schedule B names these owners of each entity listed on Schedule A. Individual
owners on Schedule A are out of scope.

- A corporation. Each shareholder that beneficially owns, can vote, or can sell
  or direct the sale of 25% or more of a class of its voting securities.
- A partnership. All general partners, and the limited and special partners that
  contributed, or can receive on dissolution, 25% or more of the capital.
- A trust. The trust and each trustee.
- A limited liability company. The members that contributed, or can receive on
  dissolution, 25% or more of the capital. Add all elected managers when the
  company has them.

A person also beneficially owns the securities of a child, stepchild,
grandchild, parent, stepparent, grandparent, spouse, sibling or in-law who
shares the same residence. The person also owns the securities that they can
acquire within 60 days through an option, a warrant or a right to purchase.

The field names differ from Schedule A. This schedule uses `status` and
`dateStatusAcquired`, Schedule A uses `titleStatus` and
`dateTitleStatusAcquired`. A parser for one schedule fails on the other.

**There is no pagination.** The whole chain arrives in one call. It is small.
The 18-row Corient response was about 4 KB.

## Example

Prompt: "Trace the ownership chain above adviser CRD 149777."

```json
{ "name": "form-adv-schedule-b-indirect-owners", "arguments": { "crd": "149777" } }
```

Morgan Stanley Smith Barney, CRD 149777, has a one-link chain. This is the full
response:

```json
[
  {
    "name": "MORGAN STANLEY",
    "ownerType": "DE",
    "entityOwned": "MORGAN STANLEY CAPITAL MANAGEMENT LLC",
    "status": "MEMBER",
    "dateStatusAcquired": "2002-10",
    "ownershipCode": "E",
    "isControlPerson": true,
    "isPublicReporting": true,
    "crd": ""
  }
]
```

`MORGAN STANLEY CAPITAL MANAGEMENT LLC` is the entity owner on Schedule A. This
row names its parent, the listed company. `isPublicReporting` is true, so you
can follow the chain into the EDGAR tools from here.

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD, and an adviser with no indirect owners, both return HTTP 200
  and `[]`. The response does not tell them apart.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md)
- [`form-adv-firms`](./form-adv-firms.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
