---
id: 16
title: Design a Load Balancer
category: Networking
topics: [l4, l7, health checks, sticky sessions, algorithms, failover]
difficulty: Easy
solution: solution.md
---

## Scene

You sit down for the interview. The interviewer sketches a box on the whiteboard and labels it `backend`. They draw a few stick figures on the left and label them `users`. They draw an arrow from the users to the box.

> *"You have a simple service serving 100 users from one box. There is a single nginx in front. Walk me from there to 1 million users, focusing on the load balancing layer at each step. What does the LB actually do, and why?"*

They sit back. "I do not want a treatise on Kubernetes. I want you to tell me, at each stage, what the load balancer is doing, why it is there, and what breaks when you outgrow it."

This is a concept question disguised as a scaling question. Most candidates skip past load balancers because "everyone just uses nginx" or "AWS has ALB, what is there to design." That is exactly the wrong instinct. The interviewer is testing whether you can explain *what an LB does at the packet level*, when L4 stops being enough, why sticky sessions are a tax not a feature, and what happens when the LB itself becomes the single point of failure.

You are going to walk from one nginx to a global anycast LB stack. At every step name what breaks, then name what fixes it.

## Step 1: clarify before you design

Take 5 minutes. Aim for at least 8 questions. The answers shape whether you need L4 or L7, whether sticky sessions matter, whether geo-distribution is on the table, and whether you can afford a managed cloud LB or have to run your own.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Protocol.** "Is this raw TCP, HTTP/1.1, HTTP/2, gRPC, WebSockets?" L4 is fine for raw TCP. HTTP/2 changes how you count "connections" for least-connections. WebSockets are long-lived and break round-robin completely. gRPC is HTTP/2-on-steroids and behaves like one long connection per client.
2. **TLS termination.** "Does the LB terminate TLS, or does it pass encrypted bytes through to the backend?" Termination at the LB centralizes certs and lets you do L7 routing. Pass-through (SNI routing only) hides traffic from the LB but loses path-based routing.
3. **Sticky sessions.** "Do backends hold per-user state in memory? Or are they stateless?" If stateless, any algorithm works. If they hold sessions (carts, WebSocket state, in-memory auth), you need either sticky routing or a shared session store, and each option has costs.
4. **Geo-distribution.** "Are users in one region, or worldwide? Do we have one data center or many?" One region: a single LB layer is enough. Worldwide: you need DNS-based or anycast routing in front of regional LBs.
5. **Traffic shape.** "What is the request rate, the bandwidth, the concurrent connection count? Is traffic spiky?" An LB that handles 1k req/s on tiny JSON is different from one handling 100 Gbps of video streams.
6. **Backend health.** "How fast do backends fail? How fast can we detect it? Is a 1-minute outage of a backend acceptable, or do we need sub-second failover?" Drives the health check interval and the algorithm choice (least-conn naturally avoids slow backends; round-robin does not).
7. **Cost sensitivity.** "Can we use a managed cloud LB (ALB, GCLB), or do we run our own (nginx, HAProxy, Envoy)?" Managed is cheaper in eng time and more expensive per byte. At a certain scale the bill flips.
8. **Failover for the LB itself.** "If the LB dies, what happens? Is one instance acceptable with a 30-second DNS failover, or do we need active-active LBs behind a virtual IP?" Most candidates forget the LB is a SPOF until pushed.

A strong candidate also names what is out of scope: WAF rules, DDoS mitigation, bot detection. Those often sit alongside the LB but are separate concerns.

</details>

## Step 2: capacity estimates

The interviewer gives you:

- Start: 100 users, ~10 requests/second, single nginx, single backend.
- End: 1 million daily active users, ~30k requests/second peak, ~5 Gbps peak bandwidth, average request 4 KB, average response 60 KB.
- HTTP/1.1 today; HTTP/2 likely later.
- Two regions to start, four eventually.
- Backends are stateless web servers; sessions live in a shared cache.

Compute at each scale:

1. Requests per second (sustained and peak).
2. Concurrent connections.
3. Bandwidth in and out at the LB.
4. New connections per second (TLS handshakes are expensive).
5. How many backend instances you would need behind the LB at each stage.

<details>
<summary><b>Reveal: the math</b></summary>

