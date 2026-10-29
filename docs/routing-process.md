# Route planning process

Design notes for the planned route optimiser (traveling salesman style, with
traffic, weather and crime weightings). Target scale used for planning: about
30,000 parcels per day across Gauteng, with up to 2,000 vehicles available.
This is a design only; nothing here is implemented or validated on real data.
See `data-sources.md` for external data sources.

## Principle

Do not hand 30,000 stops to one solver. Decompose: group parcels into
clusters, fix the clusters against real road conditions, then route each
cluster on its own. The number of clusters in the final feasible plan is the
vehicle count. 2,000 is a cap, not a target.

## 1. Pre-processing

- Consolidate parcels with the same address into one stop.
- Snap every stop to the road network. Flag stops that cannot be snapped or
  reached for manual review.
- If there are several depots, assign stops to depots first.

## 2. Lower bounds on fleet size

The larger of these wins:

- Capacity: `ceil(total load / vehicle capacity)`, per dimension (parcels,
  weight, volume).
- Time: `ceil((total service time + estimated travel time) / shift length)`.

Parcel work is usually time-bound. Large truck loads are usually capacity-bound.

Illustration only (assumed numbers, not real data): 30,000 stops, 5 min
service, 3 min driving between stops, 540 min shift gives about 445 vehicles
by the time bound, against 200 by a 150-parcel capacity bound.

## 3. Coarse clustering

- Cluster geographically with size, capacity and time limits. Plain k-means is
  not enough because it ignores those limits.
- To avoid a full 30,000 x 30,000 drive-time matrix, cluster on a coarse grid
  (for example H3 cells) using drive times between cell centres.
- Pre-cut known barriers (for example the Magaliesberg ridge, large rivers,
  the Johannesburg-Pretoria corridor) into separate zones where useful.
- Aim for slightly fewer clusters than vehicles to leave slack.

## 4. Check each cluster on the road network

Using drive times within the cluster only:

- Detour ratio: drive time / straight-line time between members.
- Connectivity: members must be linked by a reasonable road path.
- Spread: longest drive between members must stay under the limit.

Straight-line closeness is only the starting guess. Rivers, ridges,
highways and missing interchanges are caught here, so no hand-coded geography
is needed beyond hard rules for closed roads or areas not to be served together.

## 5. Collapse and rebalance

For each flagged cluster, worst first:

1. Dissolve it and release its parcels.
2. Find neighbouring clusters by road drive time, not by map adjacency.
3. Reinsert each parcel, hardest to place first, into the neighbour where it
   adds the least drive time, subject to capacity and shift limits.
4. Rebalance the affected neighbours with move and swap passes scored on real
   drive time.
5. Re-measure the touched clusters and repeat if any are flagged.

Guards: cap the number of rounds, accept only moves that lower total cost,
keep a short log of recent moves to prevent oscillation, give isolated
parcels a dedicated vehicle or a manual flag, and if neighbours cannot absorb
the load, merge with one neighbour and re-split along a better line.

## 6. Route each cluster

- Solve each cluster independently and in parallel (OR-Tools, LKH, or local
  search such as 2-opt and Or-opt).
- Apply traffic and weather as time-of-day and forecast multipliers on travel
  time. Apply crime as a risk penalty on areas or arcs, or as limits on late
  drops in high-risk zones.
- Use the same weights when clustering in step 3, so clustering and routing
  optimise the same cost.
- If a cluster has no feasible route (for example a missed time window), flag
  it and send it back through step 5.

## 7. Boundary refinement and final check

- One move and swap pass across neighbouring routes to recover quality lost
  to decomposition.
- Final feasibility check against shift limits, with repair of any breaches.

## Persistent zones and collections

Clusters can be kept as standing zones. Each zone has one driver and vehicle,
and the vehicle stays in the zone after deliveries.

- Store zones as sets of H3 cells so a location maps to a zone with one lookup.
  Snap collection addresses to the road network first.
- New collection: assign it to its zone and therefore its driver. Near a
  boundary, compare the extra drive time in each neighbouring zone and pick
  the cheaper one that still fits.
- Insert into the driver's current route by cheapest insertion, checking
  capacity, time windows and shift end. If it does not fit, hand it to a
  neighbouring zone or schedule it for the next day. Use a cut-off time for
  same-day collections.
- Reserve some spare capacity per zone, and define backup zones for absences.
- Zone record: id, cells, driver, vehicle, depot, neighbour list with drive
  times, daily load and spare capacity.

### Rebuild policy

- Rebuild on a schedule (around every 2 months as a first guess) from recent
  volumes per cell. Use last year's equivalent period before seasonal peaks.
- Also check zone health weekly and rebuild early if several zones stay
  overloaded or underused (for example above 90% or below 50% of capacity
  for three weeks).
- Between rebuilds, move only a few boundary cells at a time (for example no
  more than 5 to 10% of a zone's cells per week).
- Start any rebuild from the current zones and driver assignments, compare the
  result with the old zones, and keep the old ones if the gain is small.
- Tune the interval from how fast zone loads actually drift.

## Large truck loads

The same pipeline applies, with these differences:

- Capacity usually dominates, so clustering is capacity-first, closer to bin
  packing with a geographic preference.
- Few stops per truck over long legs, so the clustering step carries most of
  the cost.
- Loading order, axle weight limits and goods compatibility restrict which
  consignments share a truck.
- Heavy vehicles need a truck routing profile (height, weight, bridge limits,
  urban truck bans).
- Driver hours and rest limits may split a route across days.
- Return legs matter; chaining pickups with deliveries may be worth modelling.
- Tolls, fuel and distance cost weigh more than minutes in the objective.

Make the pipeline take a vehicle profile so the same steps run in parcel mode
or load mode.

## Open questions

- Depot count and locations.
- Vehicle capacities, shift length, service time per stop, delivery windows.
- Whether a managed routing service already covers the 30,000 stop case
  (limits and pricing not yet checked).
