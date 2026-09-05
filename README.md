# America PAC independent-expenditure data

Independent expenditures reported to the Federal Election Commission by
**America PAC** (committee `C00879510`), cleaned and totalled.

Updated twice daily. Every row links back to the FEC filing it came from.

## The files

### `race_totals.csv`

One row per race. This is the summary.

| column | |
| --- | --- |
| `race_id` | stable ID: `2026-H-NY-18`, `2026-S-NC`, `2024-P-US` |
| `race_name` | readable label: `NY-18 (2026)` |
| `office` / `state` / `district` | `H`/`S`/`P`, two-letter state, zero-padded district |
| `support_amount` | spent supporting candidates in this race |
| `oppose_amount` | spent opposing them |
| `total_amount` | the two added together |
| `expenditure_count` | how many expenditures |
| `last_spend_date` | most recent activity |

### `latest_spending.csv`

The 100 most recent expenditures, newest first. A slice of `transactions.csv`
with the display columns only: `spend_date`, `race_id`, `race_name`,
`candidate_name`, `support_oppose`, `amount`, `payee_name`, `purpose`,
`pdf_url`.

### `transactions.csv`

Every expenditure, one row each. This is the file the totals are built from —
filter it to a race and you get exactly the rows behind that number.

Roughly grouped, the columns are:

- **what happened** — `spend_date`, `amount`, `support_oppose`, `purpose`, `payee_name`
- **who it was about** — `race_id`, `race_name`, `candidate_id`, `candidate_name`, `office`, `state`, `district`
- **the receipt** — `transaction_id`, `sub_id`, `file_number`, `filing_form`, `report_type`, `image_number`, `pdf_url`
- **the raw dates** — `expenditure_date`, `disbursement_date`, `dissemination_date`
- **how the race was decided** — `race_id_source`, `race_disagreement`, `race_id_inline`, `race_id_candidate`

`pdf_url` opens the filing on the FEC's DocQuery.

### `reports.csv`

One row per financial report the committee filed. Where `transactions.csv` says
what was spent on candidates, this says what the committee raised, what it
spent in total, and what it held.

| column | |
| --- | --- |
| `report_type` / `report_type_full` | `Q2`, `JULY QUARTERLY` |
| `coverage_start_date` / `coverage_end_date` | the months the report covers |
| `receipt_date` | when the FEC received it |
| `total_receipts_period` | raised during the covered period |
| `total_disbursements_period` | spent during the covered period |
| `cash_on_hand_beginning_period` / `cash_on_hand_end_period` | balance at each end |
| `debts_owed_by_committee` | outstanding debts |

Nothing is disclosed between reports. The committee files a few times a year,
so the gaps between coverage periods are unreported rather than quiet.

Amended reports are dropped. The API returns both the original and the
correction, and returns a `most_recent` field but ignores `most_recent` as a
query parameter. `is_amended=false` is what removes the superseded copies.

### `report_parts.csv`

The named categories inside each report, in long form: one row per line item.
`side` is `in` or `out`, `label` is the category, `amount` is the figure for
that covered period. Join to `reports.csv` on `coverage_end_date`.

A period can have every dollar in one category, in which case the others are
zero. That is the filing, not a gap in the data.

## Two things to know before you recompute this

**1. The FEC reports the same expenditure twice.**

A committee spending close to an election files a 24- or 48-hour notice
(Form 24), then reports that same expenditure again on its next quarterly
report (Form 3X). Both copies stay in the API permanently, and both are
flagged `most_recent = true`.

Summing raw Schedule E rows for America PAC gives **$360,259,079.06**. The
real figure is **$174,637,011.09**.

An expenditure keeps its `transaction_id` across both filings, so
`committee_id` + `cycle` + `transaction_id` identifies one real expenditure.
That is what these files are grouped on.

Do not deduplicate on date, amount and payee together: `SE24.644` and
`SE24.645` are two separate $111,111 payments to the same vendor on the same
day, and collapsing them deletes real spending.

**2. Our totals run about $705,000 above the FEC's own.**

18 filings carry a candidate ID that matches nobody who ran in that cycle.
The cause is mundane: a person gets a new FEC candidate ID each time they
register a campaign, and the committee wrote down an earlier one.

```text
NJ-07   H0NJ07089  "KEAN, TOM"             <- what the filing said
        H0NJ07261  "KEAN, THOMAS H. JR."   <- his 2024 registration

WA-03   H4HI02116  "KENT, JOE"             <- a 2014 Hawaii registration
        H2WA03100  "KENT, JOSEPH"          <- his 2024 registration
```

The office, state and district the committee wrote are correct in every one of
these, so we assign the race from those fields and keep the spending. The FEC's
own `by_candidate` totals drop these rows.

Filter `transactions.csv` on `race_id_source = "unmatched_candidate_id"` to see
them. `race_id_source` records how every row was assigned:

| value | meaning |
| --- | --- |
| `candidate_id` | matched a candidate who ran that cycle; race from the FEC candidate registry |
| `unmatched_candidate_id` | ID matched nobody that cycle; race from the committee's own office fields |
| `filing_fields` | no candidate ID on the filing; race from the committee's own office fields |
| `unresolved` | no race could be determined; excluded from `race_totals.csv` |

Otherwise these numbers reconcile against the FEC's published per-candidate
totals for the completed 2024 cycle exactly, across every race.

## Source

FEC API. Independent expenditures from Schedule E (`/schedules/schedule_e/`),
candidate records from `/candidates/totals/`, and financial reports from
`/committee/{id}/reports/`. Nothing here is scraped, and no figure is derived
from free text: races come from structured FEC fields only.
