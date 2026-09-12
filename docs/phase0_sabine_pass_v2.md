# Phase 0 — Sabine Pass, one file (v2)

Supersedes v1. Two changes: vessel-type discovery now runs on **unfiltered**
data, and a draft-based identification query has been added.

Exploration only. **Do not write pipeline code in this phase.** Nothing here
belongs in `src/`. Work in `scratch/`, report results, stop.

Goal: answer seven questions against a single day of AIS before any pipeline
design is committed. Several assumptions in `docs/` were carried over from a
retired NOAA distribution or from container-era scoping and may be wrong.
This phase exists to find out.

---

## Setup

Download one file (~200 MB) to `scratch/`:

```
https://noaaocm.blob.core.windows.net/ais/csv2/csv2022/ais-2022-03-15.csv.zst
```

Keep it. Do not delete after the run — every query below uses it.

Date chosen because Sabine Pass and Freeport were both operating normally, it
is an ordinary weekday, and it sits inside the intended anchor year.

DuckDB should read `.csv.zst` directly. If it cannot, decompress with
`zstd -d` and note that — it changes the ingest design (an extra step and a
larger per-day disk footprint).

```python
import duckdb
con = duckdb.connect()
RAW = 'scratch/ais-2022-03-15.csv.zst'
```

**Do not create a `tankers` view.** v1 of this brief and §7 of the handoff
both filtered to `VesselType BETWEEN 80 AND 89` before asking which type code
LNG carriers use. That is circular: if LNG carriers report a code outside
that range, the query returns nothing and there is no way to distinguish
"none here" from "wrong filter". Type discovery must run unfiltered.

---

## Q0 — Schema and scale

**Run this first. Every query below uses PascalCase column names that may not
exist in this product or this year. Adapt to whatever Q0 returns and report
the mapping.**

```sql
DESCRIBE SELECT * FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst',
                                     sample_size = 200000);
```

```sql
SELECT COUNT(*) AS rows FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst');
```

Report exact column names, types, row count, and casing convention.

**Decides:** whether the schema table in `docs/` holds for 2022. It was
observed on a 2024 file, and paths, naming and compression are known to vary
by year.

---

## Q1 — Timezone of the timestamp column

**(a) File boundaries.**

```sql
SELECT MIN(BaseDateTime) AS first_msg, MAX(BaseDateTime) AS last_msg
FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst');
```

**(b) Daylight test.** Class B transponders are mostly recreational craft,
active in local daylight. On 2022-03-15 the US Gulf Coast is UTC-5.

```sql
SELECT HOUR(BaseDateTime) AS hr, COUNT(*) AS n
FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst')
WHERE TransceiverClass = 'B'
  AND LAT BETWEEN 26 AND 31 AND LON BETWEEN -97 AND -88
GROUP BY hr ORDER BY hr;
```

A broad peak around hours 14–21 indicates **UTC**. A peak around 09–16
indicates **local time**.

**If the histogram comes back flat, the test has failed to discriminate** —
do not guess. Report it as inconclusive and escalate; a direct check against
NOAA field documentation is needed instead.

**Decides:** correctness of every daily and monthly bucket. A 5-hour error
misassigns events near midnight — and since DOE records cargoes by departure
*date*, a midnight-boundary error becomes a false positive and a false
negative simultaneously.

---

## Q2 — Does the clip box contain Sabine Pass?

Proposed box, a hypothesis and not settled:

```
LAT  28.8  to  29.9
LON -94.2  to -93.4
```

```sql
SELECT COUNT(*) AS rows_in_box,
       COUNT(DISTINCT MMSI) AS vessels,
       ROUND(100.0 * COUNT(*) / (SELECT COUNT(*)
            FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst')), 3) AS pct_of_file
FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst')
WHERE LAT BETWEEN 28.8 AND 29.9 AND LON BETWEEN -94.2 AND -93.4;
```

Then create the one view everything downstream uses:

```sql
CREATE VIEW box AS
SELECT * FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst')
WHERE LAT BETWEEN 28.8 AND 29.9 AND LON BETWEEN -94.2 AND -93.4;
```

**Decides:** whether the box is usable, and the expected clip ratio for
capacity planning. Note this box also contains Golden Pass LNG (under
construction, opposite bank) and the Port Arthur / Beaumont petrochemical
complex. Heavy non-LNG traffic is expected, not a fault.

---

## Q3 — What vessel type do LNG carriers actually report? *(unfiltered)*

There is no LNG carrier code in AIS. The second digit of `VesselType` is a
hazard class, not a sub-type. The code must be found empirically.

```sql
SELECT VesselType,
       COUNT(*)              AS msgs,
       COUNT(DISTINCT MMSI)  AS vessels,
       ROUND(MIN(Length), 0) AS min_len,
       ROUND(MAX(Length), 0) AS max_len
FROM box
WHERE Length >= 250
GROUP BY VesselType
ORDER BY vessels DESC;
```

Report **every** type value returned, including nulls, zeros and anything
outside 80–89. Do not pre-filter.

**Decides:** the vessel-type filter, or whether one is viable at all.

---

## Q4 — Does draft separate LNG carriers from crude tankers? *(new)*

LNG is roughly half the density of crude, so LNG carriers float far shallower
for their size. Expect conventional LNGCs near 290 m length and 11–12 m
draft, against Suezmax crude tankers near 275 m and 16–17 m. Length alone
does not separate them; length plus draft should.

Draft sentinels: `0` means unavailable, and values exceeding vessel length
are known bad data. Filter both.