**Stage 1 (100 users, 10 req/s).**
- Bandwidth: 10 × 60 KB = 600 KB/s ≈ 5 Mbps. Negligible.
- Concurrent connections: tens at most.
- Backends: 1 is enough; nginx is sitting there mostly so you can swap the backend later without changing DNS.

**Stage 2 (10k users, ~500 req/s peak).**
- Bandwidth: 500 × 60 KB = 30 MB/s ≈ 240 Mbps. A single 1 Gbps NIC handles this.
- Concurrent connections: ~2,000 (10k users × ~20% active at any moment × a connection or two each).
- New conns/sec: most browsers keep-alive, so ~50/sec for new sessions. TLS cost is manageable.
- Backends: 3 to 5. One nginx still fine.

**Stage 3 (100k users, ~5k req/s peak).**
- Bandwidth: 5k × 60 KB = 300 MB/s ≈ 2.4 Gbps. Getting close to single-NIC limits.
- Concurrent connections: ~20k. nginx handles this on a beefy box but you want headroom.
- New conns/sec: ~500/sec. TLS termination starts to need a real CPU budget (each handshake is ~1-3ms of CPU on modern x86 with ECDSA certs; 500/sec × 2ms = 1 core busy just on handshakes).
- Backends: 20 to 50, partitioned by service (API, web, auth, search).

**Stage 4 (1M users, ~30k req/s peak, 5 Gbps).**
- Bandwidth: 5 Gbps sustained, 15+ Gbps peak. No single LB instance.
- Concurrent connections: 200k+. WebSocket-heavy workloads can push this to millions.
- New conns/sec: ~3k/sec. TLS at this rate is the dominant LB cost; offload to hardware or scale horizontally.
- Backends: hundreds of pods across services; LB topology is now three layers (global, regional, local).

**Key insight from the math.**

The single-LB story breaks at three different scales for three different reasons:
- **Bandwidth** breaks first if your responses are large (video, file downloads).
- **TLS CPU** breaks first if you have many short-lived clients (mobile apps, IoT).
- **Connection count** breaks first if you have long-lived connections (WebSockets, server-sent events, gRPC).

Naming which one will break first for your workload is what separates a thoughtful answer from a generic one.

</details>

## Step 3: L4 vs L7

Before you design the topology, you have to know what an LB *is*. There are two species, and they do different jobs.

Sketch the difference. What does each see on the wire? What can each do? When do you pick one over the other?

<details>
<summary><b>Reveal: L4 vs L7 in one table</b></summary>

| Aspect | L4 (transport layer) | L7 (application layer) |
|--------|----------------------|------------------------|
| What it parses | TCP/UDP headers: source IP, dest IP, ports | Full HTTP request: method, path, headers, cookies, body |
| Routing decision | By IP and port | By path, host, header, cookie, query string |
| TLS | Pass-through (SNI-aware at best) or terminate | Always terminates if it wants to read paths |
| State per connection | Just the 5-tuple | Per-request state (headers, parsed URL, cookies) |
| Throughput | High; can be done in kernel or hardware (DPDK, ASIC) | Lower; userspace HTTP parser is the bottleneck |
| Latency added | Sub-millisecond | 1 to 3 ms |
| When you want it | Raw TCP services, databases, internal RPC where the client speaks one protocol | Anything internet-facing serving HTTP, where you need path routing, A/B tests, rate-limiting per route |
| Examples | AWS NLB, GCP TCP/UDP LB, HAProxy in TCP mode, IPVS, LVS | AWS ALB, GCP HTTPS LB, nginx, HAProxy in HTTP mode, Envoy, Traefik |

**The cleanest mental model:**

- L4 is **a smart NAT**. It rewrites the destination IP/port and sends bytes along. It does not know "what" you sent.
- L7 is **a reverse proxy**. It accepts your HTTP request, parses it, decides where to send it based on what is inside, and may even modify the response.

**When you specifically need L7:**

- Path-based routing: `/api/*` to one pool, `/static/*` to another.
- Host-based routing: `acme.example.com` to tenant-A pool.
- Cookie-based stickiness: read a session cookie to pick a backend.
- Rewriting requests: strip headers, inject `X-Forwarded-For`, change paths.
- Per-route rate limits: 100 req/s on `/login` per IP, no limit on `/static`.
- A/B testing: 5% of `?variant=new` traffic to the canary pool.

