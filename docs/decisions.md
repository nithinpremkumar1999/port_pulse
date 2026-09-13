# decisions.md

Binding decisions for port_pulse. **Read this before writing code that chooses a
threshold, a filter, an algorithm or a data source.**

`literature.md` holds the evidence. This file holds the rulings. Each decision
cites the evidence that supports it via `LIT-*` IDs.

## How to use this file

**Precedence, highest first:**

1. `CLAUDE.md` hard rules and "Never" list — never overridden by anything here.
2. An `ACCEPTED` decision in this file.
3. `literature.md` findings.
4. Your own judgement.

**If no decision covers the situation:** stop and ask. Do not pick a threshold
because it seems reasonable. An undocumented threshold is the failure mode this
file exists to prevent.

**If a decision is marked `OPEN`:** it is blocking. Do not implement around it.
Say which decision is blocking and what it needs.

**If you must deviate from an `ACCEPTED` decision:** say so explicitly in your
response, give the reason, and propose an amendment. Do not deviate silently.

**Adding a decision.** New IDs are sequential within their block, never reused.
Changing a decision means adding a new one that supersedes the old; mark the old
`SUPERSEDED by D-0NN` and leave it in place. The history is the point.

## Status meanings

| Status | Meaning |
|---|---|
| `ACCEPTED` | Binding. Implement as written. |
| `PROPOSED` | Drafted, not signed off by the maintainer. Implement, but flag it in the PR description as unratified. |
| `OPEN` | Undecided and blocking. Do not proceed past it. |
| `SUPERSEDED` | Historical. Do not implement. |

## ID blocks

| Range | Area | Overflow |
|---|---|---|
| D-001–009 | Scope and framing | D-101–109 |
| D-010–019 | Ingest, bronze and silver | D-110–119 |
| D-020–029 | Port call detection (gold) | D-120–129 |
| D-030–039 | Metrics | D-130–139 |
| D-040–049 | Validation and analysis | D-140–149 |
| D-050–059 | Explicitly out of scope | D-150–159 |

**Overflow blocks.** The primary blocks are narrow and several are now fully
allocated. A decision that supersedes D-0NN takes ID D-1NN, which keeps the
lineage readable at a glance. If D-1NN is also exhausted, extend to D-2NN.

**New decisions in a full block.** The D-1NN convention above reserves overflow
IDs for supersessions, which leaves no clean ID for a genuinely *new* decision
in a full block. Rule: take the next free ID in the **primary** block of the
nearest area that still has room, and state the area in the decision header.
Never take a free D-1NN for a new decision — a reader seeing D-116 will assume
it supersedes D-016, and that inference must stay reliable. If no primary block
has room, widen the blocks rather than reusing the overflow.

---

# Scope and framing

### D-001 — port_pulse is a lead-lag study, not a nowcaster

**Status:** SUPERSEDED by D-004 | **Evidence:** SRC-NOAA-FAQ, LIT-IMF-MALTA-2019

> Retained for history. The latency analysis below is correct and carries
> forward into D-004. The conclusion — that the surviving deliverable is a
> lead-lag study — does not: see D-004.

**Do this.** Describe the project as a historical lead-lag study throughout the
codebase, README and any output. Do not claim near-real-time capability. Do not
build scheduling, alerting or live-serving components.

**Why.** The framing in `CLAUDE.md` — "port statistics are published weeks in
arrears; AIS is near real-time" — is false for this source. Marine Cadastre
publishes no live feed, adds data roughly every 90 days, and has a total lag from
collection to delivery of approximately 145–165 days. The input is slower than
the official statistics it is meant to lead. The research question ("does
congestion lead throughput, by how long, and where does it break") is unaffected
and answerable on history. The operational claim is not.

**Rejected.** Claiming real-time capability with a caveat. The caveat gets lost;
the claim is checkable and would fail.

**Consequence.** The ingest layer must be swappable so a commercial real-time
feed (MarineTraffic, Spire, ORBCOMM, exactEarth — see LIT-IMF-MALTA-2019) could
replace NOAA without touching silver and above. Treat this as an architectural
requirement, not a nice-to-have: it is the answer to "could you productionise
this?"

**Revisit if.** NOAA changes latency, or a real-time source enters scope.

---

### D-002 — Metrics are reported per vessel type group, never as a single aggregate

**Status:** PROPOSED | **Evidence:** LIT-VERSCHUUR-2021

**Do this.** Every published metric carries an explicit vessel type group
dimension. Any headline figure states which groups are in scope.

**Why.** LIT-VERSCHUUR-2021 reports global port calls down 4.4% for 2020 against
UNCTAD's 8.7%, and attributes the entire gap to vessel type scope: passenger
vessels were 66% of total port calls and fell 17%, and they excluded them. A
port-call metric's headline number is dominated by which types are counted. An
unqualified aggregate is not interpretable and is not defensible.

**Test.** No metrics table may be written without a vessel type group column.

**Note under the LNG scope (D-004).** The group that matters here — LNG
carrier — is not an AIS vessel type and cannot be read from `vessel_type`.
The dimension column is populated from the D-005 identification stack, not
from the raw field. The requirement is unchanged; its source is not.

---

### D-003 — Publish the constraint set alongside the results

**Status:** PROPOSED | **Evidence:** LIT-IMF-MALTA-2019, LIT-IMF-PORTWATCH

**Do this.** The README states, in the authors' own framing: AIS measures goods
not services, volume not value, gross trade not re-exports, and broad groups not
specific goods; AIS-derived estimates include transshipments; AIS-derived
estimates are a proxy.

**Why.** These are the limitations the IMF authors state about their own work
with better data than ours. Stating them is not hedging — omitting them is the
thing a reviewer notices.

---

### D-004 — port_pulse is a validated estimator of LNG export activity

**Status:** PROPOSED | **Evidence:** SRC-NOAA-FAQ, SRC-DOE-FE746R, OBS-PHASE0
**Supersedes:** D-001

**Do this.** Describe the project as a *validated estimator, not a live
product*. Two deliverables, both latency-independent:

1. **Accuracy** — agreement between AIS-derived cargo events and official
   records, measured **per cargo** as precision, recall and volume error.
   Never as a correlation coefficient on an aggregated series.
2. **Granularity** — berth occupancy, loading duration, turnaround, vessel
   class mix. Published statistics record that a cargo departed; they do not
   record that loading took 19 hours.

Do not build scheduling, alerting or live-serving components.

**Why.** D-001 was right that the NOAA archive lags 145–165 days and cannot
front-run official statistics. It was wrong to conclude a lead-lag study was
the surviving deliverable. Two reasons. First, official DOE figures publish
faster than the AIS input, so any measured "lead" is an artefact of the
research design, not a usable property. Second, LNG loading slots are
contracted months ahead, so queueing is rare and there is little congestion
signal for a lead to be measured in.

The estimator's lead over official publication is a property of the *method*.
The archive's delivery lag is a property of the *free distribution channel*
and is a procurement decision. State both explicitly in the README — the lag
is checkable, and stating it is a credibility asset.

**Rejected.** Nowcast framing (input is staler than the target). Lead-lag
framing (no congestion signal at LNG terminals, and the target publishes
first).

**Consequence.** The ingest layer stays swappable, per D-001's original
reasoning — a commercial feed would restore live capability without touching
silver and above. Keep that as an architectural requirement.

---

### D-005 — LNG carriers are identified by berth, then dimensions; never by draft

**Status:** PROPOSED | **Evidence:** OBS-PHASE0, LIT-HARATI-2007, SRC-DOE-FE746R

**Depends on hand-drawn geometry.** The berth polygon this decision makes
primary is hand-drawn in v1 (D-013) and explicitly not derived (D-024). D-027
then makes berth-exit timing the headline accuracy number. Taken together, a
hand-drawn polygon is now load-bearing for the D-240 result — which is
defensible, but must be stated in the writeup and sensitivity-tested per D-013
rather than left implicit.

**Do this.** Identify the LNG carrier population in this order:

1. **Berth polygon** — primary. Only LNG carriers berth at an LNG jetty. If
   the polygon is the jetty rather than the port, the geometry does the
   identification.
2. **Length and beam** — confirmation. Conventional membrane LNGCs cluster at
   290–300 m × 46–49 m. Must tolerate nulls: see the caution below.
3. **DOE tanker names** — validation only, never an input to detection.
   Matching AIS `vessel_name` against the DOE roster measures the identifier,
   it does not create it.

**Do not use `draft` to classify vessel class.**

**Why.** OBS-PHASE0 found six LNG carriers and two non-LNG tankers in the
Sabine Pass box on a single ordinary day. Dimensions separated them cleanly:
291–299 m × 46–49 m against 250 m × 44 m, with no overlap.

Draft did not separate them, and the reason is structural rather than a data
quality problem. **Draft is a load-state variable, not a vessel-class
variable.** A ballasted crude tanker and a laden LNG carrier sit at similar
draughts. OBS-PHASE0 shows this directly: ASKLIPIOS at 9.3 m and LA SEINE at
11.6 m are both LNG carriers, two metres apart, because one is in ballast and
one is laden. Any single-day snapshot observes mixed load states, so the
classes necessarily overlap.

**Rejected.** Draft as a class discriminator — the reasoning above. Rejected:
`vessel_type` as a discriminator — there is no LNG code in AIS, and
LIT-HARATI-2007 found the type field unsatisfactory in 56–74% of cases.

**Caution — static fields are not always present.** OBS-PHASE0 found one of
ten large vessels (GASLOG GIBRALTAR, a known LNG carrier) reporting neither
beam nor draft across 575 messages. The dimension filter must be
null-tolerant, and the fraction of the large-vessel population with
unusable dimensions must be reported per port-month as a coverage ceiling.

**Note.** Draft remains in scope for **loading detection** — a ballast-to-laden
change across a call is direct evidence of loading. That is a different use and
is not governed by this decision.

**Orphaned by D-153.** This note pointed at D-053, which D-153 resolved. But
D-153 rules on *volume estimation* only; it says nothing about using a draft
change as corroborating evidence that a loading occurred. Under D-027 the
departure event is defined kinematically by berth exit, so nothing in v1
requires draft for detection. The question is therefore not blocking, but it is
also not ruled on — it belongs in Open questions, not in a dangling reference.

**Test.** `test_lng_identification_is_null_tolerant`; report dimension
coverage rate per partition.

---

### D-006 — Port scope and clip boxes

**Status:** SUPERSEDED by D-106 | **Evidence:** OBS-PHASE0

> Retained for history. The Sabine Pass clip box and the contamination
> analysis carry forward verbatim. The port list does not: Houston has no
> LNG activity in the validation source. See D-106.

**Do this.** v1 covers four terminals:

| Port | Role |
|---|---|
| Sabine Pass (Cameron Parish, LA) | Primary — largest, longest history |
| Freeport (Quintana Island, TX) | 2022 outage, out-of-sample test (D-142) |
| Corpus Christi (TX) | Growth and Stage 3 ramp |
| Houston Ship Channel | Non-LNG control; congestion logic applies here only |

Sabine Pass clip box, validated in OBS-PHASE0:

```
latitude   28.8  to  29.9
longitude -94.2  to -93.5
```

**Why.** The eastern edge was moved from −93.4 to −93.5 after OBS-PHASE0. The
original edge sat 6–8 km from Cameron LNG (≈ −93.32) and Calcasieu Pass
(≈ −93.34), which is too tight for a box that must also cover offshore
waiting areas without capturing traffic bound for other LNG terminals. The
change cost 2% of rows and 3 of 206 vessels — negligible at Sabine Pass, and
it makes the one-polygon-one-terminal claim defensible rather than lucky.

**Known contamination, expected and not a fault.** The box also contains
Golden Pass LNG (under construction for most of the study window, first cargo
expected 2026) and Port Arthur / Beaumont petrochemical traffic. D-005's
berth polygon is what excludes them, not the clip box.

**Revisit if.** Golden Pass enters service within the study window — at that
point two LNG terminals share the box and berth polygons become load-bearing
for terminal attribution, not just vessel classification.

---

# Ingest, bronze and silver

### D-010 — Raw CSV is the source of record; GeoParquet is a validation oracle

**Status:** PROPOSED | **Evidence:** SRC-NOAA-FAQ

**Do this.** Ingest the Zstd-compressed daily CSV bulk files as the canonical
source. Separately, for the years where NOAA publishes cleaned GeoParquet (2024
and 2025 at time of writing, on Azure via the Marine Cadastre GitHub), download a
sample and **diff it against our own silver output** as a correctness check on
our cleaning.

**Why.** Two reasons, in order. First, the cleaning logic is the demonstrable
part of this project; consuming a pre-cleaned product removes the evidence that
we can do it. Second, and more useful: NOAA's GeoParquet has had sentinel values
and anomalies removed, which makes it an independent ground truth for our silver
layer over the overlapping period. That is a rare and free validation
opportunity — take it.

**Rejected.** GeoParquet as primary. It is described as an experimental product,
covers only recent years, and would leave the backfill on a different code path
from the recent data.

**Test.** `test_silver_matches_noaa_geoparquet` over a sample day in 2024.
Report the disagreement rate rather than asserting equality — some divergence is
expected and informative.

---

### D-011 — All time handling is UTC internally; the week boundary is port-local

**Status:** ACCEPTED | **Evidence:** SRC-NOAA-FAQ, LIT-IMF-WORLD-2020, OBS-PHASE0

**Do this.**

- Store every timestamp as UTC. No local times in bronze, silver or gold.
- Assign the **week** at the metrics layer using each port's local timezone,
  read from the port config. Never from a global constant.
- Record the week convention (ISO week, Monday start) explicitly in the metrics
  schema.
- Do **not** apply a blanket UTC offset to timestamps.

**Why.** `base_date_time` is UTC — confirmed empirically in OBS-PHASE0, not
just asserted from documentation. Official records are local. Under the LNG
scope this matters more than it did: DOE records cargoes by **departure
date** (SRC-DOE-FE746R), so a timezone error near midnight produces a false
positive and a false negative simultaneously, not a small phase shift.
LIT-IMF-WORLD-2020 handled this by shifting all AIS timestamps by six hours
across the board and matching within ±2 days, and state themselves that this
limits but does not eliminate the discrepancy. A ±2-day matching window absorbs a
blanket shift; a **weekly** metric does not. On the US West Coast, 7–8 hours of
each UTC day belongs to the previous local day; misassigning them introduces a
systematic sub-week phase error into an analysis whose entire output is a lag
measured in weeks.

**Rejected.** The IMF six-hour blanket shift. Appropriate for their ±2-day
event-matching, wrong for weekly aggregation. Also rejected: UTC week boundaries
for all ports, which is simpler but bakes in a per-port phase error that varies
with longitude and daylight saving.

**Test.** `test_week_assignment_at_local_midnight` — a synthetic departure at
23:59 and 00:01 port-local on a day and week boundary lands in the expected
buckets, for at least one US Gulf Coast port (America/Chicago, all four
terminals in D-106) in both DST and standard time. The original test named US
West and East Coast ports; under D-106 the Gulf is the only zone in scope.

---

### D-012 — The schema is a contract; assert it and fail loudly

**Status:** SUPERSEDED by D-112 | **Evidence:** SRC-NOAA-FAQ

> Retained for history. The principle is right and carries forward. The
> specific header it asserts is wrong for the `csv2` product — see D-112.

**Do this.** Validate on every ingested file that the header is exactly, in
order: `MMSI`, `BaseDateTime`, `LAT`, `LON`, `SOG`, `COG`, `Heading`,
`VesselName`, `IMO`, `CallSign`, `VesselType`, `Status`, `Length`, `Width`,
`Draft`, `Cargo`, `TransceiverClass`. On mismatch, abort the partition and raise
with the observed header in the message.

**Why.** NOAA's own FAQ uses snake_case field names (`base_date_time`,
`call_sign`, `vessel_type`) while the bulk CSVs use PascalCase. AccessAIS and the
bulk download site differ in format and structure, and a 2026 upgrade is
scheduled to unify them. Format drift is expected, not hypothetical. This also
satisfies the `CLAUDE.md` rule to fail loudly rather than degrade silently.

