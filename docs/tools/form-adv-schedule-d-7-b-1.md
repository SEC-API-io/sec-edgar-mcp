# form-adv-schedule-d-7-b-1

Return the private funds an investment adviser runs, from Schedule D Section
7.B.1.

|                 |                                            |
| --------------- | ------------------------------------------ |
| Category        | Investment advisers                         |
| Required input  | `crd`                                       |
| Returns         | a bare JSON array. No envelope, no `total`. |
| Pagination      | **None.** No `from`, `size` or `sort`.      |
| REST equivalent | `GET /form-adv/schedule-d-7-b-1/{crd}`      |

## What it does

Section 7.B.1 is the private fund report. An adviser files one section for each
hedge fund, private equity fund, real estate fund or other private fund it
advises. One item in the array is one fund.

The row is rich. It carries the fund name and ID, the country of organisation,
the exclusion relied on under the Investment Company Act, the master-feeder
links, the gross asset value, the minimum investment, the beneficial owner
count, the ownership percentages, the Form D file numbers, and the service
providers. Auditors, prime brokers, custodians, administrators and marketers
each come as a nested array.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date.

## When to use it

- Which private funds does this adviser run, and how large are they?
- What is the minimum investment in this fund?
- Who audits the fund, and is the auditor PCAOB-registered?
- Who is the prime broker, the custodian and the administrator?
- Which funds rely on the 3(c)(1) exclusion rather than 3(c)(7)?
- Which Form D filings belong to this fund?

## When to use a different tool

| Situation                                | Better tool                                             | Why                                                     |
| ---------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------- |
| You want the offering itself             | [`form-d`](./form-d.md)                                 | Use the numbers in `22-formDFileNumbers`.               |
| You want registered fund holdings        | [`form-nport`](./form-nport.md)                         | N-PORT covers registered funds, not private funds.      |
| You want separate accounts, not funds    | [`form-adv-schedule-d-5-k`](./form-adv-schedule-d-5-k.md) | Section 5.K covers managed accounts.                  |
| You want the adviser's affiliates        | [`form-adv-schedule-d-7-a`](./form-adv-schedule-d-7-a.md) | Related persons are Section 7.A.                      |
| You want the adviser's total assets      | [`form-adv-firms`](./form-adv-firms.md)                 | `FormInfo.Part1A.Item5F` holds firm-level assets.       |

## Input

| Parameter | Type   | Required | Constraints                      | Notes                                          |
| --------- | ------ | -------- | -------------------------------- | ---------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 2 to 20 characters  | The firm CRD number, for example `"793"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns a **bare JSON array**. There is no `total` and no wrapper
object. An adviser with no private funds returns `[]`.

Keys are numbered to match the question numbers on Form ADV. Each fund carries
46 keys. These are the ones most people need.

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `1a-nameOfFund`                        | string  | Fund name as filed.                                               |
| `1b-fundIdentificationNumber`          | string  | Private fund ID, format `805-##########`.                        |
| `2-lawOrganizedUnder`                  | object  | `state` and `country` of organisation.                            |
| `3a-namesOfGeneralPartnerManagerTrusteeDirector` | array | Names of the general partner or manager.               |
| `4-1-exclusionUnder3c1`, `4-2-exclusionUnder3c7` | boolean | Investment Company Act exclusion relied on.             |
| `6a` to `6d`                           | mixed   | Master-feeder structure. `6d-nameIdOfMasterFund` names the master. `8a-isFundOfFunds` is the fund-of-funds flag. |
| `10-typeOfFund`                        | object  | `selectedTypes[]` plus `otherFundType` free text. Values seen: `private equity fund`, `other private fund`. |
| `11-grossAssetValue`                   | number  | Gross asset value in dollars. A plain number, not a string.       |
| `12-minInvestmentCommitment`           | number  | Minimum investment in dollars.                                    |
| `13-numberOfBeneficialOwners`          | number  | Count of beneficial owners.                                       |
| `14-percentageOwnedByYou`              | number  | Percent owned by the adviser and its related persons.             |
| `15a-percentageOwnedByFundsOfFunds`    | number  | Percent held through funds of funds.                              |
| `16-percentageOwnedByNonUnitedStatesPersons` | number | Percent held by non-US persons.                             |
| `22-formDFileNumbers`                  | array   | Form D file numbers, format `021-######`. Empty when none.        |
| `23a-1-financialStatementsAreSubjectToAnnualAudit` | boolean | Audit answer. `23a-2` is the US GAAP answer.          |
| `23b-f-auditors[]`                     | array   | `23b-name`, `23c-location`, `23d-isIndependentPublicAccountant`, `23e-isRegistered`, `23e-boardAssignedNumber`, `23f-isSubjectToInspection`. |
| `24b-e-primeBrokers[]`                 | array   | Prime brokers. `24a-fundUsesPrimeBrokers` gates it.               |
| `25b-g-custodians[]`                   | array   | `25b-legalName`, `25d-location`, `25e-isRelatedPerson`, `25f-1-secRegistrationNumber`, `25f-2-crdNumber`. |
| `26b-f-administrators[]`               | array   | `26b-name`, `26d-isRelatedPerson`, `26e-statementsProvidedTo`.    |
| `28b-g-marketers[]`                    | array   | Marketers. `28a-fundUsesMarketers` gates it.                      |

Unanswered text fields come back as the literal string `No Information Filed`,
for example `3b-filingAdvisers`. Treat that as null.

**There is no pagination.** Every fund arrives in one call. Stifel Nicolaus,
CRD 793, returned 22 funds and about 59 KB on 2026-08-13. An adviser with many
funds will produce a large text block. See
[response format](../response-format.md).

## Example

Prompt: "List the private funds advised by CRD 344073, with their size."

```json
{ "name": "form-adv-schedule-d-7-b-1", "arguments": { "crd": "344073" } }
```

That adviser has one fund. Trimmed response:

```json
[
  {
    "1a-nameOfFund": "LIBERTY PARTNERS WEALTH MANAGEMENT",
    "1b-fundIdentificationNumber": "805-2260913442",
    "2-lawOrganizedUnder": { "state": "New York", "country": "United States" },
    "4-1-exclusionUnder3c1": true,
    "4-2-exclusionUnder3c7": true,
    "10-typeOfFund": { "selectedTypes": ["private equity fund"] },
    "11-grossAssetValue": 78960522,
    "12-minInvestmentCommitment": 50000,
    "13-numberOfBeneficialOwners": 89,
    "14-percentageOwnedByYou": 10,
    "16-percentageOwnedByNonUnitedStatesPersons": 90,
    "23a-1-financialStatementsAreSubjectToAnnualAudit": true,
    "23b-f-auditors": [
      {
        "23b-name": "INDICATOR GLOBAL",
        "23c-location": { "city": "NEW YORK", "state": "New York", "country": "United States" },
        "23d-isIndependentPublicAccountant": true,
        "23e-isRegistered": false
      }
    ]
  }
]
```

Thirty keys were removed to fit. The values shown are unchanged.

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD, and an adviser with no private funds, both return HTTP 200 and
  `[]`.
- `11-grossAssetValue` is `0` for a fund that has not launched or has wound
  down. Zero is not missing data.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-schedule-d-7-a`](./form-adv-schedule-d-7-a.md). Related persons.
- [`form-adv-schedule-d-5-k`](./form-adv-schedule-d-5-k.md). Separate accounts.
- [`form-d`](./form-d.md). The offerings behind `22-formDFileNumbers`.
- [`form-adv-firms`](./form-adv-firms.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
