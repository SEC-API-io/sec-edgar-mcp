# form-adv-schedule-d-5-k

Return the separately managed account disclosures of an investment adviser, from
Schedule D Section 5.K.

|                 |                                            |
| --------------- | ------------------------------------------ |
| Category        | Investment advisers                         |
| Required input  | `crd`                                       |
| Returns         | an object with three fixed keys. No `total`.  |
| Pagination      | **None.** No `from`, `size` or `sort`.      |
| REST equivalent | `GET /form-adv/schedule-d-5-k/{crd}`        |

## What it does

Item 5.K covers separately managed accounts, the client money an adviser runs
outside a pooled fund. Schedule D Section 5.K reports three things. It reports
how those assets split across twelve asset types. It reports assets, borrowings
and derivative exposure in three leverage buckets. It names the custodians that
hold the accounts.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date.

This tool does **not** return a count of separately managed accounts, although
the registry description promises one. The account count is in the Part 1
record, in `FormInfo.Part1A.Item5F`. Use
[`form-adv-firms`](./form-adv-firms.md) for it.

## When to use it

- What does this adviser hold for its separate accounts, by asset type?
- How much leverage do the adviser's separate accounts use?
- Which custodians hold the assets, and how much does each hold?
- Is the custodian a related person of the adviser?
- Did the asset mix move between mid-year and year end?

## When to use a different tool

| Situation                                | Better tool                                                     | Why                                                       |
| ---------------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------- |
| You want the number of accounts and total assets | [`form-adv-firms`](./form-adv-firms.md)                 | `FormInfo.Part1A.Item5F` holds those totals.               |
| You want pooled funds, not accounts      | [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md)   | Private funds are reported separately.                     |
| You want the actual positions            | [`form-13f-holdings`](./form-13f-holdings.md)                   | Section 5.K reports percentages, not securities.           |
| You want fund portfolio holdings         | [`form-nport`](./form-nport.md)                                 | N-PORT covers registered funds.                            |

## Input

| Parameter | Type   | Required | Constraints                      | Notes                                             |
| --------- | ------ | -------- | -------------------------------- | ------------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 2 to 20 characters  | The firm CRD number, for example `"341357"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns an **object**, not an array. This is the only Form ADV schedule
tool that does. The object always has the same three keys. Many advisers file an
empty shell. The shell keeps the full shape. Every asset-type key and every
leverage key is present, each percentage is the bare string `"%"`, each money
field is the bare string `"$"`, and
`3-custodiansForSeparatelyManagedAccounts` is an empty array.

| Field                                                | Type   | Meaning                                                             |
| ---------------------------------------------------- | ------ | ------------------------------------------------------------------- |
| `1-separatelyManagedAccounts.a`                      | object | Asset mix with `midYear` and `endOfYear` percentages.                |
| `1-separatelyManagedAccounts.b`                      | object | The same categories with `endOfYear` only. Only one of `a` and `b` is filled, per the Form ADV instructions. |
| `...a.i-exchangeTradedEquity` to `xii-other`         | object | The twelve asset types, in Roman-numeral keys. Each holds `midYear` and `endOfYear`. |
| `...a.other`                                         | string | Free text describing the `xii-other` bucket.                         |
| `2-borrowingsAndDerivatives.a-i-midYear`             | object | Mid-year figures. `a-ii-endOfYear` holds the year-end figures.       |
| `...regulatoryAssetsUnderManagement`                 | object | Assets in three leverage buckets, `lessThan10`, `between10And149`, `moreThan150`. |
| `...borrowings`                                      | object | Borrowings in the same three buckets.                                |
| `...derivativeExposures`                             | object | Per bucket, exposure by `interestRate`, `foreignExchange`, `credit`, `equity`, `commodity`, `other`. |
| `2-borrowingsAndDerivatives.a-i-optional`, `a-ii-optional` | string | Optional narrative. Usually an empty string.                  |
| `3-custodiansForSeparatelyManagedAccounts[]`         | array  | One item per custodian.                                              |
| `...a-legalName`, `b-businessName`                   | string | Custodian names.                                                     |
| `...c-locations[]`                                   | array  | `city`, `state`, `country`.                                          |
| `...d-isRelatedPerson`                               | boolean | True when the custodian is affiliated with the adviser.             |
| `...e-secRegistrationNumber`, `f-lei`                | string | Custodian identifiers. `f-lei` is often empty.                       |
| `...g-amountHeldAtCustodian`                         | string | Amount held, as a formatted string.                                  |

**Every number in this response is a formatted string, not a number.**
A percentage carries a space and a percent sign. Money carries a dollar sign, a
space and comma separators. Strip the symbol, the spaces and the commas before
you do arithmetic. An unanswered percentage is the bare string `"%"`. An
unanswered money field is the bare string `"$"`.

**There is no pagination.** The whole section arrives in one call. It is small.
CRD 341357 returned 2,641 bytes.

## Example

Prompt: "What do the separate accounts of adviser CRD 341357 hold?"

```json
{ "name": "form-adv-schedule-d-5-k", "arguments": { "crd": "341357" } }
```

An excerpt of the response from 2026-08-13. It shows the asset-class
breakdown in `1-separatelyManagedAccounts.a`:

```json
{
  "1-separatelyManagedAccounts": {
    "a": {
      "i-exchangeTradedEquity": { "midYear": "%", "endOfYear": "%" },
      "ii-nonExchangeTradedEquity": { "midYear": "%", "endOfYear": "%" },
      "iii-usGovernmentBonds": { "midYear": "%", "endOfYear": "%" },
      "iv-usStateAndLocalBonds": { "midYear": "%", "endOfYear": "%" },
      "v-sovereignBonds": { "midYear": "%", "endOfYear": "%" },
      "vi-investmentGradeCorporateBonds": { "midYear": "%", "endOfYear": "%" },
      "vii-nonInvestmentGradeCorporateBonds": { "midYear": "%", "endOfYear": "%" },
      "viii-derivatives": { "midYear": "%", "endOfYear": "%" },
      "ix-registeredInvestmentCompanies": { "midYear": "%", "endOfYear": "%" },
      "x-pooledInvestmentVehicles": { "midYear": "%", "endOfYear": "%" },
      "xi-cash": { "midYear": "%", "endOfYear": "%" },
      "xii-other": { "midYear": "%", "endOfYear": "%" },
      "other": ""
    }
  },
  "3-custodiansForSeparatelyManagedAccounts": []
}
```

The `b` block and the whole `2-borrowingsAndDerivatives` block were removed to
fit. The values shown are unchanged. This adviser left Section 5.K blank, so
every percentage is the bare string `"%"` and the custodian array is empty. The
shape is still complete. Read a bare `"%"` as no answer, not as zero.

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD returns the same three keys as an empty shell. It is not an
  error.
- Percentages are reported to the whole number. They do not always sum to 100.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-firms`](./form-adv-firms.md). Account counts and total assets.
- [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md). Private funds.
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