**Test.** `test_schema_contract` against the fixture.

---

### D-112 — The schema contract is snake_case, and two columns are renamed

**Status:** ACCEPTED | **Evidence:** OBS-PHASE0, SRC-NOAA-FAQ
**Supersedes:** D-012

**Do this.** Validate on every ingested file that the header is exactly, in
order: `mmsi`, `base_date_time`, `longitude`, `latitude`, `sog`, `cog`,
`heading`, `vessel_name`, `imo`, `call_sign`, `vessel_type`, `status`,
`length`, `width`, `draft`, `cargo`, `transceiver`. On mismatch, abort the
partition and raise with the observed header in the message.

Note `longitude` precedes `latitude`, and both `length` and `width` are
`BIGINT`, not `DOUBLE`. `heading` is `BIGINT`.

**Why.** OBS-PHASE0 ran `DESCRIBE` against a 2022 `csv2` file. The header is
snake_case throughout, and two columns are **renamed, not merely re-cased**:
`LAT` → `latitude`, `LON` → `longitude`, `TransceiverClass` → `transceiver`.
D-012's asserted header would have aborted every partition.

This is the D-012 principle working as intended: the contract caught a real
drift on the first file. Keep the abort-loudly behaviour.

**Caution.** The observed header is from **2022**. SRC-NOAA-FAQ documents that
paths, naming and compression vary by year and that a 2026 format unification
is planned. Do not assume this header holds across the 2020–2026 backfill.
The manifest step must record the observed header per year, and a mismatch is
a finding, not a bug to be worked around.

**Test.** `test_schema_contract` against the 2022 fixture;
`test_schema_contract_records_per_year_header`.

---

### D-013 — Port polygons come from config and are hand-drawn for v1

**Status:** PROPOSED | **Evidence:** LIT-IMF-WORLD-2020, LIT-YAN-2022, LIT-BERTH-2025

**Do this.** Port polygons live in the port config file (already required by
`CLAUDE.md`). For v1, draw them by hand from charts and satellite imagery, with a
separate **anchorage** polygon and **berth area** polygon per port where the
distinction is visible. Record the provenance and draw date of each polygon in
the config.

**Why.** LIT-IMF-WORLD-2020 argues against hand-drawn polygons — the World Port
Index lists 3,669 ports and boundaries go stale — and derives them by
unsupervised learning instead. That argument is strong at global scale. It is
weak at ours: we cover a handful of US ports, we can inspect every polygon, and
hand-drawn boundaries are auditable in a way a clustering output is not.
LIT-YAN-2022 corroborates that public port point data is unreliable (Natural
Earth omits Quanzhou Port entirely), which cuts against using an off-the-shelf
set.

Useful priors from LIT-YAN-2022 when drawing: anchorages are generally within
5 nautical miles of the port area, and a 10 km buffer around a port captures
them; over 90% of port waters are shallower than 100 m.

**Rejected.** Derived polygons for v1 — defer to D-024. Off-the-shelf port point
databases — demonstrated incomplete.

**Known risk, must be handled.** LIT-IMF-WORLD-2020 warn that vessels **transit**
polygons without calling, giving the examples of ships crossing San Francisco
Bay to reach Oakland and ships stepping on a polygon through the Singapore
Strait. Polygon entry is **not** a port call. See D-020 to D-023.

**Revisit if.** Port count grows beyond what can be hand-verified, or polygon
staleness is detected.

---

### D-014 — Sentinels are converted to NULL in silver, never dropped or imputed

**Status:** SUPERSEDED by D-114 | **Evidence:** SRC-NOAA-FAQ

> Retained for history. The intent is right and carries forward. The magic
> values it converts do not exist in the `csv2` product — see D-114.

**Do this.** In silver, map `SOG = 102.3` → NULL, `COG = 360.0` → NULL,
`Heading = 511` → NULL. Log the count converted per column per partition. Do not
drop the row — the position and timestamp are still valid. Do not interpolate a
replacement.

**Why.** These are documented "unavailable" markers, not measurements. Left as
floats they silently poison every aggregate: a single `SOG = 102.3` in an
`AVG(SOG)` over a polygon-hour is undetectable in the output and wrong. Dropping
the row loses a valid position fix. Imputing violates the `CLAUDE.md` rule
against fabricating data.

**Note.** NOAA's FAQ describes `Heading = 511` as a pre-2015 convention. It is
the NMEA standard and **does appear in current files**. Verify sentinel presence
against the data, not the documentation.

**Test.** `test_no_sentinels_in_silver`, plus `test_sentinel_counts_logged`.

---

### D-114 — The csv2 product nulls unavailable values; test for NULL, not magic numbers

**Status:** ACCEPTED | **Evidence:** OBS-PHASE0, SRC-NOAA-FAQ
**Supersedes:** D-014

**Do this.**

- Treat `sog IS NULL`, `cog IS NULL`, `heading IS NULL` as the unavailable
  markers. Do **not** test equality against `102.3`, `360.0` or `511`.
- `draft` keeps the `= 0 OR IS NULL` convention — unchanged.
- Log null counts per column per partition, as D-014 required.
- Do not drop the row; position and timestamp remain valid. Do not impute.

**Why.** OBS-PHASE0 found **zero** exact matches for all three magic values
across 90,588 rows, while `max(cog) = 359.9` and `max(heading) = 359` — both
below their sentinel values, which they would exceed if the convention were
in use. Null counts were substantial: `cog` 5,449 and `heading` 32,336. The
17.9% figure previously cited in `docs/` came from the **retired `.zip`
product**, not from `csv2`.

Had D-014 shipped as written, the silver layer would have converted nothing,
logged zero, and passed `test_no_sentinels_in_silver` — a green test proving
the opposite of what it claims. That is the exact silent-degradation failure
`CLAUDE.md` exists to prevent.

**Observed availability, Sabine Pass box, 2022-03-15 (n = 88,742):**

| Field | Null | Share |
|---|---|---|
| `sog` | 15 | 0.02% |
| `cog` | 5,449 | 6.1% |
| `heading` | 32,336 | 36.4% |

**Consequence for D-025.** `heading` is missing on over a third of messages
across the whole box. That population is dominated by small craft, so the
rate in the Class A / ≥250 m population is expected to be far lower — but it
must be measured before heading variance is relied on for berth-versus-
anchorage classification. See the caution added to D-025.

**Test.** `test_unavailable_values_are_null_not_sentinel` — asserts the
magic values are *absent* and that null counts are non-zero, so the test
fails loudly if a future year reverts to the magic-value convention.

