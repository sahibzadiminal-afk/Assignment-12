# Assignment 2: The Dirty Ledger

**Course:** ICA-F26 · The Prices Desk
**Topic:** Week 2 — Loading, cleaning and data quality
**Name:** Sahibzadi Minal Ghori  
**Student ID:** 37170

## What this folder contains

| File | Contents |
|---|---|
| `0_cover.md` | This cover page |
| `1_attempt.md` | My diagnosis and planned decisions, written **before** using any AI tool (with file hash) |
| `2_work.ipynb` | Diagnosis code, the AI's code run on a copy, my cleaning on a copy, before/after table and checks |
| `3_log_reflection.md` | Cleaning Code Court verdicts, cleaning log rows, reflection |
| `teaching_prices_messy.csv` | Original file, never edited |
| `teaching_prices_clean.csv` | Output of my cleaning (96 rows, flagged rows documented) |

## One-paragraph summary

The ledger holds 97 rows where 96 were expected (12 weekly dates × 4 items × 2 cities). I found five problems: one exact duplicate row, one blank price, a price written as `"1,250"` (about 11× its neighbours), a `9999` code, and one row labelled `oil b` that is really a mislabelled Staple A / City 1 price. I corrected one label with evidence, removed one exact duplicate, and **flagged and blanked (not guessed)** three unreliable prices while keeping their rows, so every item/city still appears exactly 12 times. The AI's code removed three real rows by dropping them and silently merged the mislabelled row into Oil B, which left five groups with the wrong row counts.

**Data note:** the file is labelled *ILLUSTRATIVE TEACHING DATA - NOT ACTUAL PBS OBSERVATIONS*. Nothing in this work describes real prices.
