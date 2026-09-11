# Literature

## Port call detection
- **Cerdeiro et al. 2020 (IMF WP/20/57)** — DBSCAN to derive port boundaries
  from AIS, then port-to-port voyages. We use hand-drawn geofences instead
  (IB-SMoT style): simpler, auditable, and sufficient for 3 known ports.
  Revisit if we scale to many ports.
- **[next paper]** — ...

## Congestion metrics
- **GSCSI-M** — decomposes port-to-port transit time into voyage lead time
  plus turnaround. Our dwell metric follows this split.

## Data quality
- **Harati-Mokhtari et al. 2007** — duplicate MMSIs, timestamp errors,
  stale retransmissions. Drives the dedup rules in silver layer.
