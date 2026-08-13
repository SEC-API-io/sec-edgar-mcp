# form-adv-schedule-d-7-a

Return the financial industry affiliations of an investment adviser, the related
persons disclosed on Schedule D Section 7.A.

|                 |                                            |
| --------------- | ------------------------------------------ |
| Category        | Investment advisers                         |
| Required input  | `crd`                                       |
| Returns         | a bare JSON array. No envelope, no `total`. |
| Pagination      | **None.** No `from`, `size` or `sort`.      |
| REST equivalent | `GET /form-adv/schedule-d-7-a/{crd}`        |

## What it does

Item 7.A asks the adviser to name every related person in the financial
industry. That covers broker-dealers, other advisers, banks, insurance
companies, commodity pool operators, futures commission merchants and sponsors
of pooled vehicles. Schedule D Section 7.A holds one row per related person,
with the control relationship, the registrations and the shared-staff answers.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date.

This is the largest Form ADV schedule for a big group. Morgan Stanley Smith
Barney, CRD 149777, returned **149 rows and about 101 KB** on 2026-08-13.

## When to use it

- Which broker-dealer is affiliated with this adviser?
- How many affiliated entities does this adviser group have?
- Does the adviser control the related person, or only share control with it?
- Which affiliates are registered with a foreign regulator?
- Do the adviser and the affiliate share supervised persons or an office?

## When to use a different tool

| Situation                                   | Better tool                                                       | Why                                                   |
| ------------------------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------- |
| You want who owns the adviser               | [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md) | Ownership is a different disclosure.        |
| You want the affiliate's own ADV record     | [`form-adv-firms`](./form-adv-firms.md)                           | Take `4a-crdNumber` and look the firm up.             |
| You want the affiliate's FOCUS report       | [`form-x-17a-5`](./form-x-17a-5.md)                               | Broker-dealer financials sit there.                    |
| You want legal subsidiaries of an issuer    | [`subsidiaries`](./subsidiaries.md)                               | Exhibit 21 is a corporate list, not a Form ADV answer. |
| You want the trade names of the adviser     | [`form-adv-schedule-d-1-b`](./form-adv-schedule-d-1-b.md)         | Other business names are Section 1.B.                  |

## Input

| Parameter | Type   | Required | Constraints                      | Notes                                             |
| --------- | ------ | -------- | -------------------------------- | ------------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 2 to 20 characters  | The firm CRD number, for example `"149777"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns a **bare JSON array**. There is no `total` and no wrapper
object. An adviser with no affiliations returns `[]`.

| Field                        | Type    | Meaning                                                            |
| ---------------------------- | ------- | ------------------------------------------------------------------ |
| `1-nameOfRelatedPerson`      | string  | Legal name of the related person.                                   |
| `2-businessName`             | string  | Name it does business under.                                        |
| `3-secFileNumber`            | string  | SEC file number, digits only, no dash.                              |
| `4a-crdNumber`               | string  | CRD of the related person. Use it with the other Form ADV tools.    |
| `4b-cikNumbers`              | array   | CIK numbers. Often empty.                                            |
| `5-typesOfRelatedPerson`     | array   | One or more type codes. See the list below.                          |
| `6-controlsRelatedPerson`    | boolean | True when the adviser controls the related person.                   |
| `7-underCommonControl`       | boolean | True when both sit under the same parent.                            |
| `8a-relatedPersonActsAsCustodian` | boolean | True when the related person holds client assets.               |
| `8b-notOperationallyIndependent`  | boolean | Custody independence answer.                                     |
| `8c-locationOfRelatedPerson` | object  | `street1`, `street2`, `city`, `state`, `zipCode`, `country`. Often all empty. |
| `9a-exemptFromRegistration`  | boolean | True when the related person is exempt. `9b-exemption` gives the reason. |
| `10a-registeredWithForeignRegulator` | boolean | Foreign registration answer. `10b-foreignRegulator` lists them. |
| `11-shareSupervisedPersons`  | boolean | True when staff are shared with the adviser.                         |
| `12-shareSameLocation`       | boolean | True when the two share an office.                                   |

Type codes in the 149 rows for CRD 149777, with the row count:
`p-sponsorOfPooledInvestmentVehicles` (115), `b-otherAdviser` (38),
`f-commodityPoolOperator` (23), `a-brokerBealer` (5), `h-bankingThriftingInstitution` (3),
`g-futuresCommissionMerchant` (2), `l-insuranceCompany` (2), `d-swapDealer` (1).

`a-brokerBealer` is **spelled that way in the API**. It is a typo for
broker-dealer. A filter on `a-brokerDealer` matches nothing. The letter prefix
matches the checkbox letter on Form ADV, so other letters exist that this
adviser did not use.

**There is no pagination.** Every related person arrives in one call. For a
large group that is roughly 100 KB of JSON in a single text block. Expect it to
fill a large part of a small context window. See
[response format](../response-format.md).

## Example

Prompt: "List the financial industry affiliates of adviser CRD 149777."

```json
{ "name": "form-adv-schedule-d-7-a", "arguments": { "crd": "149777" } }
```

CRD 149777 has 149 related persons. The first row:

```json
[
  {
    "1-nameOfRelatedPerson": "MS CAPITAL PARTNERS ADVISER INC.",
    "2-businessName": "MS CAPITAL PARTNERS ADVISER INC.",
    "3-secFileNumber": "80169426",
    "4a-crdNumber": "147521",
    "4b-cikNumbers": [],
    "5-typesOfRelatedPerson": ["b-otherAdviser", "f-commodityPoolOperator"],
    "6-controlsRelatedPerson": false,
    "7-underCommonControl": false,
    "8a-relatedPersonActsAsCustodian": false,
    "8b-notOperationallyIndependent": false,
    "9a-exemptFromRegistration": false,
    "9b-exemption": "",
    "10a-registeredWithForeignRegulator": false,
    "10b-foreignRegulator": [],
    "11-shareSupervisedPersons": true,
    "12-shareSameLocation": false
  }
]
```

The other 148 rows and the empty `8c-locationOfRelatedPerson` block were removed
to fit. The values shown are unchanged. Row two is
`MORGAN STANLEY PRIVATE EQUITY ASIA, INC.`, CRD 134366, with
`7-underCommonControl` set to true.

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD, and an adviser with no affiliates, both return HTTP 200 and
  `[]`.
- There is no way to ask for fewer rows. Plan for about 100 KB on a large group.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md). Private funds.
- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md)
- [`form-adv-firms`](./form-adv-firms.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
