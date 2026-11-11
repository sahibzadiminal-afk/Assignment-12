# 3. Cleaning Code Court, log and reflection

All numbers come from `2_work.ipynb`. The AI's code was given a plain-words description of the file (columns, types, problems); it was run on a copy and never on `df_raw`.

## The AI's code (as suggested)

```python
ai.columns = ai.columns.str.strip().str.lower()                          # L1
ai['item'] = ai['item'].str.strip().str.title()                          # L2
ai['price'] = ai['price'].astype(str).str.replace(',', '').astype(float) # L3
ai = ai.drop_duplicates()                                                # L4
ai = ai.dropna(subset=['price'])                                         # L5
ai = ai[ai['price'] < 1000]                                              # L6
ai['date'] = pd.to_datetime(ai['date'])                                  # L7
ai = ai.drop(columns=['unit', 'data_label'])                             # L8
```

## Court verdicts (one trial per line)

| Line | What it does | Verdict | Evidence (before → after) |
|---|---|---|---|
| **L1** | Strips and lower-cases column names | **Keep** | Columns were already clean lowercase; no change, no risk. `df.columns` identical before and after. |
| **L2** | Strips whitespace and Title-Cases `item` | **Modify** | Looks harmless but is the most dangerous line. It turns `oil b` into `Oil B`, creating **two** Oil B / City 1 rows on 2026-02-05 (108.5 and 318.0). The 108.5 row is really Staple A / City 1. Result: Oil B / City 1 = **13** rows, Staple A / City 1 = **11**. The item count becomes 4 and *looks* fixed, but the data is wrong. Modified: relabel that one row to `Staple A` explicitly, with evidence, and log it. |
| **L3** | Removes commas, converts price to float | **Modify** | Works mechanically: `"1,250"` becomes 1250.0, `price` is numeric. But it turns a suspicious value into a "valid-looking" one (about 11× its neighbours). Modified: strip commas (so the value can be read) and then flag 1250 as implausible and blank it. |
| **L4** | `drop_duplicates()` | **Keep** | Rows 97 → 96, exactly one removed (2026-01-08 Staple A City 1, 99.5, identical in every column). Checked that it was the only exact duplicate. (Order matters: it is only safe *after* the mislabel is fixed, otherwise it can hide or create duplicates.) |
| **L5** | Drops rows with missing price | **Reject** | Removes the Pulse C / City 2 row for 2026-01-01 (96 → 95 after L4). That deletes a real slot in the ledger, so Pulse C / City 2 has 11 weeks. It also makes "missing price" fall to 0, which hides the problem instead of reporting it. Replaced by: keep the row, price = NaN, flag "missing price in source; not imputed". |
| **L6** | Keeps only `price < 1000` | **Modify** | Removes the 9999 row and the 1250 row (95 → 93), but (a) deletes the rows so Sweetener D / City 1 and Staple A / City 2 drop to 11 weeks, (b) the 1000 cut-off is arbitrary and would let through errors such as a 999 or a 600 for a 100-priced item, and (c) nothing is recorded. Modified: blank the two prices, keep the rows, flag each with its reason. The threshold I used (500) is justified by the data (max real price = 350). |
| **L7** | Converts `date` to datetime | **Keep** | 12 distinct dates, all Thursdays, range 2026-01-01 to 2026-03-19 before and after. No dates lost or changed. |
| **L8** | Drops `unit` and `data_label` | **Reject** | The columns are constant, so dropping them looks tidy, but `data_label` is the warning that this is *ILLUSTRATIVE TEACHING DATA - NOT ACTUAL PBS OBSERVATIONS*. Removing it means a clean-looking table that could be mistaken for real statistics. `unit` records PKR per kg (illustrative). Both retained. |

### Before-and-after table

| Measure | Raw | AI code as-is | My cleaning |
|---|---|---|---|
| Rows | 97 | **93** | **96** |
| Missing price | 1 (+ `"1,250"` text, + `9999` code) | 0 (rows deleted) | 3, all flagged |
| Max price | 9999 | 350 | 350 |
| Mean price | 315.46 | 203.61 | 203.61 |
| Distinct item labels | 5 | 4 | 4 |
| Item × city groups with exactly 12 rows | 8 of 8 on paper (but Staple A / City 1 only reaches 12 because of the duplicate, and an extra `oil b` group exists) | **3 of 8** (Oil B/City 2, Pulse C/City 1, Sweetener D/City 2) | **8 of 8** |
| Valid prices changed without my knowing | – | Staple A / City 1 (02-05) now sits under Oil B | 0 (92 unflagged rows compared with raw: 0 differences) |

