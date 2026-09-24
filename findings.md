# HW1 Findings Log — NHAMCS-ED 2015

Running notes on results and interpretation, organized by assignment question.
Update as we go; use this to draft the 3–4 page write-up and slides.

## Q1. Load and Inspect the Data

**Data source:** `~/esai_2026/data/NHAMCS/nhamcsed2015.csv` (38MB CSV). Accompanying files:
`desc.txt` (full Stata-style codebook, 1,031 variable definitions) and `nhamcsed2015.pdf`
(official NHAMCS documentation).

**Load code:**
```python
import pandas as pd
df = pd.read_csv("~/esai_2026/data/NHAMCS/nhamcsed2015.csv")
print(f"Shape: {df.shape}")
```

**Raw output:**
```
DtypeWarning: Columns (59,60,75,76,270,271,272,273,274,275,276,277,278,279,585,605,625,
645,665,685,705,725,745,765,785,805,825,845,865,885,905,925,945,965,985,1005) have mixed
types.
Shape: (21061, 1031)
```

**(a) Rows and columns:**
Shape is **21,061 rows × 1,031 columns**, matching the codebook (`desc.txt` header: "obs: 21,061,
vars: 1,031") exactly. Confirms a clean, complete load with nothing dropped or duplicated.

**Data quality note on the DtypeWarning:** we identified the 36 flagged columns by name:
```python
mixed_cols = [59,60,75,76,270,271,272,273,274,275,276,277,278,279,585,605,625,645,665,685,
              705,725,745,765,785,805,825,845,865,885,905,925,945,965,985,1005]
print(df.columns[mixed_cols].tolist())
```
Result: `CAUSE2`, `CAUSE3`, `DIAG43D`, `DIAG53D`, `HDDIAG1`–`HDDIAG5` and their coded variants
(`HDDIAG13D` etc.), and `DRUGID9`–`DRUGID30`. All of these are alphanumeric medical/diagnosis
identifier codes (not the numeric analysis variables the assignment asks about), so pandas
falling back to mixed dtype here is expected and requires no fix — **not used in this
assignment, safe to ignore.**

**(b) Missingness — code:**
```python
missing = df.isna().sum().sort_values(ascending=False)
print(missing.head(20))

key_cols = ['AGE','SEX','ETHIM','WAITTIME','LOV','PATWT','VDAYR','ARRTIME',
            'PAYTYPER','HTN','DIABTYP2','OBESITY','DEPRN','TOTCHRON',
            'INJPOISAD','INTENT15','ANYIMAGE','NUMMED']
print(df[key_cols].isna().sum())
```

**Raw output (top of `missing.head(20)`):** a set of columns named `RX##V#C#` / `RX##CAT#`
(e.g. `RX30V2C4`, `RX29V3C2`, `RX27CAT3`, `RX17V3C3`, ...) each showing **21061** missing —
i.e., **100% missing, every single row.**

**Raw output (`key_cols` missingness):**
```
AGE          0
SEX          0
ETHIM        0
WAITTIME     0
LOV          0
PATWT        0
VDAYR        0
ARRTIME      0
PAYTYPER     0
HTN          0
DIABTYP2     0
OBESITY      0
DEPRN        0
TOTCHRON     0
INJPOISAD    0
INTENT15     0
ANYIMAGE     0
NUMMED       0
```

**Interpretation for the write-up:**
- The 100%-missing columns are drug-classification sub-fields for medication slots 17, 25, and
  27–30 (`RX##V#C#`/`RX##CAT#` naming pattern — these encode drug version/category codes for
  each numbered medication a patient was given). This is *not* a normal skip pattern (like "most
  patients don't have a 30th medication, so it's blank") — it's that the entire variable was
  never populated for these higher slot numbers anywhere in the 2015 file, suggesting this
  classification scheme was either discontinued or not finalized for those slots that year.
  Since none of these specific sub-fields are used by any of the 10 assignment questions
  (Q9 uses `NUMMED` and `MED1`–`MED30`, which are separate, fully-populated columns — see below),
  this is a data-quality footnote worth one sentence in the write-up, not a blocker.
- Every one of the assignment's core analysis variables shows **zero missing values** via
  `isna()`. On its face this is good news — no rows need to be dropped for missingness on AGE,
  wait time, payer type, chronic condition flags, etc.
- **Important caveat, to state explicitly in the write-up:** a result of `isna()==0` does not
  guarantee the data is fully "clean" or meaningful. NHAMCS (like many federal surveys) commonly
  encodes "unknown / blank / not asked / illegible" using **sentinel numeric codes** (e.g., -9,
  -8, -7) rather than an actual `NaN`. A column can be "0% missing" by `isna()`'s definition
  while still containing a meaningful share of non-answers hidden as ordinary-looking numbers.
  **Sentinel-code check — code:**
```python
for col in ['AGE','WAITTIME','LOV','PATWT','TOTCHRON','NUMMED']:
    print(col)
    print(df[col].value_counts().sort_index().head(5))
    print(df[col].value_counts().sort_index().tail(5))
    print()
```

  **Confirmed against the official codebook (`desc.txt` value labels), column by column:**

  - **AGE — clean, no sentinel codes.** `0` = "Under one year" (infants), `93` = "93 years and
    over" (a top-code applied for privacy at the high end, not a missing value). Every value is
    a genuine measurement. No cleanup needed.

  - **WAITTIME — significant hidden missingness.** Codebook (`WAITTIMEF`): `-9` = *Blank*
    (3,196 visits, 15.2% of the sample), `-7` = *Not Applicable* (712 visits, 3.4%). Combined,
    **18.6% of all visits (3,908 rows) have no real recorded wait time**, disguised as ordinary
    negative integers rather than `NaN`. Computing a mean/median on this column as-is would be
    badly distorted by these large negative "values." **Must clean before Q3(c).**

  - **LOV (length of visit) — same pattern, smaller scale.** Codebook (`LOVF`): `-9` = *Blank*
    (1,480 visits, 7.0%). **Must clean before Q3(d).**

  - **PATWT (survey weight) — clean, no special codes.** No missing-value labels defined for
    this variable in the codebook — every visit has a valid, usable weight. Makes sense: the
    weight is required for every population-level estimate, so it can't be allowed to be blank.

  - **TOTCHRON — hidden missingness, with an important nuance for Q6.** Codebook value label
    is oddly stored under `TOTPROCF` (not `TOTCHRONF`) and reads: `-9` = *"'None' box and all
    item fields are blank"* (339 visits, 1.6%). This specifically means the entire
    chronic-conditions section of the survey form was left unanswered — it is **not** the same
    as the respondent legitimately having zero chronic conditions (a real, valid `0`, which
    10,662 visits correctly have). **Do not lump `-9` in with real `0`s when computing Q6** —
    treat `-9` as unknown/missing, not as "no chronic conditions."

  - **NUMMED — clean, no sentinel codes.** Values range 0–30 with no negative codes anywhere.
    `0` is a legitimate "no medications administered" answer, distinct from any missing-data
    concept.

  **Cleanup code to apply before downstream questions:**
```python
df['WAITTIME_clean'] = df['WAITTIME'].replace({-9: pd.NA, -7: pd.NA})
df['LOV_clean']       = df['LOV'].replace({-9: pd.NA})
df['TOTCHRON_clean']  = df['TOTCHRON'].replace({-9: pd.NA})
```

  **Why this matters for the write-up:** this is a concrete, quantified example of exactly the
  caveat flagged above — `isna()` reported 0 missing for all of these columns, yet WAITTIME
  and LOV in particular hide substantial (18.6% and 7.0%, respectively) real missingness behind
  sentinel codes. This is worth its own short paragraph in the write-up: a naive analyst trusting
  `isna()` at face value would silently corrupt the wait-time and length-of-visit statistics.

**(c) What each row represents:** Per the assignment itself, each row = one ED visit (not one
patient — a patient could in principle appear more than once in a national sample, though at
this sample size that's not something we've separately verified with an ID check). This is
consistent with every row having visit-specific fields populated at the row level (arrival day,
arrival time, wait time, and the visit-level survey weight `PATWT`) rather than patient-level
aggregates.

**Data quality note:** `pd.read_csv` raised a `DtypeWarning` on 36 columns — all diagnosis/drug ID
code fields (`CAUSE2/3`, `DIAG43D/53D`, `HDDIAG1-5` + coded variants, `DRUGID9`–`DRUGID30`).
These are alphanumeric medical codes, not analysis variables used in this assignment — no fix
needed, safe to ignore.

---

## Q2. Weighted Totals and Sample Design

**Code:**
```python
n_unweighted = len(df)
print(f"Unweighted visits (sample size): {n_unweighted:,}")

weighted_total = df['PATWT'].sum()
print(f"Weighted total estimated U.S. ED visits (2015): {weighted_total:,.0f}")
```

**Raw output:**
```
Unweighted visits (sample size): 21,061
Weighted total estimated U.S. ED visits (2015): 136,943,181
```

**(a) Unweighted visits:** 21,061 — matches Q1's row count exactly, since each row is one visit.

**(b) Weighted national estimate:** **136,943,181 estimated ED visits nationally in 2015** —
comfortably above the assignment's "over 100 million" check, and consistent with NHAMCS's own
published 2015 documentation (~137 million ED visits that year).

**Interpretation for write-up:** each of the 21,061 sampled visits represents, on average,
`136,943,181 / 21,061 ≈ 6,502` real-world visits nationally (i.e., the mean value of `PATWT`).
This is the core mechanic of the survey design: NHAMCS samples a manageable number of visits for
detailed clinical abstraction, then assigns each one a weight so that summing weights across the
sample reconstructs an unbiased estimate of the true national total. This is the number that
should anchor the "sample vs. population" framing for the rest of the write-up — every subsequent
weighted statistic (Q3–Q9) is built on this same logic, just applied to subsets/means instead of
a simple total.

## Q3. Basic Descriptive Statistics

**Prerequisite:** uses `WAITTIME_clean` / `LOV_clean` from the Q1 sentinel-code cleanup
(`-9`/`-7` replaced with `NaN`) — using the raw `WAITTIME`/`LOV` columns here would be wrong.

**Code:**
```python
import numpy as np

def weighted_mean(x, w):
    x = x.dropna()
    w = w.loc[x.index]
    return np.average(x, weights=w)

def weighted_median(x, w):
    x = x.dropna()
    w = w.loc[x.index]
    order = np.argsort(x.values)
    x_sorted = x.values[order]
    w_sorted = w.values[order]
    cum_w = np.cumsum(w_sorted)
    cutoff = cum_w[-1] / 2.0
    idx = np.searchsorted(cum_w, cutoff)
    return x_sorted[idx]

# (a) Age
print(f"AGE — Unweighted mean/median: {df['AGE'].mean():.2f} / {df['AGE'].median():.1f}")
print(f"AGE — Weighted   mean/median: {weighted_mean(df['AGE'], df['PATWT']):.2f} / {weighted_median(df['AGE'], df['PATWT']):.1f}")

# (b) Sex and ethnicity — unweighted vs weighted shares
print(df['SEX'].value_counts(normalize=True))                       # unweighted
print(df.groupby('SEX')['PATWT'].sum() / df['PATWT'].sum())          # weighted
print(df['ETHIM'].value_counts(normalize=True))                     # unweighted
print(df.groupby('ETHIM')['PATWT'].sum() / df['PATWT'].sum())        # weighted

# (c) Median wait time (cleaned)
print(f"Unweighted median: {df['WAITTIME_clean'].median():.1f}")
print(f"Weighted median:   {weighted_median(df['WAITTIME_clean'], df['PATWT']):.1f}")

# (d) Average length of visit (cleaned)
print(f"Unweighted mean: {df['LOV_clean'].mean():.1f}")
print(f"Weighted mean:   {weighted_mean(df['LOV_clean'], df['PATWT']):.1f}")
```

**Raw output:**
```
AGE
  Unweighted — mean: 37.56, median: 34.0
  Weighted   — mean: 37.04, median: 34.0

SEX — unweighted share:      SEX — weighted share:
1 (Female)  0.551256          1 (Female)  0.554364
2 (Male)    0.448744          2 (Male)    0.445636

ETHIM — unweighted share:                ETHIM — weighted share:
1 (Hispanic/Latino)      0.158777         1 (Hispanic/Latino)      0.164934
2 (Not Hispanic/Latino)  0.841223         2 (Not Hispanic/Latino)  0.835066

WAITTIME (cleaned)
  Unweighted median: 19.0 min
  Weighted median:   18.0 min

LOV (cleaned)
  Unweighted mean: 221.9 min (~3.70 hrs)
  Weighted mean:   213.7 min (~3.56 hrs)
```

**Codebook value labels:** `SEXF`: 1 = Female, 2 = Male. `ETHIMF`: 1 = Hispanic or Latino,
2 = Not Hispanic or Latino.

**Interpretation for write-up:**

- **Age:** Nearly identical unweighted vs. weighted (mean 37.56 → 37.04, median unchanged at
  34.0). The raw sample's age composition is already close to nationally representative;
  weighting has almost no effect here.
- **Sex:** ~55% female / ~45% male in both unweighted and weighted views — essentially unchanged
  by weighting (55.1% → 55.4% female).
- **Ethnicity — the more notable demographic shift:** Hispanic/Latino share is **15.9%
  unweighted but 16.5% weighted**. This means Hispanic-associated visits are somewhat
  *underrepresented* in the raw sample relative to their true national share; weighting
  corrects this upward by ~0.6 percentage points. Worth flagging explicitly: this is a case
  where trusting the raw sample proportion would meaningfully understate this group's true
  share of ED visits nationally.
- **Wait time:** Median wait to see a provider is ~18–19 minutes, essentially unchanged by
  weighting.
- **Length of visit — the largest weighting effect of the four stats:** unweighted mean 221.9
  min vs. weighted mean 213.7 min — an ~8 minute difference (~3.7%). This suggests the raw
  sample slightly *overrepresents* longer visits relative to the true national population; the
  weighted figure (213.7 min ≈ 3.56 hrs) is the more defensible number to report as "the"
  average ED visit length nationally.

**Key patient-flow insight for the write-up:** the median wait to *see a provider* is only
~18 minutes, but the *total* visit lasts on average ~3.5+ hours. This means the overwhelming
majority of time in an ED visit happens **after** the initial triage wait — during diagnostic
workup, treatment, and disposition — not in the waiting room. Practical implication: if a
hospital wants to reduce total ED length of stay, the front-door wait time is not the main
lever; the clinical workup/treatment phase is where the time is actually being spent.

**General weighting takeaway:** across all four measures, weighting produces only modest shifts
(a few tenths of a percentage point to ~8 minutes) — evidence the sample design is reasonably
representative overall, but the two most affected estimates (ethnicity share, length of visit)
are exactly the kind of cases that justify the assignment's insistence on reporting both
unweighted and weighted numbers rather than trusting the raw sample alone.

## Q4. Demand Patterns by Time

**Codebook confirmation:**
- `VDAYRF`: 1=Sunday, 2=Monday, 3=Tuesday, 4=Wednesday, 5=Thursday, 6=Friday, 7=Saturday.
  Clean integer code, no sentinel/missing values.
- `ARRTIME`: documented as `str4` — military time, e.g. `"1430"` = 2:30pm. **Data quality note:**
  in this CSV export the leading zero was dropped for times before 10:00am (e.g., `950` instead
  of `"0950"`), confirmed by inspecting `df['ARRTIME'].unique()`. Must re-pad with `zfill(4)`
  before extracting the hour, or times like 9:50am would be misparsed.

**Code:**
```python
df['arr_hour'] = df['ARRTIME'].astype(str).str.zfill(4).str[:2].astype(int)

day_labels = {1:'Sunday', 2:'Monday', 3:'Tuesday', 4:'Wednesday',
              5:'Thursday', 6:'Friday', 7:'Saturday'}

by_day = df.groupby('VDAYR')['PATWT'].sum().rename(index=day_labels)
print(by_day.sort_values(ascending=False))

by_hour = df.groupby('arr_hour')['PATWT'].sum()
print(by_hour.sort_index())

print("Busiest day:", by_day.idxmax(), "-", f"{by_day.max():,.0f}")
print("Busiest hour:", by_hour.idxmax(), "-", f"{by_hour.max():,.0f}")
```

**Raw output — weighted visits by day of week:**
```
Monday       22,452,845
Tuesday      20,568,287
Wednesday    19,723,821
Friday       18,784,209
Thursday     18,743,470
Saturday     18,672,207
Sunday       17,998,342
```
(Sums exactly to 136,943,181 — matches the Q2 national total, a useful internal consistency
check confirming the groupby didn't drop or double-count any visits.)

**Raw output — weighted visits by arrival hour (0–23):**
```
0   4,506,626   6   2,377,504   12  8,238,141   18  8,589,298
1   2,473,330   7   3,576,795   13  7,666,168   19  7,995,955
2   2,198,571   8   4,862,647   14  7,859,572   20  7,394,421
3   1,917,072   9   6,848,652   15  7,597,133   21  6,766,962
4   1,835,272   10  8,193,821   16  7,655,578   22  6,286,263
5   1,820,212   11  7,792,006   17  7,809,440   23  4,681,742
```
Busiest day: **Monday** (22,452,845). Busiest hour: **18:00 / 6pm** (8,589,298).

**Interpretation for write-up:**

- **Day-of-week pattern — a clear "Monday effect."** Visits decline fairly steadily from Monday
  (busiest, 22.45M) through the week to Sunday (quietest, 18.00M) — Monday sees ~25% more
  visits than Sunday. This is a well-documented phenomenon in ED utilization: demand that
  accumulates over the weekend, when primary-care offices are closed, spills into the ED first
  thing Monday rather than being addressed by outpatient care over the weekend itself.
- **Hour-of-day pattern — a sustained plateau, not a narrow rush hour.** There's a deep
  overnight trough from roughly 2am–5am (~1.8–2.2M visits/hour), then a rise through the morning
  into a **long, high, relatively flat plateau from about 9am to 9pm** (consistently 7.6–8.6M
  per hour) — the day's peak (18:00, 8.59M) is only modestly higher than the plateau's other
  hours, not a sharp spike. Demand then declines steadily overnight.
- **Staffing/capacity planning implications (this is the assignment's explicit ask):**
  1. **Day-level:** staffing levels should scale down toward the weekend and scale up for
     Monday specifically — a roughly 25% swing in expected volume between the busiest and
     quietest day is large enough to justify differentiated weekly staffing rather than a flat
     schedule.
  2. **Hour-level:** because the high-demand window spans ~12 hours (9am–9pm) rather than a
     short rush, hospitals likely need a broad daytime/evening shift covering most of the
     waking day, with a legitimate opportunity to run substantially leaner overnight
     (2am–5am demand is roughly a quarter of the daytime plateau).

## Q5. Payer Mix

**Codebook confirmation — `PAYTYPERF`:** `-9` = All sources of payment blank, `-8` = Unknown,
`1` = Private insurance, `2` = Medicare, `3` = Medicaid or CHIP, `4` = Worker's compensation,
`5` = Self-pay, `6` = No charge/Charity, `7` = Other.

**Code:**
```python
pay_labels = {-9: 'Blank/Missing', -8: 'Unknown', 1: 'Private insurance', 2: 'Medicare',
              3: 'Medicaid/CHIP', 4: "Worker's comp", 5: 'Self-pay', 6: 'No charge/Charity',
              7: 'Other'}

weighted_pay = (df.groupby('PAYTYPER')['PATWT'].sum() / df['PATWT'].sum()) \
                 .rename(index=pay_labels).sort_values(ascending=False)
print(weighted_pay)

for label in ['Private insurance', 'Medicare', 'Medicaid/CHIP', 'Self-pay']:
    print(f"{label}: {weighted_pay[label]:.1%}")
```

**Raw output — weighted share of visits by payer:**
```
Medicaid/CHIP        31.2%
Private insurance    27.6%
Medicare             17.7%
Self-pay              9.0%
Unknown               8.3%
Blank/Missing         2.5%
Other                 2.2%
No charge/Charity     0.8%
Worker's comp         0.7%
```

**(a)/(b) The assignment's specific four-way comparison:**
- Private insurance: **27.6%**
- Medicare: **17.7%**
- Medicaid/CHIP: **31.2%**
- Self-pay: **9.0%**

**Interpretation for write-up:**

- **Headline finding: Medicaid/CHIP is the single largest payer category for ED visits
  nationally (31.2%)** — larger than private insurance (27.6%) and far larger than Medicare
  (17.7%). This matters because Medicaid is well documented to reimburse hospitals at
  substantially lower rates than private insurance (often near or below the actual cost of
  providing care), while private insurance is generally the most favorably-reimbursing payer.
- **Data quality caveat:** `Unknown` (8.3%) + `Blank/Missing` (2.5%) = **10.8% of visits have
  no reliably determined payer type** — roughly 1 in 9 visits. Worth stating as a limit on
  precision for the exact percentages, though the relative ranking among the known categories
  is still informative.
- **Financial performance implication (this is the assignment's explicit ask):** grouping by
  reimbursement favorability — Private insurance (27.6%, best-reimbursing) vs. everything else
  (Medicaid 31.2%, Medicare 17.7% moderate, Self-pay 9.0%, No charge/Charity 0.8%, Worker's comp
  0.7%, Other 2.2% — all lower or uncertain reimbursement) — **fewer than 3 in 10 ED visits are
  covered by the most favorably-reimbursing payer type.** Self-pay + No charge/Charity alone
  (9.8% combined) represents visits with minimal-to-no reimbursement at all — direct
  uncompensated-care exposure. Combined with Medicaid's below-market reimbursement rates, this
  payer mix implies EDs operate under real, structural financial pressure: the *majority* of
  visit volume comes from payer types that reimburse below what private insurance would pay for
  the same service.

## Q6. Chronic Conditions

**Codebook confirmation:** `HTN`, `DIABTYP2`, `OBESITY`, `DEPRN`, and 17 other individual
condition flags are all clean binary `NOYESF` fields (0=No, 1=Yes), no sentinel/missing values.
The assignment's four named columns are only examples — the codebook lists a **full set of 21
chronic-condition flags** (plus a `NOCHRON`/"none of the above" indicator) that together
determine `TOTCHRON`:
`ETOHAB` (alcohol misuse), `ALZHD` (Alzheimer's), `ASTHMA`, `CANCER`, `CEBVD` (stroke/TIA),
`CKD` (chronic kidney disease), `COPD`, `CHF` (congestive heart failure), `CAD` (coronary artery
disease), `DEPRN` (depression), `DIABTYP1`/`DIABTYP2`/`DIABTYP0` (diabetes, by type),
`ESRD` (end-stage renal disease), `HPE` (history of PE/DVT), `EDHIV` (HIV), `HYPLIPID`
(hyperlipidemia), `HTN` (hypertension), `OBESITY`, `OSA` (sleep apnea), `OSTPRSIS`
(osteoporosis), `SUBSTAB` (substance abuse). Used the full list rather than just the 4 named in
the prompt, since limiting to 4 would understate/misidentify true prevalence.

**Code:**
```python
chronic_cols = ['ETOHAB','ALZHD','ASTHMA','CANCER','CEBVD','CKD','COPD','CHF','CAD','DEPRN',
                'DIABTYP1','DIABTYP2','DIABTYP0','ESRD','HPE','EDHIV','HYPLIPID','HTN','OBESITY',
                'OSA','OSTPRSIS','SUBSTAB']

# (a) Weighted share of visits with at least one chronic condition
has_chronic = (df[chronic_cols] == 1).any(axis=1)
weighted_share_chronic = (has_chronic * df['PATWT']).sum() / df['PATWT'].sum()
print(f"Weighted share with >=1 chronic condition: {weighted_share_chronic:.1%}")

# Cross-check against TOTCHRON_clean > 0 (excludes the 339 "blank section" rows from Q1)
df['TOTCHRON_clean'] = df['TOTCHRON'].replace({-9: pd.NA})
mask = df['TOTCHRON_clean'].notna()
cross_check = ((df.loc[mask, 'TOTCHRON_clean'] > 0) * df.loc[mask, 'PATWT']).sum() / df.loc[mask, 'PATWT'].sum()
print(f"Cross-check via TOTCHRON_clean > 0: {cross_check:.1%}")

# (b) Most prevalent individual chronic condition (weighted)
prevalence = {}
for col in chronic_cols:
    prevalence[col] = (df[col] == 1).mul(df['PATWT']).sum() / df['PATWT'].sum()
prevalence = pd.Series(prevalence).sort_values(ascending=False)
print(prevalence.head(10))
```

**Raw output:**
```
Weighted share with >=1 chronic condition: 47.0%
Cross-check via TOTCHRON_clean > 0:        47.6%

HTN         23.6%
ASTHMA       9.8%
DEPRN        9.3%
HYPLIPID     8.1%
SUBSTAB      6.6%
CAD          6.0%
DIABTYP0     5.7%
COPD         5.3%
DIABTYP2     4.6%
OBESITY      3.6%
```

**Interpretation for write-up:**

- **Validation:** two independent methods (summing across 21 individual condition flags vs.
  `TOTCHRON_clean > 0`) landed within 0.6 percentage points of each other (47.0% vs. 47.6%) —
  good internal consistency, giving confidence in the result. The small residual gap likely
  reflects how the 339 "entire section blank" rows (excluded from the `TOTCHRON` cross-check
  but implicitly counted as "no condition" when summing individual flags, since those same rows
  also have all 21 flags reading 0) are handled slightly differently between the two methods.
- **Nearly half of all ED visits (47.0–47.6%) involve at least one chronic condition.** EDs are
  clearly not handling a population of purely acute, otherwise-healthy patients — a substantial
  share of ED volume carries an underlying chronic disease burden.
- **Hypertension is by far the single most common chronic condition — present in ~23.6% of all
  ED visits (roughly 1 in 4)**, more than double the next most common condition (Asthma, 9.8%).
- **Important distinction for the discussion:** hypertension is typically a *background
  comorbidity*, not the acute reason for the ED visit itself — unlike, say, Asthma, where an
  exacerbation is often the actual presenting complaint. This means the high HTN prevalence
  reflects the general disease burden of the ED-using population, not necessarily HTN-related
  emergencies specifically.
- **Role of EDs in chronic disease management (the assignment's explicit question):** with
  ~47% of visits touching at least one chronic condition and hypertension alone appearing in a
  quarter of all visits, EDs function as a major touchpoint for a chronic-disease-burdened
  population regardless of the actual presenting complaint. Practical implications: EDs likely
  need staffing/protocols equipped for chronic-disease complications and medication
  reconciliation as routine, not exceptional, parts of care — and this pattern, combined with
  the Q4 "Monday effect" (demand spike after a weekend without primary-care access), suggests
  the ED may be functioning partly as a substitute touchpoint for populations with inadequate
  access to ongoing primary/preventive care.

## Q7. Injuries and Intentionality

**Codebook confirmation:**
- `INJPOISADF`: `-9`=Blank, `-8`=Unknown, `1`=Yes, injury/trauma, `2`=Yes, overdose/poisoning,
  `3`=Yes, adverse effect of medical/surgical treatment, `4`=No, not related, `5`=Questionable
  injury status.
- `INTENT15F`: `-9`=Blank, `-8`=Unknown/intent unclear, `1`=Intentional, `2`=Unintentional,
  `4`=Questionable injury status.

**Code — part (a):**
```python
injpois_labels = {-9: 'Blank', -8: 'Unknown', 1: 'Injury/trauma', 2: 'Overdose/poisoning',
                   3: 'Adverse effect of medical/surgical tx', 4: 'Not related', 5: 'Questionable'}

weighted_injpois = (df.groupby('INJPOISAD')['PATWT'].sum() / df['PATWT'].sum()) \
                     .rename(index=injpois_labels).sort_values(ascending=False)
print(weighted_injpois)

frac_injury_related = weighted_injpois[['Injury/trauma', 'Overdose/poisoning',
                                          'Adverse effect of medical/surgical tx']].sum()
print(f"(a) Weighted share related to injury/overdose/adverse effect: {frac_injury_related:.1%}")
```

**Raw output — part (a):**
```
Not related                              62.7%
Injury/trauma                            29.5%
Unknown                                   3.7%
Adverse effect of medical/surgical tx     2.2%
Overdose/poisoning                        1.4%
Questionable                              0.3%
Blank                                     0.2%

(a) Weighted share related to injury/overdose/adverse effect: 33.1%
```

**(a) Answer: 33.1% of all ED visits nationally are related to injury/trauma, overdose/poisoning,
or an adverse effect of medical/surgical treatment.**

---

**Part (b) — initial (naive) attempt and why it needed correction:**

First pass — restricting to the three "yes" categories together (`INJPOISAD` in {1,2,3}) and
tabulating `INTENT15`:
```
Unintentional      68.4%
Blank               25.7%   <-- suspiciously large, investigated further
Intentional          5.4%
Unknown/unclear      0.5%
```

The 25.7% blank rate looked too large to be a simple data gap, so we cross-tabbed `INTENT15`
*within* each `INJPOISAD` category to check whether it was concentrated in one subgroup:
```python
crosstab = pd.crosstab(df['INJPOISAD'].map(injpois_labels), df['INTENT15'], normalize='index')
print(crosstab)
```

**Result — this revealed a methodological issue, not a data-quality issue:** `INTENT15` is
**100% blank, by design, for every `INJPOISAD` category except `Injury/trauma` and
`Overdose/poisoning`.** Specifically: `Adverse effect of medical/surgical tx` → 100% blank,
`Not related` → 100% blank, `Blank`/`Unknown` INJPOISAD → 100% blank on intent too. This makes
complete sense: an adverse effect of medical treatment is a treatment complication, not a
behavioral act that can be classified as "intentional" or "unintentional" — the survey simply
doesn't ask an intent question for that category. **Including `Adverse effect` in the intent
breakdown was the actual source of the inflated blank rate, not missing data.**

Within the two categories where intent is actually asked:
```
                        Blank    Unknown   Intentional   Unintentional  Questionable
Injury/trauma           22.0%     0.5%        5.6%          71.9%          0.0%
Overdose/poisoning       1.8%     0.0%       12.2%          86.0%          0.0%
```

**Corrected code — restrict to only the categories where intent is applicable:**
```python
mask_intent_applicable = df['INJPOISAD'].isin([1, 2])  # Injury/trauma, Overdose/poisoning only
intent_labels = {-9: 'Blank', -8: 'Unknown/unclear', 1: 'Intentional', 2: 'Unintentional', 4: 'Questionable'}
weighted_intent_clean = (df.loc[mask_intent_applicable].groupby('INTENT15')['PATWT'].sum()
                         / df.loc[mask_intent_applicable, 'PATWT'].sum()) \
                          .rename(index=intent_labels).sort_values(ascending=False)
print(weighted_intent_clean)
```

**Final raw output:**
```
Unintentional      73.3%
Blank              20.4%
Intentional          5.8%
Unknown/unclear      0.5%
```

**(b) Answer:** among injury/trauma and overdose/poisoning visits combined (the only categories
where intent is coded), **73.3% are unintentional and 5.8% are intentional**, with 20.4%/0.5%
remaining unclassified (blank/unknown). Restricting further to only the definitively-classified
cases (excluding the residual blank/unknown): **intentional ≈ 7.3%, unintentional ≈ 92.7%.**

**Interpretation for write-up:**

- **Methodological point worth stating explicitly:** a naive analysis that includes "adverse
  effect of medical/surgical treatment" when computing an intentional-vs-unintentional
  breakdown will produce a misleadingly high "blank" rate (25.7%) that looks like a data
  problem but is actually a structural/skip-pattern issue — that category isn't eligible for
  an intent classification at all. This is a good example of why checking a crosstab against a
  second variable, rather than trusting an aggregate breakdown at face value, matters.
- **Substantive finding — overdose/poisoning skews notably more intentional than general
  trauma.** Among classified cases, overdose/poisoning visits are ~12.2% intentional vs. only
  ~5.6% for injury/trauma — consistent with overdose/poisoning encompassing a meaningful share
  of self-harm/suicide-attempt presentations, a pattern not generally present in general
  physical trauma.
- **Overall:** roughly 1 in 3 ED visits (33.1%) involve injury, overdose, or an adverse
  treatment effect, and the large majority of the classifiable injury/overdose cases are
  unintentional (~92.7% among definitively classified cases) — consistent with most ED injury
  presentations being accidents rather than deliberate self-harm, though the meaningfully higher
  intentional share within overdose/poisoning specifically is a distinct, worth-flagging
  sub-pattern (relevant to behavioral health / substance-use screening resource allocation).

## Q8. Diagnostic Services

**Codebook — full list of blood-test flags (all clean binary `NOYESF` 0/1 fields):** the
assignment names `CBC`, `GLUCOSE`, `BUNCREAT`, `ELECTROL` as examples; the codebook has 16
total blood-test flags: `ABG` (arterial blood gases), `BAC` (blood alcohol), `BMP` (basic
metabolic panel), `BLOODCX` (blood culture), `BNP` (brain natriuretic peptide), `BUNCREAT`
(BUN/creatinine), `CARDENZ` (cardiac enzymes), `CBC`, `CMP` (comprehensive metabolic panel),
`DDIMER`, `ELECTROL`, `GLUCOSE`, `LACTATE`, `LFT` (liver function tests), `PTTINR`
(prothrombin time/INR), `OTHERBLD` (other blood test, catch-all). `ANYIMAGE` is a separate
single flag summarizing whether any imaging (X-ray, CT, MRI, ultrasound, other) was done.

**Code:**
```python
# (a) Any imaging
weighted_anyimage = (df['ANYIMAGE'] == 1).mul(df['PATWT']).sum() / df['PATWT'].sum()
print(f"(a) Weighted share with any imaging: {weighted_anyimage:.1%}")

# (b) Among visits with blood tests, which were most frequently ordered
blood_test_cols = ['ABG','BAC','BMP','BLOODCX','BNP','BUNCREAT','CARDENZ','CBC','CMP','DDIMER',
                    'ELECTROL','GLUCOSE','LACTATE','LFT','PTTINR','OTHERBLD']

has_blood_test = (df[blood_test_cols] == 1).any(axis=1)
weighted_share_blood = (has_blood_test * df['PATWT']).sum() / df['PATWT'].sum()
print(f"Weighted share of visits with >=1 blood test: {weighted_share_blood:.1%}")

blood_test_freq = {}
denom = (has_blood_test * df['PATWT']).sum()
for col in blood_test_cols:
    blood_test_freq[col] = ((df[col] == 1) & has_blood_test).mul(df['PATWT']).sum() / denom
blood_test_freq = pd.Series(blood_test_freq).sort_values(ascending=False)
print(blood_test_freq)
```

**Raw output:**
```
(a) Weighted share with any imaging: 47.0%
Weighted share of visits with >=1 blood test: 42.4%

Among visits with >=1 blood test, share that included each specific test:
CBC         85.4%
CMP         55.3%
OTHERBLD    46.1%
BMP         25.2%
GLUCOSE     19.2%
PTTINR      17.8%
BUNCREAT    15.7%
CARDENZ      9.8%
LFT          9.7%
ELECTROL     7.7%
BLOODCX      7.0%
BNP          5.9%
DDIMER       5.6%
BAC          4.0%
ABG          3.7%
LACTATE      3.1%
```

**(a) Answer:** 47.0% of all ED visits nationally include some form of imaging — nearly half.

**(b) Answer:** 42.4% of all visits include at least one blood test; among those,
**CBC (Complete Blood Count) is by far the most frequently ordered specific test, present in
85.4% of blood-test visits**, followed by CMP (55.3%) and the "Other blood test" catch-all
(46.1%). All remaining specific tests fall below 26%, with most in the single digits.

**Interpretation for write-up:**

- **CBC and CMP function as the two "workhorse" baseline panels** — ordered in the large
  majority of blood-test visits regardless of the presenting complaint. This is consistent with
  standard ED practice: run a broad baseline hematologic (CBC) and metabolic (CMP) screen on
  nearly any patient getting labs at all, independent of the specific reason for the visit.
- **The remaining tests are clearly targeted/condition-specific, not routine.** Cardiac enzymes
  (9.8%, suspected MI/ACS), D-dimer (5.6%, suspected PE/DVT), BAC (4.0%, suspected intoxication),
  ABG (3.7%, respiratory failure workup), and lactate (3.1%, suspected sepsis/shock) are each
  ordered in only a small minority of blood-test visits — consistent with these being ordered
  only when a specific clinical suspicion warrants them, rather than as part of a routine panel.
- **Combined resource-utilization picture:** imaging (47.0% of all visits) and blood testing
  (42.4% of all visits) together touch the large majority of ED visits nationally. This reinforces
  a broader theme (carried further in Q9's medication analysis): the typical ED visit is not a
  minimal, low-resource encounter — diagnostic workup of some kind is the norm rather than the
  exception, which has direct implications for staffing (lab/radiology throughput) and cost
  structure discussions.

## Q9. Medications

**Prerequisite:** `NUMMED` already confirmed clean in Q1 (0–30 range, no sentinel/missing
codes) — no cleanup needed before use here.

**Code:**
```python
mean_nummed = weighted_mean(df['NUMMED'], df['PATWT'])
median_nummed = weighted_median(df['NUMMED'], df['PATWT'])
print(f"Weighted mean NUMMED: {mean_nummed:.2f}")
print(f"Weighted median NUMMED: {median_nummed:.1f}")
print(f"(for comparison) Unweighted mean/median: {df['NUMMED'].mean():.2f} / {df['NUMMED'].median():.1f}")

def med_bucket(n):
    if n == 0:
        return '0 medications'
    elif n <= 2:
        return '1-2 medications'
    else:
        return '3+ medications'

df['med_bucket'] = df['NUMMED'].apply(med_bucket)
weighted_bucket = df.groupby('med_bucket')['PATWT'].sum() / df['PATWT'].sum()
print(weighted_bucket.sort_values(ascending=False))
```

**Raw output:**
```
Weighted mean NUMMED: 2.49
Weighted median NUMMED: 2.0
(for comparison) Unweighted mean/median: 2.54 / 2.0

1-2 medications    41.9%
3+ medications     37.2%
0 medications      20.9%
```

**(a) Answer:** weighted mean = 2.49 medications per visit, weighted median = 2.0. Nearly
identical to the unweighted figures (2.54 / 2.0) — weighting has minimal effect here, similar
to what was observed for `AGE` in Q3.

**(b) Answer:** 0 medications = 20.9%, 1–2 medications = 41.9%, 3+ medications = 37.2%.

**Interpretation for write-up:**

- **79.1% of all ED visits involve at least one medication** (100% − 20.9% with zero).
- **37.2% of visits involve 3 or more medications** — a substantial share receiving fairly
  intensive pharmacological treatment, not just a single dose of something simple.
- **This directly reinforces the Q8 diagnostic-services finding.** Combined picture across both
  questions: 47.0% of visits get imaging, 42.4% get blood tests, and 79.1% get at least one
  medication, with over a third getting 3+. The typical ED visit is not a quick, low-intensity
  encounter — it's a moderately-to-highly resource-intensive one involving real diagnostic
  workup and active treatment for the large majority of patients.
- **Ties together the write-up's overall staffing/operations narrative:** combined with Q4
  (a long, sustained daily high-demand window rather than a brief rush) and Q6 (nearly half of
  visits carry a chronic disease burden), this suggests EDs need to be staffed and resourced for
  sustained, moderate-to-high complexity care across the bulk of their volume — not a model
  built around a small number of severe cases and a majority of quick, simple ones.

## Q10. Correlations

**Codebook confirmation:** `DOA` (dead on arrival) and `DIEDED` (died in ED) are simple binary
`NOYESF` flags. `HDSTAT` (hospital discharge status): `-9`=Blank, `-8`=Unknown, `-7`=Not
applicable, `1`=Alive, `2`=Dead — recoded here as `died_ed = (HDSTAT==2)`.

**Bug encountered and fixed:** the Q1 cleanup of `WAITTIME`/`LOV`/`TOTCHRON` originally used
`.replace({-9: pd.NA})`, which silently converts the column to generic `object` dtype (mixed
int/`pd.NA`). This worked fine for the weighted mean/median functions (which call `.dropna()`
before doing math), but broke `.corr()`, which tries to cast the whole column to `float` first
and can't convert `pd.NA` directly — `TypeError: float() argument must be a string or a number,
not 'NAType'`. **Fix: re-clean using `np.nan` instead of `pd.NA`**, which integrates natively
with `float64`. Worth remembering for any future numeric work with these three columns.

**Code:**
```python
import numpy as np

df['WAITTIME_clean']  = df['WAITTIME'].replace({-9: np.nan, -7: np.nan})
df['LOV_clean']       = df['LOV'].replace({-9: np.nan})
df['TOTCHRON_clean']  = df['TOTCHRON'].replace({-9: np.nan})

blood_test_cols = ['ABG','BAC','BMP','BLOODCX','BNP','BUNCREAT','CARDENZ','CBC','CMP','DDIMER',
                    'ELECTROL','GLUCOSE','LACTATE','LFT','PTTINR','OTHERBLD']
df['has_blood_test'] = (df[blood_test_cols] == 1).any(axis=1).astype(int)
df['died_ed'] = (df['HDSTAT'] == 2).astype(int)

corr_vars = ['AGE', 'TOTCHRON_clean', 'NUMMED', 'WAITTIME_clean', 'LOV_clean', 'ANYIMAGE',
             'has_blood_test', 'DOA', 'DIEDED', 'died_ed']

corr_matrix = df[corr_vars].corr()
print(corr_matrix.round(2))
```

**Raw output (Pearson correlation matrix, unweighted):**
```
                 AGE  TOTCHRON  NUMMED  WAITTIME  LOV   ANYIMAGE  bloodtest  DOA   DIEDED  died_ed
AGE             1.00     0.54    0.19     -0.01   0.13    0.25      0.34    0.01   0.03    0.06
TOTCHRON_clean  0.54     1.00    0.23     -0.01   0.15    0.19      0.32   -0.00   0.01    0.05
NUMMED          0.19     0.23    1.00      0.01   0.19    0.19      0.29   -0.01   0.01    0.05
WAITTIME_clean -0.01    -0.01    0.01      1.00   0.23   -0.01      0.02   -0.00  -0.01   -0.02
LOV_clean       0.13     0.15    0.19      0.23   1.00    0.12      0.26   -0.01  -0.00    0.01
ANYIMAGE        0.25     0.19    0.19     -0.01   0.12    1.00      0.31   -0.01   0.01    0.04
has_blood_test  0.34     0.32    0.29      0.02   0.26    0.31      1.00   -0.00   0.01    0.05
DOA             0.01    -0.00   -0.01     -0.00  -0.01   -0.01     -0.00    1.00  -0.00   -0.00
DIEDED          0.03     0.01    0.01     -0.01  -0.00    0.01      0.01   -0.00   1.00   -0.00
died_ed         0.06     0.05    0.05     -0.02   0.01    0.04      0.05   -0.00  -0.00    1.00
```

**Interpretation for write-up:**

- **Strongest relationship in the matrix: `AGE` ↔ `TOTCHRON` = 0.54.** Age is a far stronger
  predictor of chronic disease burden than of anything else measured — much higher than age's
  correlation with treatment intensity (`NUMMED` 0.19) or visit length (`LOV` 0.13). Directly
  quantifies the Q6 finding that chronic disease accumulates with age.
- **A clear "treatment/diagnostic intensity" cluster:** `TOTCHRON`, `NUMMED`, `ANYIMAGE`,
  `has_blood_test`, and `LOV` are all mutually positively correlated (roughly 0.12–0.32) —
  patients with more chronic conditions receive more medications, more imaging, more blood
  tests, and stay longer, all moving together as expected clinically. `ANYIMAGE` ↔
  `has_blood_test` (0.31) confirms the two diagnostic categories cluster together too — sicker/
  more complex patients tend to get both, simpler patients tend to get neither.
- **Counterintuitive, management-relevant finding: `WAITTIME_clean` is essentially uncorrelated
  with everything except `LOV_clean` (0.23).** Age (−0.01), chronic conditions (−0.01),
  medications (0.01), imaging (−0.01) — none of these predict the initial wait to be seen. This
  suggests the front-door wait is driven primarily by **ED congestion at time of arrival**
  (consistent with the Q4 demand-timing findings — busy hours/days mean everyone waits longer,
  regardless of how sick they are), not by individual patient severity. Patient complexity does
  not appear to "jump the queue" in this data.
- **Mortality indicators (`DOA`, `DIEDED`, `died_ed`) show near-zero correlations throughout
  (−0.02 to 0.06).** This is expected, not a null result to dismiss: death in the ED is a rare
  outcome, and rare binary outcomes mechanically cap how large a Pearson correlation can get
  regardless of the true underlying relationship (low variance limits achievable correlation
  magnitude). That said, the direction and relative ranking are informative — `died_ed`
  correlates most strongly (in relative terms) with `AGE` (0.06) and `TOTCHRON`/`NUMMED` (0.05
  each), directionally consistent with older, sicker patients facing a modestly elevated
  in-ED mortality risk.
- **Ties the whole write-up together:** age drives chronic disease burden, which drives
  treatment/diagnostic intensity, which drives visit length — but none of this drives the
  *initial* wait time, which instead tracks ED congestion (Q4's demand patterns). This gives a
  coherent causal narrative for the whole assignment: demand timing (Q4) determines how long you
  wait; patient characteristics (age, chronic disease from Q6) determine how intensively you're
  treated once seen (Q8/Q9); and none of the individual patient factors meaningfully predict
  queue position at the front door.

---

**All 10 assignment questions now logged.** Next steps: draft the 3–4 page written summary and
≤5 slides from this log, and push the finalized `findings.md` (plus the analysis notebook) to
the shared repo for the team.
