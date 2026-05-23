## Solution: Design Uber / Lyft (Ride Sharing)

### TL;DR

Ride sharing is three problems wearing a trench coat. A real-time geospatial index that tracks ~1M moving drivers with 250k+ location updates per second. A matching pipeline that turns a rider's tap into a driver assignment in under 2 seconds. And a ride state machine that survives 15-25 minutes of driver cancellations, dropped packets, and backgrounded apps without billing anyone for a ride that never happened.

The design partitions cleanly. Location updates flow through a stateful driver gateway into a Redis cluster keyed by H3 hexagonal cells; that index is overwrite-in-place and never queried for history. The matching service is stateless and reads from the index, applying filters and a road-network ETA score from a separate routing service. Trip records live in a sharded Postgres (or DynamoDB) keyed by city, with a per-trip path history streamed to Kafka and durable storage for fraud and disputes.

The interesting engineering is in the seams. The driver gateway has to be stateful because it holds a WebSocket per driver, which makes deployment harder. The matching service can be greedy or batched; batched gives 5-15% better global ETA at the cost of a 500ms hold. The state machine has cancellation, no-show, and reconnect transitions that all have to be idempotent because a flaky network will deliver every event twice. Surge pricing is its own subsystem reading aggregated supply/demand per cell and emitting multipliers that the quote endpoint reads. Every hot key, from airport pickup cells to celebrity drivers near a stadium, needs the same hot-key mitigations that show up in every other large system.

### 1. Clarifying questions

Covered in `question.md`. The eight questions: scope (matching only, or also payments/surge/ETA?), scale (drivers online, riders requesting, hottest city), matching latency target, location update frequency, matching policy (greedy vs batch), cancellation handling, geographic shard model, vehicle types. The two most candidates miss: the hottest single-city number (which sizes a shard) and the matching-policy question (which forks the architecture).

### 2. Capacity estimates

Reiterated from the question:

- ~1160 trip requests/sec sustained, ~10k peak.
- ~250k idle-online location updates/sec + ~300k on-trip updates/sec = ~550k updates/sec sustained, ~1M/sec peak.
- ~1M concurrent active trips globally; biggest cities ~50-100k each.
- ~2 TB/day of durable trip-path history (compressed, aged off at 90 days).

The decisive observation: location ingest dwarfs every other workload. Keep it on its own infrastructure (the gateway and the hot index) so it does not bleed into matching latency or trip-state writes.

### 3. API design

Request a ride (rider side).

```
POST /api/v1/rides
Authorization: Bearer <rider_token>
Idempotency-Key: <uuid>

{
  "pickup":  { "lat": 40.7580, "lng": -73.9855, "address": "Times Sq" },
  "dropoff": { "lat": 40.7411, "lng": -74.0048, "address": "Chelsea" },
  "vehicle_class": "uberx",
  "payment_method_id": "pm_abc"
}
```

Responses:

| Status | Meaning | Body |
|--------|---------|------|
| 202 Accepted | Ride created in `requested` state; matching in progress | `{ "ride_id": "...", "state": "requested", "quote": { "fare_min":..., "fare_max":..., "surge": 1.2 }, "eta_seconds": 240 }` |
| 200 OK | Idempotency replay; same ride returned | same shape as 202 |
| 400 Bad Request | Invalid pickup, dropoff outside service area, etc. | |
| 402 Payment Required | Payment method invalid | |
| 409 Conflict | Rider already has an active ride | |
| 503 Service Unavailable | No drivers available in area after timeout | `{ "error": "no_drivers", "retry_after": 60 }` |

A few choices worth defending out loud:

202, not 201. The ride exists but is not yet matched. The client subscribes to a WebSocket channel for updates (`ride:{ride_id}`).

Idempotency-Key is required. Network failures cause clients to retry the submit. Without the key you bill people twice. The server stores `(idempotency_key, ride_id)` for 24h; a repeat returns the existing ride.

The quote is bracketed (`fare_min`/`fare_max`), not exact. Final fare depends on actual trip duration and any waiting time.

Driver accepts a ride.

```
POST /api/v1/rides/{ride_id}/accept
Authorization: Bearer <driver_token>

{ "driver_eta_seconds": 180 }
```

| Status | Meaning |
|--------|---------|
| 200 OK | Driver is now assigned to the ride |
| 409 Conflict | Ride already accepted by another driver, or in a state that does not allow accept |
| 410 Gone | Rider cancelled or ride expired (the dispatch offer timed out) |

Location update (driver side). Not REST; sent over the persistent WebSocket as a binary frame:

```
[1 byte msg_type=LOC]
[8 bytes driver_id]
[4 bytes lat (fixed-point)]
[4 bytes lng (fixed-point)]
[2 bytes heading]
[2 bytes speed]
[1 byte accuracy]
[8 bytes ts_ms]
[16 bytes hmac_sig]
```

~46 bytes on the wire vs ~200 bytes for the equivalent JSON. At 250k packets/sec the savings matter.

### 4. Data model

Drivers (mostly static profile, not the hot index).

