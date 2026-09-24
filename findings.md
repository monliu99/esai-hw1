# HW1 Findings Log — NHAMCS-ED 2015

Running notes on results and interpretation, organized by assignment question.
Update as we go; use this to draft the 3–4 page write-up and slides.

## Q1. Load and Inspect the Data

**(a) Rows and columns:**
- Shape: **21,061 rows × 1,031 columns** — matches the codebook (`desc.txt`) exactly.
- Each row = one ED visit (per assignment; to be confirmed with an ID-style column check).

**(b) Missingness:**
- A large block of drug-classification sub-fields (`RX##V#C#`, `RX##CAT#` for medication slots
  17, 25, 27–30) are **100% missing (21,061/21,061)** — not a skip pattern, this variable set
  appears entirely unpopulated for these higher slot numbers in the 2015 file. Not used by any
  assignment question, so no action needed, but worth a one-line mention as a data-quality note.
- All of the assignment's core variables (`AGE`, `SEX`, `ETHIM`, `WAITTIME`, `LOV`, `PATWT`,
  `VDAYR`, `ARRTIME`, `PAYTYPER`, `HTN`, `DIABTYP2`, `OBESITY`, `DEPRN`, `TOTCHRON`,
  `INJPOISAD`, `INTENT15`, `ANYIMAGE`, `NUMMED`) show **0 missing** via `isna()`.
- **Caveat to state explicitly in the write-up:** `isna()==0` does not guarantee full validity —
  NHAMCS commonly encodes "unknown/blank/not asked" as sentinel numeric codes (e.g., -9, -8, -7)
  rather than true NaN. Need to check `.value_counts()` on key columns before trusting them at
  face value. *(pending)*

**(c) What each row represents:** Each row = one ED visit (per assignment). Consistent with every
row having visit-level fields populated (arrival day/time, wait time, weight). *(no separate ID
check run, but no evidence contradicting this)*

**Data quality note:** `pd.read_csv` raised a `DtypeWarning` on 36 columns — all diagnosis/drug ID
code fields (`CAUSE2/3`, `DIAG43D/53D`, `HDDIAG1-5` + coded variants, `DRUGID9`–`DRUGID30`).
These are alphanumeric medical codes, not analysis variables used in this assignment — no fix
needed, safe to ignore.

---

## Q2. Weighted Totals and Sample Design
*(pending)*

## Q3. Basic Descriptive Statistics
*(pending)*

## Q4. Demand Patterns by Time
*(pending)*

## Q5. Payer Mix
*(pending)*

## Q6. Chronic Conditions
*(pending)*

## Q7. Injuries and Intentionality
*(pending)*

## Q8. Diagnostic Services
*(pending)*

## Q9. Medications
*(pending)*

## Q10. Correlations
*(pending)*
