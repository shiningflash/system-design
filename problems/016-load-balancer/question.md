---
id: 16
title: Design a Load Balancer
category: Networking
topics: [l4, l7, health checks, sticky sessions, algorithms, failover]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer draws a box labeled `backend`, a few stick figures labeled `users`, and an arrow between them. Then they sketch a smaller box in the middle, label it `nginx`, and lean back.

> *"One service, one box, 100 users. There is a single nginx out front. Walk me from here to 1 million users. At every step, tell me what the load balancer is actually doing, why it is there, and what breaks when you outgrow it."*

> *"I do not want a treatise on Kubernetes. I want you to tell me what the LB does at the packet level, when L4 stops being enough, why sticky sessions are a tax you pay rather than a feature you get, and what happens when the LB itself becomes the single point of failure."*

This is a concept question dressed up as a scaling question. Most candidates flinch past load balancers because "everyone uses nginx" or "ALB does it for you, what is there to design?" That is the wrong instinct. They are checking whether you can explain how an LB actually works, when each algorithm earns its keep, and how you stop the LB from being the thing that takes down your site.

You are going from one nginx on a t3.micro to a global anycast stack. At every layer, name what just broke and what fixes it.

## Step 1: clarify before you design

Take five minutes. Aim for eight questions or so. The answers shape whether you need L4 or L7, whether sticky sessions matter, whether geo-distribution is on the table, and whether you can hand the whole thing to a cloud provider or have to run it yourself.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

The eight I always ask, and why each one swings the design.

**Protocol.** Raw TCP, HTTP/1.1, HTTP/2, gRPC, or WebSockets? L4 is fine for raw TCP. HTTP/2 changes how you count "connections" because one TCP carries many requests. WebSockets break round-robin entirely because a connection lives for hours. gRPC behaves like HTTP/2 on caffeine, one long connection per client.

**TLS termination.** Does the LB terminate TLS or pass encrypted bytes through? Termination centralizes certs and lets you do L7 routing. Pass-through (SNI-only) hides traffic from the LB but you lose path routing.

**Sticky sessions.** Do backends hold per-user state in memory, or are they stateless? If stateless, every algorithm is on the table. If they hold sessions, carts, WebSocket state, in-memory auth, you pay a tax: sticky routing causes uneven load, externalizing state costs a hop per request.

**Geo-distribution.** One region or worldwide? One data center is a single LB layer. Worldwide needs DNS-based or anycast routing in front of regional LBs.

**Traffic shape.** Request rate, bandwidth, concurrent connection count, spiky or steady? An LB doing 1k req/s of tiny JSON is a different beast from one pushing 100 Gbps of video.

**Backend health.** How fast do backends fail and how fast do we need to notice? Sub-second failover and a 30-second outage are different worlds. This drives the health check interval and the algorithm choice.

**Cost sensitivity.** Managed cloud LB (ALB, GCLB) or self-run (nginx, HAProxy, Envoy)? Managed is cheaper in engineering time and more expensive per byte. At a certain bandwidth bill the math flips.

**Failover for the LB itself.** If the LB dies, what happens? Is one box with a 30-second DNS failover okay, or do you need active-active behind a VIP? Most candidates forget the LB is a SPOF until pushed.

While you are at it, name what is out of scope: WAF rules, DDoS scrubbing, bot detection. They live near the LB but are separate problems.

</details>

## Step 2: capacity estimates

The interviewer hands you the numbers.

Start: 100 users, ~10 req/s, single nginx, single backend.
End: 1M daily actives, ~30k req/s peak, ~5 Gbps peak bandwidth, average request 4 KB, response 60 KB.
HTTP/1.1 today, HTTP/2 coming. Two regions to start, four eventually. Backends are stateless; sessions live in a shared cache.

At each scale, work out: requests per second sustained and peak, concurrent connections, bandwidth in and out, new connections per second (TLS handshakes hurt), and how many backends you need behind the LB.

<details>
<summary><b>Reveal: the math</b></summary>