```sql
CREATE TABLE drivers (
    driver_id       BIGINT PRIMARY KEY,
    name            TEXT NOT NULL,
    vehicle_class   SMALLINT NOT NULL,   -- 1=uberx, 2=uberxl, 3=black, ...
    vehicle_plate   TEXT NOT NULL,
    rating_avg      NUMERIC(3,2) NOT NULL DEFAULT 5.00,
    rating_count    INT NOT NULL DEFAULT 0,
    home_city_id    INT NOT NULL,
    status          SMALLINT NOT NULL,   -- 1=active, 2=suspended, 3=offboarded
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_city_status ON drivers (home_city_id, status);
```

Read-mostly. Not in the location update path; only ride creation and post-trip aggregation read it.

Rides (the trip record).

```sql
CREATE TABLE rides (
    ride_id         BIGINT PRIMARY KEY,    -- Snowflake-style, ts + shard + seq
    rider_id        BIGINT NOT NULL,
    driver_id       BIGINT,                 -- NULL until matched
    city_id         INT NOT NULL,           -- shard key
    state           SMALLINT NOT NULL,      -- see state machine below
    pickup_lat      DOUBLE PRECISION NOT NULL,
    pickup_lng      DOUBLE PRECISION NOT NULL,
    dropoff_lat    DOUBLE PRECISION NOT NULL,
    dropoff_lng    DOUBLE PRECISION NOT NULL,
    vehicle_class   SMALLINT NOT NULL,
    requested_at    TIMESTAMPTZ NOT NULL,
    matched_at      TIMESTAMPTZ,
    picked_up_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancel_reason   SMALLINT,
    fare_cents      INT,
    surge_mult      NUMERIC(3,2) NOT NULL DEFAULT 1.00,
    payment_method  TEXT,
    idempotency_key UUID
);
CREATE UNIQUE INDEX idx_idempotency ON rides (rider_id, idempotency_key);
CREATE INDEX idx_rider_active ON rides (rider_id) WHERE state IN (1,2,3,4);
CREATE INDEX idx_driver_active ON rides (driver_id) WHERE state IN (3,4);
```

Sharded by `city_id`. Rides never cross cities mid-flight; this gives clean shard boundaries and per-city blast radius.

The partial indexes are deliberate. `idx_rider_active` enforces "one active ride per rider" with a cheap check; `idx_driver_active` does the same for drivers. After the ride completes the row no longer appears in either partial index, keeping them small.

Driver Location Index (Redis, not SQL).

```
driver:{driver_id}     HASH    fields: lat, lng, h3_r9, h3_r6, status, vehicle_class, last_update_ts
                                 TTL: 30s (refreshed on every update)

cell:{h3_r9}           SET     members: driver_ids currently in that cell
                                 (no TTL; entries removed by background sweep when driver TTL expires)

driver_inflight:{driver_id} STRING  value: ride_id of active ride
                                     (set when matched, cleared when ride completes)
```

Sharded by `h3_r6` (a coarser H3 resolution, ~6km across) so all drivers in a metropolitan neighborhood land on the same Redis shard. ~20 shards total at peak, ~5GB per shard.

### 5. Core algorithm: H3 indexing and the matching loop

#### Why H3 at resolution 9

Resolution 9 has cells with an edge length of ~174m, area ~0.1 sq km. In a dense city you get ~10-50 available drivers per cell at peak. A `kRing(cell, 1)` query covers the cell plus its 6 neighbors, a ~750m radius. That is the right scale for "find me drivers close enough to be at the rider within a few minutes."

If no match is found in the 1-ring (rare in dense areas, common in suburbs), expand to `kRing(cell, 2)` (~1.5km radius), then `kRing(cell, 3)` (~2.3km). Cap at ring 5; beyond that, ETA is too long to be acceptable and the system returns "no drivers available" to the rider.

#### The matching loop

```python
def match(ride):
    pickup_cell = h3.geo_to_h3(ride.pickup_lat, ride.pickup_lng, resolution=9)
    
    for k in [1, 2, 3, 5]:
        cells = h3.k_ring(pickup_cell, k)
        candidate_ids = redis.sunion(*[f"cell:{c}" for c in cells])
        
        # Coarse filter on cached driver attributes (no individual reads yet).
        candidates = location_index.bulk_lookup(candidate_ids)
        candidates = [
            d for d in candidates
            if d.status == AVAILABLE
            and d.vehicle_class >= ride.vehicle_class
            and d.driver_id not in ride.rider.blocked_drivers
        ]
        
        if len(candidates) == 0:
            continue
        
        # Score top N candidates with a real road-network ETA call.
        candidates = candidates[:20]   # cap to bound routing cost
        etas = routing_service.batch_eta(
            origins=[(c.lat, c.lng) for c in candidates],
            destination=(ride.pickup_lat, ride.pickup_lng)
        )
        
        scored = [
            (eta + rating_penalty(c.rating) + idle_bonus(c.last_idle_ts), c)
            for c, eta in zip(candidates, etas)
        ]
        scored.sort()
        
        best = scored[0][1]
        
        # Atomic claim. If two simultaneous matches pick the same driver, only one wins.
        if try_claim_driver(best.driver_id, ride.ride_id):
            return best.driver_id
        # else: someone else got them, loop to next-best
        for _, c in scored[1:]:
            if try_claim_driver(c.driver_id, ride.ride_id):
                return c.driver_id
    
    return None   # truly no drivers
```

`try_claim_driver` is a single Redis Lua script:

```lua
-- KEYS[1] = driver_inflight:{driver_id}, ARGV[1] = ride_id
if redis.call('SET', KEYS[1], ARGV[1], 'NX', 'EX', 30) then
    return 1
else
    return 0
end
```

