# form-adv-firms

Search the Form ADV firm register, the registration record of every
SEC-registered and state-registered investment adviser.

|                 |                                    |
| --------------- | ---------------------------------- |
| Category        | Investment advisers                |
| Required input  | `query`                            |
| Returns         | `{total, filings[]}`               |
| Pagination      | `from`, `size` (1 to 50, default 50), `sort`    |
| REST equivalent | `POST /form-adv/firm`              |

## What it does

One item in `filings[]` is the latest Form ADV filing of one adviser firm. The
row is the current Form ADV Part 1 record for that CRD number. A query for a
single CRD returns exactly one row, for example `Info.FirmCrdNb:149777`. The
tool holds no filing history. `Filing[].Dt` is the date the firm's most recent
Form ADV filing was processed. A firm amends its Form ADV from time to time,
so this date is the date of the last change, not the first registration.

The index merges two sources, SEC-registered advisers and state-registered
advisers. It holds more than 41,000 firms. The two sources produce **different
row shapes**. See Output.

Part 1 answers arrive as raw item codes under `FormInfo.Part1A`, for example
`Item5F.Q5F2C`. The registry description promises assets under management,
client types and a disciplinary summary. They are present, but unlabelled. The
description also names an IARD number. There is no separate IARD field.
`Info.FirmCrdNb` holds that number. The CRD system or the IARD system assigns
it.

## When to use it

- What is the CRD number of the adviser called Bridgewater Associates?
- Which advisers report more than $100 billion of regulatory assets?
- Where is this adviser's main office, and what is its SEC file number?
- Which firms are exempt reporting advisers rather than registered advisers?
- Did this adviser answer yes to any Item 11 disciplinary question?

## When to use a different tool

| Situation                                    | Better tool                                                                 | Why                                                        |
| -------------------------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------- |
| You want the adviser's staff, not the firm   | [`form-adv-individuals`](./form-adv-individuals.md)                         | That tool indexes adviser representatives.                 |
| You want the plain-English disclosure        | [`form-adv-brochures`](./form-adv-brochures.md)                             | Part 2 brochures are separate documents.                   |
| You want who owns the firm                   | [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md) | Ownership sits on Schedule A and Schedule B.               |
| You want the private funds it advises        | [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md)               | Fund detail sits on Schedule D, Section 7.B.1.             |
| You want the positions the adviser holds     | [`form-13f-holdings`](./form-13f-holdings.md)                               | Form ADV carries no securities holdings.                    |
| The firm is a broker-dealer                  | [`form-x-17a-5`](./form-x-17a-5.md)                                         | Broker-dealers file FOCUS reports, not Form ADV.           |

## Input

| Parameter | Type    | Required | Constraints              | Notes                                            |
| --------- | ------- | -------- | ------------------------ | ------------------------------------------------ |
| `query`   | string  | Yes      | Lucene syntax            | For example `Info.FirmCrdNb:149777`.             |
| `from`    | integer | No       | 0 to 10,000              | Offset of the first result. Default 0.           |
| `size`    | integer | No       | 1 to 50                  | Default 50.                                       |
| `sort`    | array   | No       | Elasticsearch sort array | Default `[{"Info.FirmCrdNb": {"order": "desc"}}]`. |

Query fields. The hit count on 2026-08-13 is in brackets.

| Field                             | Example                                            |
| --------------------------------- | -------------------------------------------------- |
| `Info.FirmCrdNb`                  | `Info.FirmCrdNb:149777` (1)                        |
| `Info.BusNm`                      | `Info.BusNm:"BRIDGEWATER ASSOCIATES"` (1)          |
| `Info.SECNb`                      | `Info.SECNb:"801-16048"` (1)                       |
| `MainAddr.State`                  | `MainAddr.State:NY` (4,574)                        |
| `Rgstn.FirmType`                  | `Rgstn.FirmType:ERA` (8,893)                       |
| `FormInfo.Part1A.Item5F.Q5F2C`    | `FormInfo.Part1A.Item5F.Q5F2C:[100000000000 TO *]` (270) |

The table lists common fields. Every response field is a query field, so
`Info.LegalNm`, `Filing.Dt`, `Filing.FormVrsn`,
`FormInfo.Part1A.Item2A.Q2A1` and any other item code also work. Ranges work
on numeric item codes, as the `Q5F2C` example shows. Sorting on
`FormInfo.Part1A.Item5F.Q5F2C` works too. Item 5 values are what the adviser
typed into the form. The top firm by that sort reports $11.5 trillion, which is
not credible. An unknown field returns `total: 0` with no error. See
[query language](../query-language.md).

## Output

The envelope is `{total, filings[]}`, not `{total, data[]}`. `total` is an
object, `{value, relation}`. A `relation` of `gte` means the count stopped at
10,000 and the true count is higher.

Rows come in two shapes. Both always carry `Info`, `MainAddr`, `MailingAddr`,
`Filing`, `FormInfo` and `id`. In a sample of 50 rows, 28 had the SEC shape and
22 had the state shape.

### Envelope

| Field            | Type   | Meaning                                                     |
| ---------------- | ------ | ----------------------------------------------------------- |
| `total`          | object | Hit count for the query.                                     |
| `total.value`    | number | Number of firms that match the query. Counting stops at 10,000. |
| `total.relation` | string | `eq` when the count is exact. `gte` when it stopped at 10,000. |
| `filings[]`      | array  | One item per adviser firm. Each item is the whole Form ADV record of that firm. |
| `id`             | number | Row key. Carries the same value as `Info.FirmCrdNb`.         |

### Info

| Field             | Type   | Meaning                                                    |
| ----------------- | ------ | ----------------------------------------------------------- |
| `Info`            | object | Firm identity from the IARD composite record.                |
| `Info.FirmCrdNb`  | number | CRD number that the FINRA CRD system or the IARD system assigns to the firm. The key for every other Form ADV tool. |
| `Info.SECNb`      | string | The firm's 801 or 802 number, Item 1.D.                      |
| `Info.BusNm`      | string | Primary business name, Item 1.B.                             |
| `Info.LegalNm`    | string | Legal name, Item 1.A.                                        |
| `Info.SECRgnCD`   | string | SEC regional office that holds the record. `ARO` Atlanta, `BRO` Boston, `DRO` Denver, `FWRO` Fort Worth, `CHRO` Chicago, `NYRO` New York, `LARO` Los Angeles, `PLRO` Philadelphia, `SFRO` San Francisco, `MIRO` Miami, `HQ` headquarters. |
| `Info.UmbrRgstn`  | string | `Y` when the firm registers more than one investment adviser under an umbrella registration. `N` when it does not. |

