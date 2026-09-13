# literature.md

Evidence base for port_pulse. This file records **what the sources say**.
`decisions.md` records **what we do about it**. Every threshold, algorithm and
filter in the codebase should trace to an entry here via its `LIT-*` ID.

## How to use this file

- Before choosing a threshold or algorithm, search here for the parameter name.
- When a source informs a choice, add the `LIT-*` ID to the relevant entry in
  `decisions.md`. Do not put decisions in this file.
- When a source does **not** cover something, record it under
  [Gaps in the literature](#gaps-in-the-literature) rather than inventing
  support. An uncited threshold is allowed; a falsely cited one is not.
- **Observations of our own data are not literature.** They live under
  [Own observations](#own-observations) with `OBS-*` IDs and are cited the
  same way. Keeping them separate matters: a published finding and a
  single-day measurement on one port carry very different weight, and a
  decision that rests only on an `OBS-*` ID should say so.
- Numbers in this file are attributed to their source and are **not**
  automatically applicable to port_pulse. Check the "Transplantability" note on
  each entry first — most of these studies use hourly or sub-second data, and
  ours is downsampled to one minute.

## Status of extraction

| Source | Read | Notes |
|---|---|---|
| LIT-HARATI-2007 | Full | Field-level error rates, MMSI collisions |
| LIT-EMMENS-2021 | Full | Peril taxonomy, variable table |
| LIT-MARTINCIC-2021 | Full | Status correction, voyage extraction |
| LIT-YAN-2022 | Full | Stop detection parameter set |
| LIT-BERTH-2025 | Full | Berth polygon derivation |
| LIT-IMF-WORLD-2020 | Full | Port polygons, NDC validation, cargo weight |
| LIT-IMF-MALTA-2019 | Full | Weekly indicators, cargo load index |
| LIT-IMF-PORTWATCH | Full | Netting, ballast, imputation |
| LIT-VERSCHUUR-2021 | Full | Vessel-type scope sensitivity |
| LIT-FENG-2020 | Full | Port zone time indicators |
| LIT-ANDROJNA-2021 | Full | SOTDMA physics, spoofing |
| SRC-NOAA-FAQ | Full | Dataset-specific behaviour (not peer-reviewed) |
| SRC-DOE-FE746R | Partial | DOE reporting requirements and published fields |
| OBS-PHASE0 | n/a | Own measurement, Sabine Pass, 2022-03-15 |

---

# Sources

## LIT-HARATI-2007 — AIS data reliability and human error

Harati-Mokhtari, A., Wall, A., Brooks, P., Wang, J. (2007). *Automatic
Identification System (AIS): Data Reliability and Human Error Implications.*
Journal of Navigation, 60(3), 373–389.

**What it is.** Three empirical studies of AIS field accuracy: a VTS-based study
at Liverpool (Sept–Oct 2005, 94 AIS-equipped vessels, ~6 hrs/day at high tide,
AIS compared against port VTS database sourced from Lloyd's Register); a
data-mining study (400,059 AIS reports, 1–17 March 2005, worldwide receivers,
via AISLive / Lloyd's Register-Fairplay); and a targeted surveillance study of
four suspect MMSIs in European waters (Nov 2005 – May 2006).

**Establishes the four-class model of AIS content** (per IALA 2002), which is
the organising frame for all field-level reasoning in this project:

| Class | Populated by | Fields |
|---|---|---|
| Static | Installer, once | IMO, MMSI, call sign, name, vessel type, length, beam, antenna offsets |
| Dynamic | Ship sensors, automatic | Position, UTC, COG, SOG, heading, nav status, rate of turn |
| Voyage | Crew, per voyage | Draught, cargo type, destination, ETA |
| Safety | Crew, ad hoc | Free text |

The operative implication: automatic fields fail randomly, manually entered
fields fail systematically (same error, same vessel, every voyage).

**Key findings.**

- **Vessel type is the worst field.** 74% of observed ship types were judged
  unsatisfactory in the VTS study; 56% in the data-mining study. 6% of vessels
  broadcast no type; 3% used the generic value "vessel".
- **Navigational status** was inconsistent in roughly 30% of cases (Figure 1).
  Length and beam also showed material error rates. MMSI, name and call sign
  were essentially clean in the VTS sample.
- **MMSI collisions are real and concentrated in default values.** Targeted
  surveillance of four suspect MMSIs found:

  | MMSI | Distinct ships broadcasting it |
  |---|---|
  | 1193046 | 25 |
  | 0 | 5 |
  | 1 | 3 |
  | 999999999 | 3 |

  1193046 is believed to be a transponder model's factory default, left
  unchanged at installation.
- **The vessel type category list is structurally too coarse.** Container
  vessels, car carriers and bulk carriers are all reported as "cargo". Chemical,
  petroleum and gas tankers are all reported as "tanker". Three sister high-speed
  ro-ro passenger ferries on the same route were observed broadcasting three
  different types (Cargo, HSC, Passenger).
- **"Static" fields are not static for some vessel classes.** A tug broadcasts
  type "tug" when free and switches to "towing" when it picks up a tow;
  dredgers alter type through a voyage. The authors describe this as a
  deliberate but poorly documented regulator decision, made so the navigational
  status field stays free to signal restricted manoeuvrability. At least one
  manufacturer's manual does not explain how to change static data at all.

**Transplantability.** High for the qualitative findings (which fields fail and
why). The error *rates* are from 2005–06 equipment in UK waters and should be
treated as order-of-magnitude, not current. Note that NOAA's AVIS/AVID
corrections (SRC-NOAA-FAQ) partially mitigate the type/length/width problem in
our dataset in a way that did not exist for these authors.

---

## LIT-EMMENS-2021 — The promises and perils of AIS data

Emmens, T., Amrit, C., Abdi, A., Ghosh, M. (2021). *The promises and perils of
Automatic Identification System data.* Expert Systems With Applications, 178,
114975.

**What it is.** A literature review of AIS limitations plus an empirical study of
Port of Amsterdam data, 1–29 April 2018: 21,178,375 rows × 19 columns, split into
moored (SOG = 0, 16,667,245 rows) and sailing (SOG ≥ 0.1, 4,511,130 rows).
Supplemented by practitioner interviews.

**The five perils** — use these as the canonical failure taxonomy:

1. **Noise in transmitted values.** For sailing vessels, transmitted SOG often
   does not match speed computed from consecutive positions, meaning either the
   position or the speed is wrong. Noise in positional data for moored vessels
   is comparatively small. Noise varies by vessel type; passenger vessels
   transmit the least reliable data.
2. **Equipment quality.** Terrestrial AIS has limited range but is more accurate
   than satellite AIS; S-AIS has better coverage but suffers message collision.
   Faulty installation, antenna placement, receiver unavailability and VHF
   transmission conditions all degrade quality. Interviewees reported preferring
   GPS values over AIS-transmitted SOG, and described sensor fields (COG, SOG)
   as unstable for many vessels.
3. **Redundancy / volume.** Much of the data is redundant; volume defeats manual
   inspection.
4. **Incomplete or unrealistic tracks.** Caused by receiver range limits and, for
   satellite, revisit time and latency.
5. **Failure in dense traffic.** *The peril most relevant to port_pulse.*
   Tracking in dense and complex traffic areas is difficult; many simultaneous
   transmissions in one area cause message collision, and the network becomes
   inflated leading to transmission failure. Cited across seven independent
   studies.

**Method worth reusing.** They compute Haversine distance between consecutive
AIS points per vessel to derive an implied speed, then compare it against the
transmitted SOG. The disagreement is the detector for bad position, bad
timestamp or shared MMSI.

**Variable table (their Table 1)** — note this is generic AIS, not our schema.
Includes antenna offsets A (bow), B (stern), C (port), D (starboard); COG in
tenths of a degree (0–3599); draught range −0.1 to 25.5 m; timestamp as Unix
epoch. **Length and beam were absent from their dataset entirely**, despite
being standard AIS fields — a reminder that AIS "schema" varies by provider.

**AIS switch-off vs coverage gap.** They summarise Kontopoulos et al. (2020),
who built a streaming system that distinguishes deliberate transponder switch-off
from network coverage gaps by reasoning over known coverage. Related: "Black
Hole" detection (Salmon et al. 2015) identifies persistently under-populated
grid cells from historical data to establish where absence of data is expected.

**Transplantability.** High. Single port, terrestrial AIS, one month, moored-vs-
sailing split — structurally close to our problem. Caveat: their data was not
downsampled, so their analysis of noise inside the 10-second reporting interval
is not reproducible on ours.

---

## LIT-MARTINCIC-2021 — Vessel and port efficiency metrics through validated AIS

Martinčič, T., Štepec, D., Pita Costa, J., Cagran, K., Chaldeakis, A. *Vessel and
Port Efficiency Metrics through Validated AIS data.* XLAB Research / University
of Ljubljana / Piraeus Port Authority.

**What it is.** A navigational-status validation and correction pipeline, plus
port efficiency metrics built on the corrected data. Evaluated at the Port of
Piraeus using one year of historical plus live AIS from AISHub, validated
against FAL forms held by the port authority. Ships as the PARES tool.

**The central claim.** Static fields (vessel type, MMSI, IMO) can be corrected
against dedicated maritime vessel databases. **Dynamically reported fields such
as navigational status cannot**, because no external register records what a
vessel's status was at a given moment. They cite the finding that at least 30% of
vessels transmit incorrect status, and give the diagnostic example: vessels can
only be moored at terminals, yet moored status is observed in anchorage areas and
even while sailing.

**Three correction approaches, in increasing independence from manual data:**

1. **Speed + location** against predetermined anchorage and terminal polygons.
   Downside: requires a speed threshold *and* GIS data from the port.
2. **Speed + rotation.** Omits the need for manually provided polygon data.
   *This is the approach to prefer where port GIS is unavailable.*
3. **Machine learning** (KNN and others) on reported spatial and kinematic
   fields, classifying into the appropriate status and validating against the
   reported one. At least 300 neighbours were needed for best KNN results.

**Voyage extraction rules** (their definition: a voyage is movement in the port
area, from arrival through optional anchorage stop and terminal stop to
departure). Messages grouped by MMSI and time:

| Rule | Threshold |
|---|---|
| Group consecutive messages into one voyage if gap is under | 24 hours |
| Split into two voyages if gap exceeds **and** vessel moved over | 5 hours **and** 100 metres |

Implemented as a dataframe sorted by MMSI and time, with split conditions
comparing consecutive rows.

**Data outage taxonomy** — three causes with different signatures:

| Signature | Cause |
|---|---|
| One vessel's messages missing | Transmitter fault, or deliberately switched off |
| All messages missing in an area | Terrestrial receiver problem, or area not covered |
| No data at all for a period | Provider outage, or own collection system |

Their guidance: outages can be ignored when the vessel was moored or anchored
throughout, or moving in a straight line. Otherwise correct times cannot be
extracted, and **the best solution is to mark the stop or period as having
missing data** — which aligns with the port_pulse rule that gaps stay gaps.

**Transplantability.** High, and it is the closest paper to our gold layer.
Caveat: single Mediterranean port with authority-supplied ground truth we do not
have.

---

## LIT-YAN-2022 — Extracting ship stopping information from AIS data

Yan, Z., Cheng, L., He, R., Yang, H. (2022). *Extracting ship stopping
information from AIS data.* Ocean Engineering, 250, 111004.

**What it is.** A three-stage method: stopping point identification from
trajectory features under geographic constraints → berth/anchorage mode
classification by random forest → port point extraction. Applied to the South
China Sea Silk Road region, 2017 AIS data.

**Performance.** Stopping point identification: precision 0.94, recall 0.91,
F1 0.92. Mode classification: overall accuracy 0.93, Kappa 0.87 (12 features
after screening, 5 features per node, 150 trees; beat decision tree 0.91, SVM
0.92, maximum likelihood 0.89). Port extraction: 1,067 points extracted, 1,050
correct against Google Earth imagery = 98.41%.

**The core insight.** A ship at rest is not at zero speed. Unlike a road vehicle,
a stopped ship is affected by wind, waves and currents, so its speed remains in a
small positive range determined by stopping mode and current conditions.

**Parameter set (their Table 1)** — the most directly transplantable numbers in
the knowledge base:

| Parameter | Symbol | Value | Basis |
|---|---|---|---|
| Speed threshold for "stopped" | vT | **1 knot** | Literature (Pallotta 2013, Wen 2019, Yan 2020b) + prior knowledge |
| Time gap, adjacent trajectory points | tT | 1.5 h | Tuned on their data |
| Distance, adjacent trajectory points | disT | 2 km | Literature |
| Water depth threshold | Δdep | 100 m | Derived from GEBCO |
| Buffer radius around stopping point | buf | 10 km | Literature |
| Impervious surface area threshold | Δarea | 0.25 km² | Tuned |
| Distance for merging stop segments | Δd | 2 km | Literature |
| Time gap for merging stop segments | Δt | 1 h | Literature |
| Min points in a stop segment | Δn | 10 | Literature |
| Min stop duration | Δstopt | 1.5 h | Tuned |

**Derivation of the depth threshold** (worth knowing, because it is reusable
reasoning): they buffered global port points by 10 km and intersected with GEBCO
bathymetry. Mean depth within 10 km of global ports is about 50 m and over 90% of
port waters are under 100 m, so 100 m becomes the cutoff. Anchorages are
generally within 5 nautical miles of the port area, motivating the 10 km buffer.

**Berth vs anchorage geometry.** Berth stopping segments have a **convergent**
point distribution because the vessel is physically fixed. Anchorage segments are
**scattered and tend to be circular**, because the vessel is fixed at only one
end and swings. This is the geometric basis for classifying stop type without
terminal polygons, and it corroborates the "speed + rotation" approach of
LIT-MARTINCIC-2021.

**They also establish that public port point databases are inadequate**: Natural
Earth 10 m port data omits Quanzhou Port entirely despite heavy traffic, so
matching stops against existing port points loses real stops.

**Transplantability.** The 1-knot threshold is the strongest single
recommendation in this corpus and carries three independent citations.
Caveats: the "tuned" parameters were fitted on 88,706 records from 10 ships;
`Δn = 10` assumes a message density that our one-minute-downsampled,
collision-degraded data may not deliver in a congested port.

---

## LIT-BERTH-2025 — Unsupervised port berth identification from AIS data

*Unsupervised Port Berth Identification From Automatic Identification System
Data.* Preprint, 10 June 2025.

**What it is.** DBSCAN per vessel → spatial augmentation → optional geohash
encoding → Gaussian Mixture Model → post-processed polygonal berth boundaries.
Evaluated on 11 ports including **Los Angeles**, Singapore, Antwerp, Southampton,
Algeciras, Busan, Cape Town, Gdansk, Limassol, Livorno, Auckland. Scored by
Bhattacharyya distance between GMM outputs on two vessel-exclusive data splits.

**Scope note: they restrict to cargo and tanker vessels** — the same population
as the port_pulse silver layer — and list extension to passenger and fishing
vessels as future work.

**Pipeline parameters.**

| Step | Choice | Justification |
|---|---|---|
| Interpolation period | **1 hour** | 15 and 30 min overfragment the coastline into too many berths; 2 h shows mild performance decline |
| Heading-change filter | Drop records with heading change **> 10°** | Removes manoeuvring vessels before clustering |
| DBSCAN | Per vessel, Haversine distance | Clusters where a vessel stayed ≥ `pn` periods within ε |
| DBSCAN ε prior | log-uniform, 5–70 m | Tuned by Tree Parzen Estimator |
| DBSCAN minPoints prior | uniform integer, 2–25 | Tuned by TPE |
| Spatial augmentation | 10 points/message (train), 20 (eval) | Sampled uniformly over the vessel's footprint from dimensions A/B/C/D and heading |
| Geohash precision | 9 (≈ 4.7 m × 4.7 m) | Declutters and speeds training; geohash variant outperformed non-geohash |
| Observation window | **1 month** | Monotonic improvement from 3 days → 1 week → 2 weeks → 1 month |

**Tuned values for Los Angeles** (only US port in the study, useful as a sanity
anchor):

| Variant | ε (m) | minPoints | GMM components |
|---|---|---|---|
| Non-geohash | 19.797 | 5 | 49 |
| Geohash | 42.313 | 14 | 40 |

**Findings that matter beyond the method.**

- Their models identified berths that publicly available berth labels **omit or
  misrepresent** — public berth documentation is inconsistent.
- Longer observation windows allow stricter DBSCAN hyperparameters without
  over-pruning, reducing both false positives (non-berth areas called berths) and
  false negatives.
- They note explicitly that the method uses **freely available terrestrial AIS**,
  and that regions with sparse coverage or data quality problems (they name Port
  of Ambarli) remain challenging.
- Berth usage is vessel-type specific and can be seasonal — one Cape Town berth
  served mining vessels only after the observation period.

**Transplantability.** High in structure. Two hard caveats: (a) their AIS is
reported hourly and DBSCAN `minPoints` is therefore in units of *hours*, so the
values do not transfer directly to one-minute data without rescaling;
(b) spatial augmentation requires dimensions A/B/C/D, which **NOAA collapses into
`Length` and `Width`** — the augmentation step is not reproducible on our source
without approximating the footprint from length, width and heading.

---

## LIT-IMF-WORLD-2020 — World seaborne trade in real time

Cerdeiro, D., Komaromi, A., Liu, Y., Saeed, M. (2020). *World Seaborne Trade in
Real Time: A Proof of Concept for Building AIS-based Nowcasts from Scratch.*
IMF Working Paper.

**Data.** MarineTraffic, 1 Jan 2015 – 18 April 2020, over 1 billion messages from
over 50,000 ships, **downsampled by the vendor to hourly**; effective coverage
nearer one message per ship every two hours due to message collision and lower
satellite coverage in deep ocean.

**Port boundaries are inferred, not assumed.** They argue against hand-drawn
polygons on two grounds: the National Geospatial-Intelligence Agency's World Port
Index lists 3,669 ports, and port boundaries change or new ports are built over
time, so any hand-drawn set becomes obsolete. They therefore derive polygons from
the AIS data itself by unsupervised learning.

**The false-positive problem with polygons** (directly applicable to us): vessels
*transit* polygons without calling. Their examples are ships crossing San
Francisco Bay to reach Oakland, and ships stepping on a polygon while passing
through the Singapore Strait.

**Validation against US official data.** They validate against the US Army Corps
of Engineers Navigation Data Center (NDC) national waterway data, compiled with
CBP, which records most port calls at US ports across 23 fields including vessel
IMO and date of entry. As of March 2020 the NDC series ran through end-2018.
They exclude Puerto Rico and the US Virgin Islands.

**Their matching procedure** — the reference implementation for our
AIS-to-official reconciliation:

- Search for the same IMO in AIS port visits within **± 2 days** of the NDC
  record date.
- Rationale for looking 2 days *back*: US law gives vessels up to two days to
  file an entrance statement.
- Rationale for looking 2 days *forward*: NDC records local US date while AIS
  is UTC, and **UTC can be 5–9 hours ahead depending on zone and daylight
  saving**; also a vessel may file on arrival at anchorage, before entering.
- They **shift all AIS timestamps by six hours** to limit, but not eliminate,
  this discrepancy.
- Ties broken by closest date, then by earliest record.
- Categories: matched, false negative (NDC call with no AIS visit), and by
  implication false positive.
- US-flagged ships are only 0.5% of their dataset (relevant because US-flagged
  ships coming directly from another US port without foreign goods are exempt
  from filing).

**Cargo weight estimation.**

```
cargo_it = DWT_i × (d_it − d_i,ballast) / (d_i,max − d_i,ballast)
```

- `d_it` is taken as the **last observed draught before entering a polygon**,
  on the reasoning that crews report draught carefully just before arrival to
  assure the port the vessel can enter.
- Ballast draught is not in AIS and is **imputed**: per vessel, take the ratio of
  the 1st percentile of observed draught to design draught; then take the median
  of that ratio by ship type × DWT tertile; apply to design draught.
- Outgoing cargo is set equal to the *next* port's incoming cargo, again because
  arrival draught is the more precise reading.
- Net cargo offloaded = incoming − outgoing, and may be negative.

**Transplantability.** The matching procedure and the polygon-transit warning
transfer directly and are US-specific, which is rare in this corpus. The cargo
formula does **not** transfer: it requires DWT and design draught, neither of
which is in the NOAA source (see Gaps).

---

## LIT-IMF-MALTA-2019 — Big data on vessel traffic: nowcasting trade flows

Arslanalp, S., Marini, M., Tumbarello, P. *Big Data on Vessel Traffic: Nowcasting
Trade Flows in Real Time.* IMF Working Paper. Case study: Malta, 2015–2018
(211 weeks).

**Why it matters to port_pulse.** This is the paper whose *output shape* most
resembles ours: **weekly, per-port indicators validated against official
statistics.** They explicitly frame weekly aggregation as the key advantage over
customs-based official trade data, which is monthly at best and sometimes
quarterly or annual.

**Two indicators.**

1. **Cargo number indicator** — count of incoming ships, filtered to container
   and cargo ships. Averaged about 100 ships/week for Malta. Carries no
   information about ship size or cargo load; all ships count equally. They
   smooth with a **five-term centred moving average** because the weekly series
   is visibly noisy. Seasonal trough at the start of the year, peaks around
   weeks 24–25.
2. **Cargo load indicator** —
   `CWI_t = Σ_i DWT_i,t × |d^D_i,t − d^A_i,t| / max_i(d)`
   Absolute value is taken to capture imports plus exports and to avoid negative
   values when departure draught is below arrival draught.

**Their own stated shortcomings**, which are the honest limitations to repeat:

- `max(d)` is a *local* maximum observed in port calls, not the ship's design
  draught, so the formula underestimates cargo load when the two diverge. Fix:
  use a vessel register.
- The formula assumes no trade activity when draught does not change. This is
  false for simultaneous load/unload, though such cases were under 5% of filtered
  ships in Malta.
- Draught is crew-entered and sometimes updated only before approaching the
  *next* destination rather than on departure.
- Citing Jia et al. (2015), validated against port agents' lineup reports and
  fixtures data for capesize dry bulk: **draught is unreliable per-ship but a
  sufficiently populated sample gives useful information about average payload
  and utilisation within a ship type and size category.** Aggregate only.

**Three reasons a port call goes unrecorded:** low AIS coverage around the port;
the vessel switches off its transponder; the port is small and absent from the
provider's database. They add that vessels would normally keep transponders on
during port arrivals and departures for safety reasons given port congestion.

**Why port calls beat full voyages.** Port call data is smaller, less complex,
and *more accurate*, because AIS receiving station coverage tends to be better
near ports.

**Scope limits they state for AIS-based trade estimates.** Goods not services;
volume not value; gross trade not re-exports; broad groups not specific goods.
They caution against the approach where AIS coverage near a country's ports is
poor, or where sanctions give vessels a motive to switch transponders off.

**Coverage context.** MarineTraffic operated over 3,500 terrestrial AIS
receiving stations relaying from more than 180 countries as of end-2018. Other
providers named: VT Explorer, IHS Global, exactEarth, Spire, ORBCOMM, FleetMon.

---

## LIT-IMF-PORTWATCH — Nowcasting global trade from space

Arslanalp, S. et al. *Nowcasting Global Trade from Space.* IMF Working Paper.
Documents enhancements to the IMF PortWatch platform since its beta launch in
November 2023.

**Read this before treating PortWatch as independent validation.**

- **Netting adjustment.** Container ships often load and unload during the same
  port call, so AIS observes only the net draft change and understates trade.
  PortWatch corrects this for **83 of the largest container ports (about
  two-thirds of global containerised trade) using official container throughput
  data.** A bootstrapping alternative exists for ports without throughput data.
  → *PortWatch container estimates at major ports are partly calibrated on the
  official statistics we are trying to lead. Treat as a benchmark, not as an
  independent instrument.*
- **Ballast water adjustment.** Vessels are classified "in ballast", "with
  ballast" and "laden" using micro data on ballast water capacity, to estimate
  actual payload.
- **Port database expanded** from 1,378 to 1,666 ports, reconciled against ISL,
  Lloyd's List and World Bank port lists, plus specialised oil terminals.
- **Imputation of incomplete draft.** Two techniques from Arslanalp, Koepke &
  Verschuur (2021): **backpropagation** uses the vessel's reported draft at the
  next port of call and handles **two thirds** of incomplete draft data;
  **historical averaging** covers the rest, using the vessel's average shipment
  at that port. Refinements: fall back to the same vessel type group at the same
  port when vessel history is absent, and use a **probability-weighted** average
  of positive and negative shipments separately rather than a simple mean (a
  simple mean of a vessel that loads 10% DWT half the time and unloads 10% the
  other half is zero, which is wrong).
- **Stated limitations.** AIS covers the vessels goods travel on, not the goods
  themselves. AIS-based estimates include transshipments, which can be
  significant at some ports. AIS-based estimates are a **proxy** for trade.

**Glossary definitions to use consistently** (their Annex V):

| Term | Definition |
|---|---|
| DWT | Maximum cargo in metric tons a ship can carry without compromising safety |
| Draft / draught | Vertical distance between waterline and bottom of hull |
| Maximum / design draft | Legal loading limit, marked by the Plimsoll line |
| Payload / load factor | Share of DWT occupied by paid cargo |
| IMO number | 7 digits, assigned at construction, permanent to the hull, unchanged by name/flag/owner change. **Usually but not always present in AIS** |
| MMSI | 9 digits, assigned to the AIS system on board, **always** in the signal, **not permanent** — may change with flag or ownership |
| Port call | A discrete event of a vessel arriving at and departing from a port to load/unload cargo |

---

## LIT-VERSCHUUR-2021 — COVID-19 lockdowns in high-frequency shipping data

Verschuur, J., Koks, E., Hall, J. (2021). *Global economic impacts of COVID-19
lockdown measures stand out in high-frequency shipping data.* PLOS ONE, 16(4),
e0248818.

**What it is.** Daily AIS-derived trade indicators for ~1,200 ports across 166
countries, used to estimate trade losses in Jan–Aug 2020 and to regress
individual non-pharmaceutical interventions on exports. Raw AIS from the UN
Global Platform.

**Headline results.** Port calls down 4.4% across all ports Jan–Aug 2020 vs 2019.
Estimated global maritime trade down 7.0–9.6%, equal to 206–286 Mt and
USD 225–412 bn.

**The single most useful finding for port_pulse — vessel-type scope changes the
answer.** Their 4.4% port-call decline is well below UNCTAD's 8.7% for the first
six months. They attribute the gap to vessel type inclusion: they count only the
main trade-carrying vessels, while UNCTAD also included passenger vessels, which
were **66% of total port calls** and fell hardest (−17%). *A port-call metric's
headline number is dominated by which vessel types are in scope. This must be
stated explicitly with any figure we publish.*

**Method caveat they state.** Accuracy depends on coverage of manually entered
AIS attributes, **especially vessel draft, which is less frequently reported in
developing countries.** Implies a geographic bias in any draft-based metric.

**Transplantability.** The vessel-scope finding is directly applicable and is a
strong argument for reporting our metrics by vessel type group rather than as a
single aggregate. The 2020 period is also a natural stress test: if our
congestion metric does not show the 2020 disruption, the metric is broken.

---

## LIT-FENG-2020 — Time efficiency assessment of ship movements in ports

Feng, M., Shaw, S.-L., Peng, G., Fang, Z. (2020). *Time efficiency assessment of
ship movements in maritime ports: A case study of two ports based on AIS data.*
Journal of Transport Geography, 86, 102741.

**What it is.** A space-time trajectory framework (from time geography,
Hägerstrand 1970) that decomposes a port visit into zones and statuses and
measures time in each. Shanghai Yangshan and Xiamen, May 2014, four vessel types
(container, cargo, tanker, passenger). Notes that AIS defines 24 vessel types.

**The zone/status decomposition** — this is the vocabulary to adopt for the gold
layer, because it names the states a port call passes through:

| Zone | Status |
|---|---|
| Fairway | VTS Line-to-Berth |
| Anchorage | Anchored in Anchorage during VTS Line-to-Berth |
| Berth | Moored at Berth |
| Anchorage | Anchored in Anchorage during Berth-to-VTS Line |
| Fairway | Berth-to-VTS Line |

**Their indicators.**

- **Berth Time (BT)** — six summary statistics of Moored-at-Berth duration.
- **VBT and BVT** — time **per kilometre** for the inbound and outbound fairway
  legs. Normalising by distance is what makes the measure comparable across
  ports of different geometry; a raw transit time is not.
- **VBT − BVT** — the asymmetry between inbound and outbound efficiency.
- **Counts and total duration of anchorage stops** in each direction, used as the
  delay measure.
- Berth time additionally grouped by **ship size (length)**, to control for the
  effect of carrying capacity on load/unload time.

**Transplantability.** The status decomposition transfers cleanly. Two caveats:
their zone boundaries come from **nautical chart maps and VTS lines**, which are
authoritative data we do not have and would have to approximate from port
polygons; and normalising fairway time per kilometre requires a defined route
distance, not just a polygon.

---

## LIT-ANDROJNA-2021 — AIS data vulnerability: a spoofing case study

Androjna, A. et al. (2021). *AIS Data Vulnerability Indicated by a Spoofing
Case-Study.* Applied Sciences, 11, 5015.

**What it is.** Protocol-level description of AIS plus forensic analysis of a
mass spoofing event near Elba Island, 3 December 2019, with a survey of other
documented incidents.

**Protocol facts that explain message loss** — the physical basis for
LIT-EMMENS-2021 Peril 5:

- SOTDMA, 9,600 bit/s GMSK, 256-bit messages, slot length 26.7 ms → **2,250
  messages per minute per channel**, two channels (161.975 and 162.025 MHz) →
  4,500 total. IMO performance standard requires at least 2,000 slots/min at 50%
  utilisation.
- Class A transmits at 12.5 W and may occupy up to **5 consecutive slots**;
  Class B up to **3**. Class B is therefore disadvantaged under contention.
- Nominal reporting intervals 2 s to 6 min, varying with station type, message
  group, navigational status, speed and course change. Slow ships every 10 s,
  medium 6 s, high-speed 2 s; a heading change triples transmission rate for slow
  and medium ships.
- Typical range 20 NM ship-to-ship, up to 40 NM ship-to-shore, but detection is
  not purely geometric — it depends on VHF propagation, ground conductivity,
  atmospheric conditions, receiver sensitivity, antenna attenuation, signal
  shadowing and radio interference. Range approximation `D = 2.5(√h1 + √h2)`
  gives 93.85 NM for a 1,028 m base station and a 30 m ship antenna, which is why
  very-long-range receptions are normal rather than erroneous.

**Evidence that saturation is real.** During the Elba event the spoofer generated
close to 100% VDL load, and the Monte Capanne base station reported its memory
saturated after receiving **more than 2,048 distinct AIS station reports in
9 minutes**.

**Documented spoofing patterns relevant to ports.**

| Incident | Pattern |
|---|---|
| Shanghai and 20+ Chinese coastal areas/ports, 2018–19 | "Crop circles" — signals congregated into large circles. Circle locations observed to be **oil terminals** |
| Elba Island, 3 Dec 2019 | Mass artificial targets with auto-incrementing MMSIs; a fake 40-knot vessel; an invalid MMSI (4803509) produced by truncating a generated one; MMSI 999999999 appearing as a candidate source |
| Galápagos, July 2020 | Fleet reported positions ~10,000 km from observed location; many vessels stopped transmitting for 8+ hours (confirmed by HawkEye360 RF geolocation) |
| Stena Impero, July 2019 | Tanker spoofed into altering course into Iranian waters |
| Ponce de Leon Inlet, Florida, 2020 | Four fake aids-to-navigation created from spoofed messages |

**Transplantability.** The protocol facts transfer absolutely. The spoofing
incidents are mostly non-US and mostly sanctions- or fishing-motivated;
treat the risk as low but non-zero for US ports, and note that the Florida
incident is US.

---

## SRC-NOAA-FAQ — Marine Cadastre AIS FAQ and data dictionary

NOAA Office for Coastal Management, *AIS Frequently Asked Questions*, May 2026,
and *AIS Data Dictionary* (2015–present). Not peer-reviewed; this is source
documentation for the dataset we actually ingest. **Where it conflicts with the
papers, it wins for our data.**

**Provenance.** Data originate from the US Coast Guard's Nationwide AIS (NAIS),
approximately 200 land-based receiving stations, plus US Army Corps of Engineers
data for inland waterways. **All land-based — no satellite AIS**, because
agencies that license S-AIS are barred from public redistribution.

**Coverage.** US EEZ, major inland waterways, Great Lakes and Guam. Unavailable
beyond roughly **40–50 miles from the coast**, and unavailable for remote Pacific
territories and foreign waters. **Alaska broadcast points end 29 March 2021**,
removed at the request of the Marine Exchange of Alaska.

**Processing.** Raw NMEA is down-sampled **to the nearest whole minute**.
Distributed since 2015 as Zstd-compressed daily CSVs. Accidents are preserved,
not removed. Excludes certain law enforcement and military vessels exempt for
national security; many small vessels are exempt from carriage requirements.

**Latency.** No live or near-real-time feed. New data added roughly every 90
days; **total lag from collection to delivery is approximately 145–165 days.**

**Schema, 17 columns:** `MMSI`, `BaseDateTime`, `LAT`, `LON`, `SOG`, `COG`,
`Heading`, `VesselName`, `IMO`, `CallSign`, `VesselType`, `Status`, `Length`,
`Width`, `Draft`, `Cargo`, `TransceiverClass`.

> **Contradicted by OBS-PHASE0 for the `csv2` product.** The 2022 `csv2`
> header is snake_case throughout, and two columns are renamed rather than
> re-cased: `latitude`, `longitude`, `transceiver`. The PascalCase header
> above describes the retired `.zip` distribution. See D-112.

**Sentinel values.** `COG = 360.0` means unavailable. `SOG = 102.3` means
unavailable. `Heading = 511` means unavailable (the FAQ describes this as a
pre-2015 convention, but it is the NMEA standard and **is present in current
files** — verify against data, not documentation).

> **Contradicted by OBS-PHASE0 for the `csv2` product.** None of these three
> magic values occurs in the 2022 file; unavailable values are `NULL`. The
> FAQ's own instruction — verify against data, not documentation — is what
> caught it. See D-114.

**Vessel identity corrections.**

- 2015–2023: AVIS replaced `VesselType`, `Length`, `Width` and `Draft`. **The
  original uncorrected NMEA vessel type was moved into the `Cargo` field for this
  period.**
- 2024 onward: AVID replaced AVIS. Integrates 23 state, federal and international
  sources plus final AVIS into a fuzzy logic model. Populates nulls and corrects
  gross errors in `IMO`, `CallSign`, `VesselName`, `Length`, `Width`,
  `VesselType`. **Corrects roughly 10–15% of records.**
- Vessel type codes: 0–99 standardised NMEA; 99–255 reserved; above 999 are
  4-digit AVIS service types not in the native NMEA stream. Used directly
  2015–2017; mapped to best-fit NMEA types 2018–2023. **No 1:1 mapping exists
  between the 2-digit and 4-digit schemes.**
- Type 59 means ships and aircraft of States not Parties to an armed conflict
  (ITU Resolution 18).

**Fields that do not exist in this dataset.** Voyage-related values
(destination, ETA, voyage ID) were **phased out in 2015** due to inherent
inaccuracy in crew-entered destinations. Tonnage, horsepower and fuel type are
not part of the AIS broadcast at all. Vessel ownership is not recorded.

**Expected anomalies that must not be filtered as errors.**

- **Points over land** are expected: GPS error, narrow inland waterways, and
  vessels transported by land.
- **Points far beyond radio range** usually result from anomalous propagation
  (tropospheric ducting) and from high-elevation receivers. NOAA states
  explicitly these should not be treated as erroneous or satellite-derived.
- **Small daily files** reflect real traffic variation and NAIS sensor outages.
  The USCG typically does not specify cause or duration. Most gaps are brief —
  **a few hours to a couple of days** — and usually limited to a small number of
  receiving stations.
- **Flat cargo-vessel counts over time** may reflect improved USCG reporting
  accuracy campaigns, increasing average vessel size, or route changes further
  offshore beyond NAIS range to avoid emission control areas.

**Class B.** Included throughout; the class designation is only populated
**after 2017**.

**Draft units.** 2015–present: metres at 0.1 m resolution, American spelling.
Modern entries typically report **maximum static draft**, not actual laden draft.
(2009–2014 used decimetres, with 255 meaning ≥ 25.5 m.)

**Alternative distribution.** Cleaned, analysis-ready **GeoParquet** files for
2024 and 2025 are published on Azure via the Marine Cadastre GitHub. These have
had sentinel values and anomalies eliminated. Described as an experimental
product. A 2026 upgrade is planned to unify the AccessAIS and bulk download
formats.

**Licence.** Derived from USCG NAIS under data sharing category "Level C
(Historical Data)"; historical data generally considered public domain. Cite as:
NOAA Office for Coastal Management. ([Year]). [Title of Dataset]. Marine
Cadastre. https://marinecadastre.gov.

---

## SRC-DOE-FE746R — DOE natural gas import/export reporting

US Department of Energy, Office of Fossil Energy and Carbon Management.
Form FE-746R, *Monthly Report of Natural Gas Imports and Exports*, and the
published *Natural Gas Imports and Exports Monthly* report.

**Not peer-reviewed.** An administrative reporting regime. Treat as a second
measurement of the same physical events, not as ground truth — the data is
self-reported by authorisation holders as a condition of their export
authorisation.

**Required fields per cargo.** DOE export orders require each LNG cargo to
report: the US export terminal, country of destination, **date of departure**,
**name of the LNG tanker**, supplier, volume in Mcf, price per MMBtu at the
point of exit, and the duration of the supply agreement.

**No IMO number is published.** Vessel identity is by name only. Any AIS
reconciliation must join on `vessel_name`.

**Publication.** Transaction-level detail is published as Excel files
alongside the report — the *U.S. LNG Exports and Re-Exports Transaction
Details* file carries arrival/departure date, company, docket, activity, gas
type, mode of transport, supplier, tanker, point of entry/exit, destination
and volume.

**Report discontinuity.** On **17 November 2023** FECM replaced the *LNG
Monthly* and the *Natural Gas Imports and Exports Quarterly* with the
combined *Natural Gas Imports and Exports Monthly*.

**Three categories that are not jetty loadings.**

| Category | Why it must be excluded |
|---|---|
| Split cargoes | One physical shipment whose portions have different buyers, suppliers, prices, loading ports or authorisations. Counted as **multiple cargos**, flagged `[*]`. |
| ISO container exports | LNG in containers, reported separately from LNG by vessel. |
| Re-exports | Previously-imported LNG, reported in its own table. |

**Confidentiality.** Per-cargo price is not published — only a volume-weighted
average per point of exit. Volume per cargo *is* published.

**Transplantability.** Directly applicable; this is the validation target.
See D-140 and D-044.

---

# Own observations

Measurements of port_pulse's own data. These are not literature and carry
much less weight than a published finding — typically one port on one day.
Record the sample explicitly so a reader can judge.

## OBS-PHASE0 — Sabine Pass, 2022-03-15, one national file

**Sample.** `ais-2022-03-15.csv.zst`, NOAA `csv2` product, 7,994,666 national
rows. Clip box latitude 28.8–29.9, longitude −94.2 to −93.5 (final box):
88,742 rows, 203 vessels, 1.11% of the national file. Ten vessels ≥ 250 m.

**Sample is one ordinary weekday at one terminal.** Nothing here establishes
a rate, a distribution or a seasonal pattern. It establishes the presence or
absence of specific conventions, which is what it was run for.

**Schema.** snake_case, 17 columns, `longitude` before `latitude`. Renames
against SRC-NOAA-FAQ: `LAT`→`latitude`, `LON`→`longitude`,
`TransceiverClass`→`transceiver`. `heading`, `length`, `width` are `BIGINT`.

**Timestamp is UTC.** File spans 00:00:00–23:59:59. Class B Gulf Coast
activity troughs at hours 8–11 and peaks 18–23, consistent with UTC−5 local
daylight; the local-time hypothesis predicts a peak at hours 9–16, which is
close to the observed minimum.

**Unavailable values are NULL, not magic numbers.** Zero exact matches for
`sog = 102.3`, `cog = 360.0`, `heading = 511` across 88,742 rows;
`max(cog) = 359.9`, `max(heading) = 359`, `max(sog) = 36.5`. Null counts:
`sog` 15 (0.02%), `cog` 5,449 (6.1%), `heading` 32,336 (36.4%).

**Vessel types present among ≥ 250 m vessels.** Only `80` (7 vessels), `84`
(1), `60` (2, cruise ships). No nulls, zeros or out-of-range codes at this
size threshold. **LNG carriers appeared under both 80 and 84.**

**Dimensions separate LNG carriers from other tankers; draft does not.**

| Vessel | Type | Length × beam | Draft | Class |
|---|---|---|---|---|
| ASKLIPIOS | 80 | 299 × 46 | 9.3 | LNG |
| MOL HESTIA | 80 | 298 × 48 | 9.4 | LNG |
| HELLAS ATHINA | 80 | 299 × 46 | 9.6 | LNG |
| CASTILLO DE CALDELAS | 80 | 297 × 49 | 10.0 | LNG |
| LA SEINE | 84 | 299 × 46 | 11.6 | LNG |
| GASLOG GIBRALTAR | 80 | 291 × *null* | *null* | LNG |
| DUOMO SQUARE | 80 | 250 × 44 | 10.5 | tanker |
| PACIFIC SAPPHIRE | 80 | 250 × 44 | 13.1 | tanker |
| CARNIVAL BREEZE | 60 | 305 × 37 | 8.2 | cruise |
| ADVENTURE OFTHE SEAS | 60 | 311 × 49 | 8.6 | cruise |

Length and beam separate cleanly: 291–299 × 46–49 for LNG against 250 × 44
for the two non-LNG tankers, no overlap. Draft does not: the full range
8.2–13.1 m is continuous, and the two LNG carriers furthest apart on draft
(ASKLIPIOS 9.3, LA SEINE 11.6) differ by load state, not by class. **Draft is
a load-state variable and cannot classify vessel class.** See D-005.

One of ten large vessels reported neither beam nor draft despite 575
messages. `ADVENTURE OFTHE SEAS` shows a missing space — relevant to name
matching under D-140.

**Transceiver split in box.** Class A: 187 vessels, 84,238 messages. Class B:
19 vessels, 6,350 messages.

**Box sensitivity.** Moving the eastern edge from −93.4 to −93.5 cost 1,846
rows (2.0%) and 3 vessels (1.5%), while buying 6–8 km of separation from
Cameron LNG and Calcasieu Pass. See D-006.

**Not established by this run.** Whether any of the above holds for other
years — SRC-NOAA-FAQ documents format variation by year. Whether draft
changes across a loading (no vessel ≥ 250 m showed more than one distinct
draft value in the single day). Any rate, distribution or seasonal claim.

### OBS-PHASE0B — DOE LNG transaction detail, Jan 2016 – Dec 2023

**Sample.** `3. U.S. LNG Exports and Re-Exports Details (Jan 2016 - Dec
2023).xlsx`, sheet `By Vessel and ISO Container`, 7,241 rows, downloaded
2026-09-12. Three later cumulative vintages also retrieved, the most current
covering Jan 2016 – Jun 2026.

**Publication structure.** DOE publishes one **cumulative** file per year
starting Jan 2016, not one file per year. The current vintage supersedes all
earlier ones. Observed publication lag ≈ 2 months.

**Schema.** 13 columns. `Arrival/Departure Date` is a **single** column whose
meaning depends on `Activity`; in this file `Activity` is only `Exports`
(6,288) or `Re-Exports` (953), never `Imports`, so every row is a departure.
Zero date nulls. `Volume (MMCF)` confirmed against the Notes sheet.

**No `[*]` split-cargo marker exists** in the file. Searched every string
column, file-wide and scoped.

**Tanker names.** 473 distinct. Nulls 188/7,241 (2.6%), **all** on
`Mode of Transport = ISO Container`; zero nulls on the 5,609 `Vessel` rows.
No leading/trailing whitespace, no internal double spaces, no `M/V`-style
prefixes. Observed collision pairs: `Castillo De Merida`/`Castillo DeMerida`;
`GASLOG HONGKONG`/`Gaslog Hong Kong`; `JPS Bora`/`JSP BORA`;
`Seapack Hispania`/`Seapeak Hispania`; `Vivit Arabia`/`Vivit Arabia LNG`.
Fleet naming conventions producing edit-distance-1 pairs that are
**genuinely different vessels**: the `___shu Maru` series and several others.

**Points of exit.** 15 distinct values. `Sabine Pass, LA` (2,464) is clean
and unambiguous. `Cameron, LA` (792) and `Cameron (Calcasieu Pass), LA` (257)
are different terminals in the same parish. No point-of-exit value contains
"Houston".

**Sabine Pass volumes.** Q1 2022 export cargoes cluster in roughly
2,800–3,860 MMCF. Whole-file `Volume (MMCF)`: mean 2,509.5, median 3,299.1,
sd 1,482.9 — bimodal, with a second population at 1–30 MMCF corresponding to
small-scale Caribbean and Puerto Rico distribution, not export-terminal
cargo.

**Split cargoes, Sabine Pass 2020–2023.** 1,560 rows collapse to 1,482
loadings on (Tanker, Date, Point of Exit) — 75 multi-row combos, 78 rows
removed, 5.0%. Summed volume per combo: mean 3,482.1, sd 328.7, min 2,913.3,
max 5,295.6.

**Q1 2022 Sabine Pass: 109 rows → 105 loadings** (37 / 31 / 37).

**Freeport outage, exact dates.** Last export before the gap **2022-06-07**;
first export after **2023-02-12**; 250 days. July 2022 – January 2023 are
zero. February 2023 is a partial recovery (6), reaching 26 by May 2023.

**Cross-checks against OBS-PHASE0.** `Gaslog Gibraltar` — DOE departure
2022-03-16, present in the AIS box 2022-03-15. `La Seine` — DOE departure
2022-03-14, present in the AIS box 2022-03-15. Both consistent with berth
exit plus in-box transit.

**Not established by this run.** Anything about AIS detection. Whether the
schema holds in the 2024–2026 vintages (D1–D6 were run on the 2016–2023
file). Loading duration, which is assumed from domain knowledge, not
measured.


### OBS-PHASE1-PRE — DOE vintage re-check and NOAA Q1 2022 partition index

**Sample.** `3. U.S. LNG Exports and Re-Exports Details (Jan 2016 - Jun
2026).xlsx`, 12,094 rows, versus the 2016–2023 vintage (7,241 rows). NOAA
`csv2/csv2022` blob index for 2022-01-01 to 2022-03-31, via the List Blobs
REST API with pagination confirmed complete. Run 2026-09-13.

**DOE schema is stable across vintages.** Identical sheet names and columns.
`[*]` marker still absent, confirming the OBS-PHASE0B finding was not an
artefact of one vintage.

**Sabine Pass 2020–2023 is unrevised.** 1,560 rows, 1,482 distinct (Tanker,
Date, Point of Exit) combos, 75 multi-row combos — identical in both
vintages. **Q1 2022 remains 109 rows collapsing to 105 loadings**, same four
splits.

**One malformed row, new.** Null `Arrival/Departure Date` (1 of 12,094; was 0
of 7,241) *and* a non-null `Tanker` on a `Mode of Transport = ISO Container`
row, which was 100% null in the earlier vintage. Eagle LNG Partners
Jacksonville II, Ft. Lauderdale, 2.53 MMCF. Out of scope, but the loader must
tolerate it rather than choke.

**Points of exit grew from 15 to 20 values.** New: `Plaquemines, LA` (425
rows, from 2024-12-26), `Altamira, Tamaulipas, MX` (25), `West Palm Beach,
FL` (13), `Golden Pass, TX` (3, from 2026-04-22), `Port of Savannah, GA` (1).
`Sabine Pass, LA` unchanged and still distinct from `Cameron, LA` and
`Cameron (Calcasieu Pass), LA`.

**Golden Pass LNG first cargo 2026-04-22** (AL QA'IYYAH, Belgium, 3,619.46
MMCF), then 2026-05-08 and 2026-06-25.

**Altamira, Tamaulipas, MX appears in a field DOE documents as US cities and
states.** Five sampled rows are all `Exports` by `Vessel`. Unresolved —
either a transshipment reporting pattern or a data-entry deviation. Out of
scope; recorded so it is not rediscovered.

**NOAA Q1 2022 partitions: all 90 days present.** Sizes 105.9–235.1 MB.

**Two outage-detection methods compared.** A trailing-7-day median at −2 SD
flagged ten days: 01-29, 01-30, 02-22, 02-23, 02-24, 03-12, 03-13, 03-14,
03-20, 03-21. Same-weekday baselining flags twelve: 01-29 to 01-31, 02-05 to
02-10, 03-14, 03-20 to 03-21.

Disagreements: 02-22 (190.1 MB) is above the February Tuesday median; 02-23
and 02-24 sit at their weekday medians — all three are trailing-window
artefacts. 03-12 and 03-13 are a normal weekend. **02-05 to 02-10 is a
six-day depression the trailing method missed entirely**, e.g. 02-09 at 139.7
against other February Wednesdays at 195.2, 208.7 and 179.6.

**Not established by this run.** Whether the size anomalies correspond to
actual message loss in the Sabine Pass box, or to national-level dips that
leave Gulf coverage intact. Whether the DOE schema holds for the 2024–2026
rows specifically — D4 was re-run on the 2020–2023 subset only.


---

# Parameter cross-reference

Every numeric threshold appearing in the corpus, with its source. **These are
what the papers did, not what port_pulse does** — see `decisions.md` for what we
adopted.

| Parameter | Value | Source | Adopted? |
|---|---|---|---|
| "Stopped" speed threshold | 1 knot | LIT-YAN-2022 | See D-020 |
| "Stopped" speed threshold | SOG = 0 (moored split) | LIT-EMMENS-2021 | Rejected, D-020 |
| Adjacent-point time gap | 1.5 h | LIT-YAN-2022 | See D-021 |
| Adjacent-point distance | 2 km | LIT-YAN-2022 | See D-021 |
| Min stop duration | 1.5 h | LIT-YAN-2022 | See D-022 |
| Min points per stop segment | 10 | LIT-YAN-2022 | See D-022 |
| Voyage grouping window | 24 h | LIT-MARTINCIC-2021 | See D-023 |
| Voyage split gap | 5 h **and** >100 m movement | LIT-MARTINCIC-2021 | See D-023 |
| Interpolation period | 1 hour | LIT-BERTH-2025 | Not applicable, D-024 |
| Heading-change filter | > 10° | LIT-BERTH-2025 | See D-025 |
| Geohash precision | 9 (4.7 m) | LIT-BERTH-2025 | Not adopted |
| Observation window for berth derivation | 1 month | LIT-BERTH-2025 | See D-024 |
| KNN neighbours for status classification | ≥ 300 | LIT-MARTINCIC-2021 | Not adopted, D-026 |
| AIS↔official port call match window | ± 2 days | LIT-IMF-WORLD-2020 | See D-040, D-140 |
| Minimum vessel length (LNG population) | 250 m | **uncited** — scope choice | See D-117 |
| Implied-speed quarantine | 40 knots | **uncited** — see D-018 | See D-018 |
| UTC shift applied to AIS | 6 hours | LIT-IMF-WORLD-2020 | Rejected, D-011 |
| Weekly series smoothing | 5-term centred MA | LIT-IMF-MALTA-2019 | See D-031 |
| Port buffer radius | 10 km | LIT-YAN-2022 | See D-013 |
| Anchorage distance from port | within 5 NM | LIT-YAN-2022 | See D-013 |

---

# Gaps in the literature

Things port_pulse needs that **no source in this knowledge base covers.** Do not
cite a paper for these; flag the decision as uncited in `decisions.md`.

1. **No source uses the NOAA Marine Cadastre dataset.** Every paper here uses
   MarineTraffic, AISHub, the UN Global Platform, or an unnamed provider. The
   AVIS/AVID corrections, the one-minute downsample, the removal of voyage
   fields and the 145–165 day latency are all dataset-specific and appear only in
   SRC-NOAA-FAQ.
2. **No source addresses the lead-lag question directly.** The IMF papers
   establish that AIS indicators *track* official statistics contemporaneously
   and validate against them. None tests whether an AIS congestion measure
   *leads* throughput, by how long, or under what conditions the lead breaks.
   **This is the project's actual contribution and it is uncited by design.**
3. **No source defines a congestion metric.** LIT-FENG-2020 measures time
   efficiency, LIT-MARTINCIC-2021 measures waiting and turnaround. Neither
   constructs a port-level congestion index, and neither treats the
   congestion-induced message loss problem.
4. **No source quantifies the message-loss / congestion endogeneity.**
   LIT-EMMENS-2021 establishes that dense traffic degrades AIS and
   LIT-ANDROJNA-2021 explains the mechanism, but nobody measures how much signal
   is lost as a function of vessel density, nor corrects for it.
5. **No vessel register is available.** DWT, design draught, ballast draught and
   ship class (container vs bulk vs ro-ro) are used by every IMF paper and are
   not obtainable from our source. All draught-based cargo estimation is
   therefore out of reach unless a register is added to scope.
6. **No source distinguishes container ships from other cargo ships using AIS
   alone**, because it is not possible — LIT-HARATI-2007 establishes the category
   list collapses them. This is a constraint, not an open question.
7. **IMF PortWatch methodology is documented but its schema is not.** We have
   the papers behind PortWatch (LIT-IMF-PORTWATCH) but no field-level
   documentation for the published daily port call files.
8. **No US-specific berth or anchorage polygons.** LIT-FENG-2020 uses nautical
   chart VTS lines; LIT-BERTH-2025 derives berths but needs dimensions A/B/C/D
   we do not have.
