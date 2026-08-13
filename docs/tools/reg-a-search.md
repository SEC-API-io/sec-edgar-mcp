# reg-a-search

Search every Regulation A filing at once: offering statements, annual reports
and exit reports.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /reg-a/search`                         |

## What it does

Regulation A lets a small company sell shares to the public without a full
registration. The company files a Form 1-A to open the offering, a Form 1-K each
year, and a Form 1-Z when it closes.

This tool searches all three form families in one call. One row is one filing.
The server queries the 1-A, 1-K and 1-Z indices together, so **the shape of a row
depends on its `formType`**. A 1-A row has `summaryInfo` and `issuerInfo`. A 1-K
row has `item1` and `periodOfReport`. A 1-Z row has `summaryInfoOffering` and
`certificationSuspension`. Check `formType` before you read a field.

Coverage starts 2015-06-22, the first year of Regulation A+.

## When to use it

- Show me every Reg A filing this company has ever made.
- Which Reg A issuers filed anything last week?
- Trace one offering from its 1-A through its 1-K reports to its 1-Z close.
- Which Reg A issuers are in this state?

## When to use a different tool

| Situation                           | Better tool                         | Why                                            |
| ----------------------------------- | ----------------------------------- | ---------------------------------------------- |
| You only want offering statements   | [reg-a-form-1a](./reg-a-form-1a.md) | One row shape, no `formType` guard needed      |
| You only want annual reports        | [reg-a-form-1k](./reg-a-form-1k.md) | Same reason                                    |
| You only want exit reports          | [reg-a-form-1z](./reg-a-form-1z.md) | Same reason                                    |
| The raise is a private placement    | [form-d](./form-d.md)               | Reg D offerings file Form D                    |
| The raise is a crowdfunding campaign| [form-c](./form-c.md)               | Reg CF offerings file Form C                   |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields confirmed to return rows:

| Field                                  | Example                                          | Applies to  |
| -------------------------------------- | ------------------------------------------------ | ----------- |
| `cik`                                   | `cik:1730773`                                    | all forms   |
| `companyName`                           | `companyName:"Blue Star Foods"`                  | all forms   |
| `ticker`                                | `ticker:BSFC`                                    | all forms   |
| `fileNo`                                | `fileNo:"024-12712"`                             | all forms   |
| `formType`                              | `formType:"1-K"`                                 | all forms   |
| `filedAt`                               | `filedAt:[2024-01-01 TO 2024-12-31]`             | all forms   |
| `summaryInfo.indicateTier1Tier2Offering`| `summaryInfo.indicateTier1Tier2Offering:Tier1`   | 1-A only    |

A field that belongs to one form family simply filters the others out. That is
useful: `summaryInfo.indicateTier1Tier2Offering:Tier1` returns 2,009 rows, all
of them 1-A.

`formType` matches the exact string. `formType:"1-K"` returns 2,875 rows and
does **not** include the 125 `1-K/A` amendments. Query both, or use a wildcard
such as `formType:1-K*`, which was not verified.

Counts by form type, measured on 2026-08-13:

| `formType` | Rows   |
| ---------- | ------ |
| `1-A`      | 2,326  |
| `1-A/A`    | 4,608  |
| `1-A POS`  | 2,591  |
| `1-A-W`    | 492    |
| `1-K`      | 2,875  |
| `1-K/A`    | 125    |
| `1-Z`      | 630    |
| `1-Z/A`    | 12     |

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

These fields are on every row, whatever the form type:

| Field         | Type   | Meaning                                     |
| ------------- | ------ | ------------------------------------------- |
| `id`          | string | Internal document ID                        |
| `accessionNo` | string | EDGAR accession number                      |
| `fileNo`      | string | SEC file number, `024-` or `24R-` prefixed  |
| `formType`    | string | See the table above                         |
| `filedAt`     | string | Filing timestamp with offset                |
| `cik`         | string | Issuer CIK, no leading zeros                |
| `ticker`      | string | Usually empty. Most Reg A issuers are unlisted. |
| `companyName` | string | Issuer name                                 |

These blocks appear only on the matching form type:

| Block                                                        | Form  |
| ------------------------------------------------------------ | ----- |
| `employeesInfo[]`, `issuerInfo`, `commonEquity[]`, `preferredEquity[]`, `debtSecurities[]`, `issuerEligibility`, `applicationRule262`, `summaryInfo`, `juridictionSecuritiesOffered`, `securitiesIssued[]`, `unregisteredSecuritiesAct` | 1-A |
| `periodOfReport`, `item1`, `item1Info[]`, `item2`, `summaryInfo[]` | 1-K |
| `item1`, `summaryInfoOffering[]`, `certificationSuspension[]`, `signatureTab[]` | 1-Z |

Note the collision: `summaryInfo` is an **object** on a 1-A row and an **array**
on a 1-K row. Branch on `formType` first.

For field-by-field detail, see [reg-a-form-1a](./reg-a-form-1a.md),
[reg-a-form-1k](./reg-a-form-1k.md) and [reg-a-form-1z](./reg-a-form-1z.md).

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.

## Example

Prompt: "Show me the newest Regulation A filings."

```json
{ "name": "reg-a-search", "arguments": { "query": "cik:*", "size": 1 } }
```

Response from the capture, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "id": "7d83a0661e42e659f1e1650c12ec049f",
      "accessionNo": "0001493152-26-037452",
      "fileNo": "024-12712",
      "formType": "1-A/A",
      "filedAt": "2026-08-12T17:24:40-04:00",
      "cik": "1730773",
      "ticker": "BSFC",
      "companyName": "Blue Star Foods Corp.",
      "employeesInfo": [
        { "issuerName": "Blue Star Foods Corp.", "jurisdictionOrganization": "DE", "yearIncorporation": "2017", "sicCode": 3510, "fullTimeEmployees": 6, "partTimeEmployees": 0 }
      ],
      "summaryInfo": {
        "indicateTier1Tier2Offering": "Tier2",
        "financialStatementAuditStatus": "Audited",
        "securitiesOffered": 500000000,
        "pricePerSecurity": 0.001,
        "totalAggregateOffering": 500000
      }
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- The registry description says the tool returns "status". No `status` field
  exists. Infer status from `formType`: a 1-Z means the offering closed.
- The registry says Reg A offerings are capped at $75M. That is the legal Tier 2
  ceiling, not a field in the response.
- A bare word query is accepted but the analysis is strict. `apple` returned
  zero rows. Use a field query.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [reg-a-form-1a](./reg-a-form-1a.md), [reg-a-form-1k](./reg-a-form-1k.md), [reg-a-form-1z](./reg-a-form-1z.md)
- [form-d](./form-d.md), [form-c](./form-c.md), [form-s1-424b4](./form-s1-424b4.md)
- REST docs: <https://sec-api.io/docs/reg-a-offering-statements-api>
