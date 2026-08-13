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

Schedule B names the 5% owners of the entities disclosed on
[Schedule A](./form-adv-schedule-a-direct-owners.md), then the owners of those
owners, up to the top of the chain. One item in the array is one link in the
chain.

Each row names the owner and, in `entityOwned`, the entity it owns. Follow
`entityOwned` from row to row to rebuild the ownership tree. A live response for
Corient Private Wealth, CRD 326262, ran 18 rows from the immediate parent up to
the Government of Abu Dhabi.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date.

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

| Field                | Type    | Meaning                                                             |
| -------------------- | ------- | ------------------------------------------------------------------- |
| `name`               | string  | Owner name, person or entity.                                        |
| `ownerType`          | string  | `I` individual, `DE` domestic entity, `FE` foreign entity.          |
| `entityOwned`        | string  | The entity that `name` owns. This is the link to the row below it.  |
| `status`             | string  | Status held, for example `OWNER`.                                    |
| `dateStatusAcquired` | string  | `YYYY-MM`. Month the stake began.                                    |
| `ownershipCode`      | string  | Form ADV ownership band as a letter. `D`, `E` and `F` appear in live data. The API returns the letter only, never a percentage. |
| `isControlPerson`    | boolean | True when the owner controls the entity it owns.                     |
| `isPublicReporting`  | boolean | True when the owner is a public reporting company.                   |
| `crd`                | string  | The owner's own CRD number. Usually an empty string here.            |

The field names differ from Schedule A. This schedule uses `status` and
`dateStatusAcquired`, Schedule A uses `titleStatus` and
`dateTitleStatusAcquired`. Do not reuse the same parser without checking.

**There is no pagination.** The whole chain arrives in one call. It is small.
The 18-row Corient response was about 4 KB.

## Example

Prompt: "Trace the ownership chain above adviser CRD 326262."

```json
{ "name": "form-adv-schedule-b-indirect-owners", "arguments": { "crd": "326262" } }
```

Trimmed response, verified on the REST route on 2026-08-13:

```json
[
  {
    "name": "CORIENT PARTNERS LP",
    "ownerType": "DE",
    "entityOwned": "CORIENT PRIVATE WEALTH LP",
    "status": "OWNER",
    "dateStatusAcquired": "2022-02",
    "ownershipCode": "E",
    "isControlPerson": true,
    "isPublicReporting": false,
    "crd": ""
  },
  {
    "name": "CORIENT HOLDINGS INC",
    "ownerType": "DE",
    "entityOwned": "CORIENT MANAGEMENT LLC",
    "status": "OWNER",
    "dateStatusAcquired": "2023-07",
    "ownershipCode": "F",
    "isControlPerson": true,
    "isPublicReporting": false,
    "crd": ""
  }
]
```

Sixteen rows were removed to fit. The values are unchanged.

The probe called CRD 344073 instead. That adviser reports no indirect owners, so
the capture is an empty array:

```json
{ "name": "form-adv-schedule-b-indirect-owners", "arguments": { "crd": "344073" } }
```

```json
[]
```

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
