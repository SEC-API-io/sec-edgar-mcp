# form-d

Search Form D notices, the filings that private companies and funds submit for
exempt offerings under Regulation D.

|                 |                                              |
| --------------- | -------------------------------------------- |
| Category        | Offerings and registrations                  |
| Required input  | `query`                                      |
| Returns         | `{total, offerings[]}`                       |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort` |
| REST equivalent | `POST /form-d`                               |

## What it does

The tool searches Form D and Form D/A notices. One row is one notice. A notice
tells you who is raising money, how much, from how many investors, and which
people are behind the issuer. This is the main public record of venture rounds,
private funds and other private placements.

Coverage starts 2008-09-17. The rows carry the full XML of the notice, mapped to
JSON, so nothing is summarised or rewritten.

**Read the envelope carefully. This tool returns `offerings[]`, not `data[]`.**
It is the only tool in the server that uses that key.

## When to use it

- Which companies in my state raised over $10M in a private round this year?
- How much of the announced round has actually been sold?
- Who are the officers, directors and promoters named on this raise?
- Which venture funds filed a new Form D last week?
- Did this offering accept non-accredited investors?

## When to use a different tool

| Situation                              | Better tool                         | Why                                          |
| -------------------------------------- | ----------------------------------- | -------------------------------------------- |
| The raise is a crowdfunding campaign   | [form-c](./form-c.md)               | Reg CF offerings file Form C, not Form D     |
| The raise is a Regulation A offering   | [reg-a-form-1a](./reg-a-form-1a.md) | Reg A offerings file Form 1-A                |
| The company is going public            | [form-s1-424b4](./form-s1-424b4.md) | Registered offerings file S-1 or 424B4       |
| You only want the filing index entry   | [filing-search](./filing-search.md) | Lists the filings without parsing the XML    |

## Input

| Parameter | Type    | Required | Constraints    | Notes                                     |
| --------- | ------- | -------- | -------------- | ----------------------------------------- |
| `query`   | string  | yes      | none           | Lucene syntax. A bare word is accepted.   |
| `from`    | integer | no       | 0 or more      | Offset.                                   |
| `size`    | integer | no       | 1 to 50        | Default 50. Above 50 returns HTTP 400.    |
| `sort`    | array   | no       | ES sort clause | Default `[{"filedAt":{"order":"desc"}}]`. |

Query fields:

Every property of the notice is searchable. These are the ones used most.

| Field                                                    | Example                                    |
| -------------------------------------------------------- | ------------------------------------------ |
| `primaryIssuer.cik`                                       | `primaryIssuer.cik:0002149897`             |
| `primaryIssuer.entityName`                                | `primaryIssuer.entityName:"Fusion VC"`     |
| `primaryIssuer.entityType`                                | `primaryIssuer.entityType:"Limited Partnership"` |
| `primaryIssuer.issuerAddress.stateOrCountry`              | `...stateOrCountry:CA`                     |
| `primaryIssuer.yearOfInc.value`                           | `primaryIssuer.yearOfInc.value:2026`       |
| `submissionType`                                          | `submissionType:"D/A"`                     |
| `accessionNo`                                             | `accessionNo:"0002149897-26-000001"`       |
| `filedAt`                                                 | `filedAt:[2024-01-01 TO 2024-12-31]`       |
| `offeringData.offeringSalesAmounts.totalOfferingAmount`   | `...totalOfferingAmount:[1000000 TO *]`    |
| `offeringData.offeringSalesAmounts.totalAmountSold`       | `...totalAmountSold:[100000000 TO *]`      |
| `offeringData.offeringSalesAmounts.totalRemaining`        | `...totalRemaining:[1 TO *]`               |
| `offeringData.minimumInvestmentAccepted`                  | `...minimumInvestmentAccepted:[25000 TO *]` |
| `offeringData.industryGroup.industryGroupType`            | `...industryGroupType:"Pooled Investment Fund"` |
| `offeringData.industryGroup.investmentFundInfo.investmentFundType` | `...investmentFundType:"Venture Capital Fund"` |
| `offeringData.issuerSize.revenueRange`                    | `...revenueRange:"Decline to Disclose"`    |
| `offeringData.issuerSize.aggregateNetAssetValueRange`     | `...aggregateNetAssetValueRange:*`         |
| `offeringData.investors.hasNonAccreditedInvestors`        | `...hasNonAccreditedInvestors:true`        |
| `offeringData.investors.numberNonAccreditedInvestors`     | `...numberNonAccreditedInvestors:[1 TO *]` |
| `offeringData.federalExemptionsExclusions.item`           | `...item:"06b"`                            |
| `offeringData.typeOfFiling.dateOfFirstSale.value`         | `...value:[2026-01-01 TO 2026-12-31]`      |
| `offeringData.typesOfSecuritiesOffered.isEquityType`      | `...isEquityType:true`                     |
| `offeringData.businessCombinationTransaction.isBusinessCombinationTransaction` | `...isBusinessCombinationTransaction:true` |
| `offeringData.salesCommissionsFindersFees.salesCommissions.dollarAmount` | `...dollarAmount:[1 TO *]`   |
| `offeringData.salesCompensationList.recipient.associatedBDName`   | `...associatedBDName:*`            |
| `relatedPersonsList.relatedPersonInfo.relatedPersonName.lastName` | `...lastName:Musk`                 |

Two traps in this list:

1. There is **no bare `cik` field**. `cik:*` returns zero rows. The CIK lives at
   `primaryIssuer.cik`.
2. `primaryIssuer.cik` is stored zero-padded to 10 digits.
   `primaryIssuer.cik:2149897` returns nothing.
   `primaryIssuer.cik:0002149897` returns the row.

## Output

The envelope is `{total, offerings[]}`. `total` is `{value, relation}`. A
`relation` of `gte` with `value` `10000` means "10,000 or more", not exactly
10,000.

A field is absent when the filer leaves that item of the form blank.

### Envelope

| Field            | Type   | Meaning                                                     |
| ---------------- | ------ | ----------------------------------------------------------- |
| `total.value`    | number | Count of notices that match the query. Counting stops at 10,000. |
| `total.relation` | string | `eq` if `value` is exact, `gte` if the real count is at or above `value` |
| `offerings[]`    | array  | One element per Form D or Form D/A notice                   |

### Row fields

| Field            | Type   | Meaning                                                     |
| ---------------- | ------ | ----------------------------------------------------------- |
| `id`             | string | Internal document ID                                        |
| `accessionNo`    | string | EDGAR accession number of the notice                        |
| `filedAt`        | string | Date EDGAR published the notice, with offset                |
| `submissionType` | string | `D` for a new notice, `D/A` for an amendment                |
| `testOrLive`     | string | Marks a real filing or a filer test. `LIVE` is a real filing. |
| `schemaVersion`  | string | Version of the Form D XML schema, for example `X0708`       |

### `primaryIssuer`

The company or fund that raises the money.

| Field                                                    | Type    | Meaning                                                     |
| -------------------------------------------------------- | ------- | ----------------------------------------------------------- |
| `primaryIssuer.cik`                                       | string  | Issuer CIK, zero-padded to 10 digits                        |
| `primaryIssuer.entityName`                                | string  | Legal name of the issuer                                    |
| `primaryIssuer.issuerAddress.street1`                     | string  | Street line 1                                               |
| `primaryIssuer.issuerAddress.street2`                     | string  | Street line 2, for example a suite number                   |
| `primaryIssuer.issuerAddress.city`                        | string  | City                                                        |
| `primaryIssuer.issuerAddress.stateOrCountry`              | string  | State or country code, for example `DE`, or `X0` for the UK |
| `primaryIssuer.issuerAddress.stateOrCountryDescription`   | string  | Full name of that state or country                          |
| `primaryIssuer.issuerAddress.zipCode`                     | string  | Postal code                                                 |
| `primaryIssuer.issuerPhoneNumber`                         | string  | Phone number of the issuer                                  |
| `primaryIssuer.jurisdictionOfInc`                         | string  | Jurisdiction of incorporation, for example `DELAWARE`       |
| `primaryIssuer.entityType`                                | string  | `Corporation`, `Limited Partnership`, `Limited Liability Company`, `General Partnership`, `Business Trust` or `Other` |
| `primaryIssuer.entityTypeOtherDesc`                       | string  | The legal form in words when `entityType` is `Other`        |
| `primaryIssuer.yearOfInc.value`                           | string  | Year the issuer was incorporated                            |
| `primaryIssuer.yearOfInc.withinFiveYears`                 | boolean | True if the issuer was formed in the last five years        |
| `primaryIssuer.yearOfInc.overFiveYears`                   | boolean | True if the issuer was formed more than five years ago      |
| `primaryIssuer.yearOfInc.yetToBeFormed`                   | boolean | True if the issuer is not yet formed                        |
| `primaryIssuer.issuerPreviousNameList[]`                  | array   | Former names of the issuer                                  |
| `primaryIssuer.issuerPreviousNameList[].value`            | string  | One former name. `None` when the issuer never changed name. |
| `primaryIssuer.issuerPreviousNameList[].previousName[]`   | array   | The same former names as plain strings. Some rows use this shape in place of `value`. |
| `primaryIssuer.edgarPreviousNameList[]`                   | array   | Former names of the issuer on EDGAR                         |
| `primaryIssuer.edgarPreviousNameList[].value`             | string  | One former EDGAR name. `None` when there is none.           |

### `relatedPersonsList`

The people and firms behind the issuer.

| Field                                                                              | Type    | Meaning                                                     |
| ---------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------- |
| `relatedPersonsList.over100PersonsFlag`                                             | boolean | True if the issuer has more than 100 related persons        |
| `relatedPersonsList.relatedPersonInfo[]`                                            | array   | One element per related person                              |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonName.firstName`                | string  | First name. A related firm puts part of its name here.      |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonName.middleName`               | string  | Middle name of the person                                   |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonName.lastName`                 | string  | Last name of the person, or the name of a related firm      |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonAddress.street1`               | string  | Street line 1                                               |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonAddress.street2`               | string  | Street line 2, for example a suite number                   |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonAddress.city`                  | string  | City                                                        |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonAddress.stateOrCountry`        | string  | State or country code                                       |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonAddress.stateOrCountryDescription` | string | Full name of that state or country                        |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonAddress.zipCode`               | string  | Postal code                                                 |
| `relatedPersonsList.relatedPersonInfo[].relatedPersonRelationshipList.relationship[]` | array | One or more of `Executive Officer`, `Director`, `Promoter`  |
| `relatedPersonsList.relatedPersonInfo[].relationshipClarification`                  | string  | Free text on the relationship, for example `Manager of the Issuer` |