`SET... NX EX 30` means "set only if not exists, expire in 30 seconds." If the driver is already claimed by another ride, this returns 0 and we move on. The 30-second expiry is a safety net: if the matching service crashes before completing the dispatch, the lock auto-releases and the driver is matchable again.

#### Matching flow

```
   Rider App        Ride Svc        Matching        Redis           Routing         Driver GW
       │              │               │              │               │                 │
       │ POST /rides  │               │              │               │                 │
       ├─────────────►│               │              │               │                 │
       │              │ idempotency,  │              │               │                 │
       │              │ active check, │              │               │                 │
       │              │ insert row    │              │               │                 │
       │              │ state=1       │              │               │                 │
       │ 202 + quote  │               │              │               │                 │
       │◄─────────────┤               │              │               │                 │
       │              │ dispatch.req  │              │               │                 │
       │              ├──────────────►│              │               │                 │
       │              │               │ SUNION       │               │                 │
       │              │               │ cell:X +     │               │                 │
       │              │               │ kRing(1)     │               │                 │
       │              │               ├─────────────►│               │                 │
       │              │               │◄─────────────┤               │                 │
       │              │               │ bulk_lookup, filter, cap 20  │                 │
       │              │               │              │               │                 │
       │              │               │ batch_eta(origins, dest)     │                 │
       │              │               ├──────────────────────────────►│                 │
       │              │               │◄──────────────────────────────┤                 │
       │              │               │ score, pick best             │                 │
       │              │               │              │               │                 │
       │              │               │ SET inflight NX EX 30        │                 │
       │              │               ├─────────────►│               │                 │
       │              │               │◄─────────────┤               │                 │
       │              │               │ push dispatch offer          │                 │
       │              │               ├──────────────────────────────────────────────►│
       │              │               │              │               │  send to driver │
       │              │               │              │               │                 │
       │              │               │   driver accept POST         │                 │
       │              │◄──────────────────────────────────────────────────────────────┤
       │              │ UPDATE state=1→2 WHERE state=1               │                 │
       │              │ notify rider via WebSocket                   │                 │
       │ "driver on   │               │              │               │                 │
       │  the way"    │               │              │               │                 │
       │◄─────────────┤               │              │               │                 │
```

#### Batch matching (advanced)

If the city is under heavy load, instead of matching one ride at a time, the matching service holds incoming requests for up to 500ms (the "dispatch window") and runs a Hungarian assignment over the pending batch:

```python
def batch_match(rides, drivers):
    # rides: list of pending requests in the same metropolitan area
    # drivers: candidate available drivers
    cost = [[eta(d, r.pickup) for r in rides] for d in drivers]
    # Hungarian / min-cost assignment, padded with dummy rows/cols if sizes differ
    assignment = hungarian(cost)
    return assignment
```

For up to ~50 rides and ~100 drivers, this runs in <50ms. Above that, partition by H3 sub-region and run several smaller assignments in parallel.

The 500ms hold is worth it when ETA improvements average 5-15%; it is not worth it when the city is sparse and you would be holding the request for nothing. The dispatch window is set per city by a control loop watching system-load metrics.

### 6. Architecture (detailed)

```
                                              Driver App
                                                   │
                                                   │  persistent WebSocket
                                                   │  (TLS, mTLS for driver auth)
                                                   ▼
   Rider App                              ┌──────────────────────┐
       │                                  │   Driver Gateway     │  Stateful pods. One driver
       │                                  │   (sticky routing)   │  one connection. Per pod:
       │ HTTPS                            │                      │  10-50k concurrent connections.
       ▼                                  │   - decode LOC frames│
  ┌─────────────┐                         │   - update H3 index  │
  │  API GW     │                         │   - push dispatch    │
  │  (rider)    │                         └────┬──────────┬──────┘
  └──────┬──────┘                              │          │
         │                                     │          │ on-trip pings
         │                                     ▼          ▼
         │                       ┌──────────────────┐  ┌─────────────────┐
         │                       │ Driver Location  │  │ trip.location.  │
         │                       │ Index            │  │ pings (Kafka)    │
         │                       │ Redis cluster,   │  └────────┬─────────┘
         │                       │ sharded by h3_r6 │           │
         │                       └────────┬─────────┘           │
         │                                ▲                     ▼
         │                                │              ┌──────────────┐
         ▼                                │              │ Trip History │
  ┌─────────────────┐                     │              │ Pipeline      │
  │  Ride Service   │                     │              │ (Kafka → S3   │
  │  (state machine,│                     │              │  + ClickHouse)│
  │   stateless)    │                     │              └──────────────┘
  └──┬─────────┬────┘                     │
     │         │                          │
     │         │ "find me a driver"       │
     │         ▼                          │
     │   ┌─────────────────┐              │
     │   │  Matching       │──────────────┘
     │   │  Service        │  reads candidates
     │   │  (stateless)    │
     │   └─────┬───────────┘
     │         │
     │         │ calls for ETA scoring
     │         ▼
     │   ┌─────────────────┐
     │   │  Routing /      │  OSRM or in-house. Stateless.
     │   │  ETA Service    │  Heavy CPU.
     │   └─────────────────┘
     │
     ▼
  ┌─────────────────┐
  │   Trips DB      │  Postgres or DynamoDB. Sharded by city_id.
  │                 │  Read replicas in each region.
  └─────────────────┘

  ┌─────────────────┐    ┌─────────────────┐
  │  Surge Service  │    │  Notification   │  Push to driver (dispatch),
  │  (per-cell      │    │  Service        │  push to rider (driver
  │   supply/demand)│    │                 │  arriving, in progress).
  └─────────────────┘    └─────────────────┘
```