---

### D-015 — The vessel key is (MMSI, segment), not MMSI

**Status:** PROPOSED | **Evidence:** LIT-HARATI-2007, LIT-IMF-PORTWATCH, SRC-NOAA-FAQ

**Do this.** Build a `vessel_segment_id` by splitting each `mmsi`'s message
stream where any of the following occurs:

- Implied speed between consecutive fixes exceeds the D-018 threshold.
- `vessel_type`, `length` or `width` changes mid-stream.
- A gap longer than the **state-conditioned** vessel-level threshold in D-119.

Join everything downstream on `vessel_segment_id`. Use `imo` where present as a
secondary check, never as the primary key.

**Why.** MMSI is not unique and is not stable. Not unique: LIT-HARATI-2007 found
MMSI `1193046` broadcast by 25 distinct ships, `0` by 5, `1` by 3, `999999999`
by 3. Not stable: LIT-IMF-PORTWATCH defines MMSI as assigned to the AIS system,
always present, but **not permanent** — it may change with flag or ownership —
against IMO which is permanent to the hull but only "usually" present.
SRC-NOAA-FAQ confirms transponders are transferred between owners and vessels.

**Rejected.** MMSI as primary key. IMO as primary key — absent for most tugs,
barges and domestic vessels.

**Caution.** `imo`, `call_sign`, `vessel_name`, `length`, `width` and
`vessel_type` are populated or corrected by NOAA's AVID model from 2024,
affecting roughly 10–15% of records. Uniqueness or consistency checks on these fields test AVID,
not the fleet. Do not treat agreement as validation.

**Test.** `test_vessel_segments_split_on_teleport`, using a synthetic MMSI with
two vessels' tracks interleaved.

---

### D-016 — Exclude non-vessel and known-bad MMSIs from silver

**Status:** PROPOSED | **Evidence:** LIT-HARATI-2007, LIT-ANDROJNA-2021

**Do this.** Drop from silver, logging counts and reason per `CLAUDE.md`:

| Pattern | What it is |
|---|---|
| `99MIDxxxx` | Aid to Navigation — buoy or beacon |
| `98MIDxxxx` | Craft associated with a parent ship |
| `97xxxxxxx` | SART / MOB / EPIRB devices |
| `00MIDxxxx` | Coast / base station |
| `111MIDxxx` | SAR aircraft |
| `0`, `1`, `1193046`, `999999999` | Documented default or shared identities |
| Not exactly 9 digits | Malformed |

**Why.** Aids to Navigation sit permanently inside port polygons at zero speed
and are indistinguishable from a berthed vessel to any speed-and-position rule.
Left in, every port carries a constant number of phantom occupied berths. The
default MMSIs are documented in LIT-HARATI-2007; `999999999` also appears in
LIT-ANDROJNA-2021 as a spoofing artefact and as a value shared across naval
vessels.

**Test.** `test_no_non_vessel_mmsi_in_silver`.

---

### D-017 — Filter to cargo and tanker on VesselType, and monitor the correction-regime breaks

**Status:** SUPERSEDED by D-117 | **Evidence:** SRC-NOAA-FAQ, LIT-HARATI-2007, LIT-BERTH-2025

> Retained for history. The regime-break monitoring carries forward verbatim.
> The filter itself is too broad for the LNG scope — see D-117.

**Do this.**

- Filter silver to cargo and tanker groups using **NOAA's published AIS Vessel
  Type and Group Codes table**, not hand-rolled numeric ranges. Handle the
  4-digit AVIS codes (>999) present in 2015–2017.
- Emit a per-partition diagnostic: count of rows where `VesselType != Cargo`
  (the number of records NOAA corrected that day).
- Plot weekly vessel counts with markers at **2018-01-01** and **2024-01-01**
  before trusting any cross-period comparison.

**Why.** LIT-BERTH-2025 scopes to the same population, so the choice has
precedent. But the field is unreliable and non-stationary. Unreliable:
LIT-HARATI-2007 found 74% of observed ship types unsatisfactory (56% in their
larger sample) and the category list collapses container, car carrier and bulk
carrier into "cargo". Non-stationary: SRC-NOAA-FAQ documents AVIS corrections
2015–2023 with 4-digit codes used directly 2015–2017 then mapped to best-fit NMEA
types 2018–2023, and AVID from 2024 correcting 10–15% of records. **A backfill
across those boundaries has a population discontinuity that is not a change in
traffic.**

**`Cargo` is not cargo.** For 2015–2023 the `Cargo` field holds the *original
uncorrected NMEA vessel type*, moved there when AVIS overwrote `VesselType`. Do
not use it to infer what a ship is carrying. Its only legitimate use is as an
audit trail for NOAA's corrections.

**Test.** `test_vessel_type_filter_uses_noaa_group_table`;
`test_avis_4digit_codes_handled`.

---

### D-117 — Silver filters on transceiver class and size, not on vessel type

**Status:** PROPOSED | **Evidence:** OBS-PHASE0, LIT-HARATI-2007, SRC-NOAA-FAQ
**Supersedes:** D-017

**Do this.** Filter silver to `transceiver = 'A'` and `length >= 250`. Carry
`vessel_type` through as a **reported attribute**, never as a filter
predicate. All regime-break monitoring from D-017 — the AVIS 4-digit codes,
the 2018-01-01 and 2024-01-01 markers, the `cargo`-is-not-cargo warning —
carries forward unchanged.

**Why.** Two findings from OBS-PHASE0.

Class A dominates the population that matters: 187 vessels and 84,238
messages against 19 vessels and 6,350 messages for Class B. Filtering on
transceiver class is cheap and removes the small-craft population that drives
the `heading` and `draft` missingness in D-114.

Filtering on `vessel_type` is the trap. Among ≥250 m vessels in the box, only
three type values appeared: `80` (7 vessels), `84` (1), `60` (2 cruise
ships). LNG carriers appeared under **both** 80 and 84 — LA SEINE reported 84
while five other LNGCs reported 80. Any hand-rolled range would have to
include both, at which point it is not discriminating between LNG and crude
anyway, and D-005's berth polygon is doing the real work.

**Caution.** The `length >= 250` floor is **uncited** and is a scope choice,
not a literature-backed threshold. It is set below the 290–300 m LNGC class
with margin for dimension error (LIT-HARATI-2007: 36.3% of vessels off by
1–5 m, 6.4% reporting zero length). It must be sensitivity-tested at 200 m
and 270 m, and it interacts with D-005's null-tolerance requirement — a
vessel with null `length` is excluded by this filter and must be counted in
the coverage ceiling rather than silently dropped.

**Test.** `test_vessel_type_is_not_a_filter_predicate` (static check on the
SQL); dimension-coverage rate emitted per partition.

---

### D-018 — Quarantine on implied speed, never on position plausibility

**Status:** PROPOSED | **Evidence:** LIT-EMMENS-2021, SRC-NOAA-FAQ, LIT-ANDROJNA-2021

**Do this.** Compute Haversine distance between consecutive fixes within a
vessel stream, derive implied speed, and quarantine (not delete) segments where
implied speed exceeds **40 knots**. Log quarantined rows with the reason.

Do **not** filter positions for being over land. Do **not** filter positions for
being far from shore.

**Why.** SRC-NOAA-FAQ states both anomalies are expected: points over land arise
from GPS error, narrow inland waterways and vessels moved by land; points far
beyond radio range usually result from tropospheric ducting and high-elevation
receivers, and NOAA says explicitly these should not be treated as erroneous.
LIT-ANDROJNA-2021 gives the physics — a 1,028 m base station receives a 30 m ship
antenna at nearly 94 NM against a 40 NM nominal range. A land mask or a
distance-from-shore filter would delete real port traffic and real receptions.

The detectable failure is **kinematic**. LIT-EMMENS-2021 compute Haversine
distance between consecutive points precisely to do this, and find transmitted
SOG frequently disagrees with computed speed. An implied speed of 40+ knots means
a bad position, a bad timestamp, or two vessels sharing an MMSI — all of which
D-015 needs to catch anyway.

**Rejected.** 40 knots is **uncited**. No source in the corpus gives a threshold
for this. It is chosen as comfortably above commercial vessel speeds (a
LIT-ANDROJNA-2021 spoofed target at 40 knots is described as implausible) and
must be sensitivity-tested, not defended as literature-backed.

**Test.** `test_implied_speed_quarantine`; report quarantine rate per partition.

---

### D-019 — Coverage gaps are detected, classified and reported — never filled

**Status:** SUPERSEDED by D-119 | **Evidence:** LIT-MARTINCIC-2021, SRC-NOAA-FAQ, LIT-EMMENS-2021

> Retained for history. The port-day coverage series, the outage taxonomy and
> the never-interpolate rule all carry forward verbatim. What it lacks is a
> **vessel-level** gap threshold — D-015 references "the D-019 coverage-gap
> threshold" and D-019 never defines one. See D-119.

**Do this.** Emit a per-port-per-day coverage series: message count, distinct
vessel count, and median messages per vessel-hour. Flag a gap when message count
falls outside N sigma of the trailing median. Classify each gap using
LIT-MARTINCIC-2021's taxonomy:

| Signature | Likely cause |
|---|---|
| One vessel's messages missing, others present | Transmitter fault or deliberate switch-off |
| All vessels in one area missing | Receiver outage or uncovered area |
| Nothing anywhere for a period | Provider or own-pipeline outage |

Mark affected port-weeks as degraded in the metrics output. Never interpolate.

**Why.** `CLAUDE.md` forbids filling gaps. LIT-MARTINCIC-2021 independently
reaches the same conclusion: outages can be ignored only when the vessel was
moored or anchored throughout or moving in a straight line, and otherwise the
right answer is to **mark the period as having missing data**. SRC-NOAA-FAQ
confirms gaps are real, undocumented as to cause, usually a few hours to a couple
of days, and usually limited to a few receiving stations.

**The reason this matters more here than in a normal pipeline.**
LIT-EMMENS-2021 Peril 5 establishes that AIS degrades in dense traffic — many
simultaneous transmissions cause message collision and transmission failure.
LIT-ANDROJNA-2021 gives the mechanism: 2,250 message slots per minute per
channel, and a base station saturating at over 2,048 distinct stations in
9 minutes. **Congestion is vessel density, and message loss is a function of
vessel density.** Our measurement error is correlated with the quantity being
measured, and suppresses it. The coverage series is therefore not just an ops
metric — it is a required control variable. See D-032.

**Test.** `test_gap_detection_flags_synthetic_outage`.

---

### D-119 — Gap thresholds are state-conditioned and empirically derived

**Status:** PROPOSED | **Evidence:** LIT-ANDROJNA-2021, LIT-IMF-MALTA-2019, SRC-NOAA-FAQ
**Supersedes:** D-019

**Do this.** Everything D-019 required, unchanged — the per-port-per-day
coverage series, the three-way outage taxonomy, degraded-week marking, and
never interpolating. Plus a **vessel-level** gap threshold, which D-019 omitted
and D-015 depends on:

- Derive vessel state first (D-020: `sog < 1.0` is stopped), on observed points
  only.
- Apply a gap threshold conditioned on that state and on `transceiver`.
- Set each threshold from the **observed 99th percentile** of inter-message
  intervals for that stratum, measured on real port-days. Not from the ITU
  schedule.