**When L4 is enough (and cheaper):**

- Database load balancing (e.g., a pool of read replicas behind one VIP).
- Internal RPC where all callers speak one binary protocol.
- WebSocket gateways where after the upgrade the LB is just shuffling bytes anyway.
- DDoS-scrubbing tier where you want maximum packet throughput and minimum parsing.

The common production pattern: **L4 at the edge, L7 inside**. The edge LB does TLS termination and crude DDoS sponging; the L7 LB inside does path routing and per-service policy. Each does what it is best at.

</details>

## Step 4: the architecture (sketch)

Here is the load balancing stack at non-trivial scale, with some pieces missing. Fill in the five `[ ? ]` boxes. Hint: think about what resolves a domain to an IP, what terminates TLS, what holds the list of healthy backends, who decides a backend is unhealthy, and what actually talks to the backends.

```
                Client (browser, app)
                       │
                       │  "GET /something"
                       ▼
                ┌────────────┐
                │   [ ? ]    │   (turns a hostname into an IP;
                │            │    can also do basic geo-routing)
                └─────┬──────┘
                      │
                      ▼
                ┌────────────┐
                │   [ ? ]    │   (the LB itself; accepts the connection,
                │            │    terminates TLS, picks a backend)
                └─────┬──────┘
                      │
                ┌─────┴────────────┬──────────────┐
                │                  │              │
                ▼                  ▼              ▼
            ┌───────┐         ┌───────┐      ┌───────┐
            │backend│         │backend│      │backend│
            │   A   │         │   B   │      │   C   │
            └───────┘         └───────┘      └───────┘
                                  ▲
                                  │ periodic pokes ("are you alive?")
                                  │
                          ┌───────────────┐
                          │    [ ? ]      │   (decides which backends
                          │               │    are in or out of the pool)
                          └───────────────┘

           Config:                       Optional:
           ┌────────────────┐            ┌────────────────┐
           │     [ ? ]      │            │     [ ? ]      │
           │  (the list of  │            │  (offload TLS  │
           │   backends and │            │   handshakes;  │
           │   their state) │            │   often in HW) │
           └────────────────┘            └────────────────┘
```

<details>
<summary><b>Reveal: filled-in architecture</b></summary>

```
                Client (browser, app)
                       │
                       │  "GET /something"
                       ▼
                ┌────────────┐
                │    DNS     │   Returns one of several IPs (DNS
                │  resolver  │   round-robin) or a single anycast IP.
                └─────┬──────┘
                      │
                      ▼
                ┌────────────┐
                │    Load    │   Accepts the connection.
                │  Balancer  │   Picks a backend per the algorithm.
                │ (nginx,    │   Forwards the request.
                │  HAProxy,  │
                │  Envoy)    │
                └─────┬──────┘
                      │
                ┌─────┴────────────┬──────────────┐
                │                  │              │
                ▼                  ▼              ▼
            ┌───────┐         ┌───────┐      ┌───────┐
            │backend│         │backend│      │backend│
            │   A   │         │   B   │      │   C   │
            └───────┘         └───────┘      └───────┘
                                  ▲
                                  │ GET /healthz every 5s
                                  │
                          ┌───────────────┐
                          │ Health Checker│   Marks a backend unhealthy
                          │  (in-LB or    │   after N consecutive fails.
                          │   sidecar)    │   Re-enables after M successes.
                          └───────────────┘

           Config:                       Optional:
           ┌────────────────┐            ┌────────────────┐
           │ Backend Pool   │            │ TLS Terminator │
           │  Config        │            │  (dedicated    │
           │ (static file,  │            │   process or   │
           │  service       │            │   ASIC / SmartNIC
           │  discovery,    │            │   for high     │
           │  Consul,       │            │   handshake    │
           │  Kubernetes    │            │   rates)       │
           │  endpoints)    │            │                │
           └────────────────┘            └────────────────┘
```

Pieces and what they do:

