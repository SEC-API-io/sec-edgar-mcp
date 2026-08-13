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
| `crd`     | string | Yes      | Digits only, 2 to 20 characters  | The firm CRD number, for example `"149777"`. Send it as a string. |

A one-character CRD is rejected. The tool takes no query and no paging.

## Output

The tool returns an **object**, not an array. This is the only Form ADV schedule
tool that does. The object always has the same three keys. An adviser with nothing to report returns
`{"1-separatelyManagedAccounts":{},"2-borrowingsAndDerivatives":{},"3-custodiansForSeparatelyManagedAccounts":[]}`.

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
Percentages come as `"58 %"`. Money comes as `"$ 1,733,996,722,410"`. Strip the
symbol, the spaces and the commas before you do arithmetic. An unanswered
percentage is the bare string `"%"`.

**There is no pagination.** The whole section arrives in one call. It is small.
Morgan Stanley Smith Barney returned about 3 KB.

## Example

Prompt: "What do the separate accounts of adviser CRD 149777 hold, and who custodies them?"

```json
{ "name": "form-adv-schedule-d-5-k", "arguments": { "crd": "149777" } }
```

Trimmed response, verified on the REST route on 2026-08-13:

```json
{
  "1-separatelyManagedAccounts": {
    "a": {
      "i-exchangeTradedEquity": { "midYear": "58 %", "endOfYear": "58 %" },
      "iii-usGovernmentBonds": { "midYear": "2 %", "endOfYear": "2 %" },
      "ix-registeredInvestmentCompanies": { "midYear": "26 %", "endOfYear": "25 %" },
      "xi-cash": { "midYear": "3 %", "endOfYear": "4 %" },
      "other": "STRUCTURED INVESTMENTS AND ANNUITIES"
    }
  },
  "3-custodiansForSeparatelyManagedAccounts": [
    {
      "a-legalName": "MORGAN STANLEY SMITH BARNEY LLC",
      "b-businessName": "MORGAN STANLEY",
      "c-locations": [{ "city": "PURCHASE", "state": "New York", "country": "United States" }],
      "d-isRelatedPerson": true,
      "e-secRegistrationNumber": "8 - 68191",
      "f-lei": "",
      "g-amountHeldAtCustodian": "$ 1,733,996,722,410"
    }
  ]
}
```

Eight asset types and the whole `2-borrowingsAndDerivatives` block were removed
to fit. The values shown are unchanged.

The probe called CRD 344073 instead. That adviser manages no separate accounts,
so the capture holds the three keys, empty:

```json
{ "name": "form-adv-schedule-d-5-k", "arguments": { "crd": "344073" } }
```

```json
{
  "1-separatelyManagedAccounts": {},
  "2-borrowingsAndDerivatives": {},
  "3-custodiansForSeparatelyManagedAccounts": []
}
```

## Limits and errors

- A CRD of one character, or with a non-digit, returns HTTP 404 and
  `{"status":404,"error":"Invalid CRD provided."}`.
- An unknown CRD returns the three keys, empty. It is not an error.
- Percentages are reported to the whole number. They do not always sum to 100.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-firms`](./form-adv-firms.md). Account counts and total assets.
- [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md). Private funds.
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
