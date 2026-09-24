# HW1 Findings Log — NHAMCS-ED 2015

Running notes on results and interpretation, organized by assignment question.
Update as we go; use this to draft the 3–4 page write-up and slides.

## Q1. Load and Inspect the Data

**(a) Rows and columns:**
- Shape: **21,061 rows × 1,031 columns** — matches the codebook (`desc.txt`) exactly.
- Each row = one ED visit (per assignment; to be confirmed with an ID-style column check).

**(b) Missingness:** *(pending — running `df.isna().sum()` next)*

**(c) What each row represents:** *(pending confirmation)*

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