A few things to notice while reading this:

The Driver Gateway is stateful. A WebSocket is a connection; you cannot route each frame to a random pod. Sticky routing by driver_id (consistent hash) plus a session table in Redis (which pod owns which driver) lets the Matching Service push dispatch by looking up "which pod has driver_id D?" and sending it there.

The Matching Service is stateless. It owns no connections. It reads from Redis, calls Routing, writes a claim to Redis, then asks the Driver Gateway to push the offer. Easy to scale horizontally; failures just retry.

The Routing Service is separate. Road-network ETA computation is CPU-heavy and prone to spikes. Isolating it means a routing slowdown does not crash matching; the matcher times out at, say, 300ms and falls back to haversine distance.

The Trips DB is sharded by city. Cities are natural blast-radius boundaries. NYC outage does not affect London.

The Surge Service is separate. It aggregates supply (drivers available) and demand (requests in last N seconds) per H3 cell and publishes multipliers. The quote endpoint reads from this; the matching path does not.

### 7. The match path (rider request → driver assigned)

End-to-end, what happens when a rider taps "Request Ride":

1. **t=0ms.** Client → API Gateway → Ride Service.
2. **t=20ms.** Ride Service does:
 - Idempotency check against `(rider_id, idempotency_key)`.
 - "Does rider have an active ride?" check via the partial index.
 - Look up surge multiplier for the pickup H3 cell.
 - Insert ride row with `state=1 (requested)`. Returns 202 to client with a quote.
3. **t=30ms.** Ride Service publishes `ride.requested` event to Kafka (also forwards to Matching Service directly for low latency).
4. **t=30ms.** Matching Service runs the matching loop described above.
 - **t=60ms.** SUNION on cell sets, ~5-50 candidate IDs.
 - **t=70ms.** Bulk lookup driver records from Redis, apply filters. ~10-20 remain.
 - **t=100ms.** Batch ETA call to Routing Service.
 - **t=200ms.** Score, pick best, claim via `SET NX`.
5. **t=200ms.** Matching Service publishes `dispatch.offer` event to the Driver Gateway pod owning the chosen driver.
6. **t=210ms.** Driver Gateway pushes the offer over WebSocket to the driver app.
7. **t=220ms.** Driver app shows the "new ride request" screen. Driver has up to 15 seconds to accept.
8. **Driver accepts (t=~3000ms typical).**
 - Driver app sends `POST /rides/{ride_id}/accept`.
 - Ride Service transitions state `requested → matched`, sets `driver_id`, `matched_at`.
 - Ride Service notifies the rider via their WebSocket: "Your driver is on the way."
9. **Driver does not accept (t=15000ms).**
 - Matching Service times out, releases the claim (`DEL driver_inflight:{driver_id}`), increments the driver's "ignored" counter, and re-runs matching to find the next driver. The rider sees no change in their UI other than a slightly longer wait.
 - After 3 ignores in a row, the driver is auto-set to `offline` and surfaced for a check-in (their app likely is in the background).

End-to-end latency from rider tap to driver assignment, P99 target: under 2 seconds when a driver accepts immediately. The 2-second budget breaks down to ~200ms for matching, plus the dispatch RTT, plus the driver's reaction time.

### 8. The ride state machine

This is where bugs accumulate if you handwave. The states:

```
   ┌─────────────┐
   │  requested  │  matching in progress; no driver assigned
   └──────┬──────┘
          │ driver accepts
          ▼
   ┌──────────────┐
   │   matched    │  driver_id set; driver en route to pickup
   └──────┬───────┘
          │ driver arrives + rider boards
          ▼
   ┌────────────────────┐
   │  in_progress       │  trip underway; on-trip pings being captured
   └──────┬─────────────┘
          │ driver taps "End Trip" at dropoff
          ▼
   ┌────────────────────┐
   │   completed        │  terminal. Fare charged. Receipt issued.
   └────────────────────┘

   From any non-terminal state:
   ┌────────────────────┐
   │   cancelled        │  terminal. Reason recorded. Cancel fee
   └────────────────────┘  may apply per business rules.

   Special: from `matched` only:
   ┌────────────────────┐
   │   no_show          │  terminal. Driver waited at pickup; rider didn't show.
   └────────────────────┘  Counts as a no-show cancel for billing.
```

Critical invariants:

Every transition is idempotent. Network retries are common. `POST /accept` called twice should produce the same state and not change `matched_at`.

No backward transitions. Once `in_progress`, you cannot go back to `matched`. The only exit is `completed` or `cancelled`.

One driver per ride; one ride per driver. Enforced by the partial unique indexes on `(driver_id) WHERE state IN (3,4)` and by the Redis `driver_inflight:{driver_id}` key.

Cancel reason is required. "rider cancelled," "driver cancelled," "system cancelled" (no drivers available), "no show," "fraud detected." Determines billing.