### MainAddr and MailingAddr

`MainAddr` is the principal office and place of business, Item 1.F.
`MailingAddr` is the mailing address, Item 1.G. Both hold the same eight keys.
`MailingAddr` is often empty.

| Field                                    | Type   | Meaning                        |
| ---------------------------------------- | ------ | ------------------------------- |
| `MainAddr`, `MailingAddr`                | object | Principal office address, and mailing address. |
| `MainAddr.Strt1`, `MailingAddr.Strt1`    | string | First street line.              |
| `MainAddr.Strt2`, `MailingAddr.Strt2`    | string | Second street line, such as a suite or a floor. |
| `MainAddr.City`, `MailingAddr.City`      | string | City.                           |
| `MainAddr.State`, `MailingAddr.State`    | string | State or territory code, for example `NY`. |
| `MainAddr.Cntry`, `MailingAddr.Cntry`    | string | Country name, for example `United States`. |
| `MainAddr.PostlCd`, `MailingAddr.PostlCd`| string | Postal code.                    |
| `MainAddr.PhNb`, `MailingAddr.PhNb`      | string | Telephone number.               |
| `MainAddr.FaxNb`, `MailingAddr.FaxNb`    | string | Fax number.                     |

### Registration and status

`Rgstn` and `NoticeFiled` belong to the SEC shape. `StateRgstn` and `ERA`
belong to the state shape.

| Field                            | Type   | Meaning                                 |
| -------------------------------- | ------ | ---------------------------------------- |
| `Rgstn[]`                        | array  | Registration types the firm holds with the SEC. |
| `Rgstn[].FirmType`               | string | `Registered` for a registered firm. `ERA` for an exempt reporting adviser. |
| `Rgstn[].St`                     | string | Status of that registration when the record was built. `APPROVED`, `APPROVED-120` for a 120-day approval, `SUSPENDED`, `ACTIVE`. |
| `Rgstn[].Dt`                     | string | Effective date of the SEC registration, or the date the firm filed as an exempt reporting adviser. |
| `NoticeFiled`                    | object | Active notice filings.                   |
| `NoticeFiled.States[]`           | array  | One item per state where the firm filed a notice. |
| `NoticeFiled.States[].RgltrCd`   | string | State code of the regulator, for example `FL`. |
| `NoticeFiled.States[].St`        | string | `FILED` when the notice is filed. `PENDSEC` when it is pending. |
| `NoticeFiled.States[].Dt`        | string | Effective date of that notice status.    |
| `StateRgstn`                     | object | Registrations with state regulators.     |
| `StateRgstn.Rgltrs`              | object | Wrapper around the regulator list.       |
| `StateRgstn.Rgltrs.Rgltr[]`      | array  | One item per state regulator.            |
| `ERA`                            | object | Exempt reporting adviser status with state regulators. |
| `ERA.Rgltrs`                     | object | Wrapper around the regulator list.       |
| `ERA.Rgltrs.Rgltr[]`             | array  | One item per state regulator.            |
| `StateRgstn.Rgltrs.Rgltr[].Cd`, `ERA.Rgltrs.Rgltr[].Cd` | string | State code, for example `FL`. |
| `StateRgstn.Rgltrs.Rgltr[].St`, `ERA.Rgltrs.Rgltr[].St` | string | Status with that state. `TRANSITIONING`, `APPROVED`, `CONDREST` for conditional restricted, `LIMITED`, `SUSPENDED`, `TERMREQUEST` for termination requested, `ACTIVE`. |
| `StateRgstn.Rgltrs.Rgltr[].Dt`, `ERA.Rgltrs.Rgltr[].Dt` | string | Date of that status. |
| `Filing[]`                       | array  | The latest Form ADV submission.          |
| `Filing[].Dt`                    | string | Date the most recent Form ADV filing was processed. |
| `Filing[].FormVrsn`              | string | Version of that form, for example `10/2021`. |

### FormInfo, Item 1 to Item 3

`FormInfo` is an object with `Part1A` for every firm and `Part1B` for
state-registered advisers. Paths below start at `FormInfo.Part1A`. A `Q` field
answers one question and holds `Y` or `N`, unless the type says otherwise.