- **DNS resolver.** First "load balancer" most people forget about. Returning multiple A records (DNS round-robin) gives you crude balancing at layer 0. Anycast IPs collapse this into one IP routed by BGP to the nearest LB pool.
- **Load Balancer.** The thing on the box. Terminates TLS (if L7), picks a backend per the configured algorithm, forwards. Examples: nginx, HAProxy, Envoy, Traefik for software; AWS ALB/NLB, GCP HTTPS LB, F5 BIG-IP for managed/hardware.
- **Backend pool config.** The list of backends. In old-school setups it is a static config file. In modern setups it is service discovery (Kubernetes Endpoints, Consul, etcd) and the LB watches for changes.
- **Health checker.** Active: LB calls `/healthz` on every backend every N seconds. Passive: LB watches actual request results and ejects on elevated 5xx rate. Most production LBs do both.
- **TLS terminator.** Often the same process as the LB. At very high handshake rates (mobile apps, IoT) you may offload to a dedicated TLS-only tier or to a SmartNIC/ASIC.

</details>

## Step 5: load balancing algorithms

You have a pool of backends. A request arrives. How does the LB choose? There are five algorithms you must know. Each has a workload it is best for and a failure mode that bites if you pick it for the wrong workload.

For each of the algorithms below, write down: how it works, when to use it, when it goes wrong.

1. Round robin
2. Least connections
3. IP hash
4. Consistent hash
5. Weighted round robin
6. EWMA latency (least response time)

<details>
<summary><b>Reveal: algorithms in detail</b></summary>

**1. Round robin.**

Send request 1 to backend A, request 2 to B, request 3 to C, request 4 to A, and so on.

- **When it works.** Stateless backends serving roughly uniform requests. Cheapest possible algorithm.
- **When it breaks.** If one request takes 10 seconds and another takes 10 ms, round-robin spreads slow requests evenly across all backends; one slow client can wedge every backend in turn. Round-robin also ignores backend load entirely; a backend already at 100% CPU keeps getting requests.
- **Variant: random.** Same drawbacks, plus the distribution is only uniform in expectation, not exactly.

**2. Least connections.**

Track in-flight connections per backend. Send the next request to the backend with the fewest.

- **When it works.** Variable request durations. A backend stuck on a slow request accumulates connections and naturally stops receiving new ones.
- **When it breaks.** With HTTP/2 or gRPC, one TCP connection carries many concurrent requests. "Connections" no longer correlates with load. You need "least active requests" instead, which means the LB has to parse HTTP/2 frames.
- **Implementation cost.** The LB has to keep per-backend counters and update them atomically. Cheap, but cross-LB-instance you cannot share state, so each LB only sees its own slice. At low backend count this causes imbalance.

**3. IP hash.**

Hash the client IP. Modulo the backend count. Same client always lands on the same backend.

- **When it works.** Poor-man's sticky sessions. No cookie needed. Useful when the client cannot store cookies (some IoT devices, raw TCP clients).
- **When it breaks.** NAT collapse: 10k corporate users behind one NAT all hash to one backend. That backend dies, the LB picks a new mapping, and *every one of those 10k clients lands on a different backend at once*, losing their sessions simultaneously. Mobile clients changing IPs (cell tower hops, WiFi to LTE) get rebalanced and lose stickiness anyway.

**4. Consistent hash.**

Hash the request key (often path, or user ID from a cookie) into a ring. Each backend owns a contiguous slice of the ring. Same key always lands on the same backend. Adding or removing a backend only redistributes 1/N of the keys.

- **When it works.** Cache pools, where you want repeat lookups for the same key to hit the same cache. Sharded backends, where each backend owns a partition. Any case where "same key → same backend" is more important than "even distribution."
- **When it breaks.** Hot keys: one user generates 90% of traffic; their backend melts while others idle. Solve with virtual nodes (each backend appears in many ring positions) and bounded-load consistent hashing (cap any one backend at 1.25x average; spill over to the next).

**5. Weighted round robin.**

Each backend has a weight. Backend A (weight 3) gets 3 requests per round; backend B (weight 1) gets 1.

- **When it works.** Heterogeneous backends. You added bigger boxes; give them more traffic. Canary deploys: weight the new version at 1, old version at 9; 10% of traffic to the canary.
- **When it breaks.** Weights are static; they do not reflect actual load. A weighted backend that is now overloaded keeps receiving its share. Pair weights with passive health checks to eject overloaded backends.

**6. EWMA latency (least response time / least pending).**

Track an exponentially weighted moving average of response time per backend. Send the next request to the backend with the lowest current EWMA.

