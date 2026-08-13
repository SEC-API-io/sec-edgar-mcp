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
history and no as-of date. The data updates once a day.

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
46 keys. Every key is listed below, grouped the way Form ADV groups the
questions.

### The fund, items 1 to 11

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `1a-nameOfFund`                        | string  | 1.a. Name of the private fund, as filed.                          |
| `1b-fundIdentificationNumber`          | string  | 1.b. Private fund identification number, with the `805-` prefix.  |
| `2-lawOrganizedUnder`                  | object  | 2. The state or country under whose laws the fund is organised.   |
| `2-lawOrganizedUnder.state`            | string  | The state, written in full, for example `Delaware`. Empty for a fund organised outside the United States. |
| `2-lawOrganizedUnder.country`          | string  | The country, written in full, for example `United States`.        |
| `3a-namesOfGeneralPartnerManagerTrusteeDirector[]` | array | Strings. 3.a. Names of the general partner, manager, trustee or directors, or of the persons who serve in a similar capacity. |
| `3b-filingAdvisers`                    | array or string | 3.b. Under an umbrella registration, the filing adviser and the relying advisers that sponsor or manage the fund. |
| `4-1-exclusionUnder3c1`                | boolean | 4.1. `true` when the fund qualifies for the exclusion from the definition of investment company under section 3(c)(1) of the Investment Company Act of 1940. |
| `4-2-exclusionUnder3c7`                | boolean | 4.2. The same answer for the section 3(c)(7) exclusion.           |
| `5-nameCountryOfForeignFinancialRegAuthority[]` | array | Strings. 5. The name and the country of each foreign financial regulatory authority the fund is registered with. |
| `6a-isMasterFundInMasterFeederArrangement` | boolean | 6.a. `true` when the fund is a master fund in a master-feeder arrangement. |
| `6b-nameIdOfFeederFunds[]`             | array   | 6.b. The feeder funds that invest in this fund. Item 6.a gates it. |
| `6b-nameIdOfFeederFunds[].name`        | string  | Name of the feeder fund.                                          |
| `6b-nameIdOfFeederFunds[].identificationNumber` | string | Private fund identification number of the feeder fund.   |
| `6c-isFeederFundInMasterFeederAgreement` | boolean | 6.c. `true` when the fund is a feeder fund in a master-feeder arrangement. |
| `6d-nameIdOfMasterFund`                | string  | 6.d. Name and private fund identification number of the master fund that this fund invests in. Item 6.c gates it. |
| `7a-f-feederFundDetails[]`             | array   | 7.a-f. One item per feeder fund, when the adviser files one Section 7.B.(1) for a whole master-feeder arrangement. |
| `7a-f-feederFundDetails[].7a-name`     | string  | 7.a. Name of the feeder fund.                                     |
| `7a-f-feederFundDetails[].7b-identificationNumber` | string | 7.b. Fund identification number of the feeder fund, with the `805-` prefix. |
| `7a-f-feederFundDetails[].7c-lawOrganizedUnder` | object | 7.c. The state or country under whose laws the feeder fund is organised. |
| `7a-f-feederFundDetails[].7c-lawOrganizedUnder.state` | string | The state, written in full.                    |
| `7a-f-feederFundDetails[].7c-lawOrganizedUnder.country` | string | The country, written in full.                |
| `7a-f-feederFundDetails[].7d-1-namesOfGeneralPartnerManagerTrusteeDirector[]` | array | Strings. 7.d.1. Names of the general partner, manager, trustee or directors of the feeder fund. |
| `7a-f-feederFundDetails[].7d-2-adviserActingAsSponsorManager` | string | 7.d.2. Under an umbrella registration, the filing adviser or relying adviser that sponsors or manages the feeder fund. |
| `7a-f-feederFundDetails[].7e-1-exclusionUnder3c1` | boolean | 7.e.1. `true` when the feeder fund qualifies for the section 3(c)(1) exclusion. |
| `7a-f-feederFundDetails[].7e-2-exclusionUnder3c7` | boolean | 7.e.2. The same answer for the section 3(c)(7) exclusion. |
| `7a-f-feederFundDetails[].7f-nameCountryOfForeignRegAuthority[]` | array | Strings. 7.f. The name and the country of each foreign financial regulatory authority the feeder fund is registered with. |
| `8a-isFundOfFunds`                     | boolean | 8.a. `true` when the fund is a fund of funds.                     |
| `8b-investsInFundsManagedByYouRelatedPerson` | boolean | 8.b. When 8.a is `true`, whether the fund invests in funds that the adviser or a related person manages. |
| `9-investsInSecuritiesAccordingTo6e`   | boolean | 9. Whether the fund bought, in the last fiscal year, securities issued by investment companies registered under the Investment Company Act of 1940. Money market funds stay out, to the extent Instruction 6.e allows. |
| `10-typeOfFund`                        | object  | 10. The type of the fund.                                         |
| `10-typeOfFund.selectedTypes[]`        | array   | Strings. One or more of `hedge fund`, `liquidity fund`, `private equity fund`, `real estate fund`, `securitized asset fund`, `venture capital fund` and `other private fund`. |
| `10-typeOfFund.otherFundType`          | string  | The type of the fund, when `selectedTypes` holds `other private fund`. |
| `11-grossAssetValue`                   | number  | 11. Current gross asset value of the fund, in dollars. A plain number, not a string. |