| Field                    | Type   | Meaning                                         |
| ------------------------ | ------ | ------------------------------------------------ |
| `Item1`                  | object | Firm identifying information, Item 1.            |
| `Item1.WebAddrs`         | object | Website addresses from Schedule D Section 1.I.   |
| `Item1.WebAddrs.WebAddrs[]` | array of string | Every website address the firm reports. |
| `Item1.WebAddrs.WebAddr` | string | One address out of that list.                    |
| `Item1.Q1I`              | string | The firm has one or more websites.               |
| `Item1.Q1M`              | string | The firm is registered with a foreign financial regulatory authority. |
| `Item1.Q1N`              | string | The firm is a public reporting company under Section 12 or 15(d) of the Securities Exchange Act of 1934. |
| `Item1.Q1O`              | string | The firm held $1 billion or more in assets on the last day of its most recent fiscal year. |
| `Item1.Q1ODesc`          | string | Approximate asset amount behind a `Y` in `Q1O`, for example `More than $50 billion`. |
| `Item1.Q1P`              | string | Legal Entity Identifier of the firm.             |
| `Item1.Q1F5`             | number | Number of offices other than the principal office and place of business. |
| `Item2A`                 | object | Basis for SEC registration, Item 2.A.            |
| `Item2A.Q2A1`            | string | Large advisory firm. Regulatory assets under management of $100 million or more, or SEC-registered, filing an annual updating amendment, with $90 million or more. |
| `Item2A.Q2A2`            | string | Mid-sized advisory firm with $25 million or more but less than $100 million, that is either not required to register with its home state authority or not examined by it. |
| `Item2A.Q2A4`            | string | The principal office and place of business is outside the United States. |
| `Item2A.Q2A5`            | string | Adviser or sub-adviser to an investment company registered under the Investment Company Act of 1940. |
| `Item2A.Q2A6`            | string | Adviser to a business development company under section 54 of that Act, with at least $25 million of regulatory assets under management. |
| `Item2A.Q2A7`            | string | Pension consultant for plans worth at least $200,000,000 in total, under rule 203A-2. |
| `Item2A.Q2A8`            | string | Related adviser under rule 203A-2(b) that shares its principal office with an SEC-registered adviser under common control. |
| `Item2A.Q2A9`            | string | Newly formed adviser under rule 203A-2(c) that expects to qualify for SEC registration within 120 days. |
| `Item2A.Q2A10`           | string | Multi-state adviser that must register in 15 or more states, under rule 203A-2(d). |
| `Item2A.Q2A11`           | string | Internet adviser under rule 203A-2(e).           |
| `Item2A.Q2A12`           | string | The firm holds an SEC order that exempts it from the ban on SEC registration. |
| `Item2A.Q2A13`           | string | The firm can no longer stay registered with the SEC. |
| `Item2B`                 | object | Basis for exempt reporting adviser status, Item 2.B. |
| `Item2B.Q2B1`            | string | Exempt because the firm advises only venture capital funds. |
| `Item2B.Q2B2`            | string | Exempt because the firm advises only private funds and holds less than $150 million under management in the United States. |
| `Item2B.Q2B3`            | string | The firm advises only private funds but can no longer use 2.B.(2), because its United States assets reached $150 million or more. |
| `Item3A`                 | object | Form of organization, Item 3.A.                  |
| `Item3A.OrgFormNm`       | string | How the firm is organized, for example `Limited Liability Company`. |
| `Item3A.OrgFormOthNm`    | string | The text the firm typed when it chose "other" for its form of organization. |
| `Item3B`                 | object | Fiscal year end, Item 3.B.                       |
| `Item3B.Q3B`             | string | Month in which the fiscal year ends, for example `DECEMBER`. |
| `Item3C`                 | object | Place of organization, Item 3.C.                 |
| `Item3C.StateCD`         | string | State under whose laws the firm is organized.    |
| `Item3C.CntryNm`         | string | Country under whose laws the firm is organized.  |

### Item 5A to 5C, employees and clients

| Field           | Type   | Meaning                                                   |
| --------------- | ------ | ---------------------------------------------------------- |
| `Item5A`        | object | Employee count, Item 5.A.                                  |
| `Item5A.TtlEmp` | number | Approximate number of employees.                           |
| `Item5B`        | object | Split of the employees reported in 5.A, Item 5.B.          |
| `Item5B.Q5B1`   | number | How many perform investment advisory functions, research included. |
| `Item5B.Q5B2`   | number | How many are registered representatives of a broker-dealer. |
| `Item5B.Q5B3`   | number | How many are registered with state securities authorities as investment adviser representatives. |
| `Item5B.Q5B4`   | number | How many are registered as investment adviser representatives for another adviser. |
| `Item5B.Q5B5`   | number | How many are licensed agents of an insurance company or agency. |
| `Item5B.Q5B6`   | number | How many firms or other persons solicit advisory clients for the firm. |
| `Item5C`        | object | Client counts, Item 5.C.                                   |
| `Item5C.Q5C1`   | string | Approximate number of clients that received investment advisory services in the last completed fiscal year. |
| `Item5C.Q5C1Oth`| number | Exact figure when the answer to 5.C.(1) is more than 100, rounded to the nearest 100. |
| `Item5C.Q5C2`   | number | Approximate percentage of clients that are not United States persons. |

### Item 5D, clients by type

Item 5.D reports clients by type. Keys `Q5D1x` and `Q5D2x` come from the
10/2012 form version. Keys `Q5DA` to `Q5DN` come from later versions. In the
later keys the suffix `1` is the number of clients, `2` marks fewer than five
clients, and `3` is the regulatory assets under management for that type.

| Field                | Type   | Meaning                                              |
| -------------------- | ------ | ----------------------------------------------------- |
| `Item5D`             | object | Clients by type, Item 5.D.                            |
| `Item5D.Q5D1A`, `Item5D.Q5D2A` | string | 10/2012 version, individuals other than high net worth individuals. |
| `Item5D.Q5D1B`, `Item5D.Q5D2B` | string | 10/2012 version, high net worth individuals. |
| `Item5D.Q5D1C`, `Item5D.Q5D2C` | string | 10/2012 version, banking or thrift institutions. |
| `Item5D.Q5D1D`, `Item5D.Q5D2D` | string | 10/2012 version, investment companies. |
| `Item5D.Q5D1E`, `Item5D.Q5D2E` | string | 10/2012 version, business development companies. |
| `Item5D.Q5D1F`, `Item5D.Q5D2F` | string | 10/2012 version, pooled investment vehicles other than investment companies. |
| `Item5D.Q5D1G`, `Item5D.Q5D2G` | string | 10/2012 version, pension and profit sharing plans, plan participants excluded. |
| `Item5D.Q5D1H`, `Item5D.Q5D2H` | string | 10/2012 version, charitable organizations. |
| `Item5D.Q5D1I`, `Item5D.Q5D2I` | string | 10/2012 version, corporations or other businesses not listed above. |
| `Item5D.Q5D1J`, `Item5D.Q5D2J` | string | 10/2012 version, state or municipal government entities. |
| `Item5D.Q5D1K`, `Item5D.Q5D2K` | string | 10/2012 version, other investment advisers. |
| `Item5D.Q5D1L`, `Item5D.Q5D2L` | string | 10/2012 version, insurance companies. |
| `Item5D.Q5D1M`, `Item5D.Q5D2M` | string | 10/2012 version, other client types. |
| `Item5D.Q5D1MOth`, `Item5D.Q5D2MOth` | string | 10/2012 version, the text that names those other client types. |
| `Item5D.Q5DA1`       | number | Number of clients that are individuals other than high net worth individuals. |
| `Item5D.Q5DA2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DA3`       | number | Regulatory assets under management for that type. |
| `Item5D.Q5DB1`       | number | Number of high net worth individuals.                 |
| `Item5D.Q5DB2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DB3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DC1`       | number | Number of banking or thrift institutions.             |
| `Item5D.Q5DC2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DC3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DD1`       | number | Number of investment companies.                       |
| `Item5D.Q5DD3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DE1`       | number | Number of business development companies.             |
| `Item5D.Q5DE3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DF1`       | number | Number of pooled investment vehicles, investment companies and business development companies excluded. |
| `Item5D.Q5DF3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DG1`       | number | Number of pension and profit sharing plans, plan participants excluded. |
| `Item5D.Q5DG2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DG3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DH1`       | number | Number of charitable organizations.                   |
| `Item5D.Q5DH2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DH3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DI1`       | number | Number of state or municipal government entities, government pension plans included. |
| `Item5D.Q5DI2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DI3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DJ1`       | number | Number of other investment advisers.                  |
| `Item5D.Q5DJ2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DJ3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DK1`       | number | Number of insurance companies.                        |
| `Item5D.Q5DK2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DK3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DL1`       | number | Number of sovereign wealth funds and foreign official institutions. |
| `Item5D.Q5DL2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DL3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DM1`       | number | Number of corporations or other businesses not listed above. |
| `Item5D.Q5DM2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DM3`       | number | Regulatory assets under management for that type.     |
| `Item5D.Q5DN1`       | number | Number of clients of other types.                     |
| `Item5D.Q5DN2`       | string | `Fewer than 5 clients` when that count is under five. |
| `Item5D.Q5DN3`       | number | Regulatory assets under management for those types.   |
| `Item5D.Q5DN3Oth`    | string | The text that names those other client types.         |