### `offeringData`: the offering

| Field                                                              | Type    | Meaning                                                     |
| ------------------------------------------------------------------ | ------- | ----------------------------------------------------------- |
| `offeringData.industryGroup.industryGroupType`                      | string  | Industry of the issuer, for example `Pooled Investment Fund`, `Technology`, `Real Estate`, `Energy` |
| `offeringData.industryGroup.investmentFundInfo.investmentFundType`  | string  | For a pooled investment fund: `Hedge Fund`, `Private Equity Fund`, `Venture Capital Fund` or `Other Investment Fund` |
| `offeringData.industryGroup.investmentFundInfo.is40Act`             | boolean | True if the fund is registered as an investment company under the Investment Company Act of 1940 |
| `offeringData.issuerSize.revenueRange`                              | string  | Revenue band of the issuer, from `No Revenues` to `Over $100,000,000`, or `Decline to Disclose` |
| `offeringData.issuerSize.aggregateNetAssetValueRange`               | string  | Net asset value band. A fund reports this in place of revenue. |
| `offeringData.federalExemptionsExclusions.item[]`                   | array   | Codes of the exemptions and exclusions claimed under the Securities Act, for example `06b`, `06c`, `3C`, `3C.1` |
| `offeringData.typeOfFiling.newOrAmendment.isAmendment`              | boolean | True if this notice amends an earlier notice                |
| `offeringData.typeOfFiling.newOrAmendment.previousAccessionNumber`  | string  | Accession number of the notice that this one amends         |
| `offeringData.typeOfFiling.dateOfFirstSale.value`                   | string  | Date of the first sale of securities in the offering        |
| `offeringData.typeOfFiling.dateOfFirstSale.yetToOccur`              | boolean | True if the first sale has not happened yet                 |
| `offeringData.durationOfOffering.moreThanOneYear`                   | boolean | True if the offering is planned to last more than one year  |
| `offeringData.typesOfSecuritiesOffered.isEquityType`                | boolean | Equity is offered                                           |
| `offeringData.typesOfSecuritiesOffered.isDebtType`                  | boolean | Debt is offered                                             |
| `offeringData.typesOfSecuritiesOffered.isOptionToAcquireType`       | boolean | An option, warrant or other right to acquire a security is offered |
| `offeringData.typesOfSecuritiesOffered.isSecurityToBeAcquiredType`  | boolean | The security that such a right buys is offered              |
| `offeringData.typesOfSecuritiesOffered.isPooledInvestmentFundType`  | boolean | Interests in a pooled investment fund are offered           |
| `offeringData.typesOfSecuritiesOffered.isTenantInCommonType`        | boolean | Tenant-in-common securities are offered                     |
| `offeringData.typesOfSecuritiesOffered.isMineralPropertyType`       | boolean | Mineral property securities are offered                     |
| `offeringData.typesOfSecuritiesOffered.isOtherType`                 | boolean | A security outside the listed types is offered              |
| `offeringData.typesOfSecuritiesOffered.descriptionOfOtherType`      | string  | The security named when `isOtherType` is true               |
| `offeringData.businessCombinationTransaction.isBusinessCombinationTransaction` | boolean | True if the offering goes with a merger, acquisition or exchange offer |
| `offeringData.businessCombinationTransaction.clarificationOfResponse` | string | Free text on that transaction                              |

