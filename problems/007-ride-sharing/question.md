---
id: 7
title: Design Uber / Lyft (Ride Sharing)
category: Geospatial
topics: [geohash, h3, matching, dispatch, real-time location]
difficulty: Hard
solution: solution.md
---

## Scene

The interviewer slides their laptop toward you and says:

> *Design Uber. A rider opens the app, taps "Request Ride," and within a few seconds a nearby driver accepts and starts heading their way. Walk me through how that works end to end.*

They lean back. There is a clock on the wall and they are watching the time.

This problem looks like "fetch a list of nearby drivers" but the interviewer is really asking three different things in one: how do you track a million moving vehicles in real time, how do you match a rider to one driver in under two seconds, and how do you keep the ride alive through twenty minutes of network drops, app backgrounds, and driver cancellations. Get the geospatial index wrong and the matching service grinds; get the state machine wrong and you charge people for rides that never happened.

Before you draw anything, ask what is actually in scope.

## Step 1: clarify before you design

Take 5 minutes. Do not draw boxes yet. What would you ask? Aim for at least seven questions that meaningfully change the design.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Scope.** "Are we designing matching and dispatch only, or also payments, surge pricing, ETA prediction, in-ride chat, and driver onboarding?" Ride-sharing is huge; if you try to do everything in 45 minutes you will do nothing well. A reasonable scope: rider requests a ride, system finds and assigns a driver, both sides see updates until pickup. Surge and ETA can be mentioned briefly. Payments and onboarding are explicitly out.
2. **Scale.** "How many active drivers, how many rider requests per second, what cities?" Uber-like answers: ~1M drivers online during peak, ~5M riders active, ~100M trips/day globally, single hottest city ~50k drivers. The single-city number drives shard sizing.
3. **Matching latency target.** "From tap on 'Request Ride' to a driver assigned, what is the P99?" Under 2 seconds is the bar for the rider not to abandon. Driver acceptance can add a few more seconds.
4. **Driver location update frequency.** "How often do driver apps report position?" Typical: every 4 seconds when online, every 1 second once a ride is matched and they are en route. This drives ingest throughput.
5. **Matching policy.** "Closest driver wins? Best-rated within a radius? Optimized batch assignment across multiple pending requests?" Greedy nearest is simple. Batch optimization gives better global outcomes but adds latency and complexity.
6. **Cancellations and re-matching.** "What happens if the assigned driver cancels, or never accepts? Do we re-match automatically?" Yes, almost always. The state machine must handle this without billing surprises.
7. **Geographic distribution.** "Is matching global, or sharded by city/region?" City-sharded. There is no benefit to matching a rider in Tokyo against a driver in Lagos. The shard boundary is roughly a metropolitan area.
8. **Vehicle types and pools.** "UberX vs UberXL vs UberPool? Different driver pools or one pool with filters?" One pool with a `vehicle_class` attribute and filters at match time. Pool/shared rides add a routing dimension we will not solve here.

If you walked in asking only "how do we find nearby drivers?" you missed the harder half. The state machine, the location ingest, and the city-sharding are equally important.

</details>

## Step 2: capacity estimates

Inputs from the interviewer:

- 1M drivers online globally during peak hour
- 5M riders active during peak hour
- 100M trips/day
- Driver location update every 4 seconds when online, every 1 second once on a trip
- Single hottest city: 50k drivers online (NYC, Sao Paulo on New Year's Eve)
- Average trip duration: 15 minutes
- Match latency P99 target: 2 seconds

Compute (do this before revealing):

1. Trip requests per second (sustained, peak)
2. Location updates per second (sustained, peak)
3. Bandwidth for location ingest
4. Storage for location history per day
5. Concurrent active trips at any moment

<details>
<summary><b>Reveal: the math</b></summary>

**Trip requests per second.**
100M / 86400 ≈ 1160 requests/sec sustained. Peak is 5-10x sustained (Friday night rush, surge events) → **~10k requests/sec peak**.

**Location updates per second.**
- Idle online drivers: 1M × (1/4 sec) = **250k updates/sec** sustained.
- On-trip drivers: ~30% of online drivers are on a trip on average. 300k × (1/1 sec) = **300k more updates/sec**.
- Total: **~550k updates/sec sustained**, ~1M/sec peak.

This is the dominant write workload. The matching workload is tiny by comparison.

**Bandwidth for location ingest.**
Each location packet is ~100 bytes (driver_id, lat, lng, heading, speed, accuracy, ts, signature). 550k × 100B = **55 MB/sec sustained**. Fine for ingest, but you do not write all of that to durable storage; most of it is overwrite-in-place in a fast store.

**Storage for location history per day.**
If we kept every update: 550k × 86400 × 100B = ~4.7 TB/day. We do not keep every update. Two stores:
- **Current location store (hot, overwrite-in-place):** 1M rows × ~200B = ~200MB total. Trivial.
- **Trip path history (durable, ~1 update every 4 sec during a trip):** 100M trips × ~225 location pings each × 100B = ~2 TB/day. Compressed and aged off after 90 days for fraud and dispute resolution.

**Concurrent active trips.**
100M trips/day × 15 min / 1440 min/day = ~1M trips active at any moment. Sharded by city, the biggest cities have ~50-100k concurrent active trips each.

**Key insight from the math.** The system has two extremely different workloads in one. Location updates are 250k+/sec, mostly writes to a fast in-memory store. Trip matching is ~10k/sec, computationally cheap but latency-critical. Most of the architecture exists to keep these two paths separate so they do not interfere.

</details>

## Step 3: choose a geospatial index

You need to answer two queries quickly:

- "Given a rider at (lat, lng), find all online drivers within R meters."
- "When a driver moves, update their location in the index."

Naive approach: store every driver's (lat, lng) in a B-tree on lat, then filter by lng and distance. This is awful; the index does not understand 2D locality. Real options: geohash, quadtree, H3, S2. Take 10 minutes to compare them before revealing.

<details>
<summary><b>Reveal: geospatial index comparison</b></summary>

| Index | Shape | Strengths | Weaknesses |
|-------|-------|-----------|------------|
| **Geohash** | Rectangular grid, base-32 string prefix encoding (e.g., `dr5ru`). Longer prefix = smaller cell. | Trivial to compute, trivial to shard (just hash the prefix), works in any KV store. Range queries on prefix are easy. | Cells near the equator are wider than near the poles. Neighbor cells of a corner cell may share only one character of prefix, so "find neighbors" needs a small lookup table. Two points close in space can have very different geohash prefixes if they straddle a cell boundary. |
| **Quadtree** | Recursive 4-way subdivision of a rectangle. Each region is a node. | Adapts cell size to data density (deep trees in dense areas, shallow in sparse). Good for static datasets. | Tree updates on every move are expensive. Hard to shard because the tree shape is mutable. Out of favor for real-time. |
| **H3 (Uber's choice)** | Hexagonal hierarchical grid. Resolution 0 (largest) to 15 (smallest). At resolution 9, cells are ~174m across. | Hexagons have uniform distance to all 6 neighbors (squares have 4 close + 4 diagonal, which is awkward). Built-in `kRing` function: "give me all cells within k rings of cell X" is one call. Cell sizes are roughly uniform globally. | Hexagons cannot perfectly tile the sphere; H3 uses 12 pentagons to close the gaps, which become edge cases. Indexes are 64-bit integers, not human-readable. |
| **S2** | Quadrilateral cells on a projected sphere (Hilbert curve traversal). Used by Google. | Strong locality guarantees: nearby cells have nearby IDs along the Hilbert curve, so range scans work well. Variable resolution. | Cells are not uniform in shape (some are skinnier than others). Less intuitive than hexagons. |

**Recommendation: H3 at resolution 8 or 9.**

- **Uniform neighbor distance.** A driver in the center of an H3 cell is roughly equidistant from all 6 neighboring cells. With geohash, the diagonal neighbors are √2 times farther than the edge neighbors, and your "search within R meters" code has to handle that. With H3, "give me cells within 1 ring" is one function call (`kRing(cell, 1)`) and returns exactly the 7 cells (the center plus 6 neighbors).
- **Resolution 9 (174m edge length, ~0.1 sq km area).** Small enough that "all drivers in this cell" is a tractable list (~10-100 drivers per cell in a dense city), large enough that most matches happen within one ring.
- **Sharding.** Use a coarser resolution (resolution 5 or 6, several km across) as the shard key. A shard owns all the high-resolution cells inside one coarse cell. This keeps a city's drivers on a small number of shards and avoids cross-shard queries in 95% of matches.
- **Edge cases.** When a search radius spans the boundary between two coarse cells (rider near a city edge), the search has to query both shards. The matching service handles this by issuing 2 reads in parallel.

The other valid answer is S2 (Lyft used it historically). Geohash is fine for a v1; you would graduate to H3 once you hit scale. Quadtree on a real-time-updated dataset is a wrong answer in 2025.

</details>

## Step 4: sketch the high-level architecture

Fill in the `[ ? ]` placeholders. Hint: there is a long-lived connection from driver apps for location updates and dispatch, a fast in-memory index of where drivers are, a service that runs the matching algorithm, and a state machine that owns each ride from request to completion.

```
   Rider App                                            Driver App
       │                                                      │
       │ HTTPS                                                │ persistent WebSocket
       ▼                                                      ▼
  ┌─────────────┐                                      ┌──────────────┐
  │   API GW    │                                      │   [ ? ]      │  (long-lived connections,
  └──────┬──────┘                                      │              │   ingests location, pushes dispatch)
         │                                             └──────┬───────┘
         │                                                    │
         ▼                                                    ▼
  ┌─────────────┐                                      ┌──────────────┐
  │  Ride       │                                      │   [ ? ]      │  (fast in-memory store of
  │  Service    │◄─────────────────────────────────────┤              │   "where is each driver right now")
  │  (state     │       queries: nearby drivers        └──────────────┘
  │   machine)  │
  └──────┬──────┘
         │
         │ when match needed
         ▼
  ┌─────────────┐
  │   [ ? ]     │   (runs the matching algorithm: nearby + filters + scoring)
  └──────┬──────┘
         │
         │ chosen driver
         ▼
  push dispatch via the driver gateway

  Persistent records:
  ┌──────────────────┐    ┌──────────────────┐
  │   Trips DB       │    │   [ ? ]          │  (durable per-trip path history,
  │  (ride records)  │    │                  │   for fraud, disputes, analytics)
  └──────────────────┘    └──────────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
   Rider App                                            Driver App
       │                                                      │
       │ HTTPS                                                │ persistent WebSocket
       ▼                                                      ▼
  ┌─────────────┐                                      ┌──────────────────┐
  │   API GW    │                                      │  Driver Gateway   │  Holds the WebSocket per
  │  (rider)    │                                      │  (stateful pods,  │  driver. Receives location
  └──────┬──────┘                                      │   sticky routing) │  packets, forwards dispatch
         │                                             └─────┬─────────┬──┘  offers back to the driver.
         │                                                   │         │
         │                                  location updates │         │ dispatch push
         │                                                   │         │
         ▼                                                   ▼         │
  ┌─────────────┐                                  ┌──────────────────┐│
  │  Ride       │                                  │ Driver Location  ││  Redis (or KeyDB) cluster
  │  Service    │◄────── nearby query ─────────────┤ Index (H3-keyed) ││  sharded by H3 coarse cell.
  │  (state     │                                  │ + Status flags   ││  Keys per H3 cell hold the
  │   machine)  │                                  └──────────────────┘│  set of drivers in that cell.
  └──────┬──────┘                                                      │
         │                                                              │
         │ when match needed                                            │
         ▼                                                              │
  ┌─────────────────┐                                                   │
  │  Matching       │   Pulls candidate drivers from the index,         │
  │  Service        │   applies filters (vehicle class, rating,         │
  │  (stateless)    │   pending status), scores, picks one.             │
  └────────┬────────┘                                                   │
           │                                                            │
           │ chosen driver_id ─────────────────────────────────────────►│
           │                                                            │
           ▼                                                       (sent to driver app)
  ┌─────────────────┐
  │   Trips DB       │   Postgres or DynamoDB. The canonical record
  │  (ride records)  │   of each ride. Sharded by city.
  └─────────────────┘

           ┌──────────────────────────────────────────────────────────────┐
           │                Trip Path History (durable)                    │
           │  Kafka → object store (S3) + ClickHouse for analytics.        │
           │  Per-trip pings sampled at ~1 per 4 sec while on-trip.        │
           └──────────────────────────────────────────────────────────────┘

           ┌──────────────────────────────────────────────────────────────┐
           │   Routing / ETA Service                                       │
           │  Separate. OSRM, Mapbox, or in-house. Asked for "ETA from     │
           │  point A to point B given current traffic." Used at quote     │
           │  time and during pickup tracking.                             │
           └──────────────────────────────────────────────────────────────┘
```

Component responsibilities:

- **Driver Gateway.** Stateful. Holds a WebSocket per online driver. One pod handles ~10k-50k connections. Sticky routing by driver_id so reconnects land on the same pod when possible (best-effort, not required for correctness).
- **Driver Location Index.** Redis (or KeyDB) cluster sharded by H3 coarse cell. The key is the H3 cell ID; the value is a set of driver_ids in that cell, plus side keys for per-driver status (`available`, `on_trip`, `offline`) and last-known position. Overwrite-in-place; we do not durably persist every ping here.
- **Ride Service.** Owns the ride state machine. Knows nothing about driver locations; calls the Matching Service when it needs a driver.
- **Matching Service.** Stateless. Given a pickup location, it queries the Driver Location Index for candidates, applies filters, scores them, and returns the chosen driver. Sub-second.
- **Trips DB.** Source of truth for ride records. Sharded by city so a region outage does not affect other cities.
- **Trip Path History.** Async pipeline. Driver Gateway forwards on-trip location pings to Kafka, which writes to object store and a columnar store for queries.
- **Routing / ETA Service.** Separate service for "give me the route and ETA from A to B." Heavy CPU; isolated so its load does not affect matching.

</details>

## Step 5: the matching algorithm

A rider taps "Request Ride" at location P. You need to assign one driver in under 2 seconds. What is the algorithm? Closest driver wins? Highest-rated within a radius? Bin-pack many pending requests against many available drivers using the Hungarian algorithm? Trade-offs?

<details>
<summary><b>Reveal: matching algorithm</b></summary>

Three approaches, increasing in sophistication:

**a. Greedy nearest.**

```
candidates = location_index.drivers_in_cells(kRing(cell_of(P), 2))
candidates = filter(c -> c.status == AVAILABLE and c.vehicle_class >= requested)
candidates = candidates.sort_by(eta_to(P))
return candidates[0]
```

Fast (sub-100ms). Easy to reason about. The standard answer at most companies most of the time.

Problem: locally optimal, globally bad. If two riders request simultaneously and the nearest driver happens to be near both, the second rider gets a much worse match than they would have under a paired assignment.

**b. Greedy with a brief hold.**

Same as greedy, but the matching service holds the request for ~500ms to collect any other pending requests in the same H3 area. Then it runs a small bipartite matching (Hungarian algorithm) on the batch: minimize total ETA across all pending pairs.

This is what Uber actually does in dense areas during peak. The 500ms hold is invisible to the rider (still under the 2-second target) and improves global ETA by 5-15%.

**c. Predictive matching.**

Same as (b) but the candidate pool includes drivers who are about to drop off their current rider in the next 60 seconds. The ETA for those drivers includes the remaining trip plus the deadhead. This is more efficient but adds a prediction layer (the rider's current trip ETA is itself a model).

**Recommendation: start with (a), evolve to (b).**

Filters you must apply before scoring:
- `status == AVAILABLE` (not on trip, not offline, not in "going home" mode).
- `vehicle_class` matches what the rider requested (UberX, XL, Black, etc.).
- Driver has acceptance rate above threshold (low-quality drivers deranked).
- Driver is not on the rider's block list (rare but exists).
- Driver did not just cancel a request from this rider in the last 5 minutes.

Scoring (when you have a candidate set of 5-20 drivers):
- `eta` (call Routing Service for actual road-network ETA, not straight-line distance).
- `driver_rating` (small bonus for higher rated drivers).
- `pickup_difficulty` (some streets are one-ways, hard to pull over; deprioritize matches that route the driver through known-bad pickup areas).
- `match_age` (penalize drivers who have been idle for >30 min; we want to send them work, but not at the expense of a much better match).

A common mistake: using straight-line ("haversine") distance instead of road-network ETA. A driver 200m away on the other side of a river is much farther than a driver 500m away on the same street. Always use the routing service for the final score.

**The Hungarian algorithm in 30 seconds.** Given N pending requests and N candidate drivers, you have a cost matrix `C[i][j]` = ETA from driver j to rider i. The Hungarian algorithm finds the assignment that minimizes total cost in O(N^3) time. For N up to ~50 this is sub-millisecond. Above that you bucket by H3 cell and run several smaller assignments in parallel.

</details>

## Step 6: location updates at scale

Drivers report position every 4 seconds when online, every 1 second when on a trip. With 1M online drivers, that is 250k+ packets per second arriving at your servers. They must be cheap to ingest and cheap to query. How is each update stored and routed?

<details>
<summary><b>Reveal: location update flow</b></summary>

Two parallel writes per update, each going to a different store:

**Write 1: Driver Location Index (hot, overwrite-in-place).**

This is the index that matching queries.

- The Driver Gateway receives the packet over WebSocket.
- It computes the H3 cell at resolution 9 from `(lat, lng)`.
- It updates two Redis keys:
  - `driver:{driver_id}` → hash containing `(lat, lng, h3_cell, status, vehicle_class, last_update_ts)`. Overwrite.
  - If the H3 cell changed (driver moved between cells, happens every few updates): remove from `cell:{old_h3}` set and add to `cell:{new_h3}` set. If cell did not change: no-op.

Two writes per packet, both in-memory. ~0.5ms per update. 250k updates/sec = 500k Redis ops/sec, comfortably within one Redis cluster of ~20 shards.

The `cell:{h3}` sets are how the matching service queries efficiently. "Find all drivers within 1 ring of cell X" is `SUNION cell:X cell:X_n1 cell:X_n2 ... cell:X_n6`, returning at most a few hundred driver_ids. Filter and score those.

**Write 2: Trip Path History (durable, only when on-trip).**

If the driver is on a trip:
- The Driver Gateway pushes the location to a Kafka topic `trip.location.pings`, keyed by trip_id.
- A consumer aggregates per trip and writes batches to object storage (S3) plus an analytical store (ClickHouse).
- Idle-online drivers (not on a trip) do not get persisted; their pings only update the hot index.

This split matters. We are not paying durable storage for "where was every driver every 4 seconds today"; we only pay for "where was every active trip every 4 seconds today." That is roughly 10x cheaper.

**Failure modes to call out:**

- **Stale entries.** If a driver app crashes and stops reporting, their entry in `driver:{driver_id}` stays until a TTL expires. We set TTL to 30 seconds; if no update arrives, the driver is considered offline and removed from the cell sets by a background sweep. The TTL is short enough that matching does not assign a phantom driver.
- **Hot cells.** A single H3 cell over JFK airport at 5pm can contain hundreds of drivers. The set is still small enough to read in <1ms, but the cell key becomes a hot key on its Redis shard. Mitigations: read replicas of hot cells, in-process caching of the cell read in the Matching Service for 1 second windows.
- **Out-of-order updates.** Driver app on a flaky network may deliver pings out of order. The Driver Gateway timestamps each packet at receipt and discards anything older than the current stored `last_update_ts`. Cheap and avoids the indexed location going backwards.

</details>

## Step 7: read the full solution

You have done the heavy lifting. The solution covers the ride state machine in detail (where most subtle bugs hide), surge pricing, multi-region deployment, what happens when the driver app loses connectivity mid-trip, and the dozen edge cases that only surface after real launch. Try the follow-ups before reading.

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. **The matched driver does not accept within 15 seconds.** What does the system do? Re-match to the next-best driver? Notify the rider? What if a driver consistently ignores requests?
2. **A driver is in the middle of a trip when their app loses connectivity for 90 seconds.** How does the system know the trip is still going? What does the rider see? What if the driver never reconnects?
3. **Surge pricing.** A concert lets out and 5000 riders request at once in 10 city blocks. Demand is 10x supply. How does the price multiplier get computed and applied? Which service owns it?
4. **Two riders request at almost exactly the same time, and the same driver is the best match for both.** Walk through the race condition and how you prevent double-assignment.
5. **A rider's location is in a "hot" H3 cell with 200 candidate drivers (airport, downtown).** How do you avoid filtering through all 200 on every request?
6. **The H3 cell containing a popular pickup zone (airport curb) is on a single Redis shard. It becomes a hot key.** Diagnose and mitigate.
7. **Multi-city failure.** The us-east region (which serves NYC) goes down. What happens to in-progress trips there? Can riders in unaffected cities still request?
8. **A driver is briefly very close to the pickup but driving away (just dropped off a passenger and heading home).** Should the matcher assign them? How do you signal "going off duty soon"?
9. **Fraud: a driver app is reporting fake locations to game the system (teleporting to high-surge zones).** How do you detect this without slowing the ingest path?
10. **The Routing Service (ETA) is slow or down.** Matching depends on it. How do you fail gracefully?

## Related problems

- **[News Feed (002)](../002-news-feed/question.md)**, the location update fan-out and the per-cell hot-key problem rhyme with the celebrity-follower problem in news feed; the mitigations (replicas, in-process cache, jittered TTLs) are the same.
- **[Chat System (003)](../003-chat-system/question.md)**, in-ride messaging between rider and driver is a focused chat problem with the same persistent-connection and presence concerns; the driver gateway here is the same shape as the chat gateway there.
- **[Notification System (010)](../010-notification-system/question.md)**, dispatch push to the chosen driver, "your driver is arriving" push to the rider, and SMS fallback when the app is closed all flow through the notification system.