### Item 5E to 5L, fees, assets and marketing

| Field              | Type   | Meaning                                                |
| ------------------ | ------ | ------------------------------------------------------- |
| `Item5E`           | object | How the firm is paid, Item 5.E.                         |
| `Item5E.Q5E1`      | string | A percentage of assets under management.                |
| `Item5E.Q5E2`      | string | Hourly charges.                                         |
| `Item5E.Q5E3`      | string | Subscription fees for a newsletter or periodical.       |
| `Item5E.Q5E4`      | string | Fixed fees other than subscription fees.                |
| `Item5E.Q5E5`      | string | Commissions.                                            |
| `Item5E.Q5E6`      | string | Performance-based fees.                                 |
| `Item5E.Q5E7`      | string | Another form of compensation.                           |
| `Item5E.Q5E7Oth`   | string | The text that names that other compensation.            |
| `Item5F`           | object | Regulatory assets under management, Item 5.F.           |
| `Item5F.Q5F1`      | string | The firm gives continuous and regular supervisory or management services to securities portfolios. |
| `Item5F.Q5F2A`     | number | Discretionary regulatory assets under management, in US dollars. |
| `Item5F.Q5F2B`     | number | Non-discretionary amount.                               |
| `Item5F.Q5F2C`     | number | Total amount.                                           |
| `Item5F.Q5F2D`     | number | Number of discretionary accounts.                       |
| `Item5F.Q5F2E`     | number | Number of non-discretionary accounts.                   |
| `Item5F.Q5F2F`     | number | Total number of accounts.                               |
| `Item5F.Q5F3`      | number | Part of the total in 5.F.(2)(c) that belongs to clients who are not United States persons. |
| `Item5G`           | object | Advisory services the firm provides, Item 5.G.          |
| `Item5G.Q5G1`      | string | Financial planning services.                            |
| `Item5G.Q5G2`      | string | Portfolio management for individuals or small businesses. |
| `Item5G.Q5G3`      | string | Portfolio management for investment companies.          |
| `Item5G.Q5G4`      | string | Portfolio management for pooled investment vehicles other than investment companies. |
| `Item5G.Q5G5`      | string | Portfolio management for businesses other than small businesses, or for institutional clients. |
| `Item5G.Q5G6`      | string | Pension consulting services.                            |
| `Item5G.Q5G7`      | string | Selection of other advisers.                            |
| `Item5G.Q5G8`      | string | Publication of periodicals or newsletters.              |
| `Item5G.Q5G9`      | string | Security ratings or pricing services.                   |
| `Item5G.Q5G10`     | string | Market timing services.                                 |
| `Item5G.Q5G11`     | string | Educational seminars and workshops.                     |
| `Item5G.Q5G12`     | string | Another advisory service.                               |
| `Item5G.Q5G12Oth`  | string | The text that names that other service.                 |
| `Item5H`           | object | Financial planning clients, Item 5.H.                   |
| `Item5H.Q5H`       | string | Band for the number of clients that received financial planning services in the last fiscal year, for example `1-10`. |
| `Item5H.Q5HMT500`  | number | Exact figure when that number is more than 500, rounded to the nearest 500. |
| `Item5I`           | object | Wrap fee programs, Item 5.I.                            |
| `Item5I.Q5I1`      | string | The firm sponsors a wrap fee program.                   |
| `Item5I.Q5I2A`     | number | Regulatory assets under management from acting as sponsor to a wrap fee program. |
| `Item5I.Q5I2B`     | number | Regulatory assets under management from acting as portfolio manager for a wrap fee program. |
| `Item5I.Q5I2C`     | number | Regulatory assets under management from acting as sponsor to, and portfolio manager for, the same wrap fee program. |
| `Item5J`           | object | Limits on advice and on asset reporting, Item 5.J.      |
| `Item5J.Q5J1`      | string | Item 4.B of Part 2A says the firm advises only on limited types of investments. |
| `Item5J.Q5J2`      | string | Item 4.E of Part 2A reports client assets computed by a method other than the one used for regulatory assets under management. |
| `Item5K`           | object | Separately managed account clients, Item 5.K.           |
| `Item5K.Q5K1`      | string | The firm has regulatory assets under management from clients other than those in Item 5.D.(3)(d) to (f). |
| `Item5K.Q5K2`      | string | The firm borrows for any separately managed account client it advises. |
| `Item5K.Q5K3`      | string | The firm enters derivative transactions for any such client. |
| `Item5K.Q5K4`      | string | One custodian holds ten percent or more of the regulatory assets under management that remain after the amounts in Item 5.D.(3)(d) to (f) are subtracted. |
| `Item5L`           | object | Marketing activities, Item 5.L.                         |
| `Item5L.Q5L1A`     | string | Advertisements include performance results.             |
| `Item5L.Q5L1B`     | string | Advertisements refer to specific investment advice, as rule 206(4)-1(a)(5) uses that phrase. |
| `Item5L.Q5L1C`     | string | Advertisements include testimonials, apart from those that satisfy rule 206(4)-1(b)(4)(ii). |
| `Item5L.Q5L1D`     | string | Advertisements include endorsements, apart from those that satisfy rule 206(4)-1(b)(4)(ii). |
| `Item5L.Q5L1E`     | string | Advertisements include third-party ratings.             |
| `Item5L.Q5L2`      | string | The firm gives cash or non-cash compensation, directly or indirectly, for the use of testimonials, endorsements or third-party ratings. |
| `Item5L.Q5L3`      | string | Advertisements include hypothetical performance.        |
| `Item5L.Q5L4`      | string | Advertisements include predecessor performance.         |