- Commit the observed interval distributions to `docs/sensitivity.md`.

**Why a single threshold is wrong.** NOAA down-samples to the nearest whole
minute, so every native reporting rate faster than 1/min collapses to one row
per minute. The graduated underway schedule — 2 s / 3.3 s / 6 s / 10 s / 30 s —
is unobservable in this data. The anchored and moored rates are the exception:
they are *slower* than the downsample and pass through intact.
LIT-ANDROJNA-2021 Table 2 gives **3 min** for a dual-channel Class A at anchor
or moored, against 10 s underway.

So expected inter-message interval is roughly **60 s underway and 180 s
stopped** — a threefold difference across the exact transition the pipeline
exists to detect. A 2-minute threshold marks every moored vessel permanently
gapped. A 10-minute threshold misses real underway dropouts.

**Note a conflict in the corpus.** LIT-IMF-MALTA-2019 Appendix 2 gives 6 minutes
while anchored; LIT-ANDROJNA-2021 Table 2 gives 3 min dual-channel / 6 min
single-channel, and Class A is dual-channel under SOLAS. ITU-R M.1371 adds that
a vessel at anchor or moored but moving faster than 3 kn reverts to the 10 s
rate. Two sources in our own corpus disagree by a factor of two. Resolve
empirically; adopt neither published figure.

**Rejected.** A single global gap threshold. A threshold derived from the ITU
schedule — it describes what the transponder *sent*, not what NAIS *received*,
and reception depends on VHF propagation, ground conductivity, receiver
sensitivity, antenna attenuation, shadowing and interference
(LIT-ANDROJNA-2021), plus undocumented NAIS sensor outages (SRC-NOAA-FAQ).
LIT-ANDROJNA-2021 also notes authorities can interrogate vessels for more
frequent reports, injecting variance no schedule predicts.

**Ordering constraint.** State determines expected rate; gaps are detected using
expected rate; state is derived from the data containing the gaps. Derive state
on observed points only, then apply gap rules as a second pass. Do not
co-estimate.

**Consequence under the LNG scope (D-004).** An LNG loading runs 12–24 h at
`sog < 1.0`, so a single call should produce on the order of 250–500 messages at
the 3-minute moored rate. A berth dwell with far fewer is a coverage problem,
not a short loading — and under D-240 the *departure* edge carries the headline
accuracy number, so a gap straddling departure is the expensive failure.

**Secondary use, not a detector.** The moored-to-underway transition triples
the observed message rate at departure. That is a free cross-check on the
departure timestamp. Do not promote it to a primary detector — D-030 requires
transitions to come from kinematics — but a rate change that disagrees with the
kinematic departure time by more than an hour is worth flagging.

**Test.** `test_gap_detection_flags_synthetic_outage` (from D-019, retained);
`test_gap_threshold_is_state_conditioned`;
`test_moored_vessel_at_3min_interval_not_flagged_as_gap`.

---

# Port call detection (gold)

### D-020 — "Stopped" means SOG < 1.0 knot

**Status:** PROPOSED | **Evidence:** LIT-YAN-2022

**Do this.** Use `vT = 1.0` knot as the stopped-speed threshold. Make it a config
parameter, not a literal. Run and report a sensitivity analysis at 0.5 and 2.0.

**Why.** LIT-YAN-2022 state that a ship at rest is not at zero speed — wind,
waves and currents keep it in a small positive range determined by stopping mode
and conditions — and set `vT = 1 knot` citing Pallotta et al. (2013), Wen et al.
(2019) and Yan et al. (2020b) as converging on that value in both literature and
practice. It is the best-supported single parameter in the corpus.

**Rejected.** `SOG = 0`. **Rejected:** `SOG < 0.1`, the split used by
LIT-EMMENS-2021. Their cut is appropriate for characterising noise in moored
versus sailing populations, but is not a berth-detection threshold: at 0.1 kn a
single berth stay fragments into dozens of stop/start events every time a moored
ship surges on its lines.

**Test.** `test_stop_threshold_from_config`; sensitivity results committed to
`docs/sensitivity.md`.

---

### D-021 — Trajectory adjacency thresholds

**Status:** PROPOSED | **Evidence:** LIT-YAN-2022

**Do this.** Candidate stop points require, against the previous point:
`SOG < vT` (D-020) **and** time gap `< 1.5 h` **and** distance `< 2 km`.

**Why.** Directly LIT-YAN-2022's `tT` and `disT`. Their method achieves precision
0.94 / recall 0.91 / F1 0.92 on stop point identification.

**Caveat, must be stated in any writeup.** `tT = 1.5 h` was tuned on their data —
88,706 records from 10 ships in the South China Sea — not derived. `disT = 2 km`
is literature-backed. Re-tune `tT` on our fixture rather than inheriting it, and
record the result.

---

### D-022 — Minimum stop duration is 1.5 h; point-count minimum is rescaled, not inherited

**Status:** PROPOSED | **Evidence:** LIT-YAN-2022, SRC-NOAA-FAQ

**Do this.** A stop segment qualifies if duration ≥ **1.5 h**. Do **not** adopt
LIT-YAN-2022's `Δn = 10` points minimum as written — express the minimum in
**time**, not message count, and if a point-count floor is needed, derive it from
the observed median message rate for that port-week.

**Why.** The duration threshold transfers. The point count does not. Our data is
downsampled to one minute (SRC-NOAA-FAQ), so 10 points is 10 minutes under
perfect reception — but under the congestion-correlated message loss of D-119, 10
points could span an hour or more. A fixed point-count floor therefore **silently
raises the effective minimum stop duration exactly in the conditions we care
about most.** That is a bias toward missing congestion, in the metric designed to
detect congestion.

**Test.** `test_min_stop_duration_is_time_not_count`.

---

### D-023 — Port call segmentation follows the Martinčič voyage rules

**Status:** PROPOSED | **Evidence:** LIT-MARTINCIC-2021

**Do this.** Within a port polygon, group messages by `vessel_segment_id`
(D-015), sorted by time:

- Consecutive messages with a gap **< 24 h** belong to the same port call.
- Split into two port calls where the gap **> 5 h** **and** the vessel moved
  **> 100 m** in that time.

**Why.** LIT-MARTINCIC-2021's voyage extraction rules, validated at Piraeus
against port authority FAL forms. The conjunction in the split rule is the
important part: a long gap with no movement is a moored vessel with a coverage
outage, not two calls. A long gap with movement is two calls.

**Also required: polygon entry is not a port call.** LIT-IMF-WORLD-2020 warn of
vessels transiting polygons — crossing San Francisco Bay to reach Oakland,
passing through the Singapore Strait. A port call requires a qualifying **stop**
(D-020 to D-022) inside the polygon, not merely a position inside it.

**Test.** `test_transit_does_not_produce_port_call` with a synthetic straight-line
crossing.

---

### D-024 — Berth polygons are not derived in v1

**Status:** PROPOSED | **Evidence:** LIT-BERTH-2025

**Do this.** Use hand-drawn berth-area polygons from the port config (D-013).
Do not implement the DBSCAN → augmentation → GMM pipeline in v1. Record it as
the v2 path.

**Why.** LIT-BERTH-2025 is the right method and covers Los Angeles, but two of
its steps do not transfer to our source:

1. **Spatial augmentation requires dimensions A/B/C/D** (the four antenna
   offsets) to sample points across the vessel's footprint. NOAA collapses these
   into `length` and `width` (SRC-NOAA-FAQ), so the footprint would have to be
   approximated from length, width and heading — a change to the method, not a
   reimplementation of it.
2. **Their AIS is hourly**, so DBSCAN `minPoints` is in units of hours. Their
   tuned values (Los Angeles: ε 19.797 m, minPoints 5, 49 GMM components
   non-geohash) cannot be lifted onto one-minute data without rescaling and
   re-tuning.

Their observation window finding does transfer and should be carried into v2:
performance improves monotonically from 3 days → 1 week → 2 weeks → 1 month,
because longer windows permit stricter DBSCAN parameters without over-pruning.
**Use a 1-month window.**

**Worth noting in the writeup.** LIT-BERTH-2025 found berths that public berth
labels omit or misrepresent, which is a caution against trusting any off-the-shelf
berth dataset we might otherwise reach for.

---

### D-025 — Heading variance classifies berth vs anchorage; do not adopt the 10° filter as-is

**Status:** PROPOSED | **Evidence:** LIT-YAN-2022, LIT-MARTINCIC-2021, LIT-BERTH-2025

**Do this.** Classify each qualifying stop as **berth** or **anchorage** using
positional spread and heading variance over the segment:

- Convergent positions + low heading variance → berth.
- Scattered, roughly circular positions + high heading variance → anchorage.

Do not use LIT-BERTH-2025's "drop records with heading change > 10°" rule at this
stage.

**Why.** Two independent sources converge on the same geometry. LIT-YAN-2022:
berth stopping segments have a convergent point distribution because the vessel
is physically fixed, while anchorage segments are scattered and tend to be
circular because the vessel swings on one point of attachment. LIT-MARTINCIC-2021
reach it from the kinematic side — their second status-correction approach uses
speed **and rotation**, explicitly to remove the dependency on port-supplied
anchorage and terminal GIS data, which we do not have.

The 10° rule is rejected here because it serves a different purpose in
LIT-BERTH-2025 — removing manoeuvring vessels *before* clustering, to sharpen
berth boundaries. Applied to stop classification it would discard the swinging
motion that is the anchorage signal.

**Why this matters economically.** Queue at anchor and time at berth are
different signals. Queue length plausibly leads throughput; berth time reflects
it. Collapsing them would destroy the lead the project exists to measure.

**Caution — heading availability must be measured first (D-114).** OBS-PHASE0
found `heading` null on 36.4% of messages across the whole Sabine Pass box.
That population is dominated by small craft and the rate in the Class A /
≥250 m population is expected to be much lower, but it has not been measured.
Before heading variance is used in classification, emit the null rate for the
post-D-117 population. If it exceeds roughly 10%, heading becomes a secondary
feature and positional spread carries the classification alone.

**Note under the LNG scope (D-004).** Anchorage stops are expected to be rare
at LNG terminals because loading slots are contracted ahead. The
classification is still required — it is what separates berth dwell from
everything else — but a low anchorage count is the expected result, not
evidence the classifier is broken.

**Stale under D-106.** This note previously said Houston was where the
classifier earns its keep. Houston is deferred to v2, so for the whole of v1
this classification runs only at terminals where anchorage stops are expected
to be rare. Keep the classifier — it is what separates berth dwell from
transit — but expect a low anchorage count everywhere in scope, and do not
tune it toward finding anchorages that are not there.

**Test.** `test_berth_vs_anchorage_classification` on labelled fixture segments;
`test_heading_null_rate_reported`.

---

### D-026 — `status` is a feature and a diagnostic, never a label

**Status:** PROPOSED | **Evidence:** LIT-MARTINCIC-2021, LIT-HARATI-2007

**Do this.** Derive berth/anchorage state geometrically (D-025). Then compute and
report the **agreement rate** between derived state and the reported `status`
field as a data quality metric. Never use `status` as an input to the derivation.

