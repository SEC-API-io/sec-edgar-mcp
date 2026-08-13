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
and derivative exposure across three gross notional exposure classes. It names
the custodians that hold the accounts.

The tool reads the **latest** Form ADV on file for that CRD number. There is no
history and no as-of date. The data updates once a day.

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
exposure key is present, each percentage is the bare string `"%"`, each money
field is the bare string `"$"`, and
`3-custodiansForSeparatelyManagedAccounts` is an empty array.

### `1-separatelyManagedAccounts`

Section 5.K (1) gives the asset mix of the separate accounts. Two blocks carry
it. Block `a` holds a mid-year and an end-of-year percentage per asset type.
Block `b` holds an end-of-year percentage only.

| Field                            | Type   | Meaning                                                                   |
| -------------------------------- | ------ | ------------------------------------------------------------------------- |
| `1-separatelyManagedAccounts`    | object | Section 5.K (1). The asset mix of the separately managed accounts.        |
| `1-separatelyManagedAccounts.a`  | object | Item (a). It covers regulatory assets under management of at least $10 billion, after the adviser subtracts the amounts reported in Item 5.D.3 (d-f). |
| `1-separatelyManagedAccounts.b`  | object | Item (b). It covers regulatory assets under management below $10 billion, after the same subtraction. |

The twelve asset types follow. Each key is in block `a` and in block `b`. The
paths below drop the `1-separatelyManagedAccounts.` prefix.

| Field                                                                     | Type   | Meaning                                                     |
| ------------------------------------------------------------------------- | ------ | ----------------------------------------------------------- |
| `a.i-exchangeTradedEquity`, `b.i-exchangeTradedEquity`                    | object | (i) Exchange-traded equity securities.                       |
| `a.ii-nonExchangeTradedEquity`, `b.ii-nonExchangeTradedEquity`            | object | (ii) Non exchange-traded equity securities.                  |
| `a.iii-usGovernmentBonds`, `b.iii-usGovernmentBonds`                      | object | (iii) US government and agency bonds.                        |
| `a.iv-usStateAndLocalBonds`, `b.iv-usStateAndLocalBonds`                  | object | (iv) US state and local bonds.                               |
| `a.v-sovereignBonds`, `b.v-sovereignBonds`                                | object | (v) Sovereign bonds.                                         |
| `a.vi-investmentGradeCorporateBonds`, `b.vi-investmentGradeCorporateBonds` | object | (vi) Investment grade corporate bonds.                      |
| `a.vii-nonInvestmentGradeCorporateBonds`, `b.vii-nonInvestmentGradeCorporateBonds` | object | (vii) Non-investment grade corporate bonds.         |
| `a.viii-derivatives`, `b.viii-derivatives`                                | object | (viii) Derivatives.                                          |
| `a.ix-registeredInvestmentCompanies`, `b.ix-registeredInvestmentCompanies` | object | (ix) Securities issued by registered investment companies or business development companies. |
| `a.x-pooledInvestmentVehicles`, `b.x-pooledInvestmentVehicles`            | object | (x) Securities issued by pooled investment vehicles, other than registered investment companies or business development companies. |
| `a.xi-cash`, `b.xi-cash`                                                  | object | (xi) Cash and cash equivalents.                              |
| `a.xii-other`, `b.xii-other`                                              | object | (xii) Other assets.                                          |
| `a.other`, `b.other`                                                      | string | Free text. It describes the assets in the `xii-other` bucket. |
| `a.{assetType}.midYear`                                                   | string | The share of the separate-account assets in that asset type at mid-year. A percentage string. |
| `a.{assetType}.endOfYear`                                                 | string | The same share at the end of the year.                       |
| `b.{assetType}.endOfYear`                                                 | string | The end-of-year share. Block `b` carries no `midYear` key.   |

`{assetType}` stands for any of the twelve Roman-numeral keys above.

### `2-borrowingsAndDerivatives`

Section 5.K (2) reports how the separate accounts use borrowings and
derivatives. Three blocks carry it. Each block splits its figures across three
gross notional exposure classes. `lessThan10` is below 10%, `between10And149`
is 10% to 149%, and `moreThan150` is 150% or more.

| Field                                     | Type   | Meaning                                                                |
| ----------------------------------------- | ------ | ---------------------------------------------------------------------- |
| `2-borrowingsAndDerivatives`              | object | Section 5.K (2). Use of borrowings and derivatives.                    |
| `2-borrowingsAndDerivatives.a-i-midYear`  | object | (a)(i) Mid-year figures. Item (a) covers separate-account regulatory assets under management of at least $10 billion. |
| `2-borrowingsAndDerivatives.a-ii-endOfYear` | object | (a)(ii) End-of-year figures for the same group.                      |
| `2-borrowingsAndDerivatives.b`            | object | (b) Figures for separate-account regulatory assets under management of at least $500 million but below $10 billion. This block carries no derivative exposure. |
| `2-borrowingsAndDerivatives.a-i-optional`, `a-ii-optional` | string | Narrative description of the strategies, or of the manner, in which the adviser uses borrowings and derivatives in the separately managed accounts it advises. It can be empty. |
| `2-borrowingsAndDerivatives.b-optional`   | string | The same narrative for item (b). Most responses omit the key.          |