**Stage 1: 100 users, 10 req/s.** Bandwidth is 10 × 60 KB = 600 KB/s, roughly 5 Mbps. Negligible. Tens of concurrent connections. One backend handles it; nginx is in front mostly so you can swap that backend later without touching DNS.

**Stage 2: 10k users, ~500 req/s peak.** Bandwidth is 30 MB/s, about 240 Mbps. A single 1 Gbps NIC yawns. Around 2,000 concurrent connections (10k users × ~20% active × a connection or two each). Browsers keep-alive, so maybe 50 new sessions per second. TLS cost is fine. Three to five backends.

**Stage 3: 100k users, ~5k req/s peak.** Bandwidth is 300 MB/s, around 2.4 Gbps. Getting close to single-NIC limits. About 20k concurrent connections. nginx handles this on a beefy box but you want headroom. New connections: ~500/sec. TLS now needs a real CPU budget: each handshake is 1-3 ms on x86 with ECDSA, 500/sec × 2 ms means one core busy doing nothing but handshakes. Twenty to fifty backends, partitioned by service (api, web, auth, search).

**Stage 4: 1M users, ~30k req/s peak, 5 Gbps.** Bandwidth is 5 Gbps sustained, 15+ Gbps peak. No single LB box. 200k+ concurrent connections, millions if you have WebSockets. ~3k new connections per second. TLS dominates the LB bill; offload to hardware or scale horizontally. Hundreds of backend pods; LB topology is now three layers (global, regional, local).

The big insight from the numbers: the single-LB story breaks at three different scales for three different reasons. Bandwidth breaks first if your responses are large (video, file downloads). TLS CPU breaks first if you have many short-lived clients (mobile apps, IoT). Connection count breaks first if you have long-lived connections (WebSockets, SSE, gRPC streams). Naming which one will break first for *your* workload is what separates a thoughtful answer from a generic one.

</details>

## Step 3: L4 vs L7

Before topology, you need to know what an LB *is*. There are two species and they do different jobs.

Sketch the difference on the whiteboard. What does each see on the wire? What can each decide? When do you pick one over the other?

<details>
<summary><b>Reveal: L4 vs L7 in one table</b></summary>

| Aspect | L4 (transport) | L7 (application) |
|--------|----------------|------------------|
| What it parses | TCP/UDP headers: src/dst IP and port | Full HTTP: method, path, headers, cookies, body |
| Routing decision | By IP and port | By path, host, header, cookie, query string |
| TLS | Pass-through (SNI-aware at best) or terminate | Always terminates if it wants to read paths |
| State per connection | Just the 5-tuple | Per-request state (headers, parsed URL, cookies) |
| Throughput | High; can run in kernel or hardware (DPDK, ASIC) | Lower; userspace HTTP parser is the bottleneck |
| Latency added | Sub-millisecond | 1 to 3 ms |
| When you want it | Raw TCP, databases, internal RPC | Anything internet-facing serving HTTP |
| Examples | AWS NLB, GCP TCP/UDP LB, HAProxy in TCP mode, IPVS, LVS | AWS ALB, GCP HTTPS LB, nginx, HAProxy in HTTP mode, Envoy, Traefik |

The cleanest mental model: L4 is a smart NAT. It rewrites the destination and shovels bytes. It does not know what you sent. L7 is a reverse proxy. It reads your HTTP request, decides where to send it based on what is inside, and may modify the response on the way back.

You specifically need L7 for path-based routing (`/api/*` to one pool, `/static/*` to another), host-based routing (`tenant-a.example.com` to one tenant), cookie-based stickiness, header rewriting (strip auth from logs, inject `X-Forwarded-For`), per-route rate limits, and A/B traffic splitting.

L4 is enough and cheaper for database connection pooling (a pool of read replicas behind one VIP), internal RPC where every caller speaks the same binary protocol, WebSocket gateways (after the upgrade the LB is just shuffling bytes), and DDoS scrubbing where you want maximum packet throughput and minimum parsing.

The common production pattern is L4 at the edge, L7 inside. The edge LB does TLS termination and crude DDoS sponging; the L7 LB inside does path routing and per-service policy. Each does what it is best at.

</details>

## Step 4: the architecture (sketch)

