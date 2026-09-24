# ESAI HW1 — Exploring Hospital Emergency Department Data

MGT 634 (Empirical Strategy with AI), Homework #1. Group project — see `MGT_634_HW1.pdf` for the
full assignment prompt.

**Due:** Tuesday, Sep 29, at the beginning of class.

## Overview

We're working with the 2015 National Hospital Ambulatory Medical Care Survey (NHAMCS),
Emergency Department component — a nationally representative sample of US ED visits
(21,061 rows = visits, 1,031 columns). The assignment has 10 parts, using the survey weight
`PATWT` to scale sample statistics up to national estimates, and interpreting results through a
hospital-management/operations lens (staffing, capacity planning, financial performance,
patient complexity).

## Repo contents

| File | What it is |
|---|---|
| `MGT_634_HW1.pdf` | The assignment prompt (10 questions) |
| `nhamcsed2015.pdf` | Official NHAMCS documentation |
| `desc.txt` | Full codebook — every one of the 1,031 variables, with value labels (Stata-style) |
| `20261001_hw1.ipynb` | Analysis notebook — all 10 questions, code + output |
| `findings.md` | **Start here.** Detailed findings log: code, raw output, and interpretation for every question, written so it's understandable without re-running anything |

## Data location

The actual data file (`nhamcsed2015.csv`, 38MB) is **not** in this repo — it lives on the Yale
SOM HPC:
```
~/esai_2026/data/NHAMCS/nhamcsed2015.csv
```
To reproduce/extend the analysis, open `20261001_hw1.ipynb` in Jupyter via Open OnDemand on the
HPC (see Session 3 notes if you need a refresher on getting set up there).

## Known data-cleaning gotchas (see `findings.md` for full detail)

- `WAITTIME`, `LOV`, `TOTCHRON`, and `PAYTYPER` all have **sentinel missing-value codes**
  (e.g., `-9` = Blank, `-7` = Not Applicable) that `df.isna()` will **not** catch — these must be
  replaced with `np.nan` before computing means/medians. Use `np.nan`, not `pd.NA`, when
  cleaning — `pd.NA` breaks `.corr()` (see `findings.md` Q10 for the exact error and fix).
- `ARRTIME` is stored as a number in this CSV, not a zero-padded 4-digit string as the codebook
  describes — re-pad with `.astype(str).str.zfill(4)` before extracting the hour, or times
  before 10am will be misparsed.
- Several chronic-condition-related columns (`RX##V#C#`/`RX##CAT#` medication sub-fields) are
  100% missing across all rows — not a bug, just unpopulated in this file. Not used by any
  assignment question.

## Deliverables (per syllabus + assignment prompt)

1. **3–4 page written report** (standard formatting, minimize bullet points) — draft from
   `findings.md`.
2. **Up to 5 slides** summarizing approach and key findings.
3. **Code/notebook** — `20261001_hw1.ipynb`, kept clean and commented.

## Working in this repo

- Pull before you push: `git pull` before `git push`, since we're all committing to `main`.
- Notebooks (`.ipynb`) are hard to merge if two people edit the same one at once — coordinate
  before editing `20261001_hw1.ipynb` directly, or work in a copy/separate cells.