### Item 6, other business activities

Each `Q6A` field is `Y` when the firm is actively engaged in business as the
type named.

| Field             | Type   | Meaning                                                 |
| ----------------- | ------ | -------------------------------------------------------- |
| `Item6A`          | object | Other business activities, Item 6.A.                     |
| `Item6A.Q6A1`     | string | Broker-dealer, registered or unregistered.               |
| `Item6A.Q6A2`     | string | Registered representative of a broker-dealer.            |
| `Item6A.Q6A3`     | string | Commodity pool operator or commodity trading advisor, registered or exempt. |
| `Item6A.Q6A4`     | string | Futures commission merchant.                             |
| `Item6A.Q6A5`     | string | Real estate broker, dealer or agent.                     |
| `Item6A.Q6A6`     | string | Insurance broker or agent.                               |
| `Item6A.Q6A7`     | string | Bank, a separately identifiable department or division included. |
| `Item6A.Q6A8`     | string | Trust company.                                           |
| `Item6A.Q6A9`     | string | Registered municipal advisor.                            |
| `Item6A.Q6A10`    | string | Registered security-based swap dealer.                   |
| `Item6A.Q6A11`    | string | Major security-based swap participant.                   |
| `Item6A.Q6A12`    | string | Accountant or accounting firm.                           |
| `Item6A.Q6A13`    | string | Lawyer or law firm.                                      |
| `Item6A.Q6A14`    | string | Other financial product salesperson.                     |
| `Item6A.Q6A14Oth` | string | The text that names that other product, when `Q6A14` is `Y`. |
| `Item6B`          | object | Further business activities, Item 6.B.                   |
| `Item6B.Q6B1`     | string | The firm runs another business not listed in Item 6.A, apart from giving investment advice. |
| `Item6B.Q6B2`     | string | That other business is the firm's primary business.      |
| `Item6B.Q6B3`     | string | The firm sells products or services other than investment advice to its advisory clients. |

### Item 7, affiliations and private funds

Each `Q7A` field is `Y` when the firm has a related person of the type named.
Related persons are the firm's advisory affiliates and anyone under common
control with it, foreign affiliates included.

| Field           | Type   | Meaning                                                   |
| --------------- | ------ | ---------------------------------------------------------- |
| `Item7A`        | object | Financial industry affiliations, Item 7.A.                 |
| `Item7A.Q7A1`   | string | Broker-dealer, municipal securities dealer, or government securities broker or dealer, registered or unregistered. |
| `Item7A.Q7A2`   | string | Other investment adviser, financial planners included.     |
| `Item7A.Q7A3`   | string | Registered municipal advisor.                              |
| `Item7A.Q7A4`   | string | Registered security-based swap dealer.                     |
| `Item7A.Q7A5`   | string | Major security-based swap participant.                     |
| `Item7A.Q7A6`   | string | Commodity pool operator or commodity trading advisor, registered or exempt. |
| `Item7A.Q7A7`   | string | Futures commission merchant.                               |
| `Item7A.Q7A8`   | string | Banking or thrift institution.                             |
| `Item7A.Q7A9`   | string | Trust company.                                             |
| `Item7A.Q7A10`  | string | Accountant or accounting firm.                             |
| `Item7A.Q7A11`  | string | Lawyer or law firm.                                        |
| `Item7A.Q7A12`  | string | Insurance company or agency.                               |
| `Item7A.Q7A13`  | string | Pension consultant.                                        |
| `Item7A.Q7A14`  | string | Real estate broker or dealer.                              |
| `Item7A.Q7A15`  | string | Sponsor or syndicator of limited partnerships or equivalent, pooled investment vehicles excluded. |
| `Item7A.Q7A16`  | string | Sponsor, general partner or managing member, or equivalent, of pooled investment vehicles. |
| `Item7B`        | object | Private fund advice, Item 7.B.                             |
| `Item7B.Q7B`    | string | The firm advises one or more private funds.                |

### Item 8, participation in client transactions

