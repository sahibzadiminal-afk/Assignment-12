# 1. Attempt before AI — diagnosis and planned decisions

**File audited:** `teaching_prices_messy.csv`
**SHA-256 of the original file:** `5818dad3807487d4e37f764e5d19cbaa152edd78fe4db182eb28b087c8709d35`
**Git commit (`attempt-before-AI`):** `a37867f`

## What I expected
Weekly prices, 12 dates (1 Jan – 19 Mar 2026, all Thursdays) × 4 items (Staple A, Oil B, Pulse C, Sweetener D) × 2 cities = **96 rows**, one per combination, in PKR per kg (illustrative). The file has **97 rows**, so something is already off before looking at values.

## Diagnosis (problem → code that revealed it → planned decision → reason)

| # | Problem | Code that revealed it | Planned decision | Reason |
|---|---|---|---|---|
| 1 | 97 rows instead of 96; one **exact duplicate** (2026-01-08, Staple A, City 1, 99.5 appears twice, rows 8 and 96) | `df.shape`; `df.duplicated(keep=False)` | **Correct (remove)**: keep the first copy | Identical in every column, so no information is lost. I will confirm exactly one row is removed. |
| 2 | **Blank price** (2026-01-01, Pulse C, City 2) | `df.info()` (price 96 non-null of 97); `df.isna().sum()` | **Flag and retain**; do not fill | Any number I filled in would be invented. Keeping the row keeps the 12-observations-per-group structure and shows the gap honestly. |
| 3 | **Price stored as text `"1,250"`** (2026-01-15, Staple A, City 2). This is why `price` loads as `object`. | `df.info()` (dtype object); `pd.to_numeric(..., errors='coerce')` to find unparseable cells | **Investigate, then flag/exclude the value** | Stripping the comma gives 1250, which is about 11× the other Staple A prices (99.5–122). It is not just a formatting problem; the true value is unknown, so I will not guess (e.g. 125.0 or 112.5). |
| 4 | **`9999` in Sweetener D, City 1** (2026-01-22) | `df['price'].describe()` (max 9999, mean 305 vs median 200); filter `> 500` | **Flag and exclude the value**, retain the row | 9999 is a typical "missing" code, not a price (neighbours ≈ 153–162). It would wreck means and charts if left in. |
| 5 | **Inconsistent label `oil b`** (2026-02-05, City 1, price 108.5) | `df['item'].value_counts()` (5 labels instead of 4); `groupby(['item','city']).size()` | **Investigate, then correct** | Lower-casing it to "Oil B" would be wrong: its price (108.5) is Staple-A-sized, Oil B is ≈ 318 that week, and a real Oil B / City 1 row already exists for that date. Staple A / City 1 is missing for that date, and Staple A City 2 is 114.5 (the City 2 premium is 6 everywhere else, giving 108.5). So it is a mislabelled Staple A row. Correct the label and log it. |

## Things I checked and decided to leave alone

| Item | Code | Decision | Reason |
|---|---|---|---|
| `unit` is the same in every row (PKR per kg, illustrative) | `value_counts()` | **Retain** | Constant, but it documents the unit. |
| `data_label` says ILLUSTRATIVE TEACHING DATA, same in every row | `value_counts()` | **Retain** | It is a warning about what the data is. Removing it would make fake-looking-real data. |
| Dates: 12 distinct, all Thursdays, weekly | `nunique()`, weekday check | **Retain** (convert to datetime) | No gaps or odd dates. |
| Pulse C prices barely move (200 → 207.5, in steps of 2.5) | `describe()` / reading the rows | **Retain** | Small, steady steps are plausible in illustrative data, not an error. |
| Staple A, City 1 dips from 101 to 99.5 on 2026-01-08 | reading the rows | **Retain** | A small move, no reason to doubt it. |

## Order of work (planned)
1. Keep the raw file untouched; work on `.copy()`.
2. Fix the `oil b` label *before* removing duplicates and before any aggregation.
3. Remove the one exact duplicate.
4. Parse price to numbers; blank (not guess) the three unreliable prices and flag them in a `flag` column; keep `price_raw` as an audit trail.
5. Verify: 96 rows, every item × city exactly 12 times, no valid price changed.
