# Phase 0 — Sabine Pass, one file (v2) — results

Run against `scratch/ais-2022-03-15.csv.zst` (kept, not deleted).
Executed via `scratch/phase0_run.py`. Full raw stdout in
`scratch/phase0_raw_output.txt`.

**Note on the brief filename:** `docs/phase0_sabine_pass.md` does not exist;
only `docs/phase0_sabine_pass_v2.md` was present. Ran v2 (it declares it
supersedes v1).

**Column mapping (Q0 result vs. brief's PascalCase):** all columns are
snake_case, and two are renamed, not just re-cased:

| Brief | Actual |
|---|---|
| MMSI | mmsi |
| BaseDateTime | base_date_time |
| LAT | **latitude** |
| LON | **longitude** |
| SOG | sog |
| COG | cog |
| Heading | heading (BIGINT, not DOUBLE) |
| VesselName | vessel_name |
| IMO | imo |
| CallSign | call_sign |
| VesselType | vessel_type |
| Status | status |
| Length | length |
| Width | width |
| Draft | draft |
| Cargo | cargo |
| TransceiverClass | **transceiver** |

All Q1–Q7 queries below use these adapted names. Logic, filters and
thresholds are unchanged from the brief.

---

## Q0 — Schema and scale

**Q0a — DESCRIBE**

| column_name | column_type | null | key | default | extra |
|---|---|---|---|---|---|
| mmsi | BIGINT | YES | None | None | None |
| base_date_time | TIMESTAMP | YES | None | None | None |
| longitude | DOUBLE | YES | None | None | None |
| latitude | DOUBLE | YES | None | None | None |
| sog | DOUBLE | YES | None | None | None |
| cog | DOUBLE | YES | None | None | None |
| heading | BIGINT | YES | None | None | None |
| vessel_name | VARCHAR | YES | None | None | None |
| imo | VARCHAR | YES | None | None | None |
| call_sign | VARCHAR | YES | None | None | None |
| vessel_type | BIGINT | YES | None | None | None |
| status | BIGINT | YES | None | None | None |
| length | BIGINT | YES | None | None | None |
| width | BIGINT | YES | None | None | None |
| draft | DOUBLE | YES | None | None | None |
| cargo | BIGINT | YES | None | None | None |
| transceiver | VARCHAR | YES | None | None | None |

**Q0b — row count**

| rows |
|---|
| 7994666 |

**Interpretation:** the schema table in `docs/` does not hold for 2022 —
column names are snake_case, and `LAT`/`LON` → `latitude`/`longitude`,
`TransceiverClass` → `transceiver` are renames, not just case changes. This
contradicts a docs assumption (flagged per outcome #1).

---

## Q1 — Timezone of the timestamp column

**Q1a — file boundaries**

| first_msg | last_msg |
|---|---|
| 2022-03-15 00:00:00 | 2022-03-15 23:59:59 |

**Q1b — daylight test (Class B, Gulf Coast box)**

| hr | n |
|---|---|
| 0 | 4559 |
| 1 | 4340 |
| 2 | 4234 |
| 3 | 3972 |
| 4 | 3922 |
| 5 | 4055 |
| 6 | 3976 |
| 7 | 4096 |
| 8 | 3949 |
| 9 | 3883 |
| 10 | 3767 |
| 11 | 3935 |
| 12 | 4148 |
| 13 | 4350 |
| 14 | 4613 |
| 15 | 4724 |
| 16 | 4716 |
| 17 | 4601 |
| 18 | 4853 |
| 19 | 4875 |
| 20 | 4963 |
| 21 | 4729 |
| 22 | 4708 |
| 23 | 4974 |

**Interpretation:** not flat — min 3767 (hr10) to max 4974 (hr23), ~27% range
over mean. Trough is hours 8–11 UTC (= local 3–6am under UTC-5, pre-dawn);
broad rise from hour 12 through a peak at 18–23 UTC (= local 7am through
1–6pm). Hours 9–16, the brief's local-time-peak hypothesis, are close to the
*minimum*, not a peak — directly contradicts the local-time hypothesis.
Reads as **UTC**.

---

## Q2 — Does the clip box contain Sabine Pass?

| rows_in_box | vessels | pct_of_file |
|---|---|---|
| 90588 | 206 | 1.133 |

`CREATE VIEW box` — succeeded, 0 rows returned (DDL).

**Interpretation:** box is non-empty and returns a plausible vessel count
(206) at ~1.1% of the file; usable as a starting clip.

---

## Q3 — What vessel type do LNG carriers actually report? *(unfiltered)*

| vessel_type | msgs | vessels | min_len | max_len |
|---|---|---|---|---|
| 80 | 2872 | 7 | 250 | 299 |
| 60 | 107 | 2 | 305 | 311 |
| 84 | 193 | 1 | 299 | 299 |

**Interpretation:** only 3 vessel_type values appear among Length≥250
vessels in the box — 80 and 84 (both within the assumed 80–89 tanker range)
and 60 (passenger/cruise, confirmed in Q4a as Carnival Breeze / Adventure of
the Seas — not tankers, expected noise from the box also covering
Port Arthur/Beaumont traffic). No nulls, zeros, or out-of-80–89 codes
appeared at this size threshold, but sample is tiny (10 vessels total).

---

## Q4 — Does draft separate LNG carriers from crude tankers? *(new)*

**Q4a — per-vessel draft, Length ≥ 250**

| mmsi | name | vtype | len | beam | l_over_b | min_draft | max_draft | msgs |
|---|---|---|---|---|---|---|---|---|
| 354842000 | CARNIVAL BREEZE | 60 | 305 | 37 | 8.24 | 8.2 | 8.2 | 61 |
| 311263000 | ADVENTURE OFTHE SEAS | 60 | 311 | 49 | 6.35 | 8.6 | 8.6 | 46 |
| 215906000 | ASKLIPIOS | 80 | 299 | 46 | 6.5 | 9.3 | 9.3 | 474 |
| 538009374 | MOL HESTIA | 80 | 298 | 48 | 6.21 | 9.4 | 9.4 | 553 |
| 215684000 | HELLAS ATHINA | 80 | 299 | 46 | 6.5 | 9.6 | 9.6 | 623 |
| 248720000 | CASTILLO DE CALDELAS | 80 | 297 | 49 | 6.06 | 10.0 | 10.0 | 265 |
| 636020805 | DUOMO SQUARE | 80 | 250 | 44 | 5.68 | 10.5 | 10.5 | 369 |
| 215503000 | LA SEINE | 84 | 299 | 46 | 6.5 | 11.6 | 11.6 | 193 |
| 538009200 | PACIFIC SAPPHIRE | 80 | 250 | 44 | 5.68 | 13.1 | 13.1 | 13 |
| 310757000 | GASLOG GIBRALTAR | 80 | 291 | None | None | None | None | 575 |

**Q4b — draft distribution buckets**

| draft_m | vessels |
|---|---|
| 8.0 | 2 |
| 9.0 | 3 |
| 10.0 | 2 |
| 11.0 | 1 |
| 13.0 | 1 |

**Interpretation:** **not bimodal** — draft forms one continuous run from
8.2m to 13.1m (cruise ships and tankers overlap in the same range; LA SEINE,
a known LNGC, sits at 11.6m inside the same continuum as everything else, no
gap before or after it). No vessel in the box reaches the 16–17m docs
expected for Suezmax crude tankers, so the contrast the docs assumed can't
even be tested on this box/date. 1 of 10 large vessels (GASLOG GIBRALTAR, a
known LNG carrier) has **no usable draft at all** — 10% ceiling on coverage
for this specific subset. Contradicts the docs assumption of a bimodal
separation (outcome #4).

---

## Q5 — Transceiver class split

| transceiver | msgs | vessels |
|---|---|---|
| B | 6350 | 19 |
| A | 84238 | 187 |

**Interpretation:** Class A dominates by both message and vessel count,
consistent with the docs assumption that commercial LNG carriers are Class
A — a Class A filter would drop the small-craft Class B population cheaply.

---

## Q6 — Sentinel values, split by field

| rows | sog_unavail | cog_unavail | hdg_unavail | draft_unavail | pct_sog | pct_cog | pct_hdg |
|---|---|---|---|---|---|---|---|
| 90588 | 0 | 0 | 0 | 52039 | 0.0 | 0.0 | 0.0 |

**Interpretation:** **contradicts docs.** `SOG = 102.3`, `COG = 360.0`, and
`Heading = 511` all return exactly 0 matches in the box — versus the
previously-cited 17.9% blended figure. `draft_unavail` (`draft = 0 OR draft
IS NULL`) is 52039/90588 = 57.5%, so sentinels/missingness are real in this
file, just not via these exact literal matches on sog/cog/heading. Likely
causes: float equality failing on `sog = 102.3` (stored value not bit-exact
102.3), or this 2022 product using `NULL` rather than the magic values for
unavailable SOG/COG/heading. Ran the query exactly as specified per
instructions — did not rewrite it. Needs follow-up before this sentinel
list is trusted for 2022 data.

---

## Q7 — Opportunistic: does draft change across a loading? *(optional)*

| mmsi | name | len | distinct_drafts | min_draft | max_draft | min_sog | max_sog |
|---|---|---|---|---|---|---|---|

(0 rows.)

**Interpretation:** no vessel ≥250m in the box shows more than one distinct
draft reading on 2022-03-15. Consistent with the brief's own expectation
(loading runs 12–24h, may not complete within one day) — a null result, not
a failure.

---

## Summary against the four stop conditions

1. **Column names differ from docs** — yes (see mapping table above).
2. **BaseDateTime not UTC / Q1b inconclusive** — Q1b is not flat and reads
   as UTC; local-time hypothesis is contradicted by its own predicted peak
   hours (9–16) being near the minimum.
3. **Clip box returns few/no vessels over 250m** — box returns 10 such
   vessels (7+2+1 across three vessel_type codes); non-zero, but a small
   sample from a single day.
4. **Draft unusable** — yes: not bimodal in Q4, and Q6 shows draft
   unavailable on 57.5% of messages in the box.

Two of four stop conditions are triggered (#1 and #4), plus the Q6 sentinel
anomaly for sog/cog/heading. Per the brief, this needs a design decision
before Phase 1 — not made here.

---

## Addendum — box adjusted, Q6 anomaly follow-up

Eastern edge of the box pulled in from `-93.4` to `-93.5` for separation
from Golden Pass / Port Arthur traffic:

```sql
CREATE VIEW box AS
SELECT * FROM read_csv_auto('scratch/ais-2022-03-15.csv.zst')
WHERE latitude BETWEEN 28.8 AND 29.9 AND longitude BETWEEN -94.2 AND -93.5;
```

**Box size sanity check** (was 90588 rows / 206 vessels at `-93.4`):

| rows_in_box | vessels |
|---|---|
| 88742 | 203 |

**Null counts for sog/cog/heading**

```sql
SELECT COUNT(*) FILTER (WHERE sog IS NULL)     AS sog_null,
       COUNT(*) FILTER (WHERE cog IS NULL)     AS cog_null,
       COUNT(*) FILTER (WHERE heading IS NULL) AS hdg_null,
       COUNT(*) AS rows
FROM box;
```

| sog_null | cog_null | hdg_null | rows |
|---|---|---|---|
| 15 | 5449 | 32336 | 88742 |

**Max values for sog/cog/heading**

```sql
SELECT MAX(sog) AS max_sog, MAX(cog) AS max_cog, MAX(heading) AS max_hdg
FROM box;
```

| max_sog | max_cog | max_hdg |
|---|---|---|
| 36.5 | 359.9 | 359 |

**Interpretation:** explains the Q6 anomaly. `MAX(cog) = 359.9` and
`MAX(heading) = 359` never reach the old magic sentinels (360.0, 511) —
this 2022 product has already scrubbed unavailable sog/cog/heading to
**NULL** rather than encoding them as magic values. Real missingness is
substantial (17,336/88,742 ≈ 36.4% of rows null on at least heading alone).
`draft`'s sentinel handling (`= 0 OR IS NULL`, per Q6) is unaffected by this
— separate field, separate convention. Silver-layer sentinel logic for this
product needs `IS NULL` checks for sog/cog/heading, not equality against
102.3/360.0/511.