| Field           | Type   | Meaning                                                   |
| --------------- | ------ | ---------------------------------------------------------- |
| `Item8A`        | object | Proprietary interest in client transactions, Item 8.A.     |
| `Item8A.Q8A1`   | string | The firm buys securities for itself from advisory clients, or sells securities it owns to them, that is principal transactions. |
| `Item8A.Q8A2`   | string | The firm buys or sells for itself securities that it also recommends to advisory clients, mutual fund shares excluded. |
| `Item8A.Q8A3`   | string | The firm recommends securities or other investment products in which it or a related person holds another ownership interest, apart from 8.A.(1) and (2). |
| `Item8B`        | object | Sales interest in client transactions, Item 8.B.           |
| `Item8B.Q8B1`   | string | As a broker-dealer or registered representative, the firm executes trades for brokerage customers in which advisory client securities are bought or sold, that is agency cross transactions. |
| `Item8B.Q8B2`   | string | The firm recommends that advisory clients buy securities for which it or a related person acts as underwriter, general or managing partner, or purchaser representative. |
| `Item8B.Q8B3`   | string | The firm recommends purchases or sales in which it or a related person holds another sales interest, apart from sales commissions as a broker or registered representative. |
| `Item8C`        | object | Investment or brokerage discretion, Item 8.C.              |
| `Item8C.Q8C1`   | string | The firm decides which securities are bought or sold for a client account. |
| `Item8C.Q8C2`   | string | The firm decides the amount of securities bought or sold.  |
| `Item8C.Q8C3`   | string | The firm decides the broker or dealer used.                |
| `Item8C.Q8C4`   | string | The firm decides the commission rates paid to a broker or dealer. |
| `Item8D`        | object | Item 8.D.                                                  |
| `Item8D.Q8D`    | string | The brokers or dealers behind a `Y` in 8.C.(3) are related persons. |
| `Item8E`        | object | Item 8.E.                                                  |
| `Item8E.Q8E`    | string | The firm or a related person recommends brokers or dealers to clients. |
| `Item8F`        | object | Item 8.F.                                                  |
| `Item8F.Q8F`    | string | The brokers or dealers behind a `Y` in 8.E are related persons. |
| `Item8G`        | object | Soft dollar benefits, Item 8.G.                            |
| `Item8G.Q8G1`   | string | The firm or a related person receives research or other products or services, apart from execution, from a broker-dealer or third party in connection with client securities transactions. |
| `Item8G.Q8G2`   | string | All those soft dollar benefits qualify as research or brokerage services under section 28(e) of the Securities Exchange Act of 1934. |
| `Item8H`        | object | Compensation paid for client referrals, Item 8.H.          |
| `Item8H.Q8H`    | string | 10/2012 version. The firm or a related person compensates any person for client referrals, directly or indirectly. |
| `Item8H.Q8H1`   | string | Answer to the first part of Item 8.H in later form versions. |
| `Item8H.Q8H2`   | string | Answer to the second part of Item 8.H in later form versions. |
| `Item8I`        | object | Compensation received for client referrals, Item 8.I.      |
| `Item8I.Q8I`    | string | The firm or a related person receives compensation from any person for client referrals, directly or indirectly. |

### Item 9, custody

| Field           | Type   | Meaning                                                   |
| --------------- | ------ | ---------------------------------------------------------- |
| `Item9A`        | object | Custody held by the firm, Item 9.A.                        |
| `Item9A.Q9A1A`  | string | The firm holds client cash or bank accounts.               |
| `Item9A.Q9A1B`  | string | The firm holds client securities.                          |
| `Item9A.Q9A2A`  | number | US dollar amount of those client assets.                   |
| `Item9A.Q9A2B`  | number | Number of clients whose assets it holds.                   |
| `Item9B`        | object | Custody held by related persons, Item 9.B.                 |
| `Item9B.Q9B1A`  | string | A related person holds client cash or bank accounts.       |
| `Item9B.Q9B1B`  | string | A related person holds client securities.                  |
| `Item9B.Q9B2A`  | number | US dollar amount of those client assets.                   |
| `Item9B.Q9B2B`  | number | Number of clients whose assets the related person holds.   |
| `Item9C`        | object | Custody safeguards, Item 9.C.                              |
| `Item9C.Q9C1`   | string | A qualified custodian sends account statements at least quarterly to the investors in the pooled investment vehicles the firm manages. |
| `Item9C.Q9C2`   | string | An independent public accountant audits those pooled vehicles each year, and the audited statements go to the investors. |
| `Item9C.Q9C3`   | string | An independent public accountant makes an annual surprise examination of client funds and securities. |
| `Item9C.Q9C4`   | string | An independent public accountant prepares an internal control report on custodial services, where the firm or a related person is a qualified custodian. |
| `Item9D`        | object | Qualified custodian role, Item 9.D.                        |
| `Item9D.Q9D1`   | string | The firm acts as a qualified custodian.                    |
| `Item9D.Q9D2`   | string | The firm or a related person acts as qualified custodian for its clients in connection with the advisory services it gives. |
| `Item9E`        | object | Surprise examination date, Item 9.E.                       |
| `Item9E.Q9E`    | string | Month the surprise examination started, as `YYYY-MM`, when the firm files an annual updating amendment and had such an examination in its last fiscal year. |
| `Item9F`        | object | Qualified custodian count, Item 9.F.                       |
| `Item9F.Q9F`    | number | How many persons act as qualified custodians for the firm's clients in connection with its advisory services, the firm and its related persons included. |

### Item 10 and Item 11, control and disclosure

