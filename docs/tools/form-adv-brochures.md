# form-adv-brochures

List the Form ADV Part 2 brochures an investment adviser has on file, with a
link to each document.

|                 |                                    |
| --------------- | ---------------------------------- |
| Category        | Investment advisers                |
| Required input  | `crd`                              |
| Returns         | `{brochures[]}`, with no `total`   |
| Pagination      | **None.** No `from`, `size` or `sort`. |
| REST equivalent | `GET /form-adv/brochures/{crd}`    |

## What it does

Part 2 of Form ADV is the plain-English brochure. It describes fees, services,
conflicts of interest and the adviser's disciplinary record. This tool takes one
CRD number and returns the brochures currently on file for that adviser. One
item is one brochure.

The tool returns metadata and a link. It does **not** return the brochure text.
Each `url` points to a PDF on `files.adviserinfo.sec.gov`.

The list is the current one. There is no version history and no as-of date. A
large wirehouse files many brochures, one per programme. Morgan Stanley Smith
Barney, CRD 149777, returned 12 on 2026-08-13.

## When to use it

- What does this adviser charge, and how does it disclose conflicts?
- Which advisory programmes does this firm run?
- When did the adviser last update its brochure?
- Give me a link to the current Part 2A brochure for this firm.

## When to use a different tool

| Situation                             | Better tool                                     | Why                                                    |
| ------------------------------------- | ----------------------------------------------- | ------------------------------------------------------ |
| You do not know the CRD number        | [`form-adv-firms`](./form-adv-firms.md)         | Search `Info.BusNm`, then read `Info.FirmCrdNb`.       |
| You want structured Part 1 data       | [`form-adv-firms`](./form-adv-firms.md)         | Assets, client types and Item 11 answers are in Part 1. |
| You want the separate account detail  | [`form-adv-schedule-d-5-k`](./form-adv-schedule-d-5-k.md) | Schedule D 5.K is structured, the brochure is prose. |
| You want an EDGAR document            | [`get-edgar-file`](./get-edgar-file.md)         | Brochures are on IAPD, not on EDGAR.                    |

## Input

| Parameter | Type   | Required | Constraints                        | Notes                                             |
| --------- | ------ | -------- | ---------------------------------- | ------------------------------------------------- |
| `crd`     | string | Yes      | Digits only, 1 to 10 characters    | The firm CRD number, for example `"149777"`. Send it as a string. |

The tool takes no query, no paging and no sorting. Anything else you pass is
ignored.

## Output

The envelope is `{brochures[]}`. This shape is unique to this tool. There is no
`total` and no `data` key. An empty list is `{"brochures":[]}`.

| Field           | Type   | Meaning                                                              |
| --------------- | ------ | -------------------------------------------------------------------- |
| `versionId`     | number | Brochure version ID. It is also the `BRCHR_VRSN_ID` in the URL.      |
| `name`          | string | Brochure title as filed, upper case, for example `SELECT UMA PROGRAM BROCHURE`. |
| `dateSubmitted` | string | `YYYY-MM-DD`. Normalised from the IAPD `M/D/YYYY` format.            |
| `url`           | string | Link to the PDF on `files.adviserinfo.sec.gov`.                       |

**There is no pagination.** Every current brochure comes back in one call. The
list is short. Twelve brochures returned about 2 KB.

An empty `brochures` array has two possible causes and the response does not
tell them apart. Either the adviser has no Part 2 brochure on file, which is
normal for an exempt reporting adviser, or the live lookup on
`adviserinfo.sec.gov` failed. The handler catches the error and returns an empty
list. Confirm with a firm you know has brochures before you conclude the adviser
has none.

## Example

Prompt: "List the Part 2 brochures for adviser CRD 344073."

```json
{ "name": "form-adv-brochures", "arguments": { "crd": "344073" } }
```

```json
{ "brochures": [] }
```

That firm is an exempt reporting adviser and files no Part 2 brochure. Here is
the same call for Morgan Stanley Smith Barney, CRD 149777. It was verified on
the REST route on 2026-08-13, because the probe capture used CRD 344073.

```json
{
  "brochures": [
    {
      "versionId": 1054240,
      "name": "OUTSOURCED CHIEF INVESTMENT OFFICE (OCIO)",
      "dateSubmitted": "2026-08-07",
      "url": "https://files.adviserinfo.sec.gov/IAPD/Content/Common/crd_iapd_Brochure.aspx?BRCHR_VRSN_ID=1054240"
    },
    {
      "versionId": 1054002,
      "name": "SELECT UMA PROGRAM BROCHURE",
      "dateSubmitted": "2026-08-05",
      "url": "https://files.adviserinfo.sec.gov/IAPD/Content/Common/crd_iapd_Brochure.aspx?BRCHR_VRSN_ID=1054002"
    }
  ]
}
```

Ten more brochures were removed to fit. The values are unchanged.

## Limits and errors

- A CRD with a non-digit character returns HTTP 404 and
  `{"message":"CRD invalid."}`. A CRD longer than 10 characters returns the
  same.
- An empty array is a valid answer. It is not an error.
- This tool reads `adviserinfo.sec.gov` live. It is slower than the search
  tools. The probe call took 346 ms.
- Shared behaviour is in [limits and errors](../limits-and-errors.md).

## Related

- [`form-adv-firms`](./form-adv-firms.md). Find the CRD number first.
- [`form-adv-individuals`](./form-adv-individuals.md)
- [`form-adv-schedule-d-5-k`](./form-adv-schedule-d-5-k.md)
- REST documentation:
  [Investment Adviser and Form ADV API](https://sec-api.io/docs/investment-adviser-and-adv-api)
