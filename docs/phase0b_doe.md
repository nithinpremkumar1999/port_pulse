# Phase 0B — DOE validation target

Exploration only. **Do not write pipeline code in this phase.** Nothing here
belongs in `src/`. Work in `scratch/`, report results, stop.

Goal: establish whether the DOE data can support D-140 (per-cargo matching on
tanker name and departure date) across the whole 2020–2026 window, and
resolve D-053 (volume estimation) if the data allows.

**Why this runs before the AIS backfill.** D-004 commits to per-cargo
precision, recall and volume error. Every one of those depends on DOE
publishing cargo-level detail with tanker names for the study period. That
assumption has not been tested. It costs an afternoon to test and it costs
18 GB of wasted backfill to discover late.

---

## Setup

Two things are already known and do not need discovering.

**The historical file**, covering Jan 2016 – Dec 2023 at transaction level:

```
https://www.energy.gov/sites/default/files/2024-02/3.%20U.S.%20LNG%20Exports%20and%20Re-Exports%20Details%20%28Jan%202016%20-%20Dec%202023%29.xlsx
```

**Its source page**, which lists the sibling files:

```
https://www.energy.gov/hgeo/articles/natural-gas-imports-and-exports-monthly-2023
```

DOE states the transaction-detail attributes are: arrival / departure date,
company name, docket number, docket term, activity, gas type, mode of
transport, supplier, tanker, point of entry/exit, country of destination, and
volume (MMCF). Verify this against the actual file rather than trusting it.

Work in `scratch/doe/`. Use `pandas` with `openpyxl`.

---

## D0 — Inventory: does 2024–2026 detail exist?

The known file stops at December 2023. The study window runs to 2026.

Find the equivalent transaction-detail files for 2024, 2025 and 2026. Start
from the 2023 page above and from
`https://www.energy.gov/fecm/listings/lng-reports`. The site appears to
publish one article page per year with the Excel files attached.

Report: for each year 2020–2026, whether a transaction-detail file exists,
its URL, and its date coverage. **Do not construct URLs by pattern-matching
the 2023 one** — find them from a page that links them, and say so if a year
cannot be found.

**Decides:** whether D-140 holds for the full window or only part of it.

---

## D1 — Schema of the transaction-detail file

```python
import pandas as pd
xl = pd.ExcelFile('scratch/doe/lng_details_2016_2023.xlsx')
print(xl.sheet_names)
for s in xl.sheet_names:
    df = xl.parse(s, nrows=200)
    print(s, df.shape, list(df.columns))
```

Then for the sheet holding export transactions: full row count, dtypes, and
the first 20 rows printed in full.

Answer specifically:

- Is "arrival / departure date" **one column or two**? Exports need the
  departure date (D-140). If it is one column whose meaning depends on
  `activity`, say so — that is a trap.
- What is the date column's type and format? Any nulls?
- Is `volume` in MMCF as documented?
- Is there a split-cargo flag, and how is it encoded? DOE documentation
  describes a `[*]` marker.

**Decides:** the DOE loader's schema contract, and whether D-140's join key
exists as assumed.

---

## D2 — How is Sabine Pass spelled, and how many cargoes in Q1 2022?

```python
df['point_of_exit'].value_counts()
```

Report **every** distinct point-of-exit value with a count. Do not filter to
what looks like Sabine Pass — the naming convention is unknown, terminals may
appear under more than one string, and near-misses matter.

Then, for the value(s) identifying Sabine Pass, count export cargoes per
month for 2022, and print the Q1 2022 rows in full.

**Decides:** the Phase 1 validation denominator. The 90-day fixture should
contain approximately this many loading events, and any large disagreement in
Phase 2 is a detector problem to investigate rather than a surprise.

---

## D3 — Tanker name quality

D-140 makes `vessel_name` the sole join key, and D-045 flags it as fragile on
the AIS side. Characterise the DOE side.

```python
t = df['tanker']
print(t.isna().sum(), t.nunique(), len(t))
print(t.dropna().unique()[:100])
```

Report: null/blank rate, distinct count, and anything that would break an
exact string join — case inconsistency, double spaces, punctuation, trailing
whitespace, prefixes like "LNG/C" or "M/V", or the same vessel appearing
under two spellings.

**Decides:** how aggressive D-140's fuzzy matching has to be. AIS-side
evidence already exists: OBS-PHASE0 observed `ADVENTURE OFTHE SEAS` with a
missing space.

---

## D4 — The three D-044 exclusions

For each, report how it is encoded and how many rows it removes for
Sabine Pass across 2020–2023:

1. **Split cargoes.** Find the flag. Then count how often
   (tanker, departure date, point of exit) appears more than once — that is
   the real duplication rate, whether or not the flag is reliable.
2. **ISO container exports.** Check `mode_of_transport` values.
3. **Re-exports.** Check `activity` values.

**Decides:** the D-044 exclusion logic, and the size of the recall
correction. Uncorrected split cargoes look like missed detections.

---

## D5 — Does cargo volume cluster by vessel? *(resolves D-053)*

D-053 is open: can volume be estimated without draught?

```python
g = (df[df['activity'] == <export>]
       .groupby('tanker')['volume_mmcf']
       .agg(['count', 'mean', 'std', 'min', 'max']))
g['cv'] = g['std'] / g['mean']
print(g[g['count'] >= 5].sort_values('cv').to_string())
```

Then the same grouped by vessel size class if any size proxy is available in
the file, and a histogram of volume across all export cargoes.

**Interpretation.** A low within-vessel coefficient of variation (say under
0.1) means a given carrier ships a near-constant volume, and per-class
capacity constants become empirically defensible — D-053 option (a). A high
CV means volume varies per voyage and cannot be inferred from vessel
identity — D-053 option (d), report counts only.

**Decides:** D-053, with no AIS required.

---

## D6 — Coverage of the other three terminals

Repeat D2's monthly count for Freeport, Corpus Christi and any Houston LNG
activity, 2020–2023. Report cargoes per terminal-month.

Specifically check: **does Freeport go to zero from June 2022, and when does
it resume?** D-142 makes this the out-of-sample test, and the exact dates
must come from this data rather than from memory.

**Decides:** confirmation that D-142's natural experiment is visible in the
validation source, and the exact window to test against.

---

## Reporting

Write results to `scratch/doe/phase0b_results.md`. For each question: what was
run, the result as raw tables, one line of interpretation. Flag explicitly any
result that contradicts `docs/decisions.md` or `docs/literature.md`.

Commit the downloaded Excel files to `scratch/doe/` and record their URL and
download date — DOE revises published data, and a number that cannot be
traced to a file version cannot be defended.

**Stop after reporting.** Do not start the AIS backfill. Four outcomes require
a design decision first:

1. **No transaction detail for 2024–2026** — D-004's per-cargo deliverable
   does not hold for the full window and needs a two-tier design.
2. **No tanker name column, or high null rate** — D-140's join key fails and
   the validation design needs rethinking.
3. **Departure date absent or ambiguous** — D-140 matches on the wrong event.
4. **Sabine Pass not identifiable in point-of-exit** — terminal attribution
   fails.

---

## Rules in force

- DOE is a **second measurement, not ground truth** (D-044). It is
  self-reported on Form FE-746R. Frame disagreements accordingly.
- Never impute or interpolate. A missing tanker name is an unmatchable
  record, reported as such (D-045).
- Report raw counts and tables, not summaries. Distributions will be read
  directly.
- Do not filter to expected values before reporting distinct values. The
  point of D2 and D4 is to find naming conventions that were not anticipated.