Here is the LB stack at non-trivial scale with five pieces missing. Fill in the `[ ? ]` boxes. Think about what resolves a domain to an IP, what terminates TLS, what holds the list of healthy backends, who decides a backend is unhealthy, and what optional piece handles TLS in hardware.

```
                Client (browser, app)
                       |
                       |  "GET /something"
                       v
                +------------+
                |   [ ? ]    |   (turns a hostname into an IP;
                |            |    can also do basic geo-routing)
                +-----+------+
                      |
                      v
                +------------+
                |   [ ? ]    |   (the LB itself; accepts the connection,
                |            |    terminates TLS, picks a backend)
                +-----+------+
                      |
                +-----+------------+--------------+
                |                  |              |
                v                  v              v
            +-------+         +-------+      +-------+
            |backend|         |backend|      |backend|
            |   A   |         |   B   |      |   C   |
            +-------+         +-------+      +-------+
                                  ^
                                  | periodic pokes ("are you alive?")
                                  |
                          +---------------+
                          |    [ ? ]      |   (decides which backends
                          |               |    are in or out of the pool)
                          +---------------+

           Config:                       Optional:
           +----------------+            +----------------+
           |     [ ? ]      |            |     [ ? ]      |
           |  (the list of  |            |  (offload TLS  |
           |   backends and |            |   handshakes;  |
           |   their state) |            |   often in HW) |
           +----------------+            +----------------+
```

<details>
<summary><b>Reveal: filled-in architecture</b></summary>

```
                Client (browser, app)
                       |
                       |  "GET /something"
                       v
                +------------+
                |    DNS     |   Returns one of several IPs (DNS
                |  resolver  |   round-robin) or a single anycast IP.
                +-----+------+
                      |
                      v
                +------------+
                |    Load    |   Accepts the connection.
                |  Balancer  |   Picks a backend per the algorithm.
                | (nginx,    |   Forwards the request.
                |  HAProxy,  |
                |  Envoy)    |
                +-----+------+
                      |
                +-----+------------+--------------+
                |                  |              |
                v                  v              v
            +-------+         +-------+      +-------+
            |backend|         |backend|      |backend|
            |   A   |         |   B   |      |   C   |
            +-------+         +-------+      +-------+
                                  ^
                                  | GET /healthz every 5s
                                  |
                          +---------------+
                          | Health Checker|   Marks a backend unhealthy
                          |  (in-LB or    |   after N consecutive fails.
                          |   sidecar)    |   Re-enables after M successes.
                          +---------------+

           Config:                       Optional:
           +----------------+            +-----------------+
           | Backend Pool   |            | TLS Terminator  |
           |  Config        |            |  (dedicated     |
           | (static file,  |            |   process or    |
           |  service       |            |   ASIC/SmartNIC |
           |  discovery,    |            |   for high      |
           |  Consul,       |            |   handshake     |
           |  Kubernetes    |            |   rates)        |
           |  endpoints)    |            |                 |
           +----------------+            +-----------------+
```

A quick tour of the pieces.

The DNS resolver is the first "load balancer" most people forget about. Returning multiple A records is round-robin at layer zero. Anycast IPs collapse that into one IP routed by BGP to the nearest LB pool.

The load balancer itself is the box on the diagram. It terminates TLS (if L7), picks a backend, forwards. nginx, HAProxy, Envoy, Traefik in software; AWS ALB/NLB, GCP HTTPS LB, F5 BIG-IP for managed or hardware.

The backend pool config is the list of backends. Old-school: a static file. Modern: service discovery (Kubernetes Endpoints, Consul, etcd) that the LB watches for changes.

The health checker pokes backends actively (calling `/healthz` every N seconds) and also watches real traffic passively (ejecting on elevated 5xx). Most production LBs do both.

The TLS terminator is usually just the same process as the LB. At very high handshake rates (mobile, IoT) you offload to a dedicated TLS tier or a SmartNIC/ASIC.

</details>

## Step 5: load balancing algorithms

A request arrives. The LB has a pool of backends. How does it choose? There are six algorithms worth knowing. Each has a workload it is best for and a failure mode that bites if you pick it for the wrong workload.

