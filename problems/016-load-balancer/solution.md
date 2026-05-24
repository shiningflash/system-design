## Solution: Design a Load Balancer

### The short version

A load balancer is a reverse proxy with two jobs: pick a backend for each incoming request, and keep the list of backends honest (kick out the dead ones).

The picking is the easy part. The interesting design work is everything around it. What does the LB *see* (L4 sees just IPs and ports, L7 sees the full HTTP request)? How does it decide a backend is healthy? What happens when a sticky cookie sends a user to a backend that then dies? And how do you stop the LB itself from being the thing that takes the whole site down?

The right setup is layered. DNS or anycast routes users to the nearest region. Inside each region, an L7 LB terminates TLS and does path-based routing to per-service pools. Inside each service tier, you may have a smaller L4 LB or a sidecar mesh. Each layer adds 1-3 ms of latency in exchange for one specific control point. You add layers when something specific breaks. Not before.

Scaling is straightforward but easy to get wrong. A single nginx is already a load balancer at 100 users. It is still fine at 10,000. Past that you add health checks and more backends. Then a managed cloud LB to handle TLS and DDoS. Then a global LB for geo-routing. Then sidecars for service-to-service traffic.

The trick: each step is driven by a *specific* failure. Bandwidth. TLS CPU. Connection count. Or geographic latency. Naming which one will break first for *your* workload is the whole interview.

---

### 1. The clarifying questions, recap

The eight questions are in `question.md`. Two of them drive almost the whole architecture.

**Protocol shape.** HTTP/2 with long-lived multiplexed connections is a totally different load picture from HTTP/1.1 with short connections. WebSockets change it again. Round-robin is fine for HTTP/1.1, lies to you under HTTP/2, and actively breaks for WebSockets.

**Stickiness requirement.** If backends hold no state (sessions in Redis, not memory), every algorithm is on the table. If they hold state (carts, WebSocket connections, in-memory caches), every algorithm has a tax. Sticky algorithms cause uneven load. Stateless algorithms force you to externalize state, which costs a network hop per request.

Everything else (geo, TLS, health checks, failover) follows from those two answers.

---

### 2. The math, in plain numbers

| Stage | Users | Req/sec peak | Bandwidth | Concurrent conns | New TCP/sec | What hurts |
|-------|-------|--------------|-----------|------------------|-------------|------------|
| 1 | 100 | 10 | ~5 Mbps | ~50 | ~5 | nothing; the box is bored |
| 2 | 10k | 500 | ~240 Mbps | ~2k | ~50 | NIC and nginx workers |
| 3 | 100k | 5k | ~2.4 Gbps | ~20k | ~500 | TLS CPU starts to matter |
| 4 | 1M | 30k | ~5-15 Gbps | ~200k+ | ~3k | TLS dominates, need multiple LBs |

Three ceilings to watch:

**Bandwidth.** A single 10 Gbps NIC handles about 9 Gbps usable. Video and file-heavy workloads hit this first.

**TLS handshakes.** Each handshake costs 1-3 ms of CPU. At 1,000/sec you spend 2 cores. At 10,000/sec, 20 cores doing nothing but handshakes. Fix: session resumption. When the same client returns, they reuse a "ticket" from last time. About 10x cheaper. TLS 1.3 with tickets is non-negotiable at scale.

**Connection count.** Each kept-alive connection costs file descriptors and ~10 KB of kernel memory. 1M concurrent connections needs Linux tuning (`ulimit`, `net.core.somaxconn`) and several GB of RAM just for the socket table.

Which ceiling hits first depends on your workload. Video CDN: bandwidth. IoT: TLS. Chat/WebSocket: connection count. Knowing which one is *yours* separates a generic answer from a useful one.

---

### 3. The API

A load balancer is not a REST API. It is a TCP/HTTP proxy. There are two surfaces.

The **data plane** is the LB doing its actual job: accept connection, pick backend, forward bytes. No REST endpoints here. The "API" is "speak HTTP at me and I will proxy."

The **control plane** is the admin API: add a backend, remove one, change weights, drain a backend before deploy, view current health.

```
GET    /admin/v1/pools                          # list all backend pools
GET    /admin/v1/pools/{pool}/backends          # list backends with health
POST   /admin/v1/pools/{pool}/backends          # add a backend
DELETE /admin/v1/pools/{pool}/backends/{id}     # remove a backend
PUT    /admin/v1/pools/{pool}/backends/{id}     # change weight or state

POST   /admin/v1/pools/{pool}/backends/{id}/drain   # no new conns,
                                                    # let existing finish
POST   /admin/v1/pools/{pool}/backends/{id}/ready   # re-enable

GET    /admin/v1/pools/{pool}/health            # current health snapshot
GET    /admin/v1/stats                          # request rate, errors, latency
```

The actual data model is in the LB's config file. nginx:

```nginx
upstream api_backend {
    least_conn;
    server 10.0.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 weight=1 max_fails=3 fail_timeout=30s backup;
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/ssl/api.crt;
    ssl_certificate_key /etc/ssl/api.key;
    ssl_session_cache   shared:SSL:50m;
    ssl_session_timeout 1d;

    location /api/orders/ {
        proxy_pass http://order_service;
    }
    location /api/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
    }
    location /healthz {
        return 200 "ok\n";
    }
}
```