### `offeringData`: amounts and investors

| Field                                                                   | Type    | Meaning                                                     |
| ----------------------------------------------------------------------- | ------- | ----------------------------------------------------------- |
| `offeringData.minimumInvestmentAccepted`                                 | number  | Smallest investment in dollars the issuer takes from an outside investor |
| `offeringData.offeringSalesAmounts.totalOfferingAmount`                  | number  | Dollar size of the whole offering. `-1` means indefinite.   |
| `offeringData.offeringSalesAmounts.totalAmountSold`                      | number  | Dollars sold so far                                         |
| `offeringData.offeringSalesAmounts.totalRemaining`                       | number  | Dollars still on offer. `-1` means indefinite.              |
| `offeringData.offeringSalesAmounts.clarificationOfResponse`              | string  | Free text on these amounts                                  |
| `offeringData.investors.hasNonAccreditedInvestors`                       | boolean | True if the issuer sold, or may sell, to non-accredited investors |
| `offeringData.investors.numberNonAccreditedInvestors`                    | number  | Count of non-accredited investors that already invested     |
| `offeringData.investors.totalNumberAlreadyInvested`                      | number  | Count of all investors that already invested                |
| `offeringData.salesCommissionsFindersFees.salesCommissions.dollarAmount` | number  | Sales commissions paid, in dollars                          |
| `offeringData.salesCommissionsFindersFees.salesCommissions.isEstimate`   | boolean | True if that commission figure is an estimate               |
| `offeringData.salesCommissionsFindersFees.findersFees.dollarAmount`      | number  | Finders' fees paid, in dollars                              |
| `offeringData.salesCommissionsFindersFees.findersFees.isEstimate`        | boolean | True if that fee figure is an estimate                      |
| `offeringData.salesCommissionsFindersFees.clarificationOfResponse`       | string  | Free text on the commissions and fees                       |
| `offeringData.useOfProceeds.grossProceedsUsed.dollarAmount`              | number  | Gross proceeds that go to executive officers, directors and promoters |
| `offeringData.useOfProceeds.grossProceedsUsed.isEstimate`                | boolean | True if that amount is an estimate                          |
| `offeringData.useOfProceeds.clarificationOfResponse`                     | string  | Free text on the use of proceeds                            |