For each one, write down how it works, when to use it, and when it goes wrong.

1. Round robin
2. Least connections
3. IP hash
4. Consistent hash
5. Weighted round robin
6. EWMA latency (least response time)

<details>
<summary><b>Reveal: algorithms in detail</b></summary>

**Round robin.** Request 1 to backend A, request 2 to B, request 3 to C, request 4 back to A. Cheapest possible algorithm. Works for stateless backends serving roughly uniform requests. Breaks when request durations vary: a 10-second request and a 10-ms request get distributed the same way, so one slow client wedges every backend in turn. Round-robin also ignores backend load entirely; a backend already pegged keeps getting requests. The random variant has the same drawbacks plus the distribution is only uniform in expectation.

**Least connections.** Track in-flight connections per backend. Send the next request to whoever has the fewest. Variable request durations are handled naturally: a backend stuck on something slow accumulates connections and stops receiving new ones. Breaks with HTTP/2 or gRPC because one TCP carries many concurrent requests, so "connections" stops correlating with load. You need "least active requests" instead, which means parsing HTTP/2 frames. Implementation cost is small (per-backend counter, atomic updates) but cross-LB-instance you cannot share state, so each LB only sees its own slice.

**IP hash.** Hash the client IP, modulo the backend count, same client always lands on the same backend. Poor-man's sticky sessions; no cookie needed. Useful for clients that cannot store cookies (some IoT devices, raw TCP). Breaks on NAT collapse: 10k corporate users behind one NAT all hash to one backend. That backend dies, the LB picks a new mapping, and all 10k clients land on a different backend at once, losing their sessions simultaneously. Mobile clients changing IPs (cell tower hops, WiFi to LTE) get rebalanced and lose stickiness anyway.

**Consistent hash.** Hash the request key (path, user ID from a cookie) into a ring. Each backend owns a slice. Same key always lands on the same backend. Adding or removing a backend redistributes only 1/N of keys. Works for cache pools (repeat lookups for the same key hit the same cache), sharded backends (each owns a partition), any case where "same key to same backend" beats "even distribution." Breaks on hot keys: one user generates 90% of traffic, their backend melts while the others idle. Fix with virtual nodes (each backend appears in many ring positions) and bounded-load consistent hashing (cap any one backend at 1.25x average, spill over to the next).

**Weighted round robin.** Each backend has a weight. Weight 3 gets three requests per round; weight 1 gets one. Works for heterogeneous backends (bigger boxes get more traffic) and canary deploys (weight new version at 1, old at 9, 10% to canary). Breaks because weights are static; they do not reflect actual load. An overloaded weighted backend keeps receiving its share. Pair with passive health checks to eject overloaded backends.

**EWMA latency (least response time).** Track an exponentially weighted moving average of response time per backend. Send the next request to the one with the lowest current EWMA. Works for heterogeneous backends in noisy environments: a backend getting slower (GC pause, hot disk, noisy neighbor) automatically receives less traffic until it recovers. Breaks on the death spiral: a backend that crashes instantly responds with TCP RST, the EWMA looks great (1 ms!), it gets all the traffic, and now nothing works. Pair with health checks that watch error rate, not just latency. Also sensitive to the smoothing constant; too aggressive and the LB oscillates.

The pragmatic default for HTTP services is least connections (counted as least active requests for HTTP/2) plus passive health checks. Robust, simple, failure modes well understood. Pick consistent hash for sharded state. Pick EWMA for wildly variable backend performance.

</details>

## Step 6: health checks and failover

A backend can die in many ways. It can crash (no TCP response). It can hang (TCP accepts, never responds). It can return 500s. It can be alive but serving stale data. It can be alive and fast for `/healthz` but slow for everything else. Your LB has to detect each and react.

Sketch your health-check strategy. Then answer: what happens when the LB itself dies?

<details>
<summary><b>Reveal: health check design and LB failover</b></summary>

**Active health checks.** The LB pokes each backend on a schedule.