HAProxy equivalent (you will see this in production too):

```
frontend api_frontend
    bind *:443 ssl crt /etc/ssl/api.pem alpn h2,http/1.1
    default_backend api_backend
    acl is_orders path_beg /api/orders/
    use_backend order_service if is_orders

backend api_backend
    balance leastconn
    option httpchk GET /healthz
    http-check expect status 200
    server api1 10.0.1.10:8080 check inter 5s fall 3 rise 2 weight 100
    server api2 10.0.1.11:8080 check inter 5s fall 3 rise 2 weight 100
    server api3 10.0.1.12:8080 check inter 5s fall 3 rise 2 weight 50 backup
```

---

### 4. The data model

The LB itself is mostly stateless. What state it does have is small and lives in RAM.

```
BackendPool {
    name: string
    algorithm: enum { round_robin, least_conn, ip_hash,
                      consistent_hash, ewma, weighted_rr }
    health_check: HealthCheckConfig
    backends: List<Backend>
}

Backend {
    id: string                        # "10.0.1.10:8080"
    address: ip + port
    weight: int                       # for weighted algorithms
    state: enum { healthy, unhealthy, draining, maintenance }
    consecutive_failures: int         # for active health check
    consecutive_successes: int        # for active health check
    in_flight_requests: atomic int    # for least_conn
    ewma_latency_ms: float            # for EWMA algorithm
    recent_5xx_count: ring_buffer     # for passive outlier detection
    last_check_at: timestamp
}

HealthCheckConfig {
    type: enum { http, tcp, grpc }
    path: string                      # for http; "/healthz"
    interval_ms: int                  # e.g., 5000
    timeout_ms: int                   # e.g., 1000
    unhealthy_threshold: int          # consecutive fails to eject
    healthy_threshold: int            # consecutive successes to re-add
    expected_status: List<int>        # [200] or [200, 204]
}

# Only if cookie-based stickiness is on.
StickyTable {
    map: ConcurrentMap<cookie_value, backend_id>
    ttl: duration
}
```

A few things to defend out loud:

**All of this fits in RAM.** A pool of 1,000 backends with full state is well under 1 MB. There is no database. The LB process owns the state.

**Each LB instance has its own opinion about health.** They do not share. By design. If LB #1's network path to backend X is broken but LB #2's path to X works fine, each should route to whatever it can reach. Sharing one global "X is unhealthy" opinion would turn a local network problem into a global outage.

**For sticky sessions across multiple LB instances**, the cleanest answer is: do not share state. Put the backend ID *inside the cookie itself* (signed and encrypted by the LB so clients cannot tamper with it). Any LB can read the cookie and route correctly.

---

### 5. The core algorithm

The LB's main loop is tiny.

```python
def handle_connection(conn):
    """Called once per incoming TCP connection (L4) or per HTTP request (L7)."""
    pool = match_pool_by_host_or_port(conn)

    while True:
        backend = pool.pick_backend(conn)
        if backend is None:
            return respond_503(conn, "no healthy backends")

        # Forward. If forwarding fails (backend died between health checks),
        # try the next one. Cap retries so a broken pool fails fast.
        try:
            backend.in_flight.increment()
            response = forward(conn, backend)
            backend.in_flight.decrement()
            record_outcome(backend, response.status, response.latency)
            return respond(conn, response)
        except BackendError as e:
            backend.in_flight.decrement()
            record_failure(backend, e)
            if retries >= pool.max_retries:
                return respond_502(conn, "backend exhausted")
            continue

def pick_backend(pool, conn):
    """Picks one. Returns None if pool is empty."""
    healthy = [b for b in pool.backends if b.state == "healthy"]
    if not healthy:
        return None

    if pool.algorithm == "round_robin":
        return healthy[pool.rr_counter.next() % len(healthy)]
    elif pool.algorithm == "least_conn":
        return min(healthy, key=lambda b: b.in_flight.get())
    elif pool.algorithm == "ip_hash":
        return healthy[hash(conn.client_ip) % len(healthy)]
    elif pool.algorithm == "consistent_hash":
        return pool.hash_ring.lookup(conn.routing_key)
    elif pool.algorithm == "ewma":
        return min(healthy, key=lambda b: b.ewma_latency_ms
                                          + b.in_flight.get() * EXP_REQ_MS)
    elif pool.algorithm == "weighted_rr":
        return pool.weighted_rr.next(healthy)
```

The health check loop runs in parallel, one thread per backend (or one batched checker per pool). It polls `/healthz`, increments consecutive_successes or consecutive_failures, and flips state when thresholds are crossed.

Passive outlier detection runs on every request: update the EWMA latency, push to a 5xx ring buffer, eject the backend temporarily if the recent 5xx count exceeds the threshold. Critically, the eject only happens if it would not push the pool below `max_ejection_percent`.

Three things make this safe:

**Pick-and-forward tolerates a backend dying between health checks.** If forwarding raises, the LB moves to the next backend. Capped so a fully broken pool fails fast instead of looping forever.

**Active and passive checks complement each other.** Active catches process death and obvious failures. Passive catches slow degradation that active misses (a backend returning 200 to `/healthz` while throwing 500 on real traffic).

**`max_ejection_percent` is the difference between graceful degradation and total outage.** The outlier code refuses to eject if doing so would push the pool below the floor. Better to keep sending traffic to a sick backend than to send it nowhere.