| Field             | Type   | Meaning                                                 |
| ----------------- | ------ | -------------------------------------------------------- |
| `Item10A`         | object | Control persons, Item 10.A.                              |
| `Item10A.Q10A`    | string | A person not named in Item 1.A or Schedules A, B or C controls the firm's management or policies, directly or indirectly. |
| `Item11`          | object | Disclosure information, Item 11.                         |
| `Item11.Q11`      | string | One or more of the events in Item 11 involve the firm or any of its supervised persons. |
| `Item11A`         | object | Felony questions, Item 11.A.                             |
| `Item11A.Q11A1`   | string | The firm or an advisory affiliate was convicted of, or pleaded guilty or nolo contendere to, a felony in a domestic, foreign or military court. |
| `Item11A.Q11A2`   | string | The firm or an advisory affiliate was charged with a felony. |
| `Item11B`         | object | Misdemeanor questions, Item 11.B.                        |
| `Item11B.Q11B1`   | string | Convicted of, or pleaded guilty or nolo contendere to, a misdemeanor that involved investments or an investment-related business, or fraud, false statements or omissions, wrongful taking of property, bribery, perjury, forgery, counterfeiting, extortion, or a conspiracy to commit any of these. |
| `Item11B.Q11B2`   | string | Charged with a misdemeanor listed in 11.B(1).            |
| `Item11C`         | object | SEC or CFTC actions, Item 11.C.                          |
| `Item11C.Q11C1`   | string | The SEC or the CFTC found the firm or an advisory affiliate to have made a false statement or omission. |
| `Item11C.Q11C2`   | string | Found it to have violated SEC or CFTC rules or statutes. |
| `Item11C.Q11C3`   | string | Found it to have caused an investment-related business to have its authorization denied, suspended, revoked or restricted. |
| `Item11C.Q11C4`   | string | Entered an order against it in connection with investment-related activity. |
| `Item11C.Q11C5`   | string | Imposed a civil money penalty on it, or ordered it to cease and desist. |
| `Item11D`         | object | Actions by other regulators, Item 11.D.                  |
| `Item11D.Q11D1`   | string | Another federal or state regulator, or a foreign financial regulatory authority, ever found the firm or an advisory affiliate to have made a false statement or omission, or to have been dishonest, unfair or unethical. |
| `Item11D.Q11D2`   | string | Ever found it to have violated investment-related rules or statutes. |
| `Item11D.Q11D3`   | string | Ever found it to have caused an investment-related business to have its authorization denied, suspended, revoked or restricted. |
| `Item11D.Q11D4`   | string | Entered an order against it in the past ten years in connection with an investment-related activity. |
| `Item11D.Q11D5`   | string | Ever denied, suspended or revoked its registration or license, barred it by order from association with an investment-related business, or restricted its activity. |
| `Item11E`         | object | Actions by a self-regulatory organization or exchange, Item 11.E. |
| `Item11E.Q11E1`   | string | A self-regulatory organization or commodities exchange ever found the firm or an advisory affiliate to have made a false statement or omission. |
| `Item11E.Q11E2`   | string | Ever found it to have violated the organization's rules, minor rule violations under an SEC-approved plan excluded. |
| `Item11E.Q11E3`   | string | Ever found it to have caused an investment-related business to have its authorization denied, suspended, revoked or restricted. |
| `Item11E.Q11E4`   | string | Disciplined it by expulsion, suspension, a bar from association with other members, or another restriction. |
| `Item11F`         | object | Professional authorizations, Item 11.F.                  |
| `Item11F.Q11F`    | string | An authorization to act as an attorney, accountant or federal contractor granted to the firm or an advisory affiliate was ever revoked or suspended. |
| `Item11G`         | object | Pending regulatory proceedings, Item 11.G.               |
| `Item11G.Q11G`    | string | The firm or an advisory affiliate is now the subject of a regulatory proceeding that could produce a `Y` in Item 11.C, 11.D or 11.E. |
| `Item11H`         | object | Civil judicial actions, Item 11.H.                       |
| `Item11H.Q11H1A`  | string | A court enjoined the firm or an advisory affiliate in the past ten years in connection with an investment-related activity. |
| `Item11H.Q11H1B`  | string | A court ever found that the firm or an advisory affiliate violated investment-related statutes or rules. |
| `Item11H.Q11H1C`  | string | A court ever dismissed, under a settlement agreement, an investment-related civil action brought against the firm or an advisory affiliate by a state or foreign financial regulatory authority. |
| `Item11H.Q11H2`   | string | The firm or an advisory affiliate is now the subject of a civil proceeding that could produce a `Y` in Item 11.H(1). |

### FormInfo.Part1B, state-registered advisers

Only state-registered advisers carry `Part1B`. Paths below start at
`FormInfo.Part1B`.

