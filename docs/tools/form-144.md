# form-144

Search Form 144 notices, the intent-to-sell filings that insiders submit before
they sell restricted stock.

|                 |                                                  |
| --------------- | ------------------------------------------------ |
| Category        | Ownership and insiders                           |
| Required input  | `query`                                          |
| Returns         | `{total, data[]}`. One item per Form 144 notice. |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`     |
| REST equivalent | `POST /form-144`                                 |

## What it does

Rule 144 requires an affiliate to file a notice before selling restricted or
control securities above a threshold. The notice states how many shares the
person plans to sell, through which broker, on roughly which date, and what they
already sold in the past three months. This tool searches those notices. It
holds the notices that EDGAR received in XML form, from October 2022 to present.

One item in `data[]` is one Form 144 notice. It is a plan, not a completed
trade. The sale can be smaller than stated, or never happen.

A request for `issuerInfo.issuerTicker:TSLA` with `size: 1` returns
`total.value: 75` and one notice from a Tesla officer in 2,066 bytes. He planned
to sell 2,606 shares worth $1,048,124.08.

## When to use it

- Which Tesla insiders filed notice to sell this month?
- How many shares does an executive plan to sell, and at what market value?
- What did this insider already sell in the past three months?
- Were the shares acquired from options, RSUs or an open-market purchase?

## When to use a different tool

| Situation                                  | Better tool                                                       | Why                                                                             |
| ------------------------------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| You want the sale that actually happened   | [`insider-trading`](./insider-trading.md)                         | Form 4 reports the executed trade with the real price. Form 144 is only intent. |
| You want the full board and officer roster | [`directors-and-board-members`](./directors-and-board-members.md) | Only insiders who sell file a Form 144.                                         |
| You want stakes above 5% of a company      | [`form-13d-13g`](./form-13d-13g.md)                               | 13D and 13G report beneficial ownership, not planned sales.                     |

## Input

| Parameter | Type    | Required | Constraints           | Notes                                                                                  |
| --------- | ------- | -------- | --------------------- | -------------------------------------------------------------------------------------- |
| `query`   | string  | yes      | none                  | Lucene syntax. See [query language](../query-language.md).                             |
| `from`    | integer | no       | 0 to 10000            | Offset into the result set. One query reaches 10,000 notices.                          |
| `size`    | integer | no       | 1 to 50               | Notices per call. **Defaults to 50** when you omit it.                                 |
| `sort`    | array   | no       | array of sort objects | Defaults to `[{"filedAt": {"order": "desc"}}]`.                                        |

The schema sets `additionalProperties: true`. Query fields:

- `issuerInfo.issuerTicker`. The example uses `issuerInfo.issuerTicker:TSLA`.
- `entities.ticker`. Both spellings appear in the response, which is why the
  confusion exists. `issuerInfo.issuerTicker` is the ticker of the issuer.
  `entities.ticker` sits on the EDGAR header rows, where only the issuer row
  carries it.
- `issuerInfo.issuerCik`, `issuerInfo.issuerName`,
  `issuerInfo.relationshipsToIssuer`,
  `issuerInfo.nameOfPersonForWhoseAccountTheSecuritiesAreToBeSold`, `cik`,
  `formType`, `filedAt`, `accessionNo`, `entities.fileNo`, `entities.sic`,
  `entities.stateOfIncorporation`, `securitiesInformation.approxSaleDate`. All
  present in the response body.
- The sale detail is searchable too:
  `securitiesInformation.numberOfUnitsToBeSold`,
  `securitiesInformation.aggregateMarketValue`,
  `securitiesInformation.securitiesClassTitle`,
  `securitiesInformation.securitiesExchangeName`,
  `securitiesInformation.brokerOrMarketMakerDetails.name`,
  `securitiesToBeSold.natureOfAcquisitionTransaction`,
  `securitiesToBeSold.isGiftTransaction`,
  `nothingToReportFlagOnSecuritiesSoldInPast3Months`,
  `noticeSignature.planAdoptionDates` and `noticeSignature.signature`.

## Output

The envelope is `{total, data[]}`. `total` holds `value` and `relation`.
`"eq"` is an exact count. `"gte"` at 10000 means 10,000 or more.

One item in `data[]` is one Form 144 notice. The paths below are relative to a
`data[]` item.

### Filing identity

| Field                     | Type   | Meaning                                                                              |
| ------------------------- | ------ | ------------------------------------------------------------------------------------ |
| `id`                      | string | System-internal identifier of the filing record.                                     |
| `accessionNo`             | string | Accession number that EDGAR gave to this notice.                                     |
| `previousAccessionNumber` | string | Accession number of the earlier notice that this filing amends. Only on `144/A`.     |
| `fileNo`                  | string | SEC file number of the issuer, taken from the EDGAR header.                          |
| `formType`                | string | Form type. `144` for a new notice, `144/A` for an amendment.                         |
| `filedAt`                 | string | Time when EDGAR accepted the filing, in ISO 8601.                                    |
| `remarks`                 | string | Additional comments that the filer wrote in the submission.                          |

### `entities[]`

The EDGAR header rows of the filing. The issuer row carries `(Subject)` in its
name. The insider row carries `(Reporting)`. Only the issuer row has a `ticker`.

| Field                                | Type   | Meaning                                                                                       |
| ------------------------------------ | ------ | --------------------------------------------------------------------------------------------- |
| `entities[].cik`                     | string | Central Index Key of the entity.                                                              |
| `entities[].ticker`                  | string | Stock ticker symbol of the entity.                                                            |
| `entities[].companyName`             | string | Legal name of the entity as the filing gives it.                                              |
| `entities[].irsNo`                   | string | IRS Employer Identification Number (EIN) of the entity.                                       |
| `entities[].fiscalYearEnd`           | string | Fiscal year end as four digits, month then day. `1231` is 31 December.                        |
| `entities[].stateOfIncorporation`    | string | US state or country where the entity is incorporated.                                         |
| `entities[].sic`                     | string | Standard Industrial Classification code and its industry label. The label holds HTML entities such as `&amp;`. |
| `entities[].type`                    | string | Form type of the header row. It repeats `formType`.                                           |
| `entities[].act`                     | string | Act under which the entity files reports. `33` is the Securities Act of 1933.                 |
| `entities[].fileNo`                  | string | SEC file number that tracks the filings of the entity.                                        |
| `entities[].filmNo`                  | string | Film number that the SEC gives to this one filing.                                            |

### `issuerInfo`

The company whose stock the insider plans to sell, and who plans to sell it.

| Field                                                            | Type   | Meaning                                                                                                                                                                                                                           |
| ---------------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `issuerInfo.issuerCik`                                           | string | Central Index Key of the issuer.                                                                                                                                                                                                  |
| `issuerInfo.issuerTicker`                                        | string | Stock ticker symbol of the issuer.                                                                                                                                                                                                |
| `issuerInfo.issuerName`                                          | string | Legal name of the issuer as registered with the SEC.                                                                                                                                                                              |
| `issuerInfo.secFileNumber`                                       | string | SEC file number of the registration of the issuer.                                                                                                                                                                                |
| `issuerInfo.issuerAddress.street1`                               | string | Primary street of the registered address of the issuer.                                                                                                                                                                           |
| `issuerInfo.issuerAddress.street2`                               | string | Second street line of the registered address of the issuer.                                                                                                                                                                       |
| `issuerInfo.issuerAddress.city`                                  | string | City of the registered address of the issuer.                                                                                                                                                                                     |
| `issuerInfo.issuerAddress.stateOrCountry`                        | string | State or country code of the registered address of the issuer.                                                                                                                                                                    |
| `issuerInfo.issuerAddress.zipCode`                               | string | ZIP or postal code of the registered address of the issuer.                                                                                                                                                                       |
| `issuerInfo.issuerContactPhone`                                  | string | Contact phone number given for the issuer.                                                                                                                                                                                        |
| `issuerInfo.nameOfPersonForWhoseAccountTheSecuritiesAreToBeSold` | string | Name of the person on whose behalf the securities are sold.                                                                                                                                                                       |
| `issuerInfo.relationshipsToIssuer`                               | string | How the seller relates to the issuer. `Officer` in the example response. `Director`, `10% Stockholder`, `Affiliate` and `Member of immediate family of any of the foregoing` also appear. The filer can write another description. |

### `securitiesInformation[]`

The planned sale. One item per class of security.

| Field                                                                     | Type   | Meaning                                                            |
| ------------------------------------------------------------------------- | ------ | -------------------------------------------------------------------- |
| `securitiesInformation[].securitiesClassTitle`                            | string | Title or class of the securities in the planned sale, such as `Common`. |
| `securitiesInformation[].brokerOrMarketMakerDetails.name`                 | string | Name of the broker or market maker that executes the sale.         |
| `securitiesInformation[].brokerOrMarketMakerDetails.address.street1`      | string | Primary street of the address of the broker.                       |
| `securitiesInformation[].brokerOrMarketMakerDetails.address.street2`      | string | Second street line of the address of the broker.                   |
| `securitiesInformation[].brokerOrMarketMakerDetails.address.city`         | string | City of the address of the broker.                                 |
| `securitiesInformation[].brokerOrMarketMakerDetails.address.stateOrCountry` | string | State or country code of the address of the broker.              |
| `securitiesInformation[].brokerOrMarketMakerDetails.address.zipCode`      | string | ZIP or postal code of the address of the broker.                   |
| `securitiesInformation[].numberOfUnitsToBeSold`                           | number | Number of units that the insider plans to sell.                    |
| `securitiesInformation[].aggregateMarketValue`                            | number | Total market value of the units to be sold, in US dollars.         |
| `securitiesInformation[].noOfUnitsOutstanding`                            | number | Total units of the class that the issuer has outstanding.          |
| `securitiesInformation[].approxSaleDate`                                  | string | Approximate date of the planned sale, `YYYY-MM-DD`.                |
| `securitiesInformation[].securitiesExchangeName`                          | string | Exchange on which the sale takes place, such as `NASDAQ`.          |

### `securitiesToBeSold[]`

How the insider got the shares. One item per acquisition lot.

| Field                                                | Type    | Meaning                                                                                                                       |
| ---------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `securitiesToBeSold[].securitiesClassTitle`          | string  | Title or class of the securities proposed for sale.                                                                           |
| `securitiesToBeSold[].acquiredDate`                  | string  | Date on which the insider acquired the securities, `YYYY-MM-DD`.                                                              |
| `securitiesToBeSold[].natureOfAcquisitionTransaction` | string | How the insider acquired the securities. `Restricted Stock` in the example response. `Exercise of Stock Options` and `Previously Exercised Stock Options` also appear. |
| `securitiesToBeSold[].nameOfPersonFromWhomAcquired`  | string  | Person or entity from whom the insider acquired the securities, such as `Issuer`.                                             |
| `securitiesToBeSold[].isGiftTransaction`             | boolean | `true` if the insider acquired the securities as a gift.                                                                      |
| `securitiesToBeSold[].donorAcquiredDate`             | string  | Date on which the donor acquired the securities, `YYYY-MM-DD`. It applies only to a gift.                                     |
| `securitiesToBeSold[].amountOfSecuritiesAcquired`    | number  | Number of units in the lot.                                                                                                   |
| `securitiesToBeSold[].paymentDate`                   | string  | Date on which the insider paid for the acquisition, `YYYY-MM-DD`.                                                             |
| `securitiesToBeSold[].natureOfPayment`               | string  | Method or terms of the payment, such as `Not Applicable`.                                                                     |

### Sales in the past three months

| Field                                                                  | Type    | Meaning                                                            |
| ---------------------------------------------------------------------- | ------- | -------------------------------------------------------------------- |
| `nothingToReportFlagOnSecuritiesSoldInPast3Months`                     | boolean | `true` when the filer declares no sale in the past three months.   |
| `securitiesSoldInPast3Months[].sellerDetails.name`                     | string  | Name of the seller in the past sale.                               |
| `securitiesSoldInPast3Months[].sellerDetails.address.street1`          | string  | Primary street of the address of the seller.                       |
| `securitiesSoldInPast3Months[].sellerDetails.address.street2`          | string  | Second street line of the address of the seller.                   |
| `securitiesSoldInPast3Months[].sellerDetails.address.city`             | string  | City of the address of the seller.                                 |
| `securitiesSoldInPast3Months[].sellerDetails.address.stateOrCountry`   | string  | State or country code of the address of the seller.                |
| `securitiesSoldInPast3Months[].sellerDetails.address.zipCode`          | string  | ZIP or postal code of the address of the seller.                   |
| `securitiesSoldInPast3Months[].securitiesClassTitle`                   | string  | Title or class of the securities in the past sale.                 |
| `securitiesSoldInPast3Months[].saleDate`                               | string  | Date of the past sale, `YYYY-MM-DD`.                               |
| `securitiesSoldInPast3Months[].amountOfSecuritiesSold`                 | number  | Number of units sold in the past sale.                             |
| `securitiesSoldInPast3Months[].grossProceeds`                          | number  | Gross proceeds of the past sale, in US dollars.                    |

### `noticeSignature`

| Field                                | Type             | Meaning                                                                                                            |
| ------------------------------------ | ---------------- | -------------------------------------------------------------------------------------------------------------------- |
| `noticeSignature.noticeDate`         | string           | Date on which the person signed the notice, `YYYY-MM-DD`.                                                          |
| `noticeSignature.planAdoptionDates[]` | array of string | Dates on which the person adopted the trading plans. Rule 10b5-1 sales use them. The field is optional and is absent from the example response. |
| `noticeSignature.signature`          | string           | Signature of the person who submits the notice.                                                                    |

`securitiesSoldInPast3Months[].grossProceeds` is a completed dollar amount. It
is the only realised number on the record. Everything in
`securitiesInformation[]` is a plan.

Size behaviour: one notice was 2,066 bytes. Records are small and evenly sized.
`size` defaults to 50, and 50 notices are safe to hold in context.

## Example

Prompt: "Which Tesla insiders recently filed notice to sell shares?"

```json
{
  "name": "form-144",
  "arguments": { "query": "issuerInfo.issuerTicker:TSLA", "size": 1 }
}
```

```json
{
  "total": { "value": 75, "relation": "eq" },
  "data": [
    {
      "accessionNo": "0001950047-26-005795",
      "formType": "144",
      "filedAt": "2026-06-08T17:30:09-04:00",
      "issuerInfo": {
        "issuerCik": "1318605",
        "issuerTicker": "TSLA",
        "nameOfPersonForWhoseAccountTheSecuritiesAreToBeSold": "VAIBHAV TANEJA",
        "relationshipsToIssuer": "Officer"
      },
      "securitiesInformation": [
        {
          "numberOfUnitsToBeSold": 2606,
          "aggregateMarketValue": 1048124.08,
          "approxSaleDate": "2026-06-08",
          "securitiesExchangeName": "NASDAQ"
        }
      ],
      "noticeSignature": {
        "noticeDate": "2026-06-08",
        "signature": "/s/ Vaibhav Taneja"
      }
    }
  ]
}
```

Trimmed. The full response also holds `entities[]`, the broker block,
`securitiesToBeSold[]` and one prior sale of 3,000 shares for $1,350,000 gross.

## Limits and errors

- A Form 144 is a notice of intent to sell. It does not record a completed sale.
  [`insider-trading`](./insider-trading.md) reports the outcome.
- `approxSaleDate` is approximate by design and often equals the filing date.
- The ticker field is `issuerInfo.issuerTicker` here. On other tools it is
  `ticker`, `entities.ticker` or `issuer.tradingSymbol`.
- Only `size` is validated. A value above 50 fails with `Maximum 'size' limit of
50 exceeded.` A malformed query is passed straight to the search engine.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`insider-trading`](./insider-trading.md)
- [`form-13d-13g`](./form-13d-13g.md)
- [`directors-and-board-members`](./directors-and-board-members.md)
- REST docs: [Form 144 API](https://sec-api.io/docs/form-144-restricted-sales-api)