---

### 6. The architecture, drawn out

Here is the full stack at 1 million users.

```
                          User worldwide
                                |
                                v
                       +-----------------+
                       |       DNS       |   Returns anycast IP, or
                       |  (Route 53,     |   geo-aware A record per region.
                       |  Cloudflare)    |   TTL 60s for fast failover.
                       +--------+--------+
                                |
                                v
                  +---------------------------+
                  |    Anycast IP via BGP     |   Same IP advertised from
                  |  (Cloudflare, AWS Global  |   every region. BGP routes
                  |  Accelerator, GCP Premium |   client to nearest healthy
                  |  Tier)                    |   region.
                  +-------------+-------------+
                                |
        +-----------------------+-----------------------+
        |                       |                       |
        v                       v                       v
  +-------------+         +-------------+         +-------------+
  | Region:     |         | Region:     |         | Region:     |
  | us-east     |         | eu-west     |         | ap-south    |
  |             |         |             |         |             |
  | +---------+ |         | +---------+ |         | +---------+ |
  | | Edge L4 | |         | | Edge L4 | |         | | Edge L4 | |
  | |   LB    | |         | |   LB    | |         | |   LB    | |
  | | (NLB,   | |         | |         | |         | |         | |
  | | DDoS    | |         | |         | |         | |         | |
  | | scrub)  | |         | |         | |         | |         | |
  | +----+----+ |         | +----+----+ |         | +----+----+ |
  |      |      |         |      |      |         |      |      |
  |      v      |         |      v      |         |      v      |
  | +---------+ |         | +---------+ |         | +---------+ |
  | | L7 LB   | |         | | L7 LB   | |         | | L7 LB   | |
  | | (ALB,   | |         | |         | |         | |         | |
  | | Envoy,  | |         | |         | |         | |         | |
  | | nginx)  | |         | |         | |         | |         | |
  | | TLS +   | |         | |         | |         | |         | |
  | | path    | |         | |         | |         | |         | |
  | | routing | |         | |         | |         | |         | |
  | +--+--+--++ |         | +---------+ |         | +---------+ |
  |    |  |  |  |         |             |         |             |
  |    v  v  v  |         |             |         |             |
  | +---+ +--+ +--+       | per-service |         | per-service |
  | |api| |au| |st|       | pools +     |         | pools +     |
  | |   | |th| |at|       | sidecar     |         | sidecar     |
  | +---+ +--+ +--+       | mesh        |         | mesh        |
  |  pods (each   |       |             |         |             |
  |  with Envoy   |       |             |         |             |
  |  sidecar)     |       |             |         |             |
  +---------------+       +-------------+         +-------------+
```

What each layer does and *why* it earns its slot:

**DNS** is the first place where balancing happens (whether you call it that or not). Returning multiple A records to a hostname is round-robin at layer zero. Geo-aware DNS (Route 53 latency routing) is poor-man's geo-LB and works when anycast is unavailable.

**Anycast** announces the same IP from every region using BGP. ISPs route each client to the nearest LB pool. Failover is automatic and silent: a region's BGP session drops, traffic shifts to the next-nearest in seconds.