Implementation: model as a database row with a `state` column plus an audit trail in a separate `ride_events` table. Transitions are conditional updates:

```sql
UPDATE rides
SET state = 2, driver_id = $driver, matched_at = NOW()
WHERE ride_id = $ride AND state = 1
RETURNING state;
```

If `RETURNING` yields 0 rows, the transition failed (already matched, or cancelled). The driver gateway treats this as "you lost the race" and discards the offer.

### 9. Location update path

Already covered in detail in `question.md` Step 6. The key points:

- Two writes per update: the hot Redis index (overwrite-in-place) and, only for on-trip drivers, the durable Kafka stream for path history.
- Cell membership updates lazily. If a driver moves but stays in the same H3 r9 cell, only the `driver:{driver_id}` hash updates. The expensive set membership change happens only when the cell changes.
- TTL on driver keys. 30 seconds. If a driver app goes silent, the driver is auto-removed from candidate pools.
- On-trip frequency. 1Hz instead of 0.25Hz so the rider's "where is my driver?" view updates smoothly. The bandwidth cost is bounded because only ~30% of online drivers are on a trip.
- Out-of-order rejection. The Driver Gateway holds the last accepted `ts_ms` per driver and discards older packets. Cheap and avoids the indexed location going backwards on a flaky network.

One detail not yet covered: batching from the driver app. The driver app actually buffers location samples and sends them in bursts every 4 seconds, not as a continuous stream. This reduces TCP overhead and saves battery. The server cares only about the latest sample for the index; older ones in the batch go straight to the on-trip path history if applicable.

### 10. Scaling

#### a. Driver Gateway

- Stateful pods, each holding 10-50k WebSocket connections.
- 1M concurrent drivers → ~30-50 pods at peak.
- Sticky routing via a session table: `gateway_session:{driver_id} → pod_id`, refreshed on connect/reconnect. The Matching Service looks up the pod by driver_id when it needs to push dispatch.
- Pod failure: connections drop, drivers reconnect (apps retry exponentially). Capacity is over-provisioned 30% to absorb reconnects without a thundering herd.

#### b. Driver Location Index (Redis cluster)

- Sharded by H3 r6 cell. ~20 shards.
- Hot shards (containing NYC, SF, Mumbai) get more memory and CPU than cold ones (Wyoming).
- Read replicas of each shard: 2 replicas per primary in the same region. Matching service round-robins reads across replicas.
- Sustained ops: 500k writes/sec (location updates × 2 writes each) + ~50k reads/sec (matching queries). Within capacity of a 20-shard cluster.

#### c. Hot cells

The airport cell. The stadium cell during a game. These cells have hundreds of drivers and are queried many times per second. Mitigations, in order:

1. In-process cache on Matching Service. Cache the result of `cell:{h3}` reads for 1 second. Within a second, the membership barely changes. This cuts Redis read load on hot cells by 10-100x.
2. Read replicas. As mentioned.
3. Sub-shard the hot cell. If even with replicas the hot cell is the bottleneck, split it into `cell:{h3}:bucket0`, `cell:{h3}:bucket1`, etc., keyed by `hash(driver_id) mod N`. The Matching Service unions all buckets. Adds a small read cost but distributes writes.

#### d. Matching Service

- Stateless. Auto-scale on request rate.
- One instance per city is a reasonable starting point; bigger cities get more instances behind a city-level load balancer.
- Throughput per instance: ~500 matches/sec (each match takes ~200ms wall clock but most is waiting on Redis and Routing; concurrency is high).
- Total: ~50-100 instances globally at peak.

#### e. Trips DB

- Sharded by `city_id`. Each city is one shard logically; large cities have multiple shards by `hash(ride_id)` within the city.
- Read replicas in each region; writes go to the primary in the city's home region.
- Storage: 100M trips/day × 500B/row = ~50GB/day. Across cities, manageable per-shard.

#### f. Trip Path History

- Kafka cluster with topic `trip.location.pings`, partitioned by `trip_id`.
- Consumer writes to S3 in 5-minute batches, partitioned by `(date, city_id)`.
- A second consumer aggregates into ClickHouse for analytics.
- Retention: 90 days hot in ClickHouse for fraud and disputes; long-term in S3 indefinitely (regulatory).

### 11. Reliability

Driver Gateway pod loss. All drivers on that pod disconnect. Apps retry with exponential backoff plus jitter (start at 1s, max at 30s). New pod absorbs reconnects. During the ~10s window, those drivers are not matchable; their TTL expires within 30s and they fall out of the cell sets cleanly. In-progress rides on those drivers are not affected because the ride state lives in the Trips DB, not the gateway.

Driver app loses connectivity mid-trip. WebSocket disconnects. The trip stays `in_progress` because no event has transitioned it. The rider's app shows "driver location stale" after 30 seconds. If the driver reconnects within 5 minutes, on-trip pings resume (the gap is interpolated for ETA display; the actual fare is computed from the dropoff event, not the pings). If the driver does not reconnect within 15 minutes, an alert is raised; ops can manually mark the trip `cancelled` with reason `driver_disconnect`.

Redis cluster loss. Catastrophic for matching. The cluster is replicated within-region; failover takes ~30 seconds during which matching is degraded (rides go into `requested` state and wait). On full cluster loss, location updates buffer in the Driver Gateway memory for up to 60 seconds; if the cluster does not recover, drivers are temporarily marked unmatchable and riders see "no drivers available."