- **When it works.** Heterogeneous backends in a noisy environment. Backends that get slower (GC pause, hot disk, bad neighbor on shared hardware) automatically receive less traffic until they recover.
- **When it breaks.** "Death spiral": a backend that crashes instantly responds with TCP RST. The EWMA looks great (1 ms!) and it gets all the traffic. Pair with health checks that watch error rate, not just latency. Also: EWMA is sensitive to the smoothing constant; too aggressive and the LB oscillates.

**The pragmatic default for HTTP services: least connections (with HTTP/2 counted as least active requests) plus passive health checks.** It is robust, simple, and the failure modes are well understood. Pick consistent hash if you have sharded state on the backends. Pick EWMA if your backends have wildly variable performance.

</details>

## Step 6: health checks and failover

A backend can die in many ways. It can crash (no TCP response). It can hang (TCP accepts, never responds). It can return 500s. It can be alive but serving stale data. It can be alive and fast for /healthz but slow for everything else. Your LB has to detect each of these and react.

Sketch your health-check strategy. Then answer: what happens when the LB itself dies?

<details>
<summary><b>Reveal: health check design and LB failover</b></summary>

**Active health checks.**

The LB periodically pokes each backend.

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

Choices to think about:

- **Interval.** Too short = wasted health check traffic and a backend deploying its app gets flapped in and out. Too long = slow detection. 5 to 10 seconds is the usual sweet spot.
- **Threshold.** 3 failures before ejection prevents flapping on a single hiccup. 2 successes before re-add prevents flapping on partial recovery.
- **What `/healthz` returns.** A real `/healthz` checks: can I reach my database? Can I reach my cache? Am I past startup warmup? Returning 200 just because the process is running is a lie that lets a broken backend keep serving.
- **Deep vs shallow.** A shallow check (`return 200`) catches process death. A deep check (verify DB connectivity) catches a broader set of problems but also fails as a group when the DB is down, which can cause the LB to eject every backend at once. Most teams ship both: shallow for the LB's frequent check, deep for monitoring dashboards.

**Passive health checks (outlier detection).**

The LB watches real traffic. If a backend returns elevated 5xx, eject it temporarily.

```
# Envoy outlier detection
outlier_detection:
  consecutive_5xx: 5            # eject after 5 consecutive 5xx
  interval: 10s                 # re-evaluation interval
  base_ejection_time: 30s       # eject for at least 30s
  max_ejection_percent: 50      # never eject more than half the pool
```

The `max_ejection_percent` is critical. Without it, a bad deploy or a downstream-dependency outage flips every backend to 5xx, the LB ejects all of them, and you have a hard outage. Capping at 50% means even in the worst case you still send traffic somewhere, even if it is to a broken backend; better a partial outage than a total one.

**What about the LB itself?**

If the LB is one box, it is a single point of failure. Two patterns to fix this:

1. **Active-passive with a virtual IP (VIP).** Two LB instances; one holds a VIP. The standby watches the active via VRRP or keepalived. On failure, the standby grabs the VIP. Failover in ~1 second. Same IP, same DNS, no client change. The downside: only one LB is doing work at a time; the other sits idle.
2. **Active-active with multiple IPs and DNS or anycast.** N LB instances all serve traffic. DNS returns multiple A records, or BGP anycasts a single IP to all of them. Clients land on whichever is reachable. If one dies, DNS clients retry on the next record; anycast clients are silently re-routed to the nearest survivor by BGP.

For internet-facing services at any non-trivial scale, active-active anycast is the default. Active-passive VIP is common in corporate data centers because it requires less BGP and DNS infrastructure.

**Cascading failure pitfall.**

When health checks eject too aggressively, you can create a thundering herd: backend A dies, the LB shifts all of A's traffic to B and C. B and C are now at 1.5x their previous load and start to time out. The LB ejects them too. Now there are no backends. The LB returns 503 to every client. Clients retry. The retries overload the next backend that comes up. The system is in a worse state than if A had just stayed in the pool serving 5xx.

Fixes:
- `max_ejection_percent: 50` (already mentioned). Never eject everyone.
- Circuit breakers on the backend side: when load exceeds a threshold, immediately return 503 (cheap) instead of trying to serve and timing out (expensive).
- Client-side retries with backoff and jitter, not tight loops.