**Edge L4 LB** is the first line of defense. Terminates TCP, absorbs DDoS, hands off to L7. Often a managed service (AWS NLB, Cloudflare's edge). Optimized for raw packet throughput, not for parsing.

**L7 LB** is where TLS termination lives. Parses HTTP. Does path and host routing. Applies per-route rate limits. Injects headers like `X-Forwarded-For`. nginx, HAProxy, Envoy, ALB.

> Why two layers of load balancing? Because the edge LB does one job extremely well: terminate TLS and route to the right region. The L7 LB inside the region does the smart stuff: path-based routing, sticky sessions, header-based A/B tests. Mixing the two into one box creates a single point of failure that is also slower.

**Service tier.** Each backend service has its own pool. The L7 LB routes to it by path.

**Sidecar mesh (Envoy, Linkerd, Istio).** For service-to-service traffic *inside* the cluster. Every pod has its own Envoy sidecar. That sidecar is the LB for outbound traffic. Removes the central LB hop for internal calls. Adds 1-2 ms per hop but gives you per-service mTLS, retries, and observability for free.

---

### 7. A request's journey, end to end

There is no separate "read path" and "write path" inside the LB. There is one path: request in, pick backend, forward, response out.

But it is worth tracing one request through all the layers to see where the latency comes from.

A user in Sydney hits `api.example.com`:

```
Browser    DNS    Anycast/BGP   Edge L4    L7 LB     Backend
   |        |          |           |         |          |
   |  resolve api      |           |         |          |
   |------->|          |           |         |          |
   |<-------|          |           |         |          |
   | returns anycast IP|           |         |          |
   |        |          |           |         |          |
   | TCP SYN to anycast IP         |         |          |
   |------------------>|           |         |          |
   |        |  BGP routes to ap-south        |          |
   |        |          |---------->|         |          |
   |        |          |  5-tuple hash       |          |
   |        |          |  to L7 LB instance  |          |
   |        |          |           |-------->|          |
   |        |          |           |         |          |
   |  TLS handshake (1-RTT, or 0-RTT with ticket)       |
   |<================================>|     |          |
   |        |          |           |         |          |
   |  HTTP request "GET /api/orders/123"     |          |
   |--------------------------------------->|          |
   |        |          |           |  parse |          |
   |        |          |           |  match path       |
   |        |          |           |  pick backend     |
   |        |          |           |  (least_conn)     |
   |        |          |           |         |  forward|
   |        |          |           |         |-------->|
   |        |          |           |         |  process|
   |        |          |           |         |<--------|
   |        |          |           |         |  200 OK |
   |        |          |           |<--------|         |
   |  200 OK streams back                    |         |
   |<---------------------------------------|         |
```

Step by step, with latency budgets:

- **DNS resolution.** Cached <1 ms, uncached ~30 ms. Only the first connect pays this.
- **Anycast routing.** BGP routes the SYN to ap-south. ~10 ms round trip from Sydney.
- **Edge L4 LB.** Picks one of N edge instances by 5-tuple hash (same connection always lands on same instance). Forwards to L7. <1 ms.
- **L7 LB TLS handshake.** TLS 1.3. With a session ticket: 0-RTT. Without: 1 RTT. 0-15 ms.
- **L7 LB parse and route.** Match `/api/orders/123` to the order_service pool. <1 ms.
- **L7 LB backend selection.** Pool uses least_connections. Picks pod #7. <1 ms.
- **L7 LB to backend.** Reuse a TCP connection from the keepalive pool. <1 ms inside the region.
- **Backend processes.** 5-50 ms.
- **Response streams back.** Dominated by network send time.

Total: 30-100 ms for a fresh client, 10-50 ms for a returning client (DNS cached, TLS resumed). The LB layers added maybe 5 ms. That is the price for control, observability, and smart routing.

---

### 8. The scaling journey: 10 users to 1 million

This is the part interviewers care about most. Four stages. Each is driven by a *specific* failure, not by anticipated load. Build nothing before you need it.

#### Stage 1: 100 users, single nginx, one backend

```
[Browser] -> [nginx :443] -> [backend :8080]
```

That single nginx is already a load balancer. It:

- Terminates TLS so the backend speaks plain HTTP.
- Does a trivial "round-robin" across one backend (a no-op, but the config is ready).
- Does a trivial `/healthz` check.

Why put nginx in front from day 1? Three reasons:

- You can swap the backend (deploy on port 8081, switch upstream, no DNS change).
- You can add a second backend later without restructuring.
- TLS lives in one place, so cert renewal does not need a backend deploy.

Cost: ~$5/month for a t3.micro running both nginx and the backend. Or less on a hobby tier.

What you do **not** build yet: no real health checker, no failover (the box dies, the service is down; for 100 users that is fine), no sticky sessions.

#### Stage 2: 10,000 users, 3-5 backends, real health checks

**What just broke:** the single backend pegs CPU during bursts. Killing it for deploys takes the site down for 30 seconds.

**The fixes:**

- Scale backends to 3-5 instances behind the same nginx.
- Switch the algorithm to `least_conn`. Round-robin is okay, but `least_conn` handles variable request times better.
- Add active health checks every 5 seconds against `/healthz`. Eject after 3 failures, re-add after 2 successes.
- For deploys, drain one backend at a time (`down` in nginx config, reload, redeploy, bring it back).
- Add `proxy_next_upstream` so the LB silently retries on a different backend if the first one fails.

```nginx
upstream api {
    least_conn;
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}
```

What you **do not** build yet: no second LB instance. One nginx is still the SPOF. For 10,000 users, a 30-second DNS failover for the rare event is acceptable. No L4 in front. No anycast or geo-routing.

Cost: ~$50-100/month. A small LB box plus 3-5 backend boxes.

#### Stage 3: 100,000 users, two-layer LB, regional managed LB

**What just broke** (several things at once):

- TLS CPU on the single nginx is biting. 500 new connections/sec × 2 ms = one whole CPU core just on handshakes.
- One nginx is now a SPOF you cannot accept. A 30-second failover is too much.
- The team is splitting the monolith into services (api, auth, search, static). The single nginx config is becoming a wall of `location` blocks.
- Script-kiddies are noticing you. You want basic DDoS protection.

**The fixes:**

- Add a managed cloud L7 LB at the edge (ALB, GCP HTTPS LB, or Cloudflare in front). It terminates TLS, absorbs basic DDoS, routes to per-service target groups.
- Add an internal L7 LB per service tier (Envoy or nginx). Each service team owns their pool config.
- Sticky sessions for the few services that need them (cart, real-time UI). Cookie-based, 1-hour TTL.
- Active-active LBs. ALB is multi-AZ by default. For self-run, two instances behind a VIP or DNS round-robin.
- Turn on HTTP/2 between client and LB. This cuts new-connection rate by about 10x for browsers.

Envoy config for the internal L7 tier has clusters per service, each with `lb_policy: LEAST_REQUEST`, http_health_check on `/healthz` (interval 5s, unhealthy_threshold 3, healthy_threshold 2), and outlier_detection (consecutive_5xx 5, base_ejection_time 30s, max_ejection_percent 50).

What you **do not** build yet: no global LB (still one region). No sidecar mesh.

Cost: ~$1-5k/month. ALB at ~$25/month + per-GB charges (~$0.008/GB), internal LB pods, more backends.

#### Stage 4: 1,000,000 users, global + regional + local + sidecar mesh

**What just broke:**

- Users in Asia, Europe, the Americas all need <100 ms latency. One us-east LB cannot do that.
- TLS handshake CPU at 30,000 req/s is now thousands of dollars per month if you run your own. Managed TLS is cheaper per byte.
- Service-to-service traffic hits the central L7 LB on the way out *and* back. Internal latency budgets are eaten by LB hops.
- Cross-region failover is too slow because DNS TTL is the bottleneck.

**The fixes:**

- Add a global LB (anycast or geo-DNS): Cloudflare, AWS Global Accelerator, GCP Premium Tier. Same IP, routed by BGP to the nearest healthy region. Sub-second failover (BGP withdraw) instead of DNS-TTL-bound.
- Regional LBs stay the same as stage 3.
- Add a sidecar mesh (Envoy, Linkerd, Istio). Every pod gets a sidecar. Service-to-service calls go: pod -> local sidecar -> remote sidecar -> remote pod. The sidecar is the LB for outbound traffic, which removes the central L7 hop for internal traffic. Adds 1-2 ms per hop but gains per-service mTLS, retries, circuit breakers, and per-call observability.
- Bounded-load consistent hashing for any pool with stateful affinity (cache nodes, session-affinity tiers). Spreads hot keys.
- TLS offload to hardware or SmartNIC at the edge.
- Slow start for new backends: ramp weight from 0 to 100% over 30 seconds so cold caches do not get hammered.

Each layer adds latency in exchange for one control point:

| Layer | Latency added | What you gain |
|-------|---------------|---------------|
| DNS + Anycast | ~5-20 ms (first connect only) | Geo-routing, regional failover |
| Edge L4 | <1 ms | DDoS scrubbing, TLS offload |
| Regional L7 | 1-3 ms | Path routing, TLS, per-route policy |
| Sidecar | 1-2 ms per hop | mTLS, observability, retries |
| Backend pod | request time | the actual work |

A request crosses three or four LB layers before hitting the backend. Each layer earns its slot. You do not add one "in case."

Cost: ~$10-100k/month depending on bandwidth, regions, and how much is managed.

#### What you would do at 10x scale (10M+ users)

Honestly, the stage-4 architecture *is* what FAANG companies run. At 10M+ you are not changing the topology. You are doing more of the same:

- More regions.
- More sophisticated outlier detection.
- Custom-built LBs (Google's Maglev, Facebook's Katran) instead of off-the-shelf. Saves real money on CPU.

The interesting work shifts from "design the LB" to "tune the LB at scale": kernel bypass (DPDK, XDP), zero-copy forwarding, custom NIC offloads. None of that changes the interview answer.

---

### 9. Reliability

**The LB itself dies.** Covered above. Two patterns: active-passive VIP with keepalived (one LB works, one stands by, ~1-second failover) and active-active with anycast (all LBs work, BGP shifts traffic on failure, sub-second).

**A backend dies.** Active health checks notice within 1-3 intervals (5-15 seconds). The LB stops sending it new requests. In-flight requests on the dead backend may fail; `proxy_next_upstream` retries them on a healthy backend.

**The whole backend pool dies.** `max_ejection_percent: 50` stops the LB from ejecting everyone. Worst case the LB returns 503 while a human investigates. Better the LB tells the truth (503) than tries to forward to nowhere and hangs.

**A backend is alive but slow.** This is the dangerous one. Active checks pass (it answers `/healthz`). Passive checks pass (it returns 200s, just slowly). The slow backend accumulates connections. With round-robin you keep sending it more.

The fix: `least_conn` avoids it naturally (most in-flight). EWMA avoids it explicitly (worst latency). Add request timeouts at the LB so a slow backend cannot tie up LB workers forever.

**Cascading failure / thundering herd.** Backend A is slow. LB ejects it. Traffic shifts to B and C. They run hot. LB ejects them too. Now no backends. LB returns 503. Clients retry. The retries hammer the next backend that comes up.

The fixes:

- `max_ejection_percent: 50` (never eject everyone).
- Backend-side circuit breakers (when load > threshold, return 503 fast rather than serving slowly).
- Client-side retries with exponential backoff and jitter.
- LB-side request hedging, *carefully* (sending the same request to a second backend if the first is slow can either save tail latency or melt a pool; use only with caps).

**DNS failure / regional outage.** If a region is BGP-withdrawn, anycast handles failover automatically. If DNS fails (rare), clients with cached entries keep working until TTL expires; new clients are stranded. This is why DNS TTL matters: 60 seconds is the usual production setting.

**TLS cert expiry.** A cert silently expiring takes the whole service down. Automate renewal (Let's Encrypt + cert-manager). Alert on certs expiring within 30 days. The number of outages caused by expired certs is embarrassing.

---

### 10. Observability

What must be instrumented from day one:

| Metric | Why it matters |
|--------|----------------|
| `lb.request.rate` per backend | The headline distribution metric; uneven values mean uneven balancing |
| `lb.request.error_rate` per backend | A single backend spiking errors is a candidate for ejection |
| `lb.request.latency` p50/p95/p99 per backend | Find slow backends before EWMA / outlier ejects them |
| `lb.healthy_backend_count` per pool | Drops below threshold = page someone |
| `lb.ejected_backend_count` per pool | Spikes = something pool-wide is wrong, possibly the health check itself |
| `lb.tls.handshakes_per_sec` | TLS CPU bound; tells you when to scale or offload |
| `lb.tls.session_resumption_rate` | If low (<70%), clients are not benefiting from session tickets |
| `lb.connection.active_count` | File descriptor and memory pressure indicator |
| `lb.connection.new_per_sec` | Should be much less than request rate (keep-alive working) |
| `lb.upstream.retry_rate` | If high, backends are flaky; the LB is masking it |
| `lb.upstream.queue_depth` | Requests waiting for an upstream connection |
| `lb.bandwidth.in / out` | NIC saturation watch |

Per-route metrics in addition to per-backend: `lb.request.rate{path="/api/orders/*"}`. Lets you see if one route is overwhelming the pool.

**Page on:** `healthy_backend_count < 50% of total` for 1 minute. LB instance unresponsive. TLS handshake error rate > 1%.

**File a ticket on:** latency p99 regression > 30%. `ejected_backend_count > 0` sustained for 10 minutes. Bandwidth > 70% of NIC capacity.

---

### 11. Gotchas the senior interviewer is listening for

Some of these only come out when the interviewer asks "what happens if...". The senior candidate brings them up unprompted.

**Sticky sessions cause uneven load.** Power users (admins, long sessions) cluster on the same backends. Fix: cap session TTL, allow rebalancing, prefer stateless backends with shared session storage.

**Slow backends starve LB workers.** A slow backend ties up workers waiting for its response. Fix: aggressive timeouts. Eject slow backends on latency, not just 5xx.

**TLS termination CPU.** ECDSA certs are ~5x cheaper than RSA at the same security level. TLS 1.3 with session tickets gives 0-RTT for returning clients. Without these, LB CPU climbs linearly with new-connection rate.

**HTTP/2 breaks `least_conn`.** One TCP connection carries many concurrent requests. Every backend looks like it has 1 connection. Count *active requests* instead (Envoy's `LEAST_REQUEST`).

**`X-Forwarded-For` trust.** If the LB does not strip incoming XFF and append carefully, a malicious client spoofs their IP. Always `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for` (appends, does not replace). Backend trusts only the rightmost N IPs where N = trusted proxy hops.

**Health check thundering herd.** 100 LBs × 1,000 backends × every 5 seconds = 20,000 req/s on `/healthz`. Deep checks DoS your own DB.

**DNS round-robin and TTL.** Browsers cache DNS aggressively (sometimes forever). DNS round-robin is balancing in expectation, not enforcement. Do not rely on it for failover.

**Draining and graceful shutdown.** When you remove a backend, it must stop accepting new requests but finish in-flight ones. Skip this and you drop requests.

**Anycast and connection state.** BGP can re-route mid-connection if a route flaps. Long-lived TCP (WebSockets, gRPC streams) can break. Fix: pin the connection to a specific LB IP using a 302 or a global LB's client affinity feature.

**WebSockets and round-robin.** A WebSocket is one long connection. Round-robin spreads the initial connect evenly, but all the traffic over that connection lives on one backend for hours. Use `least_conn` for connect, accept some imbalance, rebalance on disconnect.

---

### 12. Follow-up answers

These are the questions a senior interviewer is listening for. Each answer is short on purpose. The depth is in the *why*.

**1. Sticky sessions and uneven load.**

Cookie-based stickiness gives perfect affinity but creates skew when power users cluster. Options, in order of how much you give up:

- **Cap session TTL.** Re-balance after, say, 1 hour. Brief inconvenience (cart sometimes re-resolves) in exchange for periodic rebalance.
- **Bounded-load stickiness.** If the target backend is at >1.25x average load, fall back to a different one and re-emit the cookie. Slightly disrupts stickiness for the unlucky few but caps the imbalance.
- **External session store.** Backend becomes stateless. Sessions live in Redis. Any backend handles any request. Pays a Redis hop per request (~1 ms), and the LB is back to round-robin or least_conn. This is the right answer for new systems. Legacy systems often cannot pay the rewrite cost.

The choice depends on what the session contains. Tiny session (auth token): externalize. Large session (in-progress workflow with megabytes): stickiness, accept the skew, monitor outliers.

**2. TLS termination cost.**

In order of cost:

- **Enable TLS 1.3 with session tickets.** Returning clients do 0-RTT, about 10x cheaper than fresh handshakes. Free; just enable.
- **Use ECDSA certs instead of RSA.** ECDSA-P256 is ~5x faster than RSA-2048. Free; cert change only.
- **Tune session resumption cache.** Bigger cache holds more clients between visits. Costs a bit of RAM.
- **Scale the LB horizontally.** More instances = more CPU cores doing handshakes. Linear cost.
- **TLS offload to a dedicated tier.** Run a TLS-terminating proxy in front. L7 LB behind it speaks plain HTTP. Splits the CPU burden cleanly.
- **SmartNIC / ASIC offload.** Some cloud providers offer hardware TLS (AWS Nitro, Cloudflare's stack). Highest fixed cost, lowest per-handshake cost. Worth it only above ~10k handshakes/sec.

Most teams stop at step 3 or 4.

**3. HTTP/2 and least connections.**

With HTTP/2, each client opens *one* TCP connection and multiplexes many requests over it. The LB's `least_conn` sees: every backend has 1 connection. The first backend (lowest ID) wins ties consistently; all new connects land there. After warm-up, one backend has 100 connections, the others have 1 each.

Fix: switch from "least connections" to "least active requests." Envoy's `LEAST_REQUEST`. nginx's `least_conn` is too literal with HTTP/2. The LB must parse HTTP/2 frames and count in-flight streams per backend, not connections.

If the LB is L4 and cannot parse HTTP/2, switch to consistent hashing on a per-request key, or move to an L7 LB.

**4. WebSockets and uneven distribution.**

WebSocket connect is one round trip. After that, the LB just shuffles bytes. Users connect once and stay connected for hours.

With 10k users and 5 backends on round-robin, the initial spread is even (2k per backend). But as users disconnect and reconnect, the spread drifts based on disconnect patterns. If a backend bounces (deploy, crash), all its connections reconnect and round-robin sends them all to the next backend, doubling its load.

Fixes:

- `least_conn` for the connect step. Backends with more accumulated connections get fewer new ones.
- Drain gracefully on bounce. Close connections in waves over several minutes, not all at once.
- Cap connections per backend. If a backend hits 5,000, the LB stops sending new ones and returns 503. Client retries to a different backend.
- Consider a dedicated WebSocket gateway layer separate from your HTTP LB. Different scaling story, different operational model.

**5. Slow backend starving the pool.**

`least_conn` is the simplest fix. The slow backend accumulates in-flight requests. Once it has more than the others, the LB stops sending it new ones. It naturally drains.

Combine with `proxy_read_timeout 30s` (or whatever your hard cap is). After 30 seconds the LB gives up, frees the worker, the request fails. Better than tying up a worker forever.

Add passive outlier detection on latency: if a backend's p99 is >3x pool median for 1 minute, eject for 30 seconds, let it cool off, bring back gradually.

Backend-side circuit breaker: when its own dependency (slow disk, slow DB) is sick, return 503 immediately rather than serving slowly. The LB ejects, the backend recovers, the LB re-admits.

EWMA explicitly avoids slow backends but is harder to tune. Start with `least_conn` + timeouts.

**6. DNS TTL.**

- Long TTL: clients cache longer, fewer DNS queries, slower to react to changes.
- Short TTL: clients re-resolve often, faster failover, more DNS load.

Production default: 60 seconds. Failover happens within 1-2 minutes worst case. Some traffic-heavy services use 30s. Internal services where you have more control can use longer (300s).

For an emergency LB IP change: drop TTL to 30s 24 hours before so resolver caches have short TTLs by the time you flip. After the change, raise TTL back.

The real fix is to not change LB IPs. Anycast IPs do not change; they just stop being advertised from a failed region. The IP-change problem is mostly a non-anycast problem.

**7. Cross-region failover.**

Layers, top to bottom:

- **Anycast (if used).** BGP advertisement from the failed region is withdrawn (manually or automatically by health check). Routers learn within seconds. Clients silently routed to the next-nearest region. Sub-second on good networks. Up to 30s on slow ones.
- **Geo-DNS (if used instead).** Route 53's latency-routing or geolocation policies have health checks. When the us-east check fails, Route 53 returns eu-west's IP. Bound by DNS TTL (60s).
- **Regional LB.** Each region's LB is internally HA. Failure within a region is handled by the regional LB.
- **Client retries.** Clients with cached DNS still hit us-east for up to TTL seconds. They get connection-refused and retry. SDKs with retry logic eventually succeed against eu-west.

End-to-end: anycast = ~5-30s. Geo-DNS = ~60-300s. The slowest piece is whatever your DNS TTL is.

**8. Path-based routing for a monolith split.**

Add a route to the L7 LB:

```nginx
location /api/orders/ {
    proxy_pass http://order_service;
}
location /api/ {
    proxy_pass http://monolith;
}
```

Order matters. More specific paths first.

Migration plan (the strangler-fig pattern):

- **Phase 1:** deploy `order_service` running in parallel with the monolith. Both can handle `/api/orders/*`.
- **Phase 2:** route a small slice (5%) using `split_clients`:

```nginx
split_clients "${request_id}" $backend {
    5%   order_service;
    *    monolith;
}
```

- **Phase 3:** monitor metrics. If `order_service` handles 5% cleanly, raise to 25%, then 50%, then 100%.
- **Phase 4:** remove order-handling from the monolith. Route 100% of `/api/orders/*` to `order_service`.

The LB is the cutover point. Rollback is one config change.

**9. Health check storm.**

200 LB instances × 500 backends ÷ 5s = 20,000 req/s on `/healthz`. If checks are deep (DB query), you DoS your own DB.

Fixes:

- **Shallow `/healthz`** returning 200 from the process. Catches process death, not deep issues. Deep checks (DB connectivity) run on a slower interval (60s) and report to a different system, not the LB.
- **EDS / xDS push instead of pull.** Modern LBs (Envoy) can subscribe to a service-discovery system that *pushes* updates. The LB is told "backend X is unhealthy." The discovery system does the actual health checking centrally (once, not 200 times). Cuts traffic by 200x.
- **Sampling.** Each LB only checks a random subset of backends each interval. Rotates over time. Cuts per-backend load proportionally.
- **Outsource to the platform.** In Kubernetes, `kubelet` runs liveness/readiness probes once per node. The LB watches Endpoints. Backends are health-checked once by the platform, not N times by N LBs.

In practice, EDS/xDS plus shallow checks gets you 99% of the way. The platform handles the rest.

**10. LB dropping connections during a deploy.**

Two failure modes:

- New backends register and start receiving traffic before they are warm. First requests hit a cold cache or unfinished init. Clients see errors.
- Old backends are killed mid-request. In-flight requests fail.

Correct deploy sequence:

1. **New backend starts.** Process boots. App initializes (DB, caches, etc.).
2. **Readiness probe.** Backend's `/ready` returns 503 until init finishes. Kubernetes/LB does not include it in the pool until `/ready` is 200. This is separate from `/healthz`, which only indicates liveness.
3. **Slow start.** LB ramps the new backend's weight from 0 to 100% over 30 seconds. Avoids hammering a cold backend.
4. **Old backend drain.** Mark as `draining`. No new connections accepted. In-flight requests finish.
5. **Old backend stop.** After drain timeout (e.g., 30s), kill the old backend. If a request is still in flight after 30s, it gets dropped (acceptable; clients retry).
6. **Repeat per backend.** Rolling update. Never lose more than `(replicas - max_unavailable)` at once.

Kubernetes does most of this with readiness probes + `preStop` hooks + `terminationGracePeriodSeconds`. Envoy and Istio do slow-start when configured. The mistake is skipping any of: readiness probe, slow start, drain. Each skipped step is a class of dropped requests.

---

### 13. Trade-offs worth saying out loud

**Hardware LB vs software LB vs cloud-managed.**

- **Hardware (F5, Citrix NetScaler):** high upfront cost, very high throughput per unit, complex to operate, locked to one vendor. Common in enterprises with regulatory requirements.
- **Software (nginx, HAProxy, Envoy):** commodity hardware, open source, full control, you operate it. Default for most teams.
- **Cloud-managed (ALB, GCLB):** zero operations, pay per request and per GB, less flexible config. Default for cloud-native teams below a certain bandwidth bill. Above ~10s of TB/month outbound, self-managed gets cheaper.

**L4 vs L7.** L4 is faster, cheaper, dumber. L7 is slower, more expensive, smarter. Pick L7 if you need anything HTTP-aware (path routing, cookies, headers, per-route policy). Pick L4 for raw TCP or when you need the throughput and do not need parsing.

**Sidecar mesh vs centralized LB.**

- **Centralized:** one LB tier in front of all services. Easy to reason about. Single config point. Single failure point.
- **Sidecar:** every pod has a proxy. Decisions are local. Full per-call observability. More moving parts. More memory overhead (~50 MB per sidecar × thousands of pods = real money).

Sidecar wins for service-to-service inside the cluster. Centralized wins for ingress from the internet. Most large systems use both.

**Sticky sessions vs stateless backends.** Stickiness is operationally simpler (no shared state to operate). Stateless is operationally cleaner (any LB algorithm works, any backend serves any request). New systems should default to stateless with an external session store. Legacy systems are often stuck with stickiness.

**What you would revisit at 10x scale.** Move to a custom LB if cloud LB costs dominate (Google built Maglev, Facebook built Katran for exactly this reason). Push more logic into the sidecar mesh; let the central L7 do less. TLS 1.3 with 0-RTT everywhere, including internal hop boundaries. Pre-compute hash rings for consistent-hash pools at config change time, not per-request.

---

### 14. Common interview mistakes

Most weak answers fall into one of these.

**Treating "load balancer" as a single thing.** The LB is layered. Saying "we use nginx" without acknowledging the DNS/anycast layer in front and the sidecar layer behind shows you have not thought about topology.

**No mention of L4 vs L7.** Knowing the difference is the most basic concept question on this topic. Skip it and you lose the round.

**Round-robin everywhere.** Round-robin is the wrong default for variable-duration requests, HTTP/2, WebSockets, and pretty much any modern workload. `least_conn` (or its HTTP/2 equivalent, least active requests) is a better default.

**Ignoring sticky session costs.** Stickiness is easy to enable, hard to live with. "We'll use cookie-based stickiness" without discussing the load skew and the backend-failover reshuffle is a junior answer.

**No mention of health check storms.** At non-trivial scale, the LB tier polling backends becomes its own DoS vector. EDS/xDS or platform-managed probes (Kubernetes) is the senior answer.

**Forgetting the LB is a SPOF.** Active-active anycast or active-passive VIP. Either is acceptable. Not addressing it is not.

**TLS termination handwave.** "Terminate at the LB" is correct but incomplete. Mention session tickets, ECDSA, and at scale, hardware offload.

**Cascading failure / thundering herd not mentioned.** This is the most common production LB incident. `max_ejection_percent` and backend-side circuit breakers are the answer.

**No drain step in deploys.** Rolling deploys without drain drop requests. A senior candidate names readiness probes, slow start, and drain explicitly.

**Treating sidecar mesh as a buzzword.** If you mention service mesh, explain *why* (removes central LB hop for internal traffic, per-service mTLS and observability), not just *that*. Otherwise it sounds like architecture-by-fashion.

If you can hit seven of these ten without prompting, you are interviewing at senior level on the topic. The three that separate strong from average answers: L4/L7 framing, sticky session costs, and health check storms. Those are what the senior architect listens for.