Routing Service slow or down. Matching service times out at 300ms and falls back to haversine distance (straight-line) for scoring. Quality drops (drivers across rivers may be ranked too high) but the system still functions. A health check on the routing service flips a circuit breaker; while open, all matches use haversine.

Trips DB primary failure. Failover to replica (~30 sec). During the window, new ride requests return 503 in that city. In-progress rides in that city cannot transition state (no acceptance, no completion). Other cities are unaffected.

Whole region failure. Drivers and riders in that region's cities are stranded. Cross-region failover is not feasible because location state is in-region; a NYC driver is not useful for matching a Tokyo ride. The mitigation is region redundancy within a continent (us-east + us-west for North America) and DNS-based failover for new traffic; in-progress trips in the failed region are lost (cancelled with reason `system_outage`) and refunded.

### 12. Observability

| Metric | Why |
|--------|-----|
| `match.latency.p99` by city | Headline SLO; >2s for 5 min = page |
| `match.success_rate` by city | % of requests that get a driver assigned within timeout |
| `match.no_drivers_rate` | If >5% in a city, supply problem (raise driver incentives) |
| `dispatch.accept_rate` | % of dispatch offers accepted; <40% = drivers gaming the system or bad matching |
| `dispatch.accept_latency.p50` | How long drivers take to accept; sharp rise = app problem |
| `location.ingest_rate` by region | Should be roughly drivers_online * (1/4 + on_trip_fraction * 0.75) |
| `location.stale_driver_count` | Drivers whose last update is >30s old; spikes suggest gateway problem |
| `redis.hot_cell.ops` for known hot cells (airport, stations) | Should not climb above shard capacity |
| `routing.latency.p99` | Affects match latency directly |
| `state_machine.illegal_transitions` | Should be near zero; nonzero = client bug or replay attack |
| `trip.in_progress_no_pings.count` | Trips with no location update in 60s; possible network or fraud |
| `surge.multiplier.max` by city | Sudden 5x surge in a sleepy area = sensor problem or real event |

Page on: match P99 >2s for 5 min in any top-20 city; match success rate <90% for 5 min; gateway disconnect rate >5%/min; trip state DB unavailable. File a ticket on: dispatch accept rate <40%; hot cell ops >80% of shard capacity; illegal transitions >0.

### 13. Follow-up answers

**1. The matched driver does not accept within 15 seconds.**

The Matching Service holds a soft claim on the driver via `driver_inflight:{driver_id}` with a 30-second TTL. When the dispatch times out at 15s:

- Driver Gateway notifies Matching Service.
- Matching Service explicitly releases the claim (`DEL driver_inflight:{driver_id}`) and increments a per-driver `ignored_count` counter in Redis.
- Matching Service re-runs the matching loop, excluding this driver from the candidate set for the next 5 minutes (so they do not get reoffered the same ride).
- The rider's UI updates "Still finding your driver..." but state stays `requested`. No fee.

If a driver ignores 3 dispatches in a row, their status is auto-flipped to `offline`. The driver app receives a notification ("You seem inactive. Are you still online?"). This prevents drivers who left their phone unattended from holding queue slots indefinitely.

**2. Driver app loses connectivity mid-trip for 90 seconds.**

The ride is in `in_progress`. The state machine does not care about the connection; only about state-change events (`pickup`, `complete`, `cancel`).

What happens:
- Driver Gateway sees the WebSocket dropped. Marks the driver `connection_lost` in the session table but does not change ride state.
- Location pings stop arriving. The rider's app shows "GPS lost" overlay after 30 seconds (UX choice).
- The driver's app queues location samples locally up to ~10 minutes worth.
- When the driver reconnects (90 seconds later), the gateway accepts the queued samples. They are timestamped, so out-of-order arrivals are handled. The on-trip path history gets backfilled.
- If the driver never reconnects: after 15 min with no pings during an `in_progress` ride, ops gets an alert. They review and either mark the trip `completed` with a best-guess fare based on the partial path, or `cancelled` with refund.

The fare is computed from the dropoff event submitted by the driver, not from the location pings. Lost connectivity does not invalidate the trip as long as the driver eventually submits the dropoff.

**3. Surge pricing.**

Surge is a multiplier on the base fare quote, computed per H3 cell from supply and demand:

```
multiplier = clip(demand_rate / supply_count, min=1.0, max=5.0)
```

Where `demand_rate` is ride requests in the cell in the last 60 seconds, and `supply_count` is available drivers in the cell.

Architecture:
- Surge Service is a separate service. It consumes the `ride.requested` event stream and joins against the Driver Location Index (snapshot every 10 seconds).
- Outputs per-cell multipliers to a key `surge:{h3_r8}` in Redis with a 30-second TTL.
- The Ride Service reads `surge:{cell_of(pickup)}` when generating a quote. If missing or stale, defaults to 1.0.

Why H3 r8 (coarser) for surge cells: surge is regional, not block-level. A 5x multiplier on one street block but 1x on the next would look bizarre to riders. H3 r8 is ~3km across, which feels like "this neighborhood is busy."

Why the 1.0 minimum: a "0.7x discount" would be marketed differently and lives in promotions, not surge.