### Ownership, items 12 to 16

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `12-minInvestmentCommitment`           | number  | 12. Minimum investment commitment required of an investor in the fund, in dollars. |
| `13-numberOfBeneficialOwners`          | number  | 13. Approximate number of the fund's beneficial owners.           |
| `14-percentageOwnedByYou`              | number  | 14. Approximate percent of the fund that the adviser and its related persons beneficially own. |
| `15a-percentageOwnedByFundsOfFunds`    | number  | 15.a. Approximate percent of the fund that funds of funds beneficially own in aggregate. |
| `15b-salesAreLimited`                  | boolean | 15.b. When the fund qualifies for the section 3(c)(1) exclusion, whether sales of the fund go to qualified clients only. |
| `16-percentageOwnedByNonUnitedStatesPersons` | number | 16. Approximate percent of the fund that non-United States persons beneficially own. |

### Advisory services, items 17 to 20

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `17a-isSubadviser`                     | boolean | 17.a. Whether the adviser is a subadviser to the fund.            |
| `17b-nameAndSecFileNumber`             | string  | 17.b. Name and SEC file number of the adviser of the fund. Item 17.a gates it. |
| `18a-investmentAdvisersAdviseFund`     | boolean | 18.a. Whether investment advisers other than the ones listed in 3.b advise the fund. |
| `18b-otherAdvisers[]`                  | array   | Strings. 18.b. The name and the SEC file number of each other adviser to the fund. Item 18.a gates it. |
| `19-clientsAreSolicited`               | boolean | 19. Whether the adviser solicits its clients to invest in the fund. |
| `20-percentageClientsInvestedInFund`   | number  | 20. Approximate percent of the adviser's clients that have invested in the fund. |

### Private offering, items 21 and 22

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `21-fundReliedOnExemption`             | boolean | 21. Whether the fund has ever relied on an exemption from registration of its securities under Regulation D of the Securities Act of 1933. |
| `22-formDFileNumbers[]`                | array   | Strings. 22. The fund's Form D file numbers, format `021-######`. Item 21 gates it. Empty when none. |