**Why.** Two independent studies, roughly two decades apart, agree that around
30% of vessels transmit incorrect navigational status. LIT-MARTINCIC-2021 give
the diagnostic example: vessels can only be moored at terminals, yet moored
status appears in anchorage areas and while sailing. They also make the structural
point that static fields can be corrected against vessel databases but **dynamic
self-reported fields cannot**, because no external register records what a
vessel's status was at a given moment.

**Expected result.** Agreement near 70% independently reproduces the literature
and is worth reporting. **Agreement near 99% means `status` has leaked into the
derivation** — treat it as a bug signal, not a success.

**Not adopted:** LIT-MARTINCIC-2021's third approach (KNN on spatial and
kinematic fields, ≥ 300 neighbours). The geometric features are interpretable and
testable; revisit only if geometric classification underperforms.

**Test.** `test_status_not_used_in_derivation` (static check on the SQL);
agreement rate emitted per partition.

---

# Metrics

### D-030 — Congestion metrics must be robust to message loss

**Status:** PROPOSED | **Evidence:** LIT-EMMENS-2021, LIT-ANDROJNA-2021

**Do this.** Prefer metrics derived from **state transitions** (arrival and
departure events, vessel-hours integrated from those events) over metrics derived
from **sampling** (mean concurrent vessel count per minute).

**Why.** Under message collision a dropped message looks like an absent ship, so
a sampled concurrency count understates congestion precisely when congestion is
highest. A metric built from arrival and departure transitions survives sparse
sampling, because the transition only needs to be observed once.

**Rejected.** Mean concurrent vessels sampled per minute as the primary metric.
Acceptable as a secondary metric *if* published alongside the D-119 coverage
series.

**Terminology under D-004 and D-033.** This decision predates the LNG scope and
says "congestion". Congestion is not a v1 deliverable: D-004's granularity
output is loading duration and inter-arrival spacing, and D-033 shows berth
occupancy is saturated at Sabine Pass. The ruling is unchanged and still
binding — transition-derived metrics over sampled ones — but read "congestion
metric" as "granularity metric" throughout.

---

### D-031 — Weekly series are reported raw; smoothing is presentation only

**Status:** PROPOSED | **Evidence:** LIT-IMF-MALTA-2019

**Do this.** Store raw weekly values. Apply a **five-term centred moving average**
for charts only, and label it. Never feed smoothed values into the lead-lag
estimation.

**Why.** LIT-IMF-MALTA-2019 use a five-term centred MA on their weekly cargo
number indicator because the weekly series carries a substantial noise component
— precedent for the choice of window. But a **centred** moving average uses
future values, which would leak information into a lead-lag estimate and
manufacture the result the project is testing for. Presentation only, and that
distinction must be explicit in the code.

**Note under D-004.** The lead-lag estimation this decision protects is no
longer a deliverable. The ruling stands regardless — smoothed values must not
feed any estimate, and a centred moving average still uses future data — but
the test name should be read as protecting the accuracy and granularity
metrics, not a lag estimate.

**Test.** `test_leadlag_input_is_unsmoothed`.

---

### D-032 — Publish the message-density diagnostic alongside every congestion metric

**Status:** PROPOSED | **Evidence:** LIT-EMMENS-2021, LIT-ANDROJNA-2021

**Do this.** For every port-week, publish messages-per-vessel-per-hour beside the
congestion metric. Report their correlation. If it is materially negative,
state so and discuss it as a measurement floor on the congestion estimate.

**Why.** This is the project's most serious internal validity threat (see D-119).
It cannot be eliminated with this data. It can be measured, reported, and
reasoned about — and doing so is the difference between having read the
literature and having understood it. No source in the corpus quantifies this
relationship, so the diagnostic is our own contribution, not a replication.

---

# Validation and analysis

### D-040 — AIS-to-official matching uses a ±2 day window on IMO

**Status:** SUPERSEDED by D-140 | **Evidence:** LIT-IMF-WORLD-2020

> Retained for history. The window reasoning carries forward. The join key
> does not: DOE publishes tanker *names*, not IMO numbers — see D-140.

**Do this.** When reconciling derived port calls against official records, match
on `IMO` within **±2 days** of the official record date. Break ties by closest
date, then by earliest record. Classify every record as matched, false negative
(official call with no AIS equivalent) or false positive. Publish all three
counts.

**Why.** This is LIT-IMF-WORLD-2020's procedure, and their rationale for each
side of the window is specific and US-applicable: **2 days back** because US law
gives vessels up to two days to file an entrance statement; **2 days forward**
because official records use local US date while AIS is UTC (5–9 hours ahead
depending on zone and DST), and because a vessel may file on arrival at anchorage
before entering the port.

**Note.** They match on IMO, not MMSI. Our IMO coverage is incomplete
(SRC-NOAA-FAQ), so the matchable population is a subset — quantify it and say so.

---

### D-140 — Match AIS laden departures to DOE cargoes on tanker name and departure date

**Status:** SUPERSEDED by D-240 | **Evidence:** SRC-DOE-FE746R, LIT-IMF-WORLD-2020

> Retained for history. The join key, the departure-not-arrival finding and
> the ±2 day window all carry forward. The instruction to match fuzzily does
> not — it is actively unsafe on this name population. See D-240.

**Do this.** The matching key is **(export terminal, LNG tanker name, departure
date)**, within a ±2 day window. Classify every record as matched, false
negative (DOE cargo with no AIS equivalent) or false positive (AIS departure
with no DOE cargo). Publish all three counts and the volume error on matches.

**Why the key changed.** DOE's reporting order requires each cargo to report
the export terminal, country of destination, date of departure, name of the
LNG tanker, supplier and volume in Mcf (SRC-DOE-FE746R). It does **not**
publish IMO numbers. `vessel_name` is therefore the only available join key —
which is unfortunate, because it is a static field NOAA's AVID model
populates and corrects from 2024 (SRC-NOAA-FAQ). Name matching must be fuzzy
(case, spacing, punctuation; OBS-PHASE0 observed `ADVENTURE OFTHE SEAS` with
a missing space) and the match rate reported as a diagnostic.

**Why the event changed.** DOE records the **departure**, not the arrival. The
detector's departure edge is therefore load-bearing for the headline accuracy
number and the arrival edge is not. Threshold choices should be tuned
accordingly, and a departure misdated across a day boundary counts as both a
false positive and a false negative.

**Why the window transfers.** LIT-IMF-WORLD-2020's ±2 days remains
appropriate: 2 days back for filing latency, 2 days forward for the local-
versus-UTC date discrepancy (D-011).

**The key is also fragile, not just imperfect.** `vessel_name` is a type 5
field broadcast once per 6 minutes and is the first thing lost at the edge of
reception — see D-045. Departures with a null name must be counted as
*unmatchable*, separately from false negatives, or the headline recall number
absorbs a reception problem and reports it as a detector problem.

**Test.** `test_doe_match_is_fuzzy_on_name`; publish match rate, precision,
recall, volume error **and unmatchable count** per terminal-month.

---

### D-044 — DOE records need three exclusions before matching

**Status:** SUPERSEDED by D-144 | **Evidence:** SRC-DOE-FE746R

> Retained for history. The three exclusions and the "second measurement,
> not ground truth" framing carry forward. The `[*]` split-cargo marker it
> specifies does not exist in the transaction file. See D-144.

**Do this.** Before matching under D-140, from the DOE transaction-level data:

1. **Deduplicate split cargoes** on (tanker, departure date, terminal). DOE
   defines a split cargo as one physical shipment whose portions have
   different buyers, suppliers, prices, loading ports or authorisations, and
   **counts it as multiple cargos**, flagged `[*]`.
2. **Exclude ISO container exports.** DOE separates LNG by vessel from LNG in
   ISO containers. The latter is not a jetty loading.
3. **Exclude re-exports** of previously-imported LNG, which DOE reports in a
   separate table.

Log the count removed at each step.

**Why.** Uncorrected, split cargoes appear as DOE rows with no AIS equivalent
and depress recall by an amount that has nothing to do with the detector. A
single vessel loading can be two or three DOE rows.

**Also required in the README: DOE is a second measurement, not ground truth.**
The data is self-reported by authorisation holders on Form FE-746R
(SRC-DOE-FE746R). When AIS and DOE disagree, the honest framing is that two
independent measurements disagree and here is the likely cause — not that the
pipeline was wrong.

**Source discontinuity.** On 17 November 2023 FECM replaced the *LNG Monthly*
with *Natural Gas Imports and Exports Monthly*. The 2020–2026 study window
straddles that change. Build the DOE loader for two formats from the start.

---

### D-045 — `vessel_name` is a type 5 field; measure its null rate as a match-failure bias

**Status:** PROPOSED | **Evidence:** LIT-ANDROJNA-2021, SRC-NOAA-FAQ, OBS-PHASE0

**Do this.** Emit, per terminal-month, the share of **departure events** whose
`vessel_name` is null or blank, alongside the D-240 match rate. Report it as a
ceiling on achievable recall. A departure with no name is an unmatchable event,
not a detector failure, and the two must be reported separately.

Do the same for `imo`, `length` and `width` — D-117 already requires
dimension-coverage reporting; this extends it to the departure event
specifically and to the join key.

**Why.** The CSV row is a denormalised join of two message streams arriving at
different rates. Position, `sog`, `cog` and `heading` come from Class A position
reports (types 1/2/3) at up to 60/min before downsampling. Everything static —
`vessel_name`, `imo`, `call_sign`, `vessel_type`, `length`, `width`, `draft` —
comes from a **type 5 message broadcast once per 6 minutes**
(LIT-ANDROJNA-2021 Table 2). NOAA forward-fills between them.

Three consequences, in increasing severity:

1. **Static columns are not "as of" their row's timestamp.** They are the last
   received type 5, up to six minutes stale — or, from 2024, AVID's imputation
   rather than a broadcast at all (SRC-NOAA-FAQ).
2. **Type 5 is a longer message** — 424 bits over two slots against one — so it
   is more fragile at range and under VHF contention. LIT-ANDROJNA-2021 states
   directly that vessels at the edge of AIS range are detected but no static
   ship data is available.
3. **Therefore null static fields are not missing at random.** They correlate
   with marginal reception, which correlates with distance from receiver and
   with VHF congestion. OBS-PHASE0 already observed the pattern: GASLOG
   GIBRALTAR, a known LNG carrier, reported neither beam nor draft across 575
   messages.

**Why this one is load-bearing rather than housekeeping.** D-240 makes
`vessel_name` the sole join key to DOE, because DOE publishes tanker names and
not IMO numbers. That key is a type 5 field. A departing carrier at the seaward
edge of the clip box may have position but no name — and a departure is exactly
where the vessel is heading *away* from the receiver. **The D-240 headline
accuracy number inherits type 5's fragility**, and without this diagnostic an
unmatchable event is indistinguishable from a missed one.

**Rejected.** Back-filling `vessel_name` from earlier in the same
`vessel_segment_id`. It is tempting, cheap, and would improve the match rate —
and it is imputation, which `CLAUDE.md` forbids. It would also inflate the
headline number by construction. If it is ever adopted it must be reported as a
separate, clearly labelled variant, never as the primary result.

**Test.** `test_departure_events_report_name_null_rate`; publish
unmatchable-count separately from false-negative count in every D-240 table.

### D-041 — Treat IMF PortWatch as a benchmark, not an independent instrument