```
# nginx-style
upstream api {
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
}

# Envoy-style health check config
health_checks:
  - timeout: 1s
    interval: 5s
    unhealthy_threshold: 3      # 3 consecutive failures to eject
    healthy_threshold: 2        # 2 consecutive successes to re-add
    http_health_check:
      path: "/healthz"
      expected_statuses: [200]
```

A few knobs to think about. **Interval:** too short wastes traffic and flaps backends during deploys; too long means slow detection. 5 to 10 seconds is the usual sweet spot. **Threshold:** three failures before ejection prevents single-hiccup flapping; two successes before re-add prevents flapping on partial recovery. **What `/healthz` returns:** a real one checks the DB, the cache, and whether warmup finished. Returning 200 just because the process is alive is a lie that lets a broken backend keep serving. **Deep vs shallow:** shallow (`return 200`) catches process death. Deep (verify DB) catches more but fails as a group when the DB dies, ejecting every backend at once. Ship both, shallow for the LB's fast poll, deep for monitoring dashboards.

**Passive health checks (outlier detection).** The LB watches real traffic and ejects backends that misbehave.

```
# Envoy outlier detection
outlier_detection:
  consecutive_5xx: 5            # eject after 5 consecutive 5xx
  interval: 10s                 # re-evaluation interval
  base_ejection_time: 30s       # eject for at least 30s
  max_ejection_percent: 50      # never eject more than half the pool
```

`max_ejection_percent` is the line between a degradation and a full outage. Without it, a bad deploy or a downstream-dependency failure flips every backend to 5xx, the LB ejects everyone, and the site is dark. Capping at 50% means you keep sending traffic somewhere even in the worst case. Better a partial outage than a total one.

**What about the LB itself?** One LB is a SPOF. Two patterns fix this.

*Active-passive with a virtual IP.* Two LB instances, one holds the VIP. The standby watches via VRRP or keepalived. On failure, the standby grabs the VIP. Failover in about a second. Same IP, same DNS, no client change. Downside: only one LB does work, the other idles.

*Active-active with anycast or multiple IPs.* N LB instances all serve traffic. DNS returns multiple A records, or BGP anycasts a single IP to all of them. Clients land on whichever is reachable. If one dies, DNS clients retry the next record; anycast clients are silently re-routed by BGP.

For internet-facing services at non-trivial scale, active-active anycast is the default. Active-passive VIP shows up in corporate data centers because it needs less BGP and DNS infrastructure.

**Cascading failure pitfall.** When health checks eject too aggressively you create a thundering herd. Backend A dies, the LB shifts A's traffic to B and C. B and C are now at 1.5x load and start timing out. The LB ejects them too. Now there are no backends, the LB returns 503 to every client, clients retry, the retries overload the next backend that comes up. The system is in a worse state than if A had stayed in the pool serving 5xx.

Fixes: `max_ejection_percent: 50` (never eject everyone), circuit breakers on the backend side (when load exceeds a threshold, return 503 immediately instead of trying to serve and timing out), and client-side retries with backoff and jitter rather than tight loops.

</details>

## Step 7: write the config

You have the architecture and the algorithm. Now write a minimal but production-shaped config for the L7 LB. Two services behind one nginx: `/api/orders/*` to an `orders` pool of 3 backends, everything else to a `web` pool of 5 backends. TLS at the LB. Health checks on `/healthz`. Sticky sessions on `web` (cookie-based), `least_conn` on `orders`.

Take ten minutes. The point is to feel where the config gets noisy and where the defaults bite.

<details>
<summary><b>Reveal: nginx config and where the defaults bite</b></summary>

