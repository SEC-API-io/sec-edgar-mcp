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

Coverage starts 2016-05-16, when Form C in XML format became mandatory. The
index holds `C` (new offering), `C/A` (amendment), `C-U` (progress update),
`C-AR` (annual report), `C-AR/A` (amended annual report), `C-TR` (termination
report) and the matching withdrawal types `C-W`, `C/A-W`, `C-U-W`, `C-AR-W`,
`C-AR/A-W` and `C-TR-W`.

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
| `from`    | integer | no       | 0 to 10000     | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

Every field of the parsed filing is searchable. These are the ones used most.

| Field                                                          | Example                                     |
| -------------------------------------------------------------- | ------------------------------------------- |
| `cik`                                                           | `cik:1810055`                               |
| `ticker`                                                        | `ticker:*`                                  |
| `companyName`                                                   | `companyName:"Rad Technologies"`            |
| `formType`                                                      | `formType:"C-AR"`                           |
| `accessionNo`                                                   | `accessionNo:"0001493152-26-037428"`        |
| `fileNo`                                                        | `fileNo:"020-37290"`                        |
| `filedAt`                                                       | `filedAt:[2025-01-01 TO 2025-12-31]`        |
| `periodOfReport`                                                | `periodOfReport:[2025-01-01 TO 2025-12-31]` |
| `issuerInformation.companyName`                                 | `...companyName:"DEALMAKER SECURITIES LLC"` |
| `issuerInformation.crdNumber`                                   | `...crdNumber:000315324`                    |
| `issuerInformation.issuerInfo.legalStatus.jurisdictionOrganization` | `...jurisdictionOrganization:DE`        |
| `issuerInformation.issuerInfo.legalStatus.legalStatusForm`      | `...legalStatusForm:Corporation`            |
| `offeringInformation.securityOfferedType`                       | `...securityOfferedType:"Common Stock"`     |
| `offeringInformation.offeringAmount`                            | `...offeringAmount:[100000 TO *]`           |
| `offeringInformation.maximumOfferingAmount`                     | `...maximumOfferingAmount:[1000000 TO *]`   |
| `offeringInformation.overSubscriptionAccepted`                  | `...overSubscriptionAccepted:true`          |
| `annualReportDisclosureRequirements.revenueMostRecentFiscalYear` | `...revenueMostRecentFiscalYear:[1 TO *]`  |

Here `cik` is the bare field and takes the CIK without leading zeros. That
differs from [form-d](./form-d.md), which needs `primaryIssuer.cik` zero-padded
to 10 digits.

`issuerInformation.companyName` is the **funding portal**, not the issuer. The
issuer name is the top-level `companyName`.

## Output

The envelope is `{total, data[]}`. `total` is `{value, relation}`. A `relation`
of `gte` with `value` `10000` means "10,000 or more".

### Envelope

| Field            | Type    | Meaning                                                     |
| ---------------- | ------- | ----------------------------------------------------------- |
| `total.value`    | integer | Number of filings that match the query                      |
| `total.relation` | string  | `eq` for an exact count, `gte` when the count stops at 10000 |
| `data[]`         | array   | The matched filings, at most 50 per response                |

### Filing identity

| Field            | Type   | Meaning                                                                |
| ---------------- | ------ | ---------------------------------------------------------------------- |
| `id`             | string | Internal document ID                                                   |
| `accessionNo`    | string | EDGAR accession number                                                 |
| `fileNo`         | string | SEC file number. It ties together all filings of one offering process. Example `020-37290`. |
| `formType`       | string | `C`, `C/A`, `C-U`, `C-AR`, `C-AR/A`, `C-TR` and the withdrawal types    |
| `filedAt`        | string | Time EDGAR accepted the filing, with offset                            |
| `periodOfReport` | string | Period the report covers, `YYYY-MM-DD`. On an annual report it is the fiscal year end. |
| `cik`            | string | Issuer CIK, no leading zeros                                           |
| `ticker`         | string | Ticker of the issuer at the time of filing, when the issuer is listed. Usually empty. |
| `companyName`    | string | Issuer legal name as given in the filing                               |

### `issuerInformation`