| Field                          | Type   | Meaning                                    |
| ------------------------------ | ------ | ------------------------------------------- |
| `Item2`                        | object | Bond and capital information, Part 1B Item 2.B. |
| `Item2.Q1B2B1`                 | string | Name of the insurance company that issues the firm's bond. |
| `Item2.Q1B2B2`                 | string | Amount of the bond.                         |
| `Item2.Q1B2B4`                 | string | `Y` when the firm meets its home state's minimum capital requirements, where the home state sets them. |
| `DsclrQstns`                   | object | Disclosure questions, Part 1B Item 2.C to 2.F. |
| `DsclrQstns.Q1B2C`             | string | A bonding company ever denied, paid out on, or revoked a bond for the firm, an advisory affiliate or a management person. |
| `DsclrQstns.Q1B2D`             | string | Unsatisfied judgments or liens stand against the firm, an advisory affiliate or a management person. |
| `DsclrQstns.Q1B2E1`            | string | An arbitration claim over $2,500 against the firm, an advisory affiliate or a management person, now or in the past, that involved an investment or an investment-related business or activity. |
| `DsclrQstns.Q1B2E2`            | string | Such a claim that involved fraud, a false statement or an omission. |
| `DsclrQstns.Q1B2E3`            | string | Such a claim that involved theft, embezzlement or other wrongful taking of property. |
| `DsclrQstns.Q1B2E4`            | string | Such a claim that involved bribery, forgery, counterfeiting or extortion. |
| `DsclrQstns.Q1B2E5`            | string | Such a claim that involved dishonest, unfair or unethical practices. |
| `DsclrQstns.Q1B2F1`            | string | A civil, self-regulatory or administrative proceeding, now or in the past, in which the firm, an advisory affiliate or a management person was found liable, that involved an investment or an investment-related business or activity. |
| `DsclrQstns.Q1B2F2`            | string | Such a proceeding that involved fraud, a false statement or an omission. |
| `DsclrQstns.Q1B2F3`            | string | Such a proceeding that involved theft, embezzlement or other wrongful taking of property. |
| `DsclrQstns.Q1B2F4`            | string | Such a proceeding that involved bribery, forgery, counterfeiting or extortion. |
| `DsclrQstns.Q1B2F5`            | string | Such a proceeding that involved dishonest, unfair or unethical practices. |
| `ItemG`                        | object | Other business activities, Part 1B Item 2.G. |
| `ItemG.Q1BG1TaxPrprr`          | string | The firm is actively engaged in business as a tax preparer. |
| `ItemG.Q1BG1IssueScrts`        | string | As an issuer of securities.                 |
| `ItemG.Q1BG1XcldPoolInvmt`     | string | As a sponsor or syndicator of limited partnerships or equivalent, pooled investment vehicles excluded. |
| `ItemG.Q1BG1PoolInvmt`         | string | As a sponsor, general partner or managing member, or equivalent, of pooled investment vehicles. |
| `ItemG.Q1BG1ReAdvsr`           | string | As a real estate adviser.                   |
| `ItemG.Q1B2G2`                 | string | Description of any further business the firm, an advisory affiliate or a management person runs, outside Item 6.A of Part 1A and Item 2.G(1) of Part 1B, with the approximate time spent on it. |
| `ItemH`                        | object | Investments made on financial planning advice, Part 1B Item 2.H. |
| `ItemH.Q1B2HScrtsNvsmt`        | string | Securities investments made on those services at the end of the last fiscal year. |
| `ItemH.Q1B2HScrtsNvsmtAm`      | number | Exact amount when those securities investments pass $5,000,000. |
| `ItemH.Q1B2HNScrtsNvsmt`       | string | Non-securities investments on the same basis. |
| `ItemH.Q1B2HNScrtsNvsmtAm`     | number | Exact amount when those non-securities investments pass $5,000,000. |
| `ItemI`                        | object | Custody, Part 1B Item 2.I.                  |
| `ItemI.Q1B2I1`                 | string | The firm withdraws advisory fees straight from client accounts. |
| `ItemI.Q1B2I1A`                | string | It sends a copy of its invoice to the custodian or trustee at the same time as to the client. |
| `ItemI.Q1B2I1B`                | string | The custodian sends clients quarterly statements that show all disbursements from the account, the advisory fee included. |
| `ItemI.Q1B2I1C`                | string | Clients give written authorization for direct payment from the accounts the custodian or trustee holds. |
| `ItemI.Q1B2I2Ai`               | string | The firm or a related person acts as general partner, managing member or equivalent for a pooled investment vehicle that it advises, or whose investors it advises. |
| `ItemI.Q1B2I2AiiAtrny`         | string | An attorney is engaged to approve each payment or transfer from that pooled vehicle account. |
| `ItemI.Q1B2I2AiiCpa`           | string | A certified public accountant is engaged for that role. |
| `ItemI.Q1B2I2AiiOth`           | string | Another independent party is engaged for that role. |
| `ItemI.Q1B2I2AiiOthTx`         | string | Description of that other independent party. |
| `ItemI.Q1B2I2B`                | string | The firm or a related person is investment adviser and trustee of the same trust, or trustee of a trust whose beneficiaries are advisory clients. |
| `ItemI.Q1B2I3`                 | string | The firm requires prepayment of more than $500 per client, six months or more in advance. |
| `ItemJ`                        | object | Sole proprietor qualifications, Part 1B Item 2.J. |
| `ItemJ.Q1BJ1A`                 | string | The sole proprietor passed the Series 65 examination on or after 1 January 2000. |
| `ItemJ.Q1BJ1B`                 | string | The sole proprietor passed the Series 66 examination on or after 1 January 2000, and passed the Series 7 examination at any time. |
| `ItemJ.Q1BJ2A`                 | string | The sole proprietor holds an investment advisory professional designation. |
| `ItemJ.Q1BJ2BCfp`              | string | Certified Financial Planner, CFP.           |
| `ItemJ.Q1BJ2BCfa`              | string | Chartered Financial Analyst, CFA.           |
| `ItemJ.Q1BJ2BChfc`             | string | Chartered Financial Consultant, ChFC.       |
| `ItemJ.Q1BJ2BCic`              | string | Chartered Investment Counselor, CIC.        |
| `ItemJ.Q1BJ2BPfs`              | string | Personal Financial Specialist, PFS.         |
| `ItemJ.Q1BJ2BNone`             | string | None of the designations above.             |
| `ItemK`                        | object | Legal status, Part 1B Item 2.K.             |
| `ItemK.Q1B2K1`                 | string | Date the firm obtained its legal status, when it is not a sole proprietorship. |

The `QxxNN` codes are the question numbers on Form ADV. An item object is empty
when the adviser did not answer it. The exempt reporting adviser in the example
response has empty `Item5A` to `Item5L`. Item numbers can hold answers from more
than one form version, so a row can carry the 10/2012 keys, the later keys, or
both.

`size` counts firms and caps at 50. `from` moves the window. A `from` plus
`size` above 10,000 returns `{"total":{"value":0},"filings":[]}` with no error
and no `relation` key. The JSON arrives as one text block. See
[response format](../response-format.md).

## Example

Prompt: "Show me the most recently registered adviser in the Form ADV register."

```json
{ "name": "form-adv-firms", "arguments": { "query": "Info.FirmCrdNb:*", "size": 1 } }
```

```json
{
  "total": { "value": 10000, "relation": "gte" },
  "filings": [
    {
      "Info": {
        "SECRgnCD": "NYRO",
        "FirmCrdNb": 344073,
        "SECNb": "802-137266",
        "BusNm": "LIBERTY PARTNERS WEALTH MANAGEMENT INC.",
        "LegalNm": "LIBERTY PARTNERS WEALTH MANAGEMENT INC."
      },
      "MainAddr": {
        "Strt1": "244 MADISON AVENUE",
        "City": "NEW YORK",
        "State": "NY",
        "Cntry": "United States",
        "PostlCd": "10016"
      },
      "Rgstn": [{ "FirmType": "ERA", "St": "ACTIVE", "Dt": "2026-08-11" }],
      "Filing": [{ "Dt": "2026-08-11", "FormVrsn": "10/2021" }],
      "id": 344073
    }
  ]
}
```

Keys were removed to fit. The values are unchanged. The default sort is CRD
descending, so the first row is the newest registrant.

## Limits and errors

- `size` above 50 returns HTTP 400 with
  `Maximum 'size' limit of 50 exceeded.`
- Omitting `size` returns 50 firms, not 10.
- `Rgstn` is absent from state-shaped rows. Those rows carry `StateRgstn` and
  `ERA` instead.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-individuals`](./form-adv-individuals.md),
  [`form-adv-brochures`](./form-adv-brochures.md)
- [`form-adv-schedule-a-direct-owners`](./form-adv-schedule-a-direct-owners.md),
  [`form-adv-schedule-d-7-b-1`](./form-adv-schedule-d-7-b-1.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