### Auditors, item 23

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `23a-1-financialStatementsAreSubjectToAnnualAudit` | boolean | 23.a.1. Whether the fund's financial statements go through an annual audit. |
| `23a-2-financialStatementsPreparedWithUsGaap` | boolean | 23.a.2. When 23.a.1 is `true`, whether the statements follow US GAAP. |
| `23b-f-auditors[]`                     | array   | 23.b-f. One item per auditing firm. Item 23.a.1 gates it.         |
| `23b-f-auditors[].23b-name`            | string  | 23.b. Name of the auditing firm.                                  |
| `23b-f-auditors[].23c-location`        | object  | 23.c. The office of the auditing firm that is responsible for the audit of the fund. |
| `23b-f-auditors[].23c-location.city`   | string  | City of that office.                                              |
| `23b-f-auditors[].23c-location.state`  | string  | State or province of that office, written in full.                |
| `23b-f-auditors[].23c-location.country` | string | Country of that office, written in full.                          |
| `23b-f-auditors[].23d-isIndependentPublicAccountant` | boolean | 23.d. Whether the auditing firm is an independent public accountant. |
| `23b-f-auditors[].23e-isRegistered`    | boolean | 23.e. Whether the auditing firm is registered with the Public Company Accounting Oversight Board. |
| `23b-f-auditors[].23e-boardAssignedNumber` | string | 23.e. The number the Public Company Accounting Oversight Board assigned to the firm. `23e-isRegistered` gates it. |
| `23b-f-auditors[].23f-isSubjectToInspection` | boolean | 23.f. When `23e-isRegistered` is `true`, whether the firm goes through regular inspection by the Public Company Accounting Oversight Board under its rules. |
| `23g-financialStatementsDistributedToInvestors` | boolean | 23.g. Whether the audited financial statements for the last completed fiscal year go to the fund's investors. |
| `23h-reportsIncludeUnqualifiedOpinions` | string | 23.h. Whether every report the auditing firm made for the fund since the last annual updating amendment holds an unqualified opinion. Values: `yes`, `no`, `report not yet received`. |

### Prime brokers, item 24

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `24a-fundUsesPrimeBrokers`             | boolean | 24.a. Whether the fund uses one or more prime brokers.            |
| `24b-e-primeBrokers[]`                 | array   | 24.b-e. One item per prime broker. Item 24.a gates it.            |
| `24b-e-primeBrokers[].24b-name`        | string  | 24.b. Name of the prime broker.                                   |
| `24b-e-primeBrokers[].24c-1-secRegistrationNumber` | string | 24.c. SEC registration number of the prime broker, when it is registered with the SEC. |
| `24b-e-primeBrokers[].24c-2-crdNumber` | string  | 24.c. CRD number of the prime broker, if any.                     |
| `24b-e-primeBrokers[].24d-location`    | object  | 24.d. The office of the prime broker that the fund uses most.     |
| `24b-e-primeBrokers[].24d-location.city` | string | City of that office.                                             |
| `24b-e-primeBrokers[].24d-location.state` | string | State or province of that office, written in full.              |
| `24b-e-primeBrokers[].24d-location.country` | string | Country of that office, written in full.                      |
| `24b-e-primeBrokers[].24e-actsAsCustodian` | boolean | 24.e. Whether this prime broker also acts as custodian for some or all of the fund's assets. |

### Custodians, item 25

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `25a-fundUsesCustodians`               | boolean | 25.a. Whether the fund uses any custodian to hold some or all of its assets. The prime brokers of item 24 count. |
| `25b-g-custodians[]`                   | array   | 25.b-g. One item per custodian. Item 25.a gates it.               |
| `25b-g-custodians[].25b-legalName`     | string  | 25.b. Legal name of the custodian.                                |
| `25b-g-custodians[].25c-businessName`  | string  | 25.c. Primary business name of the custodian.                     |
| `25b-g-custodians[].25d-location`      | object  | 25.d. The office of the custodian that is responsible for custody of the fund's assets. |
| `25b-g-custodians[].25d-location.city` | string  | City of that office.                                              |
| `25b-g-custodians[].25d-location.state` | string | State or province of that office, written in full.                |
| `25b-g-custodians[].25d-location.country` | string | Country of that office, written in full.                        |
| `25b-g-custodians[].25e-isRelatedPerson` | boolean | 25.e. Whether the custodian is a related person of the firm.     |
| `25b-g-custodians[].25f-1-secRegistrationNumber` | string | 25.f. SEC registration number of the custodian, when it is a broker-dealer and has one. |
| `25b-g-custodians[].25f-2-crdNumber`   | string  | 25.f. CRD number of the custodian, if any.                        |
| `25b-g-custodians[].25g-legalEntityIdentifier` | string | 25.g. Legal entity identifier of the custodian. The adviser gives it when the custodian is not a broker-dealer, or is a broker-dealer with no SEC registration number. |

