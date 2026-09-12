# Learned

## Possible points of failure

- **Congestion** — when there is congestion in the polygon port, many transmissions in the
  same area lead to collision of messages, and the network becomes inflated resulting in
  transmission failure. Congestion is when there are many vessels in a specific area,
  therefore message failure is a function of vessel density.

  - **Things to keep in mind:**
    1. track messages-per-vessel-per-hour metric inside the polygon, alongside the
       congestion metric, if they are negatively correlated, it proves the above.
    2. metrics developed must be robust to message loss, vessel-hours estimated from
       arrival/departure transitions survives sparse sampling
- **Data**

  - **Primary Key** - [MMSI, Segment], however, Segment break on implausible discontinuities.
    Eventhough IMO is fixed at ship construction, from 2024, NOAA's AVID model populates
    this column with NULL.
  - **Berth Occupancy** - Buoy has MMSI prefix *99MIDxxxx*, crafts associated with parent ship
    such as lifeboats (*98MIDxxxx*), SART, MOB, EPIRB devices (*970/972/974...*), coast/base
    stations (*00MIDxxxx*), and SAR aircraft (*111MIDxxx*) inside the polygon have permanent
    SOG 0 (berthed). Every port has a few permanently occupied phantom berths and your baseline
    is inflated by a constant.
  - **LAT/LON:**
    1. Points on land: GPS positional error, vessels in narrow inland waterways, and vessels
       physically transported by land all produce coordinates ashore. Important to keep this to
       accurately estimate port traffic.
    2. Points implausibly far offshore: high-elevation receivers routinely detect commercial
       vessels at great distance.
    3. Genuine spoofing: since the study is focused on only US waters, which is less suspectible
       to spoofing, we can de-prioritise the need for solutions for this in our study.
       **Solutions:** Filter on derived speed between consecutive fixes. Emmens et al. computed
       Haversine distance between consecutive points per vessel specifically to do this, and found
       that transmitted SOG frequently disagrees with computed speed — meaning either the position
       or the speed is noisy. That disagreement is your detector. A vessel whose implied speed exceeds
       ~40 knots has a bad position, a bad timestamp, or a shared MMSI, and all three warrant quarantine
       rather than deletion.
  - **Speed-over-Ground (SOG):** 102.3 means unavailable. A stopped ship is not a ship at zero knots,
    it stays within a small positive range determined by whether it's anchored or berthed and by the
    current state.
  - **Course-over-Ground (COG) and Heading:** COG = 360.0 and Heading = 511 mean unavailable, same
    sentinel-poisoning risk as SOG. A vessel at anchor swings around its anchor as the tide and wind
    turn, heading varies through a wide arc while position traces a circle. A vessel at a berth is
    physically fixed, heading is near-constant and position barely moves. So Heading variance over a
    stop segment, plus positional spread, gives you berth vs anchorage classification without needing
    hand-drawn terminal polygons.
  - **VesselType and Cargo:** VesselType is the field with the worst error rate in the literature and
    the most tampering by NOAA. Tugs and dredgers, "navigational status" is encoded in the ship type
    field rather than the status field. A tug reads as type "tug" when free, and switches to "towing" when
    it picks up a tow. Container vessel, car carrier and bulk carrier are all identified as cargo, and
    chemical tanker, petroleum tanker and gas carrier are all identified as tanker.
    1. Cargo is not cargo. For 2015–2023 it holds the original raw vessel type. Do not use it to infer what
    a ship is carrying. "Cargo/tanker vessels" selects on a field whose correction regime changed in 2018 and
    again in 2024. Between 2015 and 2023, AVIS replaced vessel_type, length, width and draft, and the original
    uncorrected NMEA type code was moved into the cargo field for that period. From 2024, AVID does the same
    job differently.
    2. AIS type 70–79 is "cargo". But since we are focusing our study on LNG/Gas shipping, we need to scope
    our study by either:
    (a) validate against total tonnage or total cargo tonnage rather than Twenty-Foot Equivalent Unit (TEU)
    (b) use Length/Width as a crude vessel-class proxy, since post-Panamax box ships have a fairly
    distinctive beam.
  - **Status:** Two independent studies, two decades apart, come to the conclusion that Status is not reliable
    measure for occupancy. Vessels can only be moored at terminals, yet sometimes report moored status in
    anchorage areas, or even while sailing. We derive occupancy geometrically
    (position + speed + heading variance, per Yan and Martinčič), then use Status as a cross-check, and report
    the disagreement rate as a data quality metric.
  - **Length and Width:** Derived from the transponder's four antenna offsets — A (bow), B (stern), C (port),
    D (starboard). NOAA collapses them to Length = A+B and Width = C+D. Treat as an ordinal band
    (Panamax / post-Panamax / feeder) rather than a continuous number, since the individual values are unreliable
    but the bands are mostly right.
  - **TransceiverClass:** Filter to Class A (cargo/tanker population), it's the population with mandatory carriage
    under SOLAS, and it's the population least affected by slot contention.