Surge is contentious; in production it has anti-gaming logic (drivers cannot teleport to high-surge cells to claim the bonus on rides started elsewhere), smoothing (the multiplier cannot jump from 1.0 to 5.0 in one tick; it ramps), and rider notifications ("Fares are higher right now due to high demand. Pay 2.1x?").

**4. Two riders request at almost the same time, same best-match driver.**

The race:
- Ride A and Ride B both start matching at t=0 and t=10ms.
- Both pull the same candidate driver D as best.
- Both attempt `SET driver_inflight:D NX`.

Only one wins. Say A wins at t=200ms. B's `SET NX` returns 0 at t=210ms.

B's Matching Service detects the conflict and moves to the next-best candidate. If B has more than one good option, the rider barely notices (maybe a 100ms longer match). If B has only one viable driver (sparse area), B waits 1-2 more seconds and tries again.

The lock is held in Redis with a 30-second TTL. If A's matching service crashes before sending the dispatch, the lock expires and D becomes claimable.

The "lost-race" case can also happen at the database level if two matching services try to write `driver_id` to two different rides for the same driver. The partial unique index `idx_driver_active` prevents this:

```sql
UPDATE rides SET state=2, driver_id=$D, matched_at=NOW()
WHERE ride_id=$R AND state=1;
```

If another row already has `driver_id=$D` with state in (3,4), this update would violate the partial unique index and error. The Matching Service treats this as "lost-race," releases its Redis claim, and retries with the next driver.

Belt-and-suspenders: Redis claim for fast-path mutual exclusion, DB partial unique index for ultimate correctness.

**5. Hot cell with 200 candidate drivers.**

Two costs to bound:
- The SUNION across cells returns up to 200 candidate IDs.
- For each, we do a bulk-lookup of driver attributes from Redis.
- For the top 20 by some cheap proxy (e.g., last-known position cached on the matching service), we call the Routing Service for true ETA.

Bounding strategies:
- Cheap pre-filter first. Before fetching driver attributes, sample the candidate set: take the 50 nearest by haversine distance (computed from cached positions in the matching service). Only fetch attributes for those 50.
- Score-and-cap. Apply filters (vehicle class, status, rating) and cap to 20 before the Routing Service call.
- Cache cell membership. As mentioned, in-process cache with 1-second TTL. A cell read may be served hundreds of times from memory in a busy second.

Result: the hot cell adds a few ms of in-process work, not a Redis round-trip per candidate.

**6. Hot key on a popular pickup zone (airport curb).**

Symptoms: one Redis shard hits CPU 100%, `cell:{airport_h3}` reads dominate, every other key on that shard sees degraded P99.

Mitigations in order:
1. In-process cache on the Matching Service (1-second TTL). Cuts reads on hot cells by 10-100x.
2. Read replicas of the shard. Matching service round-robins reads across replicas.
3. Sub-shard the hot cell. Replace one `cell:{h3}` set with `cell:{h3}:bucket{0..15}` by `hash(driver_id) mod 16`. Reads become 16-way SUNION (still cheap), writes are distributed.
4. Move the hot cell to a dedicated shard. Tag certain cells (airports, train stations) at provisioning and pin them to a high-resource shard with extra replicas.

These layer. Most installs need just (1) and (2); (3) and (4) come up when an event happens that the system was not provisioned for.

**7. Region failure (us-east goes down).**

In-progress trips in us-east cities are lost. The state machine for those rides cannot transition because the Trips DB shards live in us-east. Mitigation:
- Trips DB is replicated across AZs within us-east, so AZ failures are absorbed.
- A full us-east failure (rare but happens) triggers cross-region failover to a hot standby in us-east-2 if available. Replication lag of ~5 seconds means up to 5 seconds of rides may be lost.
- Riders in unaffected cities (Europe, Asia, us-west) are fully functional; cities are independent.
- Riders in us-east see the app refuse to start new rides during the outage; in-progress rides show "service interrupted" and are auto-cancelled with full refund.

Multi-region for matching is hard because location state is in-region and not useful elsewhere. Multi-region for trip records is feasible at higher cost (active-active per city is overkill; active-passive with replication is common).

**8. Driver near pickup but driving away (off duty or just dropped off).**

The driver's `status` field in the Driver Location Index distinguishes:
- `AVAILABLE`: eligible to receive dispatches.
- `ON_TRIP`: currently driving a rider.
- `GOING_OFFLINE`: driver tapped "Sign off after this ride."
- `OFFLINE`: not eligible.

A driver who just dropped off a rider transitions to `AVAILABLE` automatically. If they tap "End shift," they go to `GOING_OFFLINE` for the duration of any current ride and then `OFFLINE`. A `GOING_OFFLINE` driver is not in the candidate pool.

What about a driver who is `AVAILABLE` but heading in the wrong direction? That is a real problem. Two approaches:
- Heading-aware scoring. Penalize candidates whose heading vector points away from the pickup. Easy with the `heading` field already in each ping.
- Predictive ETA. Use the routing service to compute "if this driver continues their current heading and speed, what is the route and ETA to pickup?" The driver's current motion is baked into the ETA.

In practice, the routing service already handles this if it is given the driver's current location and asked for ETA to the pickup; the road network will reflect any one-way streets and U-turns. The simpler haversine fallback misses this, which is one more reason to prefer real routing.

**9. Fraud: driver app reporting fake locations.**

