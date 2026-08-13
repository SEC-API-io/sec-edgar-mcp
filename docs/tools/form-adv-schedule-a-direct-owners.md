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
more of the adviser, plus the officers and directors who control it. One item in
the array is one such person or entity.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date. You get the current picture.

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

| Field                     | Type    | Meaning                                                          |
| ------------------------- | ------- | ---------------------------------------------------------------- |
| `name`                    | string  | Owner name. `LAST, FIRST MIDDLE` for a person, legal name for an entity. |
| `ownerType`               | string  | `I` individual, `DE` domestic entity, `FE` foreign entity.       |
| `titleStatus`             | string  | Title or status held, for example `SHAREHOLDER` or `EXECUTIVE VICE PRESIDENT & DIRECTOR`. |
| `dateTitleStatusAcquired` | string  | `YYYY-MM`. Month the title or stake began.                       |
| `ownershipCode`           | string  | Form ADV ownership band as a letter. `NA` and `E` appear in live data. The API returns the letter only, never a percentage. |
| `isControlPerson`         | boolean | True when the person controls the adviser.                        |
| `isPublicReporting`       | boolean | True when the owner is a public reporting company.                |
| `crd`                     | string  | The owner's own CRD number. Empty string when it has none.        |

**There is no pagination.** The whole schedule arrives in one call. It is small.
Stifel Nicolaus, CRD 793, returned 14 rows and about 3 KB on 2026-08-13.

## Example

Prompt: "Who directly owns adviser CRD 344073?"

```json
{ "name": "form-adv-schedule-a-direct-owners", "arguments": { "crd": "344073" } }
```

The full response from the capture:

```json
[
  {
    "name": "GRAVES, DAVID GAVIN",
    "ownerType": "I",
    "titleStatus": "DIRECTOR",
    "dateTitleStatusAcquired": "2026-07",
    "ownershipCode": "E",
    "isControlPerson": true,
    "isPublicReporting": false,
    "crd": "8326994"
  }
]
```

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
