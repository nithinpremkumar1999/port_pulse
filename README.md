# port_pulse

Turning raw AIS vessel positions into weekly port congestion metrics,
and testing whether they lead officially reported port throughput.

Work in progress.



Bronze Layer: Clip Polygon, deliberately loose polygon, since its job is to remove 98%+ of data and focus on nearby areas of the port.
Gold Layer: Berth Polygon, must be tight, signalling loading. Waiting Polygon, signalling queueing.

Gold Layer clipping cannot be made at Bronze level without months of data, therefore, a generous net is casted at Bronze layer.