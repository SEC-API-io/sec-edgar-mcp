# form-c

Search Form C filings, the offering and reporting documents of Regulation
Crowdfunding campaigns.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, data[]}`                            |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-c`                               |

## What it does

The tool searches the Form C family. One row is one filing. A row names the
issuer, the funding portal that hosts the campaign, the security offered, the
target and maximum raise, the deadline, and a short set of financials for the
last two fiscal years.

Coverage starts 2016-05-16, which is when Regulation Crowdfunding began. A
50-row sample of the newest filings held these form types: `C` (new offering),
`C/A` (amendment), `C-U` (progress update), `C-AR` (annual report) and `C-W`
(withdrawal).

Sections are omitted, not nulled, when the form type does not use them. A `C-AR`
row has no `offeringInformation` block, because an annual report offers nothing.

## When to use it

- Which crowdfunding campaigns launched this month, and for how much?
- How much revenue did this crowdfunding issuer report last fiscal year?
- Which issuers used a given funding portal, such as DealMaker or Wefunder?
- What price per share is this campaign asking, and when is the deadline?
- Which campaigns accept oversubscription?

## When to use a different tool

| Situation                            | Better tool                         | Why                                      |
| ------------------------------------ | ----------------------------------- | ---------------------------------------- |
| The raise is a private placement     | [form-d](./form-d.md)               | Reg D offerings file Form D              |
| The raise is a Regulation A offering | [reg-a-form-1a](./reg-a-form-1a.md) | Reg A offerings file Form 1-A            |
| The company is going public          | [form-s1-424b4](./form-s1-424b4.md) | Registered offerings file S-1 or 424B4   |
| You want the campaign narrative text | [extractor](./extractor.md)         | Returns the document text, not the XML   |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields confirmed to return rows:

| Field                                                          | Example                                     |
| -------------------------------------------------------------- | ------------------------------------------- |
| `cik`                                                           | `cik:1810055`                               |
| `companyName`                                                   | `companyName:"Rad Technologies"`            |
| `formType`                                                      | `formType:"C-AR"`                           |
| `filedAt`                                                       | `filedAt:[2025-01-01 TO 2025-12-31]`        |
| `issuerInformation.companyName`                                 | `...companyName:"DEALMAKER SECURITIES LLC"` |
| `issuerInformation.issuerInfo.legalStatus.jurisdictionOrganization` | `...jurisdictionOrganization:DE`        |
| `offeringInformation.maximumOfferingAmount`                     | `...maximumOfferingAmount:[1000000 TO *]`   |

Here `cik` is the bare field and takes the CIK without leading zeros. That
differs from [form-d](./form-d.md), which needs `primaryIssuer.cik` zero-padded
to 10 digits.

`issuerInformation.companyName` is the **funding portal**, not the issuer. The
issuer name is the top-level `companyName`.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

| Field                                              | Type    | Meaning                                              |
| -------------------------------------------------- | ------- | ---------------------------------------------------- |
| `id`                                                | string  | Internal document ID                                 |
| `accessionNo`                                       | string  | EDGAR accession number                               |
| `fileNo`                                            | string  | SEC file number, for example `020-37290`             |
| `formType`                                          | string  | `C`, `C/A`, `C-U`, `C-AR`, `C-W`                     |
| `filedAt`                                           | string  | Filing timestamp with offset                         |
| `periodOfReport`                                    | string  | Present on `C-AR` rows                               |
| `cik`                                               | string  | Issuer CIK, no leading zeros                         |
| `ticker`                                            | string  | Usually empty. Reg CF issuers are rarely listed.     |
| `companyName`                                       | string  | Issuer name                                          |
| `issuerInformation.issuerInfo.nameOfIssuer`         | string  | Issuer name as filed on the form                     |
| `issuerInformation.issuerInfo.legalStatus`          | object  | `legalStatusForm`, `jurisdictionOrganization`, `dateIncorporation` |
| `issuerInformation.issuerInfo.issuerAddress`        | object  | `street1`, `city`, `stateOrCountry`, `zipCode`       |
| `issuerInformation.issuerInfo.issuerWebsite`        | string  | Issuer website                                       |
| `issuerInformation.companyName`                     | string  | Funding portal or broker-dealer name                 |
| `issuerInformation.crdNumber`                       | string  | Portal CRD number                                    |
| `issuerInformation.natureOfAmendment`               | string  | Present on `C/A` rows                                |
| `offeringInformation.securityOfferedType`           | string  | `Common Stock`, `Debt`, `Other`, and similar         |
| `offeringInformation.noOfSecurityOffered`           | number  | Units offered                                        |
| `offeringInformation.price`                         | number  | Price per unit                                       |
| `offeringInformation.offeringAmount`                | number  | Target raise in dollars                              |
| `offeringInformation.maximumOfferingAmount`         | number  | Maximum raise in dollars                             |
| `offeringInformation.deadlineDate`                  | string  | `MM-DD-YYYY`                                         |
| `offeringInformation.compensationAmount`            | string  | Free text describing the portal's fee                |
| `annualReportDisclosureRequirements`                | object  | Two-year financials, see below                       |
| `signatureInfo`                                     | object  | `issuerSignature`, `signaturePersons[]`              |

`annualReportDisclosureRequirements` holds paired fields with the suffixes
`MostRecentFiscalYear` and `PriorFiscalYear`. The prefixes are `totalAsset`,
`cashEqui`, `actReceived`, `shortTermDebt`, `longTermDebt`, `revenue`,
`costGoodsSold`, `taxPaid` and `netIncome`. It also holds `currentEmployees` and
`issueJurisdictionSecuritiesOffering[]`, a list of state codes.

Dates inside the parsed blocks use `MM-DD-YYYY`. Only the top-level `filedAt`
and `periodOfReport` use ISO order. Do not mix them.

Paging is real but shallow. `from` plus `size` must stay at or below 10,000.

## Example

Prompt: "Show me the newest Regulation Crowdfunding filings."

```json
{ "name": "form-c", "arguments": { "query": "cik:*", "size": 1 } }
```

Response from the capture, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "data": [
    {
      "id": "15ffe73a6749ad9f904d580bc5c05534",
      "accessionNo": "0001493152-26-037428",
      "fileNo": "020-37290",
      "formType": "C/A",
      "filedAt": "2026-08-12T17:12:38-04:00",
      "cik": "1810055",
      "ticker": "",
      "companyName": "Rad Technologies Inc.",
      "issuerInformation": {
        "natureOfAmendment": "The purpose of this non-material amendment is to modify the equity perks.",
        "companyName": "DEALMAKER SECURITIES LLC",
        "crdNumber": "000315324"
      },
      "offeringInformation": {
        "securityOfferedType": "Other",
        "securityOfferedOtherDesc": "Class B Common Stock",
        "noOfSecurityOffered": 9333,
        "price": 1.05,
        "offeringAmount": 9995.64,
        "maximumOfferingAmount": 5000000,
        "deadlineDate": "04-30-2027"
      }
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"data":[]}` with
  no error. An empty result there does not mean there is no more data.
- The registry description promises a "use of proceeds" field. No such field
  exists in the capture or in the Node SDK example. Treat that as wrong.
- Unlike [form-s1-424b4](./form-s1-424b4.md) and [form-8k](./form-8k.md), this
  tool does not demand a colon in `query`.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [form-d](./form-d.md), [reg-a-form-1a](./reg-a-form-1a.md), [form-s1-424b4](./form-s1-424b4.md)
- [filing-search](./filing-search.md), [extractor](./extractor.md)
- REST docs: <https://sec-api.io/docs/form-c-crowdfunding-api>