The overall mean is identical for the AI and for me (203.61) because both removed the same bad values. That is a good example of why a single summary number is not evidence that cleaning was correct: the AI's version has 3 fewer observations and one value under the wrong item. The row counts per group are what expose it.

**Count check.** After my cleaning every item/city appears exactly **12** times (96 rows). After the AI's code, the counts are Oil B/City 1 = 13, Staple A/City 1 = 11, Staple A/City 2 = 11, Pulse C/City 2 = 11, Sweetener D/City 1 = 11.

## Cleaning log (rows for the final cleaned file)

| # | Row(s) | Problem | Detected by | Decision | Reason / evidence | Effect |
|---|---|---|---|---|---|---|
| 1 | 2026-02-05, City 1, item `oil b`, 108.5 | Inconsistent label; actually a Staple A price | `item.value_counts()` (5 labels), `groupby(['item','city']).size()` | **Correct** → `Staple A` | Price is Staple-A-sized (Oil B ≈ 318); a real Oil B / City 1 row already exists that date; Staple A / City 1 was missing for that date; City 2 − City 1 = 6 for all 45 other date/item pairs where both exist, and 114.5 − 108.5 = 6. | 1 label changed, flagged in `flag` |
| 2 | 2026-01-08, Staple A, City 1, 99.5 (second copy, row 96) | Exact duplicate | `duplicated(keep=False)`; row count 97 vs 96 expected | **Remove** (keep first) | Identical in all six columns; exactly 1 row removed. | 97 → 96 rows |
| 3 | 2026-01-01, Pulse C, City 2 | Missing price | `isna().sum()`; `info()` | **Flag, retain, no imputation** | Filling it would be inventing data. Row kept so the group still has 12 weeks. | price = NaN, flagged |
| 4 | 2026-01-15, Staple A, City 2, `"1,250"` | Text price; implausible magnitude | `info()` (object dtype); `to_numeric(errors='coerce')`; `describe()` | **Flag, exclude value, retain row** | After removing the comma it is 1250, about 11× its neighbours (City 1 same week = 103; Staple A range 99.5–122). True value unknown (can't tell 125.0 from 112.5), so no correction. | price = NaN, original text kept in `price_raw` |
| 5 | 2026-01-22, Sweetener D, City 1, `9999` | Sentinel / missing-value code | `describe()` (max 9999, mean far above median); filter `> 500` | **Flag, exclude value, retain row** | Neighbours ≈ 153–162; 9999 is a placeholder, not a price. | price = NaN, flagged |
| 6 | `unit`, `data_label` columns | Constant columns | `value_counts()` | **Retain** | They carry the unit and the ILLUSTRATIVE DATA warning. | none |
| 7 | `date` column | Stored as text | `info()` | **Convert** to datetime | 12 valid Thursdays, no gaps. | dtype change only |

Audit trail in `teaching_prices_clean.csv`: `price` (cleaned numeric), `price_raw` (original text) and `flag` (reason for any change).

## Reflection

**Where my decisions differed from the AI's.** Three differences mattered. (1) I kept rows with bad prices and blanked the *value*; the AI deleted whole rows (L5, L6), which hid three real observations' existence and made groups uneven (11 weeks instead of 12). (2) I treated `oil b` as a data problem to investigate, not a spelling problem. L2's Title Case makes the label look right while moving a Staple A price into Oil B. (3) I kept `data_label`; L8 would have stripped the only warning that the numbers are illustrative.

**Which was better supported.** Mine, because each decision has a piece of evidence beside it: a row count, a `value_counts()`, the City 2 − City 1 gap of 6, the neighbouring prices. The AI's lines were short and plausible but came with no evidence and no record of what they changed. The AI was right on L1, L4 and L7, and those were verified rather than assumed.

**What I would have missed alone.** The AI's code highlighted for me that dropping or recasing can be silent: I would probably have lower-cased `oil b` to `Oil B` and moved on, and the count check (13 vs 12) is what exposed that. I might also have been tempted to "repair" `"1,250"` by guessing `125.0`. Comparing group counts after cleaning, not just looking at the mean, was the check that mattered most.

**What I still can't know.** I cannot recover the true values for the blank, `"1,250"` and `9999` cells; that needs the original source. The `oil b` correction is strongly supported but is still an inference, so it is flagged in the file rather than hidden. And because the data is illustrative teaching data, none of this says anything about real prices.