Fake locations get submitted to game surge zones, "deadhead" bonus, or to harvest free rides. Detection without slowing ingest:

- Plausibility checks at the gateway. If a driver's reported position jumps >1km between two pings 4 seconds apart, flag it. Real drivers do not teleport. This check is in-process, sub-ms.
- Cross-check against the trip path. For on-trip drivers, the GPS path should look like a road. A path that cuts across buildings is suspect. Run a nightly batch job against the Trip History store.
- Device attestation. Use Play Integrity / DeviceCheck to verify the location was reported from a real device, not a mock-location app. Bake this into the WebSocket handshake; suspicious devices get a flag that downstream services react to (lower match priority, more aggressive fraud review).
- Surge-zone teleportation. A driver who appears in a high-surge zone but had no recent path leading there gets flagged. Pings going through reasonable road segments are weighted positively in the fraud model.

None of these sit in the synchronous match path. Fraud is mostly an async detection problem. The synchronous protection is just the plausibility check.

**10. Routing Service slow or down.**

Match path falls back to haversine distance (straight-line) for scoring.

```python
try:
    etas = routing_service.batch_eta(origins, dest, timeout=300ms)
except (TimeoutError, ServiceUnavailable):
    etas = [haversine(o, dest) / AVG_SPEED for o in origins]
    metrics.inc("matching.routing_fallback")
```

Quality drops because haversine ignores the road network. A driver across a river may score better than a driver one street away. But the system still functions and most matches are still acceptable (in dense urban grids, haversine correlates well with real driving distance).

A circuit breaker around the Routing Service trips after sustained failures and switches all matches to haversine for a cool-down period (30 seconds), then probes to see if the service is back.

For ETA presented to riders (the "your driver will arrive in 4 minutes" estimate), the fallback is "your driver will arrive in a few minutes": fuzzier text instead of a wrong number.

### 14. Trade-offs worth saying out loud

Greedy match vs batch match. Greedy is simpler, faster, locally optimal. Batch (with a 500ms hold) gives 5-15% better global ETA in dense areas. Ship greedy first, add batched matching as an experiment per-city, keep the dispatch window tunable.

Why H3 not geohash. Hexagonal neighbor uniformity matters when most matches are within 1-2 rings. Geohash neighbors have non-uniform distance, which means more rings of search to cover the same physical area. Operationally, H3 also has the `compact`/`uncompact` API for multi-resolution queries, a small but real win.

Why stateful Driver Gateway. Stateless is the dogma, but a WebSocket cannot be stateless. The alternative (pull-based long polling) costs 3-5x more bandwidth and battery. Stateful with sticky routing and a session table is the right answer.

Why not use the Trips DB as the source of truth for driver location. Could be done. But 250k writes/sec to a relational DB is much more painful than 250k overwrite-in-place ops to Redis. Location is "current truth, not historical truth"; treating it as a cache, not a record, is the right shape.

Surge complexity. A real surge system has half a dozen knobs: demand smoothing, anti-gaming, regional caps, special events, regulatory compliance (some cities cap surge). The architecture above is the simple core; production has a lot more under the hood.

What I would revisit at 10x scale.
- Per-region matching services with cross-region "border" handling for trips that cross metro areas.
- Tiered location index: hot (last 60s) in memory, warm (last 5 min) in Redis, archived in object storage. Drivers idle for >5 min get a cheaper representation.
- Edge-deployed matching for hot cities. Round-trip from rider to a central data center is 50-100ms; pushing matching to a city-local micro data center could cut P99 by half.
- Replace haversine fallback with a precomputed road-distance table for hot O-D pairs (airport to downtown is asked thousands of times per day; just memoize it).

### 15. Common interview mistakes

Most weak answers fall into one of these:

"Just store driver locations in Postgres with a GiST index." Possible for v1 in one city. Falls over at 250k writes/sec sustained. Call this out as the toy answer and graduate to a Redis-backed H3 index for scale.

"Use the haversine formula." Acceptable as a fallback. As the primary scoring function, it ignores the road network and assigns drivers on the wrong side of a river. The interviewer is checking whether you know the routing layer is separate.

No state machine. A solution that talks about matching but never names the ride states (requested, matched, in_progress, completed, cancelled, no_show) is incomplete. The state machine is where most subtle production bugs live.

Ignoring idempotency. Network retries on `POST /rides` cause double-bills. Idempotency keys are required, not optional.

No mention of the hot cell problem. Every city has an airport. Every airport is a hot cell. If you cannot describe how to keep that one Redis key alive under load, you are missing a real operational concern.

Treating the Routing Service as a free function call. It is the heaviest dependency in the match path. Bounding its concurrency, timeout, and fallback is essential.

Forgetting about cancellations. Half of the state machine is cancel paths. If you only describe the happy path, you are leaving the interviewer to imagine the rest.

Confusing matching latency with acceptance latency. Matching is <200ms. Driver acceptance can take 5-15 seconds. The 2-second P99 target applies to "from rider tap until matching service has picked a driver and dispatched the offer," not "until driver has tapped Accept." Be clear which you are measuring.

Overengineering the multi-region story. Cities are independent. You do not need a globally-distributed multi-master location index. Per-region, per-city sharding is the right answer.

If you can hit 9 of these, you are interviewing at staff level. The most common gaps in real interviews are the state machine, hot cells, and the routing service fallback.
