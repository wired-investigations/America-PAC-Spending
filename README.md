# America PAC independent-expenditure data

Independent expenditures reported to the Federal Election Commission by
**America PAC** (committee `C00879510`), cleaned and totalled by WIRED.

Updated twice daily from the FEC. Every row links back to the filing it came
from.

## The files

### `expenditures.csv`

Every expenditure, one row each, in the 2024 and 2026 cycles. Every total in
the other files is built from these rows: filter to a race and you get exactly
the payments behind its number.

Roughly grouped, the columns are:

- **what happened** — `spend_date`, `amount`, `support_oppose`, `purpose`, `payee_name`
- **who it was about** — `race_id`, `race_name`, `candidate_id`, `candidate_name`, `fec_candidate_name`, `party`, `office`, `state`, `district`
- **the receipt** — `transaction_id`, `sub_id`, `file_number`, `filing_form`, `report_type`, `image_number`, `pdf_url` (opens the filing on the FEC's site)
- **the dates** — `expenditure_date`, `disbursement_date`, `dissemination_date`
- **the election** — `election_type`: `G2026` is the 2026 general, `S2025` a 2025 special election
- **how the race was decided** — `race_id_source`, `race_disagreement`, `race_id_inline`, `race_id_candidate`

`candidate_name` is the name the candidate campaigns under (`Ken Paxton`);
`fec_candidate_name` is the FEC's own string (`PAXTON, WARREN KENNETH JR`).
See `display_names.csv` below.

### `race_totals.csv`

One row per race in the 2026 cycle: `race_id` (`2026-H-NY-17`, `2026-S-TX`),
the supported and opposed candidates and their parties, `support_amount`,
`oppose_amount`, `total_amount`, `expenditure_count` and `last_spend_date`.

A race can have spending aimed at more than one candidate on the same side, in
a primary for instance, so the candidate columns name the one who drew the
most on that side. A blank oppose side means nothing was spent opposing
anyone in that race.

The FEC's 2026 cycle also covers 2025, so this file includes the April 2025
Florida special elections (`2025-H-FL-01`, `2025-H-FL-06`).

### `embed_*.csv`

The five files WIRED's candidate-spending graphic reads, already added up so
the graphic does no arithmetic of its own. They cover only races held in 2026
(the midterms), so the 2025 Florida specials are left out, and their dates are
the day each payment was made (`disbursement_date`).

| file | |
| --- | --- |
| `embed_races.csv` | one row per race, with its newest payment |
| `embed_payments.csv` | one row per expenditure, newest first within each race, with a `filing_url` |
| `embed_states.csv` | one row per state, split by chamber and by supporting/opposing |
| `embed_summary.csv` | one row: the graphic's headline figures and shares |
| `embed_daily.csv` | running totals by day, nationally and by state, for its charts |

Totals are in whole dollars, rounded once at the smallest level shown and then
added, so a state's Senate and House figures always sum to its total.
`expenditures.csv` keeps the cents.

### `display_names.csv`

How each FEC candidate ID is named in these files: `candidate_id`,
`display_name`. FEC names are often not the names candidates go by ("FEELY,
THOMAS JAMES" campaigns as Jay Feely), so WIRED compiled display names from
the congressional roster, certified state ballot lists, campaign committee
names and news coverage, and reviews them. A candidate not on this list
appears under the FEC's name, reordered (`Ryan Edward Mackenzie`).

## Two things to know before you recompute this

**1. The FEC reports the same expenditure more than once.**

A committee spending close to an election files a 24- or 48-hour notice
(Form 24), then reports that same expenditure again on its next quarterly
report (Form 3X). Both copies stay in the FEC's data permanently.

As of Oct. 1, 2026, summing the FEC's 2,992 raw rows for America PAC gives
**$382,498,258.23**; the 1,682 real expenditures behind them total
**$196,876,190.26**.

An expenditure keeps its `transaction_id` across both filings, so
`committee_id` + `cycle` + `transaction_id` identifies one real expenditure.
That is what these files are grouped on.

Don't deduplicate on date, amount and payee together: `SE24.644` and
`SE24.645` are two separate $111,111 payments to the same vendor on the same
day, and collapsing them deletes real spending.

**2. Our totals run about $730,000 above the FEC's own.**

Nineteen filings carry a candidate ID that matches nobody who ran in that
cycle. The cause is mundane: a person gets a new FEC candidate ID each time
they register a campaign, and the committee wrote down an earlier one.

```text
NJ-07   H0NJ07089  "KEAN, TOM"             <- what the filing said
        H0NJ07261  "KEAN, THOMAS H. JR."   <- his 2024 registration

WA-03   H4HI02116  "KENT, JOE"             <- a 2014 Hawaii registration
        H2WA03100  "KENT, JOSEPH"          <- his 2024 registration
```

The office, state and district the committee wrote are correct in every one of
these, so we assign the race from those fields and keep the spending. The FEC's
own per-candidate totals drop these rows.

Filter `expenditures.csv` on `race_id_source = "unmatched_candidate_id"` to see
them. `race_id_source` records how every row was assigned:

| value | meaning |
| --- | --- |
| `candidate_id` | matched a candidate who ran that cycle; race from the FEC candidate registry |
| `unmatched_candidate_id` | ID matched nobody that cycle; race from the committee's own office fields |
| `filing_fields` | no candidate ID on the filing; race from the committee's own office fields |
| `unresolved` | no race could be determined; left out of the totals |

Otherwise these numbers match the FEC's published per-candidate totals for the
completed 2024 cycle exactly, race for race.

## Source

The FEC API: independent expenditures from Schedule E
(`/schedules/schedule_e/`) and candidate records from `/candidates/totals/`.
Nothing is scraped, and no race is inferred from free text: races come from
structured FEC fields only.