| Field                                                              | Type    | Meaning                                                                       |
| ------------------------------------------------------------------ | ------- | ----------------------------------------------------------------------------- |
| `issuerInformation.isAmendment`                                     | boolean | True when the filing amends an earlier offering statement                     |
| `issuerInformation.progressUpdate`                                  | string  | Short update on how the offering progresses. Present on `C-U` rows.           |
| `issuerInformation.natureOfAmendment`                               | string  | The material changes the amendment makes. Such a change can force investors to reconfirm. Present on `C/A` rows. |
| `issuerInformation.issuerInfo.nameOfIssuer`                         | string  | Full legal name of the issuer as it appears on the form                       |
| `issuerInformation.issuerInfo.legalStatus.legalStatusForm`          | string  | Legal form of the issuer: `Corporation`, `Limited Partnership`, `Limited Liability Company`, `General Partnership`, `Business Trust` or `Other` |
| `issuerInformation.issuerInfo.legalStatus.legalStatusOtherDesc`     | string  | The legal form in words when the form is `Other`                              |
| `issuerInformation.issuerInfo.legalStatus.jurisdictionOrganization` | string  | State or country where the issuer is organized                                |
| `issuerInformation.issuerInfo.legalStatus.dateIncorporation`        | string  | Date the issuer was incorporated or organized, `MM-DD-YYYY`                   |
| `issuerInformation.issuerInfo.issuerAddress.street1`                | string  | First line of the issuer street address                                       |
| `issuerInformation.issuerInfo.issuerAddress.street2`                | string  | Second line of the issuer street address                                      |
| `issuerInformation.issuerInfo.issuerAddress.city`                   | string  | City of the issuer                                                            |
| `issuerInformation.issuerInfo.issuerAddress.stateOrCountry`         | string  | State or country of the issuer address                                        |
| `issuerInformation.issuerInfo.issuerAddress.zipCode`                | string  | Postal code of the issuer address                                             |
| `issuerInformation.issuerInfo.issuerWebsite`                        | string  | Website of the issuer                                                         |
| `issuerInformation.isCoIssuer`                                      | boolean | True when a co-issuer takes part in the offering                              |
| `issuerInformation.coIssuers[].isEdgarFiler`                        | boolean | True when the co-issuer has its own EDGAR registration                        |
| `issuerInformation.coIssuers[].coIssuerCik`                         | string  | CIK of the co-issuer                                                          |
| `issuerInformation.coIssuers[].nameOfCoIssuer`                      | string  | Legal name of the co-issuer                                                   |
| `issuerInformation.coIssuers[].coIssuerLegalStatus.legalStatusForm` | string  | Legal form of the co-issuer. Same values as the issuer.                       |
| `issuerInformation.coIssuers[].coIssuerLegalStatus.legalStatusOtherDesc` | string | The legal form in words when the co-issuer form is `Other`                |
| `issuerInformation.coIssuers[].coIssuerLegalStatus.jurisdictionOrganization` | string | State or country where the co-issuer is organized                    |
| `issuerInformation.coIssuers[].coIssuerLegalStatus.dateIncorporation` | string | Date the co-issuer was incorporated or organized, `MM-DD-YYYY`               |
| `issuerInformation.coIssuers[].coIssuerAddress.street1`             | string  | First line of the co-issuer street address                                    |
| `issuerInformation.coIssuers[].coIssuerAddress.street2`             | string  | Second line of the co-issuer street address                                   |
| `issuerInformation.coIssuers[].coIssuerAddress.city`                | string  | City of the co-issuer                                                         |
| `issuerInformation.coIssuers[].coIssuerAddress.stateOrCountry`      | string  | State or country of the co-issuer address                                     |
| `issuerInformation.coIssuers[].coIssuerAddress.zipCode`             | string  | Postal code of the co-issuer address                                          |
| `issuerInformation.coIssuers[].coIssuerWebsite`                     | string  | Website of the co-issuer                                                      |
| `issuerInformation.companyName`                                     | string  | Funding portal or broker-dealer name. This is the intermediary, not the issuer. |
| `issuerInformation.commissionCik`                                   | string  | CIK of that intermediary                                                      |
| `issuerInformation.commissionFileNumber`                            | string  | SEC file number of that intermediary                                          |
| `issuerInformation.crdNumber`                                       | string  | CRD number of that intermediary                                               |

### `offeringInformation`

| Field                                                | Type    | Meaning                                                                    |
| ---------------------------------------------------- | ------- | -------------------------------------------------------------------------- |
| `offeringInformation.securityOfferedType`            | string  | Type of security offered: `Common Stock`, `Preferred Stock`, `Debt` or `Other` |
| `offeringInformation.securityOfferedOtherDesc`       | string  | The security in words when the type is `Other`                             |
| `offeringInformation.noOfSecurityOffered`            | number  | Target number of securities in the offering                                |
| `offeringInformation.price`                          | number  | Price per security, or the fixed value that sets the offering price        |
| `offeringInformation.priceDeterminationMethod`       | string  | The method that sets the price when the price is not a fixed value         |
| `offeringInformation.offeringAmount`                 | number  | Target offering amount in dollars                                          |
| `offeringInformation.maximumOfferingAmount`          | number  | Maximum offering amount in dollars, when it differs from the target        |
| `offeringInformation.overSubscriptionAccepted`       | boolean | True when the offering accepts oversubscriptions                           |
| `offeringInformation.overSubscriptionAllocationType` | string  | How the extra securities go out: `Pro-rata basis`, `First-come, first-served basis` or `Other` |
| `offeringInformation.descOverSubscription`           | string  | The allocation rule in words when the type is `Other`                      |
| `offeringInformation.deadlineDate`                   | string  | Date by which the issuer must meet the target amount, `MM-DD-YYYY`         |
| `offeringInformation.compensationAmount`             | string  | Amount or percentage the intermediary gets for the offering. Free text.    |
| `offeringInformation.financialInterest`              | string  | Any other direct or indirect interest the intermediary holds in the issuer, or an arrangement to get one |