**Status:** PROPOSED | **Evidence:** LIT-IMF-PORTWATCH

**Do this.** Use PortWatch for comparison and sanity-checking. Do **not** treat
agreement with PortWatch as independent validation of a container-trade signal at
a major container port.

**Why.** LIT-IMF-PORTWATCH documents that the netting adjustment for container
ships is implemented for **83 of the largest container ports, covering about
two-thirds of global containerised trade, using official container throughput
data**. At exactly those ports, PortWatch's container estimates are partly
calibrated on the official statistics port_pulse is trying to lead. Agreement
would be partly circular.

PortWatch also applies **backpropagation** (using the next port's reported draft,
handling two-thirds of incomplete draft data) and **historical averaging** for the
remainder. Both are imputation, which `CLAUDE.md` forbids in our pipeline. The
two series are therefore not constructed on comparable rules, and differences
should not be read as one being wrong.

---

### D-042 — Validate against 2020 as a natural experiment

**Status:** SUPERSEDED by D-142 | **Evidence:** LIT-VERSCHUUR-2021

> Retained for history. The reasoning — that a metric failing to show a known
> exogenous shock is broken — carries forward in full. The shock changes:
> under the LNG scope a terminal-level outage is sharper than a global
> pandemic. See D-142.

**Do this.** Run the full pipeline over 2019–2021 and check that the congestion
metric registers the COVID-19 disruption. Compare direction and rough magnitude
against LIT-VERSCHUUR-2021 (global port calls −4.4% Jan–Aug 2020 vs 2019, for
trade-carrying vessels only).

**Why.** A metric that fails to show 2020 is broken, and this is a cheap, sharp
falsification test available before any lead-lag estimation. Compare on
trade-carrying vessels only — per D-002 and LIT-VERSCHUUR-2021's own finding that
scope drives the headline number.

**Caution.** Their figure is global and provider-sourced; US-only, NOAA-sourced,
port-level numbers will differ. Check sign and order of magnitude, not equality.

---

### D-142 — The Freeport 2022 outage is the out-of-sample validation

**Status:** PROPOSED | **Evidence:** SRC-DOE-FE746R, LIT-VERSCHUUR-2021
**Supersedes:** D-042

**Do this.** Fit and tune the detector on Sabine Pass and Corpus Christi only.
Then run it unmodified over Freeport across 2022–2023 and check that cargo
events collapse to approximately zero from the June 2022 incident and recover
on the documented timeline. Verify the exact dates against DOE records rather
than from memory.

**Why.** This is a stronger test than D-042's COVID check on three counts. The
effect is near-total rather than a few percent, so statistical power is not a
concern. The terminal is held out of tuning, so it is genuinely out of sample.
And the ground truth is exact — DOE records the cargo count at that terminal
month by month — rather than a global figure requiring an order-of-magnitude
comparison.

**This is the headline result.** A detector that reproduces an eight-month
terminal shutdown it was never shown is worth more than any correlation
coefficient. Put it at the top of the README.

**Caution.** Freeport is excluded from tuning *and* from any threshold
sensitivity analysis. If a threshold is adjusted after seeing Freeport
results, the test is no longer out of sample and must be described as
in-sample in the writeup.

---

### D-043 — Adopt the Feng port-state vocabulary

**Status:** PROPOSED | **Evidence:** LIT-FENG-2020

**Do this.** Name port call states consistently: `inbound_transit`,
`anchored_inbound`, `at_berth`, `anchored_outbound`, `outbound_transit`.

**Why.** LIT-FENG-2020's decomposition (VTS Line-to-Berth, Anchored in Anchorage
during inbound, Moored at Berth, Anchored in Anchorage during outbound,
Berth-to-VTS Line) is a clean and published vocabulary, and using it makes the
gold schema legible to anyone who knows the literature.

**Not adopted:** their per-kilometre normalisation of transit legs (VBT, BVT).
It requires a defined route distance and a VTS line from nautical charts, neither
of which we have. Their berth-time-by-ship-size breakdown is worth revisiting if
`length` proves reliable enough.

---

# Explicitly out of scope

### D-050 — No draught-based cargo or trade volume estimation

**Status:** PROPOSED | **Evidence:** LIT-IMF-WORLD-2020, LIT-IMF-MALTA-2019, LIT-IMF-PORTWATCH

**Do this.** Do not implement cargo weight or trade volume estimation. Metrics are
congestion and timing only.

**Why.** Three reasons, any one sufficient.

1. **The inputs do not exist.** Both IMF formulas need DWT and design draught.
   SRC-NOAA-FAQ states tonnage is not part of the AIS broadcast and must come
   from third-party vendors. No vessel register is in scope.
2. **The methods require imputation that `CLAUDE.md` forbids.**
   LIT-IMF-WORLD-2020 impute ballast draught from a type × DWT-tertile median
   ratio; LIT-IMF-PORTWATCH backpropagate draft from the next port and fall back
   to historical averaging. These are best practice for a *level* estimate and
   are incompatible with our no-fabrication rule.
3. **The field is weak.** SRC-NOAA-FAQ notes modern entries typically report
   maximum static draft rather than laden draft. LIT-IMF-MALTA-2019, citing Jia
   et al. (2015), conclude draught is unreliable per-ship and usable only in
   aggregate within a type and size category. LIT-VERSCHUUR-2021 note draft is
   less frequently reported in some regions, introducing geographic bias.

**The rule conflict is deliberate and should be stated in the writeup.** Our
no-imputation rule is correct for a pipeline whose output is a *timing* claim,
where fabricated values would manufacture the very leads being tested. The IMF
rule is correct for a platform whose output is a *level* estimate. Being able to
articulate that distinction is the point.

**Scope narrowed under D-004.** This decision forbids **draught-based**
estimation, and that prohibition stands. It does not settle whether LNG cargo
volume can be estimated by other means — per-class capacity constants, or a
vessel capacity register. That question was D-053 and is now **resolved by
D-153**: per-vessel historical median with a temporal holdout, benchmarked
against a global constant. Do not read D-050 as having closed it, and do not
read D-153 as reopening draught — D-050 stands unchanged.

**Ratification note.** D-153 is `ACCEPTED` and explicitly rests on this
decision standing. An accepted decision resting on a proposed one is an
inversion worth closing: either ratify D-050 or demote D-153.

---

### D-051 — Container ships are not separately identifiable; the validation target adapts

**Status:** WITHDRAWN — no longer in scope under D-004
**Evidence:** LIT-HARATI-2007, SRC-NOAA-FAQ

> Retained for history. Container throughput is not a validation target under
> the LNG scope, so the TEU mismatch this decision was blocking on no longer
> arises. The underlying finding — that AIS type codes cannot isolate a
> commercial vessel class — carries forward into D-005 and D-117, where the
> same problem recurs for LNG carriers and is solved by berth geometry rather
> than by a type filter. This decision no longer blocks anything.

**The problem.** AIS type 70–79 is "cargo". LIT-HARATI-2007 establishes the
category list collapses container vessels, car carriers and bulk carriers into a
single code, and this is a property of the AIS specification, not a data quality
issue. It cannot be fixed by cleaning.

If the validation target is container throughput in **TEU** — which is what most
US port authorities publish monthly, and what a reader will assume was used —
then a vessel population including bulkers, break-bulk and ro-ro is being
regressed against a container-only series. The IMF papers avoid this by joining
commercial vessel registers for ship class; we have none.

**Options, needs a ruling:**

| Option | Trade-off |
|---|---|
| (a) Validate against **total cargo tonnage** rather than TEU | Cleanest story, no proxy needed. Fewer US ports publish it, and it is a less familiar headline. |
| (b) Use `Length`/`Width` bands as a vessel-class proxy | Keeps TEU as the target, but the proxy is unvalidated and the dimension fields are AVID-imputed for 10–15% of records. |
| (c) Add a free vessel register to scope | Solves it properly. Adds a dependency and an ingest path. |

**Recommendation:** (a). It is defensible without a proxy, and "I changed the
validation target because AIS cannot isolate container ships, and here is the
source that says so" is a stronger answer than an unexplained approximation.

**This decision blocks D-040** and the choice of throughput series. Resolve
before writing the validation layer.

---

### D-053 — LNG cargo volume estimation method

**Status:** RESOLVED by D-153 | **Evidence:** SRC-DOE-FE746R, LIT-IMF-WORLD-2020, OBS-PHASE0

**The problem.** D-004 commits to reporting volume error per cargo. That
requires an AIS-side volume estimate. D-050 forbids the draught-based route.
No vessel capacity register is in scope, and AIS does not broadcast capacity.

**Options, needs a ruling:**

| Option | Trade-off |
|---|---|
| (a) Per-class capacity constants from `length`/`width` bands | No new dependency. The bands are unvalidated and D-114 shows some LNGCs report no beam at all. |
| (b) Ballast-to-laden `draft` change as a relative signal | Uses a field already present, and OBS-PHASE0 shows the ballast/laden spread is real (9.3 m vs 11.6 m). But it is per-ship unreliable (LIT-IMF-MALTA-2019) and edges toward what D-050 forbids. |
| (c) Add a free vessel capacity register | Solves it properly. New dependency and ingest path. |
| (d) Drop volume; report cargo counts only | Honest and cheap. Loses one of the two D-004 deliverables. |

**Note.** (b) is not automatically barred by D-050 — D-050 forbids *imputing*
a missing draught and deriving an absolute cargo weight from it. Using an
observed change in a reported field as a relative indicator is a different
operation. Whether that distinction survives scrutiny is exactly what needs
deciding.

**Recommendation:** (d) for v1, with (a) explored and reported as an
experiment. Cargo count precision and recall is already a strong result and
does not depend on this.

**Resolve before writing the validation layer.**

---

### D-052 — No spoofing detection in v1

**Status:** PROPOSED | **Evidence:** LIT-ANDROJNA-2021

**Do this.** Do not implement spoofing detection. Rely on the D-018 implied-speed
quarantine, which catches the crude cases incidentally.

**Why.** LIT-ANDROJNA-2021's documented incidents are concentrated outside US
waters and are motivated by sanctions evasion and illegal fishing — Shanghai
"crop circles" over oil terminals, the Galápagos fishing fleet, the Stena Impero.
The one US case (four fake aids-to-navigation at Ponce de Leon Inlet, Florida,
2020) is an AtoN spoof, which D-016 excludes anyway. Risk is low but non-zero.

**Revisit if.** A port-week shows an implausible vessel count spike that the
D-119 coverage diagnostics do not explain. The signature to look for is a burst
of near-sequential MMSIs appearing simultaneously.

### D-106 — Port scope: three LNG terminals; Houston deferred to v2

**Status:** ACCEPTED | **Evidence:** OBS-PHASE0, OBS-PHASE0B
**Supersedes:** D-006

**Do this.** v1 covers three terminals:

| Port | DOE point-of-exit string | Role |
|---|---|---|
| Sabine Pass, LA | `Sabine Pass, LA` | Primary — largest, longest history |
| Freeport, TX | `Freeport, TX` | 2022 outage, out-of-sample test (D-142) |
| Corpus Christi, TX | `Corpus Christi, TX` | Growth and Stage 3 ramp |

Terminal attribution on the DOE side is **exact string equality** on the
values above. Never substring, never case-insensitive `contains`.

Sabine Pass clip box, unchanged from D-006:

```
latitude   28.8  to  29.9
longitude -94.2  to -93.5
```

**Why Houston is out.** OBS-PHASE0B searched every column of the DOE file for
"Houston": five rows match and all five are the tanker `Gaslog Houston`. No
point-of-exit value contains Houston. Houston therefore validates nothing
against the D-004 deliverables, while carrying the largest download of the
four candidates. The congestion metrics it would exercise (D-025, D-030) are
not D-004 deliverables.

Houston remains the right v2 addition — it demonstrates the port abstraction
generalises to a non-LNG port, which is an engineering claim worth making —
but it is cheap to add once the pipeline exists and expensive to carry now.

**The Cameron trap.** Cameron Parish, LA hosts three distinct LNG terminals.
DOE distinguishes them:

| DOE string | Terminal | In scope? |
|---|---|---|
| `Sabine Pass, LA` | Sabine Pass Liquefaction (Cheniere) | Yes |
| `Cameron, LA` | Cameron LNG (Sempra) | No |
| `Cameron (Calcasieu Pass), LA` | Calcasieu Pass LNG (Venture Global) | No |

`CLAUDE.md` describes Sabine Pass as "(Cameron Parish, LA)". That names the
parish, not the terminal, and any code matching on "Cameron" will pull in two
wrong terminals.

**Coherence note worth keeping.** These are the *same two terminals* that
forced the clip box's eastern edge from −93.4 to −93.5 in D-006. The same
confusion appears independently on the AIS side as geographic bleed and on
the DOE side as string collision. Both are now handled; the symmetry is a
useful sanity check that the terminal boundary is drawn in the right place.

**Test.** `test_terminal_attribution_is_exact_match` — asserts no `contains`,
`startswith` or case-folded comparison appears in terminal-matching code;
asserts `Cameron, LA` and `Cameron (Calcasieu Pass), LA` map to no in-scope
terminal.

---

### D-144 — Collapse DOE rows to physical loadings; no marker exists

**Status:** PROPOSED | **Evidence:** OBS-PHASE0B, SRC-DOE-FE746R
**Supersedes:** D-044

**Do this.** Before any matching, collapse DOE export rows to one row per
physical loading by grouping on **(Tanker, Arrival/Departure Date, Point of
Entry or Exit)** and summing `Volume (MMCF)`. Retain the constituent
destination countries as an array; do not discard them.

Apply the other two exclusions and **log all three counts even when zero**:

| Exclusion | Mechanism | Sabine Pass 2020–23 |
|---|---|---|
| Split cargoes | Group-and-sum, above | 1560 → 1482 rows (5.0%) |
| ISO containers | `Mode of Transport != 'Vessel'` | 0 rows |
| Re-exports | `Activity != 'Exports'` | 0 rows |

**The comparison unit is loadings, not rows.** Q1 2022 Sabine Pass is **105
loadings** (37 / 31 / 37 by month), not the 109 rows the raw file contains.
Four Q1 splits: Qogir 01-26, LNG Adventure 02-10, Amberjack LNG 02-21, La
Seine 03-14.

**Why the mechanism changed.** D-044 specified DOE's `[*]` split-cargo flag as
the primary method. OBS-PHASE0B searched every string column for `[*]` and
for a bare `*` and found zero matches, twice — file-wide and again scoped to
Sabine Pass. The marker is a footnote convention in DOE's published PDF
tables; it does not appear in the transaction-detail Excel. The grouping
check D-044 specified as a secondary cross-check is in fact the only
mechanism available.

**Why the grouping is trustworthy.** Across the 75 multi-row combos in the
Sabine Pass 2020–2023 subset, summed volume has mean 3,482 MMCF and standard
deviation 329 — squarely inside the normal single-cargo range. If these were
genuine duplicate reports rather than one loading split across destinations,
summed volumes would be roughly double a single cargo. They are not.

**Caution.** Zero ISO-container and zero re-export rows at Sabine Pass is a
property of this terminal, not of the dataset — both are non-zero at the
small Florida and Puerto Rico points of exit. Do not write exclusion logging
that assumes non-zero counts, and treat a non-zero count at an in-scope
terminal as a finding to investigate rather than a number to subtract.

**Test.** `test_doe_rows_collapsed_before_matching`;
`test_sabine_pass_q1_2022_is_105_loadings` as a regression fixture;
`test_all_three_exclusions_logged_including_zero`.

---

### D-240 — Name matching is normalisation plus a curated alias table, never edit distance

**Status:** ACCEPTED | **Evidence:** OBS-PHASE0B, OBS-PHASE0
**Supersedes:** D-140

**Do this.** Three stages, in order:

1. **Normalise both sides.** Uppercase, strip all whitespace and
   punctuation. `Castillo De Merida` and `Castillo DeMerida` both become
   `CASTILLODEMERIDA`; AIS `GASLOG GIBRALTAR` and DOE `Gaslog Gibraltar`
   both become `GASLOGGIBRALTAR`.
2. **Apply a hand-curated alias table** held in version control, one row per
   mapping, each with a `reason` field. Roughly a few dozen entries.
3. **Report the residual as unmatchable** under D-045. Never force a match.

**Do not use edit distance, Levenshtein, Jaro-Winkler, phonetic hashing, or
any similarity threshold.**

**Why.** OBS-PHASE0B found both of these in the same 473-name population:

| Pair | Edit distance | Truth |
|---|---|---|
| `Seapack Hispania` / `Seapeak Hispania` | 1 | **Same vessel** (typo) |
| `Bishu Maru` / `Bushu Maru` | 1 | **Different vessels** |

No threshold accepts the first and rejects the second. Any matcher tuned
tight enough to catch the typos will silently merge distinct vessels — and
the LNG fleet is full of systematic naming conventions that make this
common, not rare: the `___shu Maru` series (Bishu, Bushu, Enshu, Esshu,
Seishu, Nohshu, Sohshu, Shinshu), `Maran Gas ___`, `___ Knutsen`,
`BW Pavilion ___`, `Castillo de ___`, `Seapeak ___`, `Hoegh ___`.

A silent merge is worse than a miss. It inflates the match rate while
corrupting vessel identity, and it is invisible in the headline number.

**What each stage catches.** Normalisation alone resolves
`Castillo De Merida`/`DeMerida`, `GASLOG HONGKONG`/`Gaslog Hong Kong`, and
the AIS-versus-DOE casing difference — with no false positives, because
removing whitespace and case cannot merge two genuinely different strings.
The alias table is needed for `JPS Bora`/`JSP BORA` (transposition),
`Seapack`/`Seapeak Hispania` (typo, plus a probable Teekay→Seapeak rebrand)
and `Vivit Arabia`/`Vivit Arabia LNG` (suffix).

**Everything else from D-140 carries forward:** the join key is (terminal,
tanker name, departure date) with a ±2 day window; the matched event is the
**departure**, not the arrival; every record is classified as matched, false
negative, false positive or unmatchable, and all four are published.

**Caution — build the alias table from observations, not anticipation.** The
473 DOE names are one side. The AIS side will contribute its own variants,
and OBS-PHASE0 already found `ADVENTURE OFTHE SEAS` there. Entries are added
when an unmatched pair is observed and inspected, never pre-emptively. The
table's growth over the backfill is itself a reportable diagnostic.

**Test.** `test_no_similarity_matching` — static check that no edit-distance
or phonetic library is imported in the matching module;
`test_alias_entries_have_reasons`; `test_bishu_and_bushu_do_not_match`;
publish unmatched count alongside precision and recall.

---

### D-153 — Cargo count is the deliverable; volume is a benchmarked extension

**Status:** ACCEPTED | **Evidence:** OBS-PHASE0B, SRC-DOE-FE746R
**Resolves:** D-053

**Do this.** Report cargo count precision and recall as the headline D-004
accuracy result. Report volume error as a secondary extension with **three
estimators side by side**:

| # | Estimator | Role |
|---|---|---|
| (i) | Global constant — median volume of all in-scope cargoes | Baseline |
| (ii) | Per-vessel historical median, **fit on 2020–2022, applied 2023–2026** | Main |
| (iii) | Dimension-class median from AIS `length`/`width` | Fallback for vessels unseen in the fit window |

Report (i) always, so a reader can see the marginal value of (ii) and (iii).
Report seen-vessel and unseen-vessel errors separately.

**Why (ii) needs the holdout.** Predicting a vessel's cargo volume from that
vessel's DOE history and then validating against DOE is circular — AIS
contributes only identity and timing, so the reported error measures DOE's
own per-voyage variance, not pipeline accuracy. A temporal holdout makes it a
legitimate out-of-sample estimator.

**Why (i) is mandatory.** OBS-PHASE0B shows Sabine Pass cargoes cluster in
roughly 2,800–3,860 MMCF. LNG carriers are built to fill, so cargo size is
close to a physical constant. A flat median may already achieve single-digit
percentage error, in which case the sophisticated estimator adds very little
— and saying so plainly is more credible than reporting (ii) alone and
implying the accuracy was earned.

**Rejected.** Draught-derived volume — D-050 stands unchanged. Per-vessel
lookup without a holdout — circular, as above. Dropping volume entirely —
with the constant baseline it costs almost nothing to include, and the
comparison is itself the interesting result.

**Caution.** The whole-population CV of 0.188 reported in OBS-PHASE0B is
contaminated by a second population — small-scale Caribbean and Puerto Rico
distribution shipping 1–30 MMCF out of Miami, Ft. Lauderdale, Penuelas, San
Juan and Ponce. None of those are in-scope terminals. Compute all volume
statistics on in-scope terminals only, or the baseline will look artificially
hard to beat.

**Test.** `test_volume_fit_window_excludes_validation_period`;
`test_all_three_estimators_reported_together`.

---

### D-027 — The departure event is berth exit, not clip-box exit

**Status:** PROPOSED | **Evidence:** OBS-PHASE0B, OBS-PHASE0

**Do this.** The departure timestamp is the **last observation inside the
berth polygon before sustained movement away from it**. It is not the last
observation inside the D-106 clip box.

**Why.** The clip box extends well offshore to cover waiting areas, so a
vessel remains inside it for hours after leaving the jetty. OBS-PHASE0B gives
two independent cross-checks from a single AIS day:

| Vessel | DOE departure | Present in AIS box |
|---|---|---|
| LA SEINE | 2022-03-14 | 2022-03-15 |
| GASLOG GIBRALTAR | 2022-03-16 | 2022-03-15 |

Both are consistent with berth-exit timing and transit time inside the box.
Neither is consistent with box-exit as the event definition.

**Consequence.** D-240's ±2 day window exists to absorb reporting latency and
the local-versus-UTC date discrepancy (D-011). It must not be spent absorbing
transit time, or the window stops being a tolerance and becomes a fudge
factor. This also makes berth polygon precision directly load-bearing for the
headline accuracy number, which is the strongest argument yet for spending
Phase 1 effort on the density map.

**Test.** `test_departure_is_berth_exit_not_box_exit` on a synthetic track
that leaves the berth and lingers in the box for six hours.

---

### D-033 — Berth occupancy is saturated at Sabine Pass; report duration and spacing

**Status:** PROPOSED | **Evidence:** OBS-PHASE0B, LIT-FENG-2020

