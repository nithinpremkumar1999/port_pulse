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

| Range | Area |
|---|---|
| D-001–009 | Scope and framing |
| D-010–019 | Ingest, bronze and silver |
| D-020–029 | Port call detection (gold) |
| D-030–039 | Metrics |
| D-040–049 | Validation and analysis |
| D-050–059 | Explicitly out of scope |

---

# Scope and framing

### D-001 — port_pulse is a lead-lag study, not a nowcaster

**Status:** PROPOSED | **Evidence:** SRC-NOAA-FAQ, LIT-IMF-MALTA-2019

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

**Status:** PROPOSED | **Evidence:** SRC-NOAA-FAQ, LIT-IMF-WORLD-2020

**Do this.**

- Store every timestamp as UTC. No local times in bronze, silver or gold.
- Assign the **week** at the metrics layer using each port's local timezone,
  read from the port config. Never from a global constant.
- Record the week convention (ISO week, Monday start) explicitly in the metrics
  schema.
- Do **not** apply a blanket UTC offset to timestamps.

**Why.** `BaseDateTime` is UTC. Port authority throughput is local.
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

**Test.** `test_week_assignment_at_local_midnight` — a synthetic arrival at
23:59 and 00:01 port-local on a week boundary lands in the expected weeks, for at
least one US West Coast and one US East Coast port, in both DST and standard
time.

---

### D-012 — The schema is a contract; assert it and fail loudly

**Status:** PROPOSED | **Evidence:** SRC-NOAA-FAQ

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

**Status:** ACCEPTED (follows `CLAUDE.md`) | **Evidence:** SRC-NOAA-FAQ

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

### D-015 — The vessel key is (MMSI, segment), not MMSI

**Status:** PROPOSED | **Evidence:** LIT-HARATI-2007, LIT-IMF-PORTWATCH, SRC-NOAA-FAQ

**Do this.** Build a `vessel_segment_id` by splitting each MMSI's message stream
where any of the following occurs:

- Implied speed between consecutive fixes exceeds the D-018 threshold.
- `VesselType`, `Length` or `Width` changes mid-stream.
- A gap longer than the D-019 coverage-gap threshold.

Join everything downstream on `vessel_segment_id`. Use `IMO` where present as a
secondary check, never as the primary key.

**Why.** MMSI is not unique and is not stable. Not unique: LIT-HARATI-2007 found
MMSI `1193046` broadcast by 25 distinct ships, `0` by 5, `1` by 3, `999999999`
by 3. Not stable: LIT-IMF-PORTWATCH defines MMSI as assigned to the AIS system,
always present, but **not permanent** — it may change with flag or ownership —
against IMO which is permanent to the hull but only "usually" present.
SRC-NOAA-FAQ confirms transponders are transferred between owners and vessels.

**Rejected.** MMSI as primary key. IMO as primary key — absent for most tugs,
barges and domestic vessels.

**Caution.** `IMO`, `CallSign`, `VesselName`, `Length`, `Width` and `VesselType`
are populated or corrected by NOAA's AVID model from 2024, affecting roughly
10–15% of records. Uniqueness or consistency checks on these fields test AVID,
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

**Status:** PROPOSED | **Evidence:** SRC-NOAA-FAQ, LIT-HARATI-2007, LIT-BERTH-2025

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

**Status:** ACCEPTED (follows `CLAUDE.md`) | **Evidence:** LIT-MARTINCIC-2021, SRC-NOAA-FAQ, LIT-EMMENS-2021

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
perfect reception — but under the congestion-correlated message loss of D-019, 10
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
   into `Length` and `Width` (SRC-NOAA-FAQ), so the footprint would have to be
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

**Test.** `test_berth_vs_anchorage_classification` on labelled fixture segments.

---

### D-026 — `Status` is a feature and a diagnostic, never a label

**Status:** PROPOSED | **Evidence:** LIT-MARTINCIC-2021, LIT-HARATI-2007

**Do this.** Derive berth/anchorage state geometrically (D-025). Then compute and
report the **agreement rate** between derived state and the reported `Status`
field as a data quality metric. Never use `Status` as an input to the derivation.

**Why.** Two independent studies, roughly two decades apart, agree that around
30% of vessels transmit incorrect navigational status. LIT-MARTINCIC-2021 give
the diagnostic example: vessels can only be moored at terminals, yet moored
status appears in anchorage areas and while sailing. They also make the structural
point that static fields can be corrected against vessel databases but **dynamic
self-reported fields cannot**, because no external register records what a
vessel's status was at a given moment.

**Expected result.** Agreement near 70% independently reproduces the literature
and is worth reporting. **Agreement near 99% means `Status` has leaked into the
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
Acceptable as a secondary metric *if* published alongside the D-019 coverage
series.

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

**Test.** `test_leadlag_input_is_unsmoothed`.

---

### D-032 — Publish the message-density diagnostic alongside every congestion metric

**Status:** PROPOSED | **Evidence:** LIT-EMMENS-2021, LIT-ANDROJNA-2021

**Do this.** For every port-week, publish messages-per-vessel-per-hour beside the
congestion metric. Report their correlation. If it is materially negative,
state so and discuss it as a measurement floor on the congestion estimate.

**Why.** This is the project's most serious internal validity threat (see D-019).
It cannot be eliminated with this data. It can be measured, reported, and
reasoned about — and doing so is the difference between having read the
literature and having understood it. No source in the corpus quantifies this
relationship, so the diagnostic is our own contribution, not a replication.

---

# Validation and analysis

### D-040 — AIS-to-official matching uses a ±2 day window on IMO

**Status:** PROPOSED | **Evidence:** LIT-IMF-WORLD-2020

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

**Status:** PROPOSED | **Evidence:** LIT-VERSCHUUR-2021

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
`Length` proves reliable enough.

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

---

### D-051 — Container ships are not separately identifiable; the validation target adapts

**Status:** OPEN — **blocking for D-040 and the choice of throughput series**
**Evidence:** LIT-HARATI-2007, SRC-NOAA-FAQ

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
D-019 coverage diagnostics do not explain. The signature to look for is a burst
of near-sequential MMSIs appearing simultaneously.

---

# Open questions

| ID | Question | Blocks |
|---|---|---|
| D-051 | Container vs TEU mismatch — which validation target? | Validation layer, D-040 |
| — | Which US ports are in v1? Determines polygon effort and whether LIT-BERTH-2025's Los Angeles parameters are a usable anchor. | D-013 |
| — | Study period start. 2018-01-01 gives a populated `TransceiverClass` (D-017) and avoids the AVIS 4-digit code era; earlier gives more observations. | D-017, backfill scope |
| — | Is `tT = 1.5 h` (D-021) re-tuned on our fixture, or inherited? | D-021 |