```nginx
# /etc/nginx/conf.d/api.conf

upstream orders {
    least_conn;
    server 10.0.1.20:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.21:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.22:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

upstream web {
    # Cookie-based stickiness; "route" cookie maps client to server.
    # Free nginx only has ip_hash; sticky cookie is nginx-plus or a third-party module.
    # Shown here as it would appear in nginx-plus or HAProxy.
    sticky cookie srv_id expires=1h path=/;
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.13:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.14:8080 max_fails=3 fail_timeout=30s;
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/ssl/api.crt;
    ssl_certificate_key /etc/ssl/api.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_session_cache   shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets on;

    # Timeouts: defaults are too long for an LB.
    proxy_connect_timeout 2s;
    proxy_send_timeout    10s;
    proxy_read_timeout    30s;

    # Headers the backend needs to know the real client.
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Retry on a different upstream if the first one returns transient errors.
    proxy_next_upstream         error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries   2;
    proxy_next_upstream_timeout 5s;

    # Keepalive to backend.
    proxy_http_version 1.1;
    proxy_set_header Connection "";

    location /api/orders/ {
        proxy_pass http://orders;
    }

    location / {
        proxy_pass http://web;
    }

    location = /healthz {
        access_log off;
        return 200 "ok\n";
    }
}

# Redirect plain HTTP to HTTPS.
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}
```

Where the defaults bite, in roughly the order they will burn you.

`proxy_read_timeout` defaults to 60s. Way too long. A hung backend ties up an nginx worker for a full minute. Set it to your actual P99 plus headroom.

`proxy_connect_timeout` also defaults to 60s. A dead backend that does not RST (just black-holes the SYN) makes the LB wait a minute before failing over. Set to 2s.

`proxy_next_upstream` defaults to `error timeout`, which does not include `http_502 http_503 http_504`. So if a backend returns 503, the LB happily forwards it to the client instead of trying another backend. Almost always you want the retry.

No `proxy_next_upstream_tries` limit means a request that fails on every backend retries through all of them, burning seconds per failed request. Cap at 2 or 3.

No `keepalive` on the upstream means nginx opens a new TCP connection to the backend per request. Adds 1-2 RTT per request and burns ephemeral ports. Always set keepalive.

Default SSL session cache is around 10k sessions. For a busy LB you want 50m or more. Without it, every returning client does a fresh handshake.

`ssl_protocols` defaults vary by nginx version; old defaults include TLS 1.0/1.1, both deprecated. Pin it to `TLSv1.2 TLSv1.3` explicitly.

The config is short. The number of decisions in it is large. Every default that "just works" in dev becomes a production incident at some scale.

</details>

## Follow-up questions

Try each in 2 to 4 sentences before reading the solution.

1. **Sticky sessions and uneven load.** You enable cookie-based stickiness so users keep landing on the same backend (in-memory cart). One backend ends up with the heaviest users. How do you mitigate without losing stickiness?
2. **TLS termination cost.** Your LB CPU is pegged at 80% and you trace it to TLS handshakes. What are your options, in order of cost?
3. **HTTP/2 multiplexing and least connections.** You switch backends to HTTP/2. Suddenly the LB sends all traffic to one backend. Why, and what do you change?
4. **WebSockets.** You add a WebSocket feature. Each user opens one long-lived connection. After deploying, traffic distribution is wildly uneven for hours. What happened?
5. **Slow backend starving the pool.** One backend has a slow disk. Requests to it take 30s instead of 30ms. Round-robin keeps sending it requests; nginx workers pile up. What algorithm or config fixes this?
6. **DNS TTL.** You set DNS TTL to 1 hour and your LB IP changes during an emergency. Clients still hit the old IP for an hour. What is the right TTL, and what is the trade-off?
7. **Cross-region failover.** Your us-east region is down. How does traffic get to eu-west, and how long does it take? Walk through the layers.
8. **Path-based routing for a monolith split.** You are splitting a monolith. `/api/orders/*` should go to the new order-service, everything else stays on the monolith. What changes in the LB, and how do you migrate without breaking clients?
9. **Health check storm.** 200 LB instances each polling 500 backends every 5 seconds = 20k req/s of `/healthz` traffic. How do you reduce this without losing health visibility?
10. **LB dropping connections during a backend deploy.** New backends register before they are ready; old backends are killed mid-request. What is the right deploy sequence?

## Related problems

- **[Distributed Cache (009)](../009-distributed-cache/question.md)** uses the same consistent-hash routing pattern internally. Understanding consistent hashing here makes the cache problem easier.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)** has the LB tier at the center of read scaling. Same algorithms, different workload shape.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md)** puts the LB in front of a write path, where stickiness choices change how partitioning behaves.