</details>

## Step 7: write the config

You have the architecture and the algorithm. Now write a minimal but production-shaped config for the L7 LB. Two services behind one nginx: `/api/orders/*` to an `orders` pool of 3 backends, everything else to a `web` pool of 5 backends. TLS terminated at the LB. Health checks on `/healthz`. Sticky sessions for `web` (cookie-based), `least_conn` for `orders`.

Take 10 minutes. The point is to feel where the config gets noisy and where the defaults bite.

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
    # Free nginx only has ip_hash; sticky cookie is nginx-plus or use third-party module.
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
    proxy_next_upstream     error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
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

**Where defaults bite:**

1. **`proxy_read_timeout` defaults to 60s.** Way too long. A hung backend ties up an LB worker for a full minute. Set to your actual P99 + headroom.
2. **`proxy_connect_timeout` defaults to 60s.** A dead backend that does not RST (just black-holes) makes the LB wait a minute before failing over. Set to 2s.
3. **`proxy_next_upstream` defaults to `error timeout`.** Does not include `http_502 http_503 http_504`. So if a backend returns 503, the LB happily forwards it to the client instead of trying another backend. Almost always you want the retry.
4. **No `proxy_next_upstream_tries` limit.** Without it, a request that fails on every backend retries through all of them, potentially seconds of wasted work per failed request. Cap at 2 or 3.
5. **No `keepalive` on the upstream.** Without it, nginx opens a new TCP connection to the backend per request. Adds 1-2 RTT to every request and burns ephemeral ports. Always set keepalive.
6. **Default SSL session cache is tiny.** ~10k sessions. For a busy LB you want 50m or more. Without it, every returning client does a fresh handshake.
7. **`ssl_protocols` defaults vary by nginx version.** Old defaults include TLS 1.0/1.1, which are deprecated. Explicitly set `TLSv1.2 TLSv1.3`.

The config is short. The number of decisions in it is large. Every default that "just works" in the dev environment becomes a production incident at some scale.

</details>

## Follow-up questions

Try each in 2 to 4 sentences before reading the solution.

1. **Sticky sessions and uneven load.** You enable cookie-based stickiness so users keep landing on the same backend (in-memory cart). One backend ends up with the heaviest users. How do you mitigate without losing stickiness?
2. **TLS termination cost.** Your LB CPU is pegged at 80% and you trace it to TLS handshakes. What are your options, in order of cost?
3. **HTTP/2 multiplexing and least connections.** You switch the backends to HTTP/2. Suddenly the LB sends all traffic to one backend. Why, and what do you change?
4. **WebSockets.** You add a WebSocket-based feature. Each user opens one long-lived connection. After deploying, traffic distribution is wildly uneven for hours. What happened?
5. **Slow backend starving the pool.** One backend has a slow disk. Requests to it take 30s instead of 30ms. Round-robin keeps sending it requests; nginx worker threads pile up on it. What algorithm or config fixes this?
6. **DNS TTL.** You set DNS TTL to 1 hour and your LB IP changes during an emergency. Clients still hit the old IP for an hour. What is the right TTL, and what is the trade-off?
7. **Cross-region failover.** Your us-east region is down. How does traffic get to eu-west, and how long does it take? Walk through the layers.
8. **Path-based routing for a monolith split.** You are splitting a monolith. `/api/orders/*` should go to the new order-service, everything else stays on the monolith. What changes in the LB, and how do you migrate without breaking clients?
9. **Health check storm.** You have 200 LB instances each polling 500 backends every 5 seconds. The backends are seeing 20k req/s of `/healthz` traffic. How do you reduce this without losing health visibility?
10. **The LB is dropping connections during a deploy of the backend pool.** New backends register before they are ready; old backends are killed mid-request. What is the right deploy sequence?

## Related problems

- **[Distributed Cache (009)](../009-distributed-cache/question.md)**, which uses the same consistent-hash routing pattern internally. Understanding consistent hashing here makes the cache problem easier.
- **[Read-Heavy System Patterns (017)](../017-read-heavy-patterns/question.md)**, where the LB tier is central to scaling reads. The algorithms here are the same; the workload shape changes.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md)**, where the LB sits in front of a write path and the choice of stickiness affects how the partitioning behaves.