### `offeringData.salesCompensationList`

The brokers and finders paid to sell the offering. The object is empty when the
issuer paid nobody.

| Field                                                                                 | Type    | Meaning                                                     |
| ------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------- |
| `offeringData.salesCompensationList.over100RecipientFlag`                              | boolean | True if more than 100 recipients take compensation          |
| `offeringData.salesCompensationList.recipient[]`                                       | array   | One element per paid recipient                              |
| `offeringData.salesCompensationList.recipient[].recipientName`                         | string  | Name of the recipient                                       |
| `offeringData.salesCompensationList.recipient[].recipientCRDNumber`                    | string  | CRD number of the recipient                                 |
| `offeringData.salesCompensationList.recipient[].associatedBDName`                      | string  | Broker-dealer the recipient works with                      |
| `offeringData.salesCompensationList.recipient[].associatedBDCRDNumber`                 | string  | CRD number of that broker-dealer                            |
| `offeringData.salesCompensationList.recipient[].recipientAddress.street1`              | string  | Street line 1                                               |
| `offeringData.salesCompensationList.recipient[].recipientAddress.city`                 | string  | City                                                        |
| `offeringData.salesCompensationList.recipient[].recipientAddress.stateOrCountry`       | string  | State or country code                                       |
| `offeringData.salesCompensationList.recipient[].recipientAddress.zipCode`              | string  | Postal code                                                 |
| `offeringData.salesCompensationList.recipient[].statesOfSolicitationList[]`            | array   | States where the recipient solicits investors               |
| `offeringData.salesCompensationList.recipient[].statesOfSolicitationList[].state`      | string  | State code                                                  |
| `offeringData.salesCompensationList.recipient[].statesOfSolicitationList[].description` | string | Full name of that state                                     |
| `offeringData.salesCompensationList.recipient[].statesOfSolicitationList[].value`      | string  | The state entry as filed                                    |
| `offeringData.salesCompensationList.recipient[].foreignSolicitation`                   | boolean | True if the recipient solicits outside the United States    |

