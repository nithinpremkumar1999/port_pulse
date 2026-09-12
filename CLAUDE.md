# port_pulse

## What this is

port_pulse reconstructs terminal-level LNG export activity from raw AIS
vessel positions, validated cargo-by-cargo against official records, and
surfaces operational detail that published statistics do not contain.

An LNG carrier arrives in ballast, berths, loads, departs laden. Laden
departures multiplied by vessel capacity approximate export volume. Official
figures publish with a lag; the estimator observes the event as it happens.

**Framing — read this before writing any README copy.** The free NOAA archive
delivers with a lag of roughly 75–165 days depending on position in the
quarterly release cycle (measured at 165 days on 2026-09-12). This project is
therefore a **validated estimator, not a live nowcast**. The estimator's lead
over official publication is a property of the method; the archive's delivery
lag is a property of the free distribution channel. Never describe port_pulse
as nowcasting or real-time without qualifying the source latency — an
informed reader will check.

Two claims that survive regardless of latency:

1. **Accuracy** — how closely AIS-derived cargo counts and volumes match
   official records, measured per cargo, not by correlation.
2. **Granularity** — berth occupancy hours, loading duration, turnaround,
   waiting time, vessel-class mix. Official statistics report that a cargo
   left; AIS shows it took 19 hours to load and waited 4 hours for a berth.
   Nobody publishes this.

This is a *throughput estimator*, not a congestion model. LNG loading slots
are contracted and scheduled, so queueing is rare and congestion carries
little signal. Congestion logic applies to Houston only.

## Ports

| Port | Why |
|---|---|
| Sabine Pass (Cameron Parish, LA) | Largest, longest continuous history |
| Freeport (Quintana Island, TX) | 2022 outage — exogenous shock, out-of-sample test |
| Corpus Christi (TX) | Growth plus Stage 3 ramp |
| Houston Ship Channel | **Control** — congested multi-commodity port, not LNG |

Houston is a deliberate contrast, not a fourth LNG terminal. It is also the
largest data volume of the four.

Ports come from a config file (polygons, berth geometry, metadata). Never
hard-code a port.

## Data

**Source:** NOAA Marine Cadastre AIS, CC0 public domain, US coastal waters.
Terrestrial receivers only — no satellite data is redistributable.
~8M rows per national daily file.

**Host:** Azure blob storage, e.g.
`https://noaaocm.blob.core.windows.net/ais/csv2/csv2026/ais-2026-04-01.csv.zst`

**Availability:** confirmed 2020 through March 2026. Paths, naming and
compression vary by year — the 2026 pattern does NOT generalise. The old
`coast.noaa.gov/htdata/...` paths 404 despite still being listed; a directory
listing is not evidence a file exists.

**Before any backfill:** build a date manifest by HEAD request across the
full range, recording URL, format and status per date. Treat this as a
pipeline step, not a one-off script.

**Compression:** `.zst` files should be readable directly by DuckDB without
unzipping. Verify — if it works, peak disk drops from ~2 GB to ~320 MB per
day and the unzip step disappears.

**Validation:** DOE LNG Monthly report (cargo-level — vessel name, terminal,
volume, destination) is the primary ground truth, enabling per-cargo
precision/recall rather than aggregate correlation. EIA monthly export
statistics as a secondary aggregate check. IMF PortWatch only for Houston,
and only if it resolves these terminals — verify before relying on it.

**Coverage note:** every downloadable AIS day already has published DOE and
EIA figures against it, so the entire history can be validated with no
truncation at the recent end.

`data/` is gitignored except a single-day sample fixture under
`tests/fixtures/`, which CI and `make demo` run against.

## The identification problem

**There is no AIS vessel type code for "LNG carrier."** Type 80–89 covers all
tankers. LNG carriers must be identified by some combination of:

- dimensions (conventional membrane vessels cluster near 290–300 m length,
  45–49 m beam; Q-Flex/Q-Max larger)
- berth geometry (only LNG carriers berth at LNG jetties)
- vessel-name matching against the DOE cargo roster

Unsolved. Check `docs/decisions.md` before implementing any filter.

## Stack

- Python 3.11
- DuckDB as the warehouse
- pandas only from the aggregated layer onwards (not in bronze/silver)
- pytest

## Hard rules

- **Never load a raw daily AIS file into memory.** DuckDB reads the
  compressed file directly with filters pushed into the scan.
- **Ingest loop**: download one day -> clip to port polygons -> write Parquet
  partition -> delete raw file. Peak disk bounded regardless of backfill
  length.
- **Partitions are idempotent**: rerunning a date produces identical output;
  skip if the partition already exists.
- **Check free disk before each download**; fail loudly rather than filling
  the disk.
- **Bronze is the archive of record.** Upstream paths have already moved once
  and format shifts by year. Once written, bronze partitions must be backed
  up outside `data/`. Losing them is not a rerun.
- **Port calls span multiple days.** Detection cannot run per daily partition
  — it needs a multi-day window or open-call state carried across runs. This
  is the key architectural constraint.

## Known data quirks

- Sentinel values: `SOG = 102.3` and `COG = 360.0` both mean "unavailable"
- `COG` and `Heading` are noise on stationary vessels — unusable in detection
  logic at rest
- Message frequency collapses when moored (~1/min under way, gaps of an hour
  or more at berth). Gap thresholds must be set from the stationary tail, not
  the overall distribution
- Duplicate/spoofed MMSIs — impossible position jumps within a single minute
- Corrupt positions far outside US coastal waters; needs a bounds filter
- `Status` (0 under way, 1 at anchor, 5 moored, 15 undefined) is
  crew-reported and patchy. Candidate validation label, not a detector
- `Draft` is crew-reported. Whether ballast/laden transitions are reliably
  updated is an open question — verify before relying on it

## Pipeline layers

1. **bronze** — clipped raw AIS (port polygons only)
2. **silver** — tankers, cleaned (sentinels handled, dupes resolved,
   bounds-filtered)
3. **gold** — loading cycle events: ballast arrival, berth, laden departure
4. **metrics** — per-terminal estimated export volume plus operational
   metrics (berth hours, loading duration, turnaround, waiting time)

## Never

- Commit anything from `data/` except the fixture
- Silently drop rows. Every filter logs rows in, rows out, and the reason
- Fabricate or interpolate missing data. Gaps stay gaps and get reported
- Invent a threshold. If a needed decision isn't in `docs/decisions.md`, stop
  and ask
- Describe the pipeline as real-time or nowcasting without stating the
  source lag

## Methodology

Threshold and algorithm choices live in `docs/decisions.md`, with sources in
`docs/literature.md`. Read `docs/decisions.md` before implementing anything
involving a threshold or detection rule.

## Conventions

- SQL lives in `.sql` files, not inline strings
- Every pipeline stage has a test against the sample fixture
- Type hints on public functions

## Commands

- Activate env: `source .venv/bin/activate` (nothing works without it)
- Tests: `pytest`
- Single-day demo against the fixture: `make demo`
- Backfill: `python -m src.ingest --start YYYY-MM-DD --end YYYY-MM-DD`

## Stretch goal

A live AIS source (e.g. AISStream.io — free WebSocket, bounding-box
subscription) running on the same pipeline would demonstrate the architecture
is source-agnostic and close the latency gap. Requires verifying Gulf coast
coverage first. Two sources with different latencies sharing a bronze schema
introduces real reconciliation problems (backfill vs stream, late-arriving
data, overlap deduplication) — treat as a genuine design exercise, not a
bolt-on.