```sql
SELECT MMSI,
       ANY_VALUE(VesselName)  AS name,
       ANY_VALUE(VesselType)  AS vtype,
       ROUND(ANY_VALUE(Length), 0) AS len,
       ROUND(ANY_VALUE(Width),  0) AS beam,
       ROUND(ANY_VALUE(Length) / NULLIF(ANY_VALUE(Width), 0), 2) AS l_over_b,
       ROUND(MIN(Draft) FILTER (WHERE Draft BETWEEN 1 AND 30), 1) AS min_draft,
       ROUND(MAX(Draft) FILTER (WHERE Draft BETWEEN 1 AND 30), 1) AS max_draft,
       COUNT(*) AS msgs
FROM box
WHERE Length >= 250
GROUP BY MMSI
ORDER BY max_draft;
```

Ordering by draft makes any gap in the distribution visible by eye. Then
bucket it:

```sql
SELECT FLOOR(max_draft) AS draft_m, COUNT(*) AS vessels
FROM (
  SELECT MMSI, MAX(Draft) FILTER (WHERE Draft BETWEEN 1 AND 30) AS max_draft
  FROM box WHERE Length >= 250 GROUP BY MMSI
) WHERE max_draft IS NOT NULL
GROUP BY draft_m ORDER BY draft_m;
```

**Decides:** whether draft is a usable discriminator, and how much work the
berth polygon has to do. Report whether the draft distribution for large
vessels is bimodal, and what fraction of large vessels report no usable
draft at all — that fraction is the ceiling on this method's coverage.

---

## Q5 — Transceiver class split

```sql
SELECT TransceiverClass, COUNT(*) AS msgs, COUNT(DISTINCT MMSI) AS vessels
FROM box GROUP BY TransceiverClass;
```

**Decides:** the value of a Class A filter in silver. All commercial LNG
carriers are Class A.

---

## Q6 — Sentinel values, split by field

The existing "17.9%" figure is unsplit and probably misleading.

```sql
SELECT COUNT(*) AS rows,
       COUNT(*) FILTER (WHERE SOG     = 102.3) AS sog_unavail,
       COUNT(*) FILTER (WHERE COG     = 360.0) AS cog_unavail,
       COUNT(*) FILTER (WHERE Heading = 511)   AS hdg_unavail,
       COUNT(*) FILTER (WHERE Draft   = 0 OR Draft IS NULL) AS draft_unavail,
       ROUND(100.0 * COUNT(*) FILTER (WHERE SOG = 102.3) / COUNT(*), 3) AS pct_sog,
       ROUND(100.0 * COUNT(*) FILTER (WHERE COG = 360.0) / COUNT(*), 3) AS pct_cog,
       ROUND(100.0 * COUNT(*) FILTER (WHERE Heading = 511) / COUNT(*), 3) AS pct_hdg
FROM box;
```

`Heading = 511` may already have been nulled by NOAA processing. Report
whichever is true.

**Decides:** the honest per-field figure. `SOG` is load-bearing and cannot be
imputed. `COG` is unusable at rest regardless.

---

## Q7 — Opportunistic: does draft change across a loading? *(optional)*

Only meaningful if a vessel both berthed and departed within this single day.
LNG loading typically runs 12–24 hours, so this may find nothing. Run it; a
null result is not a failure.

```sql
SELECT MMSI, ANY_VALUE(VesselName) AS name,
       ROUND(ANY_VALUE(Length), 0) AS len,
       COUNT(DISTINCT Draft) AS distinct_drafts,
       ROUND(MIN(Draft) FILTER (WHERE Draft BETWEEN 1 AND 30), 1) AS min_draft,
       ROUND(MAX(Draft) FILTER (WHERE Draft BETWEEN 1 AND 30), 1) AS max_draft,
       ROUND(MIN(SOG) FILTER (WHERE SOG != 102.3), 1) AS min_sog,
       ROUND(MAX(SOG) FILTER (WHERE SOG != 102.3), 1) AS max_sog
FROM box
WHERE Length >= 250
GROUP BY MMSI
HAVING COUNT(DISTINCT Draft) > 1
ORDER BY (max_draft - min_draft) DESC;
```

**Decides:** early evidence on whether crews maintain `Draft`, which
underpins both cargo-volume estimation and ballast/laden detection.

---

## Reporting

Write results to `scratch/phase0_results.md`. For each question: the query
run, the result, one line of interpretation. Flag explicitly any result that
contradicts an assumption in `docs/`.

**Stop after reporting.** Do not proceed to backfill, ingest code, or polygon
work. Four outcomes require a design decision first:

1. Column names or types differ from `docs/`
2. `BaseDateTime` is not UTC, or Q1(b) is inconclusive
3. The clip box returns few or no vessels over 250 m
4. Draft is unusable — either mostly missing, or not bimodal for large
   vessels

---

## Rules in force

- Never load the raw file into memory; DuckDB reads it directly with filters
  pushed into the scan.
- Never silently drop rows. Every filter reports rows in, rows out, reason.
- Never impute or interpolate. Gaps stay gaps and get reported.
- Sentinels: `SOG = 102.3`, `COG = 360.0`, `Heading = 511`, `Draft = 0`.
  Draft above 30 m at these terminals is bad data, not a deep vessel.
- Ports come from config, never hard-coded. The box above is exploratory and
  does not get committed to `src/`.
- Do not filter by `VesselType` anywhere in this phase. That filter is an
  output of Q3, not an input to it.