### `offeringData.signatureBlock`

| Field                                                    | Type    | Meaning                                                     |
| -------------------------------------------------------- | ------- | ----------------------------------------------------------- |
| `offeringData.signatureBlock.authorizedRepresentative`    | boolean | True if the signer signs as an authorized representative of the issuer |
| `offeringData.signatureBlock.signature[]`                 | array   | One element per signature                                   |
| `offeringData.signatureBlock.signature[].issuerName`      | string  | Issuer name over the signature                              |
| `offeringData.signatureBlock.signature[].signatureName`   | string  | The signature text, for example `/s/ Kirk Carson`           |
| `offeringData.signatureBlock.signature[].nameOfSigner`    | string  | Printed name of the person who signed                       |
| `offeringData.signatureBlock.signature[].signatureTitle`  | string  | Title that person holds at the issuer                       |
| `offeringData.signatureBlock.signature[].signatureDate`   | string  | Date of the signature                                       |

`from` plus `size` must stay at or below 10,000. That is the deepest you can
page.

## Example

Prompt: "Show me the newest private placements that raised at least $1 million."

```json
{
  "name": "form-d",
  "arguments": {
    "query": "offeringData.offeringSalesAmounts.totalOfferingAmount:[1000000 TO *]",
    "size": 1
  }
}
```

Response, trimmed for length:

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "offerings": [
    {
      "schemaVersion": "X0708",
      "submissionType": "D",
      "testOrLive": "LIVE",
      "primaryIssuer": {
        "cik": "0002149897",
        "entityName": "Fusion VC SPV Skapion LLC",
        "jurisdictionOfInc": "DELAWARE",
        "entityType": "Limited Liability Company",
        "yearOfInc": { "withinFiveYears": true, "value": "2026" }
      },
      "offeringData": {
        "industryGroup": { "industryGroupType": "Pooled Investment Fund" },
        "minimumInvestmentAccepted": 0,
        "offeringSalesAmounts": { "totalOfferingAmount": 1402000, "totalAmountSold": 1402000, "totalRemaining": 0 },
        "investors": { "hasNonAccreditedInvestors": false, "totalNumberAlreadyInvested": 11 }
      },
      "accessionNo": "0002149897-26-000001",
      "filedAt": "2026-08-13T09:13:20-04:00",
      "id": "5be684f8ec109058266aca6744ee6997"
    }
  ]
}
```

## Limits and errors

- `size` above 50 gives HTTP 400 with a message naming the 50 limit.
- `from` plus `size` above 10,000 returns `{"total":{"value":0},"offerings":[]}`
  with no error. An empty result there does not mean there is no more data.
- `total.value` of exactly 10000 with `relation: "gte"` is a counting ceiling.
  Narrow the query if you need a real count.
- Unlike [form-s1-424b4](./form-s1-424b4.md) and [form-8k](./form-8k.md), this
  tool does not demand a colon in `query`.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [form-c](./form-c.md), [reg-a-form-1a](./reg-a-form-1a.md), [form-s1-424b4](./form-s1-424b4.md)
- [filing-search](./filing-search.md), [edgar-entities](./edgar-entities.md)
- REST docs: <https://sec-api.io/docs/form-d-xml-json-api>