### `annualReportDisclosureRequirements`

Each financial field comes as a pair. The suffix `MostRecentFiscalYear` gives
the last fiscal year. The suffix `PriorFiscalYear` gives the year before it.
All amounts are in dollars.

| Field                                                                    | Type   | Meaning                                                        |
| ------------------------------------------------------------------------ | ------ | -------------------------------------------------------------- |
| `annualReportDisclosureRequirements.currentEmployees`                     | number | Headcount the issuer reports                                   |
| `annualReportDisclosureRequirements.totalAssetMostRecentFiscalYear`       | number | Total assets at the end of the last fiscal year                |
| `annualReportDisclosureRequirements.totalAssetPriorFiscalYear`            | number | Total assets at the end of the year before                     |
| `annualReportDisclosureRequirements.cashEquiMostRecentFiscalYear`         | number | Cash and cash equivalents, last fiscal year                    |
| `annualReportDisclosureRequirements.cashEquiPriorFiscalYear`              | number | Cash and cash equivalents, year before                         |
| `annualReportDisclosureRequirements.actReceivedMostRecentFiscalYear`      | number | Accounts receivable or a like measure such as trade receivables, last fiscal year |
| `annualReportDisclosureRequirements.actReceivedPriorFiscalYear`           | number | Accounts receivable, year before                               |
| `annualReportDisclosureRequirements.shortTermDebtMostRecentFiscalYear`    | number | Short-term debt, last fiscal year                              |
| `annualReportDisclosureRequirements.shortTermDebtPriorFiscalYear`         | number | Short-term debt, year before                                   |
| `annualReportDisclosureRequirements.longTermDebtMostRecentFiscalYear`     | number | Long-term debt, last fiscal year                               |
| `annualReportDisclosureRequirements.longTermDebtPriorFiscalYear`          | number | Long-term debt, year before                                    |
| `annualReportDisclosureRequirements.revenueMostRecentFiscalYear`          | number | Revenue or sales, last fiscal year                             |
| `annualReportDisclosureRequirements.revenuePriorFiscalYear`               | number | Revenue or sales, year before                                  |
| `annualReportDisclosureRequirements.costGoodsSoldMostRecentFiscalYear`    | number | Cost of goods sold, last fiscal year                           |
| `annualReportDisclosureRequirements.costGoodsSoldPriorFiscalYear`         | number | Cost of goods sold, year before                                |
| `annualReportDisclosureRequirements.taxPaidMostRecentFiscalYear`          | number | Taxes paid, last fiscal year                                   |
| `annualReportDisclosureRequirements.taxPaidPriorFiscalYear`               | number | Taxes paid, year before                                        |
| `annualReportDisclosureRequirements.netIncomeMostRecentFiscalYear`        | number | Net income, last fiscal year. A loss is negative.              |
| `annualReportDisclosureRequirements.netIncomePriorFiscalYear`             | number | Net income, year before                                        |
| `annualReportDisclosureRequirements.issueJurisdictionSecuritiesOffering[]` | array of strings | Two-letter codes of the states and territories where the issuer plans to offer the securities |

### `signatureInfo`

| Field                                          | Type   | Meaning                                                     |
| ---------------------------------------------- | ------ | ----------------------------------------------------------- |
| `signatureInfo.issuerSignature.issuer`         | string | Name of the issuer, or of the person who signs for it       |
| `signatureInfo.issuerSignature.issuerSignature` | string | The signature text of the issuer                           |
| `signatureInfo.issuerSignature.issuerTitle`    | string | Title of the person who signs for the issuer                |
| `signatureInfo.signaturePersons[].personSignature` | string | The signature text of an added signer                   |
| `signatureInfo.signaturePersons[].personTitle` | string | Title of that signer                                        |
| `signatureInfo.signaturePersons[].signatureDate` | string | Date that signer signed, `MM-DD-YYYY`                     |

Dates inside the parsed blocks use `MM-DD-YYYY`. Only the top-level `filedAt`
and `periodOfReport` use ISO order. The two formats are not interchangeable.

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.

## Example

Prompt: "Show me the newest Regulation Crowdfunding filings."

```json
{ "name": "form-c", "arguments": { "query": "cik:*", "size": 1 } }
```

Response, trimmed for length:

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
  exists in any response.
- Unlike [form-s1-424b4](./form-s1-424b4.md) and [form-8k](./form-8k.md), this
  tool does not demand a colon in `query`.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [form-d](./form-d.md), [reg-a-form-1a](./reg-a-form-1a.md), [form-s1-424b4](./form-s1-424b4.md)
- [filing-search](./filing-search.md), [extractor](./extractor.md)
- REST docs: <https://sec-api.io/docs/form-c-crowdfunding-api>
