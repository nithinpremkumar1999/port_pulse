# port_pulse

## What this is

port_pulse turns raw AIS vessel position data into weekly port congestion
metrics, then tests whether those metrics *lead* officially reported port
throughput. Port statistics are published weeks in arrears; AIS is near
real-time. The core question: does congestion derived from AIS lead
reported volumes, by how long, and where does the relationship break?

## Data

- **Source**: NOAA Marine Cadastre AIS (CC0 public domain), US coastal
  waters, ~1-2GB per daily file.
- **Validation**: IMF PortWatch daily port calls and published port
  authority throughput.
- `data/` is gitignored except a single-day sample fixture under
  `tests/fixtures/`, which CI and `make demo` run against.

## Stack

- Python 3.11
- DuckDB as the warehouse
- pandas only from the aggregated layer onwards (not in bronze/silver)
- pytest

## Hard rules

- **Never load a raw daily AIS file into memory.** DuckDB reads
  Parquet/CSV directly with filters pushed into the scan.
- **Ingest loop**: download one day -> clip to port polygons -> write
  Parquet partition -> delete raw file. Peak disk usage must stay bounded
  regardless of backfill length.
- **Partitions are idempotent**: rerunning a date produces identical
  output; skip if the partition already exists.
- **Check free disk before each download**; fail loudly rather than
  filling the disk.
- **Known data quirks** to account for: sentinel values (COG 360.0, SOG
  102.3 both mean "unavailable"), duplicate MMSIs, receiver coverage gaps
  lasting hours to days.

## Pipeline layers

1. **bronze** — clipped raw AIS (port polygons only)
2. **silver** — cargo/tanker vessels, cleaned (sentinels handled, dupes
   resolved)
3. **gold** — port call events
4. **metrics** — weekly per-port congestion metrics

## Conventions

- SQL lives in `.sql` files, not inline strings.
- Every pipeline stage has a test against the sample fixture in
  `tests/fixtures/`.
- Type hints on public functions.

## Commands

- Activate env: `source .venv/bin/activate` (always; nothing works without it)
- Tests: `pytest`
- Single-day demo against the fixture: `make demo`
- Backfill: `python -m src.ingest --start YYYY-MM-DD --end YYYY-MM-DD`

## Never

- Commit anything from `data/` except the fixture.
- Silently drop rows. Every filter logs rows in, rows out, and the reason.
- Fabricate or interpolate missing data. Gaps stay gaps and get reported.
- Hard-code a port. Ports come from a config file with polygons and metadata.

## Methodology

Threshold and algorithm choices live in `docs/decisions.md`, with sources in
`docs/literature.md`. Before implementing anything involving a threshold or
detection rule, read `docs/decisions.md`. If a needed decision isn't recorded
there, stop and ask rather than inventing a value.