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
history and no as-of date. The data updates once a day.

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

Each related person carries 17 keys. The key numbers match the question numbers
in Item 7.A. Every boolean is the filer's `Yes` or `No` answer to that question.

### Related person

| Field                        | Type    | Meaning                                                            |
| ---------------------------- | ------- | ------------------------------------------------------------------ |
| `1-nameOfRelatedPerson`      | string  | Legal name of the related person.                                   |
| `2-businessName`             | string  | Primary business name of the related person.                        |
| `3-secFileNumber`            | string  | SEC file number of the related person, if any. It starts with `801`, `8`, `866` or `802`, and carries no dash. |
| `4a-crdNumber`               | string  | CRD of the related person, if any. Use it with the other Form ADV tools. |
| `4b-cikNumbers`              | array   | CIK numbers of the related person, if any. Often empty.              |
| `4b-cikNumbers[]`            | string  | One CIK, digits only and without leading zeros. Use it with the EDGAR tools. |
| `5-typesOfRelatedPerson`     | array   | The categories the filer checked for this related person.            |
| `5-typesOfRelatedPerson[]`   | string  | One category code. See the table below.                              |
| `6-controlsRelatedPerson`    | boolean | True when the adviser controls the related person, or the related person controls the adviser. |
| `7-underCommonControl`       | boolean | True when the adviser and the related person are under common control. |
| `8a-relatedPersonActsAsCustodian` | boolean | True when the related person acts as a qualified custodian for the adviser's clients, for the advisory services the adviser gives them. |
| `8b-notOperationallyIndependent`  | boolean | True when the adviser has overcome the presumption that it is not operationally independent from the related person under rule 206(4)-2(d)(5), and so needs no surprise examination of the client funds and securities held there. It applies only when the adviser is registering or registered with the SEC and `8a-relatedPersonActsAsCustodian` is true. |
| `8c-locationOfRelatedPerson` | object  | Office location of the related person. The key is in every row. The filer fills it in when `8a-relatedPersonActsAsCustodian` is true, so it is empty in most rows. |
| `9a-exemptFromRegistration`  | boolean | True when the related person is an investment adviser that is exempt from registration. |
| `9b-exemption`               | string  | The exemption the related person relies on, when `9a-exemptFromRegistration` is true. Free text, for example `EXEMPT REPORTING ADVISER` or `ABA 2005 NO ACTION LETTER`. Empty otherwise. |
| `10a-registeredWithForeignRegulator` | boolean | True when the related person is registered with a foreign financial regulatory authority. |
| `10b-foreignRegulator`       | array   | The foreign financial regulatory authorities the related person is registered with. Empty when `10a-registeredWithForeignRegulator` is false. |
| `10b-foreignRegulator[]`     | string  | One authority, as `Country - Authority`, for example `United Kingdom - Financial Conduct Authority`. An authority outside the Form ADV list carries the prefix `Other - `. |
| `11-shareSupervisedPersons`  | boolean | True when the adviser and the related person share supervised persons. |
| `12-shareSameLocation`       | boolean | True when the two share the same physical location.                  |

### `8c-locationOfRelatedPerson`

The address of the office of the related person. All six keys are strings, and
all six are in every row. They are empty in most rows.

| Field                                | Type   | Meaning                                            |
| ------------------------------------ | ------ | -------------------------------------------------- |
| `8c-locationOfRelatedPerson.street1` | string | First line of the street address.                   |
| `8c-locationOfRelatedPerson.street2` | string | Second line, such as a floor or a suite. Empty when the address has one line. |
| `8c-locationOfRelatedPerson.city`    | string | City, in upper case.                                |
| `8c-locationOfRelatedPerson.state`   | string | State or province, written in full, for example `Utah`. |
| `8c-locationOfRelatedPerson.zipCode` | string | Postal code.                                        |
| `8c-locationOfRelatedPerson.country` | string | Country, written in full, for example `United States`. |

### Category codes in `5-typesOfRelatedPerson`

The letter prefix matches the checkbox letter in Item 7.A.5. There are sixteen
codes, `a` to `p`. The last column counts the 149 rows of CRD 149777.

| Code                                          | Form ADV category                                                                        | Rows |
| --------------------------------------------- | ---------------------------------------------------------------------------------------- | ---- |
| `a-brokerBealer`                              | (a) Broker-dealer, municipal securities dealer, or government securities broker or dealer. | 5    |
| `b-otherAdviser`                              | (b) Other investment adviser, including a financial planner.                               | 38   |
| `c-municipalAdvisor`                          | (c) Registered municipal advisor.                                                          | 0    |
| `d-swapDealer`                                | (d) Registered security-based swap dealer.                                                 | 1    |
| `e-swapParticipant`                           | (e) Major security-based swap participant.                                                 | 0    |
| `f-commodityPoolOperator`                     | (f) Commodity pool operator or commodity trading advisor, registered or exempt.             | 23   |
| `g-futuresCommissionMerchant`                 | (g) Futures commission merchant.                                                           | 2    |
| `h-bankingThriftingInstitution`               | (h) Banking or thrift institution.                                                         | 3    |
| `i-trustCompany`                              | (i) Trust company.                                                                         | 0    |
| `j-accountant`                                | (j) Accountant or accounting firm.                                                         | 0    |
| `k-lawyer`                                    | (k) Lawyer or law firm.                                                                    | 0    |
| `l-insuranceCompany`                          | (l) Insurance company or agency.                                                           | 2    |
| `m-pensionConsultant`                         | (m) Pension consultant.                                                                    | 0    |
| `n-realEstateBroker`                          | (n) Real estate broker or dealer.                                                          | 0    |
| `o-sponsorExcludingPooledInvestmentVehicles`  | (o) Sponsor or syndicator of limited partnerships, or the equivalent, excluding pooled investment vehicles. | 0 |
| `p-sponsorOfPooledInvestmentVehicles`         | (p) Sponsor, general partner or managing member, or the equivalent, of pooled investment vehicles. | 115 |

`a-brokerBealer` is **spelled that way in the API**. It is a typo for
broker-dealer. A filter on `a-brokerDealer` matches nothing.

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