**Do this.** D-004's granularity deliverable is the **loading duration
distribution** and **inter-arrival spacing**, not berth occupancy. Compute
occupancy anyway and publish it as a diagnostic, not as a finding.

**Why.** OBS-PHASE0B gives 105 loadings in Q1 2022 at Sabine Pass — 1.17 per
day. At typical LNG loading durations of 12–24 hours across a small number of
berths, the terminal is close to continuously occupied. An occupancy metric
that sits near its ceiling in every period carries almost no information, and
presenting a flat line as a result invites the obvious question.

**Occupancy is still worth computing** for the opposite reason: at a
saturated terminal, an occupancy figure that is *not* near the ceiling
indicates missed calls. It is a detector health check.

**Caution.** This is arithmetic from DOE cargo counts, not a measurement.
Loading duration has not been observed yet. Confirm in Phase 1 and revise if
the observed duration distribution is materially shorter than assumed.

### D-007 — Study window is 2020-01-01 to 2026-03-31

**Status:** PROPOSED | **Evidence:** OBS-PHASE1-PRE, OBS-PHASE0B
**Amends the window assumed in:** D-106

**Do this.** The study window closes **2026-03-31**. Terminal scope is
unchanged from D-106 (Sabine Pass, Freeport, Corpus Christi).

**Why.** OBS-PHASE0B found Golden Pass LNG entering service with its first
cargo on **2026-04-22**. Golden Pass sits inside the Sabine Pass clip box by
construction — D-106 already records this and flags it as an explicit
"revisit if" trigger. From that date the box contains two active LNG
terminals, and terminal attribution stops being a property of the box and
becomes a property of the berth polygons.

That is not merely more work. Berth polygons are derived from observed
stationary density (D-013), and **no amount of data derives a polygon for a
terminal that was not operating during the derivation window.** Golden Pass
berths cannot be located from 2022 data. Including 2026 Q2 would mean a
polygon set valid for part of the window only, silently attributing Golden
Pass loadings to Sabine Pass.

**A second constraint lands on nearly the same date.** D-001's latency
analysis puts the NOAA archive 145–165 days behind real time. From the
current date that places the newest available partition around the start of
April 2026 regardless. Verify this against the `csv2/csv2026` index before
treating it as settled, but if it holds, the window end is forced by the
archive and Golden Pass merely coincides.

**Rejected.** Time-varying polygons with effective-from dates, and Golden
Pass as a fourth terminal. Both are more correct and neither is worth ten
weeks of the stalest data in the series. This is the better v2 change, not a
v1 requirement.

**Revisit if.** v2 scope opens. Golden Pass (from 2026-04-22) and Plaquemines
(from 2024-12-26, 425 cargoes to 2026-06-30) are the two obvious additions,
and Plaquemines needs no clip-box change at all — it is on the Mississippi,
outside every box in scope.

**Unblocking condition.** This stays `PROPOSED` until the `csv2/csv2026` index
is checked. The entry currently argues the window end is *also* forced by the
145–165 day archive latency, which is inference from D-001, not measurement.
If the newest available partition is near the start of April 2026, promote to
`ACCEPTED` and keep both arguments. If the archive reaches June 2026, delete
the latency paragraph and accept on the Golden Pass argument alone — which
stands unaided.

**Test.** `test_no_partition_after_window_end`.

---

### D-028 — *[Ingest]* Outage detection baselines on the same weekday, never a trailing window

**Status:** ACCEPTED | **Evidence:** OBS-PHASE1-PRE
**Area:** ingest and silver — ID borrowed from the detection block, which the
ID rule directs when the primary block is full.

**Do this.** Flag a candidate partition outage by comparing a day's file size
against the **median of the same weekday** over a fixed reference window
(the containing quarter, or a trailing eight same-weekday observations).
State the deviation threshold explicitly and log every flagged day with its
deviation.

**Do not use a trailing N-day median.** It fails in both directions.

**Why — false positives.** Commercial traffic has genuine weekly
seasonality, so a trailing window straddling a weekend has its median pulled
down only partially and flags the weekend by construction. In OBS-PHASE1-PRE,
a trailing-7-day method flagged 2022-02-22, 02-23 and 02-24. Against their own
weekday medians, 02-22 is *above* median and the other two sit on it. Three
normal days flagged.

**Why — false negatives, which is worse.** A sustained degradation becomes
its own baseline within one window length and then disappears. The same
method missed **2022-02-05 to 02-10 entirely** — six consecutive days, four
of them weekdays, 17–30% below their weekday medians. By 02-07 the trailing
window had already absorbed the drop.

**Observed effect on the two methods, Q1 2022:**

| Method | Days flagged | True positives | False positives | Missed |
|---|---|---|---|---|
| Trailing 7-day median, −2 SD | 10 | 7 | 3 | 6 |
| Same-weekday median | 12 | 12 | 0 | 0 |

**Caution — this is a proxy for a proxy.** Compressed byte count confounds
vessel count, message count and compression ratio, and a national-level dip
need not touch the Gulf receivers at all. A size-derived suspect list is a
**hypothesis to confirm**, never a conclusion. Confirm it in the partition
itself with in-box message counts per day, and record both figures.

**Relationship to D-119.** D-119 governs coverage gaps measured *from message
data* after ingest. This decision governs partition screening *from the blob
index* before download. They are complementary: this one tells you which days
to look at, D-119 tells you what actually happened in them. A day flagged
here that D-119 finds healthy is a false positive to record, not to hide.

**Test.** `test_outage_detection_uses_weekday_baseline` (static check that no
trailing-window median appears in screening code);
`test_sustained_degradation_is_flagged` on a synthetic six-day depression;
`test_normal_weekend_is_not_flagged`.

---

### D-046 — Thresholds are tuned on the clean subset; the degraded-day list is committed before ingest

**Status:** ACCEPTED | **Evidence:** OBS-PHASE1-PRE

**Ratified before the Q1 2022 fixture was ingested.** That sequencing is the
substance of this decision, not a formality — see rule 1.

**Do this.** Two rules, both binding on Phase 1.

1. **Commit the suspect-day list before ingesting the partitions.** It lives
   in `docs/` with the deviation figures that produced it and the date it was
   written.
2. **Tune every empirical threshold on the clean subset only** — D-021's
   `tT`, D-022's minimum stop duration, D-119's state-conditioned gap
   percentiles. Validate against **all** days in the fixture, clean and
   suspect together, and report matched-versus-missed split by the two
   groups.

**Why rule 1.** If the detector finds 96 of 105 loadings and the outage list
is produced afterwards, "nine misses fell on outage days" is unfalsifiable
and reads as excuse-making. Committed beforehand, the identical claim is a
prediction that happened to hold. Same evidence, entirely different standing.

**Why rule 2.** Thresholds tuned on degraded days silently compensate for
missing data — a gap percentile fitted through an outage widens to absorb it,
and the detector then appears to work. That failure looks exactly like
success and is invisible in the headline number.

**Q1 2022 suspect list (12 of 90 days, 13%):**

| Days | Deviation vs same weekday |
|---|---|
| 2022-01-29 to 01-31 | −45%, −40%, −33% |
| 2022-02-05 to 02-10 | −17% to −30% |
| 2022-03-14 | −29% |
| 2022-03-20 to 03-21 | −30%, −41% |

Clean subset: 78 days.

**Why the fixture keeps its outages rather than moving to a cleaner quarter.**
D-119 requires an outage taxonomy and degraded-week marking. That code cannot
be tested on clean data. A fixture containing real degradation is more useful
than one without, provided tuning and validation are separated as above.

**Caution — the list must be able to fail.** If detection performance on the
78 clean days is no better than on the 12 suspect days, the outage hypothesis
is wrong: the suspect list is withdrawn and the misses are a detector problem.
Do not defend a pre-registered list against contrary evidence; that would
convert an honesty mechanism into a rationalisation.

**Test.** `test_tuning_excludes_suspect_days`; report accuracy split by day
group in every D-240 table.

**Not a test — a record.** Rule 1 is enforced by the git history, not by the
suite. A test comparing commit dates fails on a fresh clone or after a rebase,
for reasons unrelated to correctness. Instead, record both dates in the README:
the commit that added the suspect list, and the commit that added the first
fixture partition. A reader can verify the order in one command.

---

# Open questions

| ID | Question | Blocks |
|---|---|---|
| — | Do the D-021 / D-022 thresholds (`tT = 1.5 h`, min stop 1.5 h) survive on LNG loading cycles, which run 12–24 h? Re-tune on the fixture, do not inherit. | D-021, D-022 |
| — | Is the D-117 `length >= 250` floor right? Sensitivity at 200 m and 270 m. | D-117 |
| — | What is the observed inter-message interval distribution, split by state and `transceiver`, on the Sabine Pass fixture? D-119 cannot be given numbers until this is measured, and two corpus sources disagree by a factor of two. | D-119, and D-015 via its gap reference |
| — | What share of departure events carry a usable `vessel_name`? This is the ceiling on D-240 recall and it has not been measured. | D-045, D-240 |
| — | Is a ballast-to-laden `draft` change used as corroborating evidence for a detected loading? D-153 resolved volume estimation but not this. Not blocking — D-027 defines the event kinematically — but currently unruled. | D-005, D-027 |
| — | Does the newest `csv2/csv2026` partition fall near April 2026? Determines whether D-007 accepts on two arguments or one. | D-007 |
| — | Ratification order. Several `ACCEPTED` decisions rest on `PROPOSED` ones — see the note below. | D-004, D-050, D-117, D-144, D-027 |

**Resolved since the last revision:** volume estimation (D-053 → D-153);
Houston (D-106 — deferred to v2); study period start (D-007 — 2020-01-01 to
2026-03-31); DOE split-cargo mechanism (D-144); name matching strategy
(D-240); outage screening method (D-028); tuning discipline (D-046).

**Earlier:** D-051 (withdrawn — container scope retired); framing (D-004);
sentinel convention (D-114); schema contract (D-112); timezone (D-011);
vessel-level gap threshold (D-119 — rule set, numbers still to be measured).

---

# Ratification order

Four decisions are `ACCEPTED` while decisions they depend on are still
`PROPOSED`. That inverts the dependency: precedence rule 2 makes an `ACCEPTED`
decision binding, so a binding decision currently rests on an unratified one.

| Accepted | Rests on | Status of dependency |
|---|---|---|
| D-106, D-153, D-240 | D-004 (framing) | PROPOSED |
| D-153 | D-050 (no draught-based estimation) | PROPOSED |
| D-240 | D-144 (the 105-loading figure) | PROPOSED |
| D-114, D-112 | D-010 (raw CSV is source of record) | PROPOSED |

None of this blocks Phase 1, which produces polygons and thresholds rather
than detections. D-004 and D-050 are the two worth ratifying first: D-004
because everything rests on it, D-050 because D-153 names it explicitly.
D-144 and D-027 should be ratified before Phase 2, per the note in each.

**A maintenance gap this exposed.** Supersession preserves history but leaves
forward references dangling — this revision repaired seven live references to
`SUPERSEDED` decisions (D-006→D-106 ×2, D-019→D-119 ×4, D-140→D-240 ×5,
D-053→D-153 ×2) across six live decisions. Nothing was catching them. A short script asserting that every
`D-NNN` cited outside a "Retained for history" block points at a decision whose
status is not `SUPERSEDED` or `RESOLVED` would catch the next one for free.