### Administrators, items 26 and 27

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `26a-fundUsesAdministrators`           | boolean | 26.a. Whether the fund uses an administrator other than the adviser's firm. |
| `26b-f-administrators[]`               | array   | 26.b-f. One item per administrator. Item 26.a gates it.           |
| `26b-f-administrators[].26b-name`      | string  | 26.b. Name of the administrator.                                  |
| `26b-f-administrators[].26c-location`  | object  | 26.c. Location of the administrator.                              |
| `26b-f-administrators[].26c-location.city` | string | City of the administrator.                                     |
| `26b-f-administrators[].26c-location.state` | string | State or province of the administrator, written in full.      |
| `26b-f-administrators[].26c-location.country` | string | Country of the administrator, written in full.              |
| `26b-f-administrators[].26d-isRelatedPerson` | boolean | 26.d. Whether the administrator is a related person of the firm. |
| `26b-f-administrators[].26e-statementsProvidedTo` | string | 26.e. Whether the administrator makes and sends investor account statements to the fund's investors. Values: `all investors`, `some investors`, `no investors`. |
| `26b-f-administrators[].26f-statementsSentBy` | string | 26.f. Who sends the investor account statements to the rest of the investors, when 26.e is `some investors` or `no investors`. It reads `not applicable` when no statements go out. |
| `27-percentageOfAssetsValuedNotByRelatedPerson` | number | 27. Percent of the fund's assets, by value, that a person who is not a related person of the adviser valued in the last fiscal year. An administrator is one such person. |

### Marketers, item 28

| Field                                  | Type    | Meaning                                                          |
| -------------------------------------- | ------- | ---------------------------------------------------------------- |
| `28a-fundUsesMarketers`                | boolean | 28.a. Whether the fund uses someone other than the adviser or its employees to market the fund. A placement agent, consultant, finder, introducer, municipal advisor or other solicitor counts. |
| `28b-g-marketers[]`                    | array   | 28.b-g. One item per marketer. Item 28.a gates it.                |
| `28b-g-marketers[].28b-isRelatedPerson` | boolean | 28.b. Whether the marketer is a related person of the firm.      |
| `28b-g-marketers[].28c-name`           | string  | 28.c. Name of the marketer.                                       |
| `28b-g-marketers[].28d-secNumber`      | string  | 28.d. SEC file number of the marketer, when it is registered with the SEC. It starts with `801`, `8` or `866`. |
| `28b-g-marketers[].28d-crdNumber`      | string  | 28.d. CRD number of the marketer, if any.                         |
| `28b-g-marketers[].28e-location`       | object  | 28.e. The office of the marketer that the fund uses most.         |
| `28b-g-marketers[].28e-location.city`  | string  | City of that office.                                              |
| `28b-g-marketers[].28e-location.state` | string  | State or province of that office, written in full.                |
| `28b-g-marketers[].28e-location.country` | string | Country of that office, written in full.                         |
| `28b-g-marketers[].28f-marketsThroughWebsites` | boolean | 28.f. Whether the marketer markets the fund through one or more websites. |
| `28b-g-marketers[].28g-websiteAddresses` | string | 28.g. The website addresses. Item 28.f gates it.                  |

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

Thirty-three keys were removed to fit. The values shown are unchanged.

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