The three blocks share the tables below. `{block}` stands for `a-i-midYear`,
`a-ii-endOfYear` or `b`. The paths drop the `2-borrowingsAndDerivatives.`
prefix.

| Field                                                | Type   | Meaning                                                            |
| ---------------------------------------------------- | ------ | ------------------------------------------------------------------ |
| `{block}.regulatoryAssetsUnderManagement`            | object | (1) Regulatory assets under management, split across the three classes. |
| `{block}.regulatoryAssetsUnderManagement.lessThan10` | string | The assets in accounts with a gross notional exposure below 10%. A money string. |
| `{block}.regulatoryAssetsUnderManagement.between10And149` | string | The assets in accounts with a gross notional exposure of 10% to 149%. |
| `{block}.regulatoryAssetsUnderManagement.moreThan150` | string | The assets in accounts with a gross notional exposure of 150% or more. |
| `{block}.borrowings`                                 | object | (2) Borrowings, split across the same three classes.                |
| `{block}.borrowings.lessThan10`                      | string | Borrowings of the accounts in the below 10% class. A money string.  |
| `{block}.borrowings.between10And149`                 | string | Borrowings of the accounts in the 10% to 149% class.                |
| `{block}.borrowings.moreThan150`                     | string | Borrowings of the accounts in the 150% or more class.               |

Only the two item (a) blocks report derivative exposure. `{aBlock}` stands for
`a-i-midYear` or `a-ii-endOfYear`. `{class}` stands for `lessThan10`,
`between10And149` or `moreThan150`.

| Field                                          | Type   | Meaning                                                                   |
| ---------------------------------------------- | ------ | ------------------------------------------------------------------------- |
| `{aBlock}.derivativeExposures`                 | object | (3) Derivative exposure, split across the three classes and by derivative type. |
| `{aBlock}.derivativeExposures.lessThan10`      | object | Derivative exposure of the accounts with a gross notional exposure below 10%. |
| `{aBlock}.derivativeExposures.between10And149` | object | Derivative exposure of the accounts with a gross notional exposure of 10% to 149%. |
| `{aBlock}.derivativeExposures.moreThan150`     | object | Derivative exposure of the accounts with a gross notional exposure of 150% or more. |
| `{aBlock}.derivativeExposures.{class}.interestRate` | string | (a) Interest rate derivative exposure of that class. A percentage string. It can pass 100%. |
| `{aBlock}.derivativeExposures.{class}.foreignExchange` | string | (b) Foreign exchange derivative exposure of that class.        |
| `{aBlock}.derivativeExposures.{class}.credit`  | string | (c) Credit derivative exposure of that class.                             |
| `{aBlock}.derivativeExposures.{class}.equity`  | string | (d) Equity derivative exposure of that class.                             |
| `{aBlock}.derivativeExposures.{class}.commodity` | string | (e) Commodity derivative exposure of that class.                        |
| `{aBlock}.derivativeExposures.{class}.other`   | string | (f) Derivative exposure of that class that fits none of the five types above. |

### `3-custodiansForSeparatelyManagedAccounts`

Section 5.K (3) names the custodians. The array is empty when the adviser
reports none.

| Field                                                | Type    | Meaning                                                            |
| ---------------------------------------------------- | ------- | ------------------------------------------------------------------ |
| `3-custodiansForSeparatelyManagedAccounts`           | array   | One item per custodian of the separately managed accounts.          |
| `3-custodiansForSeparatelyManagedAccounts[].a-legalName` | string | Legal name of the custodian.                                    |
| `3-custodiansForSeparatelyManagedAccounts[].b-businessName` | string | Primary business name of the custodian.                      |
| `3-custodiansForSeparatelyManagedAccounts[].c-locations[]` | array | The offices of the custodian that are responsible for custody of the assets. |
| `3-custodiansForSeparatelyManagedAccounts[].c-locations[].city` | string | City of the office, in upper case.                       |
| `3-custodiansForSeparatelyManagedAccounts[].c-locations[].state` | string | State or province of the office, written in full, for example `New York`. |
| `3-custodiansForSeparatelyManagedAccounts[].c-locations[].country` | string | Country of the office, written in full, for example `United States`. |
| `3-custodiansForSeparatelyManagedAccounts[].d-isRelatedPerson` | boolean | `true` when the custodian is a related person of the firm, `false` otherwise. |
| `3-custodiansForSeparatelyManagedAccounts[].e-secRegistrationNumber` | string | SEC registration number, when the custodian is a broker-dealer and has one. Format `8 - #####`. |
| `3-custodiansForSeparatelyManagedAccounts[].f-lei` | string | Legal entity identifier. The adviser gives it when the custodian is not a broker-dealer, or is a broker-dealer with no SEC registration number. It is often empty. |
| `3-custodiansForSeparatelyManagedAccounts[].g-amountHeldAtCustodian` | string | The separate-account regulatory assets under management that the custodian holds. A money string. |

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
