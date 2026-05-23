---
id: 14
title: Design a Coupon Code Redemption System
category: Concurrency
topics: [uniqueness, single-use, abuse prevention, expiration, atomic operations]
difficulty: Easy
solution: solution.md
---

## Scene

It is the day before Black Friday. Marketing walks into your standup with a slide deck.

> *"Tomorrow at 9am we are dropping BLACKFRI100 for 100 percent off our flagship product. First 1000 customers only. We expect 10,000 users hitting the redeem button in the same second. Oh, and we are also mailing a unique code to each of our 200,000 newsletter subscribers, single use, expires in 30 days. Can you build that today?"*

They smile. The CTO looks at you. You have one afternoon.

This problem looks like a CRUD app. It is not. The interesting bits: how do you give the code to exactly the first N customers and no more, how do you stop the same user from redeeming the same code fifty times, how do you survive Redis dying mid-transaction without handing the code to two people, and how do you stop someone who reverse-engineered your code format from brute-forcing the namespace.

Most candidates jump to "store coupons in a table, mark them used." That answer breaks the moment two requests arrive in the same millisecond. The good design names the race condition first, then fixes it.

## Step 1: clarify before you design

Five minutes. Aim for eight questions. Each one should change the design materially if answered differently.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

The first fork is *single-use or reusable?* One redemption only (a gift card) versus many redemptions up to a cap (a generic promo). Single-use means one row, one redeem, throw away. Reusable means a counter, an atomic decrement, and a per-user dedup story.

The second fork sits inside reusable: *can the same user redeem SAVE10 once a day, once ever, or unlimited?* Without this answer you will let one user drain the whole campaign. It drives whether you need a `(code, user)` dedup table.

*Are codes stackable?* Two coupons in one cart, both percentage discounts, do they compound or sum? Stacking rules live in the validate path, not the redeem path, and they change the API shape.

*Expiration semantics.* Hard expiry on a wall clock, or N hours after issue per user? Timezone? "Midnight expiry" interpreted as PST versus UTC has cost real companies millions.

*Offline redemption.* If an in-store POS goes offline and syncs hours later, you need conflict resolution and a tentative-claim model. If everything is online, you can refuse anything you cannot validate live.

*Cancellation and refund.* If the order is cancelled after the coupon was redeemed, does the code come back to life? Refunds without code-release create angry support tickets; refunds with code-release create a parallel race.

*Generation pattern.* Generic shared codes like SAVE10? Per-user unique codes mailed individually? A pre-generated pool of single-use codes that any user can claim? The three patterns have completely different storage and scaling shapes; pick one or design for all three.

*Abuse model.* Same user trying every guess? Codes leaked to a deal forum? Bots scraping the namespace? Determines whether you need rate limiting, device fingerprinting, allowlists, or all three.

A senior candidate adds two more. *Does the cart page show "this code is valid" before checkout, separate from actually consuming it?* A validate endpoint is standard for good UX but it leaks information to scrapers. And *does finance need to know exactly which codes were used by which users on which orders, for accounting and fraud review?* If yes, you need an immutable redemption log.

</details>

## Step 2: capacity estimates

Marketing gives you the numbers:

- Launch burst: BLACKFRI100 drops at 9am. 10,000 redeem requests in the first second. 1000 win, 9000 must be told "sold out".
- Steady state: ~50,000 redemptions per day across normal promos.
- ~50 active campaigns at any time. Each holds 10k to 10M codes.
- Per-user codes total: ~200M codes across all campaigns, ~5 year retention.
- Validate-to-redeem ratio: ~5 validate calls per redeem. Users paste a code into the cart, bounce, come back.

Compute these on paper before reading the reveal:

1. Peak QPS at launch
2. Steady QPS across the day
3. Validate QPS (steady, with the 5x ratio)
4. Storage for all per-user codes
5. Hot working set size during a campaign launch

<details>
<summary><b>Reveal: the math</b></summary>

**Peak QPS at launch.** 10,000 requests in 1 second = 10,000 QPS spike. This is the design pressure point. It lasts seconds, not hours, but every request must be served correctly: 1000 win, 9000 lose, no double-wins, no skipped slots.

**Steady QPS.** 50,000 / 86400s is about 0.6 redemptions/sec sustained. Peak business hours maybe 5/sec. Trivial.

**Validate QPS.** Five times redeem is ~3/sec sustained, ~25/sec peak. Also trivial. The launch burst applies here too: 50,000 validate calls at 9am because users paste the code in the cart before clicking redeem.

**Storage.** 200M codes at ~200 bytes per record (code, campaign_id, user_id, state, timestamps, index overhead) is ~40GB. Redemption log at ~100 bytes per redemption times 50k/day times 365 times 5 years is ~9GB. Round to 50GB total. Still one machine.

**Hot working set during launch.** One campaign is the hot row. One key. The pool of 1000 codes if you pre-generate, also tiny. Maybe 100KB of actively-hot data.

The system is small. The architecture exists for two reasons: surviving the 10,000 QPS burst correctly on one hot key, and stopping abuse. Database throughput and storage are non-issues; the design pressure is concurrency and integrity.

</details>

## Step 3: code types

Before you draw anything, decide what a "code" looks like. The format dictates storage, lookup speed, brute-force resistance, and how you generate them. Three serious patterns.

| Pattern | Example | Use case |
|---------|---------|----------|
| **Generic shared** | `SAVE10`, `WELCOME20` | Public promos, marketing campaigns. One code, many redemptions, cap on total. |
| **Unique per-user** | `UID-7A2F-9B3C-1D4E` | Newsletter codes, referral rewards, gift cards. One code, one redemption, mailed individually. |
| **Pre-generated pool** | `BLACKFRI-AB7K-2X9P` | "First 1000 get a code" drops. Many codes, each single-use, claimed in order. |

Fill in the comparison below before reading the reveal.

<details>
<summary><b>Reveal: pattern comparison and when to pick which</b></summary>

| Pattern | Storage | Lookup | Brute-force resistance | Race conditions |
|---------|---------|--------|------------------------|----------------|
| **Generic shared** | One row per code. Counter for total redemptions. | O(1) by PK. | Weak. SAVE10 is guessable. Rate limit per user is the only defense. | Atomic decrement on the counter. Per-user dedup via a `(code, user)` unique key. |
| **Unique per-user** | One row per (code, intended_user). | O(1) by PK. | Strong. UID-7A2F-9B3C-1D4E has ~10^14 possible values, infeasible to guess. | One redemption ever, simple `UPDATE WHERE state = 'unused' RETURNING`. |
| **Pre-generated pool** | One row per code, plus an "unclaimed" index. | O(1) by PK to validate. Atomic pop from the pool to claim. | Strong if the code is long and random. | The hard race. Two users hit the same pool at the same instant; only one can claim each code. |

A few format rules carry regardless of pattern. Use 8 to 12 characters of base32, dropping 0/O, 1/I/L, U because humans confuse them. That gives 32^10 or about 10^15 codes, plenty for any campaign. Prefix the code with the campaign tag (`BLACKFRI-`) so validate calls route without a join and mistype detection becomes easier. For human-typed codes, a checksum suffix catches typos before you hit the database; for machine-mailed ones it is dead weight. Always normalize to uppercase so `save10` and `SAVE10` are the same code.

The recommendation: support all three patterns in one schema. Each campaign has a `type` (`shared`, `unique`, `pool`). The data model differs slightly per type but the API surface is the same. We design for this in the solution.

</details>

## Step 4: incomplete architecture diagram

Sketch what the system looks like. Fill in the five `[ ? ]` boxes. The placeholders mark: what stops abuse before it hits your service, what holds the authoritative state, what makes the launch-burst hot path fast, what records what actually happened for finance, and what tells downstream systems "a redemption happened so apply the discount."

```
                Client (web, mobile, POS terminal)
                              │
                              ▼
                    ┌──────────────────┐
                    │   [ ? ]          │  auth, per-user rate limit,
                    │                  │  bot detection
                    └────────┬─────────┘
                             │
        validate path        │       redeem path
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        ┌─────────────┐            ┌─────────────┐
        │ Coupon      │            │ Coupon      │
        │ Service     │            │ Service     │
        │ (read)      │            │ (write)     │
        └──────┬──────┘            └──┬───┬──────┘
               │                      │   │
               ▼                      ▼   │
        ┌─────────────┐         ┌──────────┐
        │  [ ? ]      │         │  [ ? ]   │  single-key atomic ops
        │             │         │          │  for the launch burst
        └─────────────┘         └────┬─────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  [ ? ]       │  source of truth,
                              │              │  campaigns + codes
                              │              │  + redemptions
                              └──────┬───────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  [ ? ]       │  notify cart / checkout
                              │              │  that discount applies
                              └──────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                Client (web, mobile, POS terminal)
                              │
                              ▼
                    ┌──────────────────┐
                    │  API Gateway     │  auth, per-user + per-IP rate
                    │  + Rate Limiter  │  limit, WAF for bot signatures
                    └────────┬─────────┘
                             │
        validate path        │       redeem path
                             │
                ┌────────────┼────────────┐
                │                         │
                ▼                         ▼
        ┌─────────────┐            ┌─────────────┐
        │ Coupon      │            │ Coupon      │
        │ Service     │            │ Service     │
        │ (read)      │            │ (write)     │
        └──────┬──────┘            └──┬───┬──────┘
               │                      │   │
               ▼                      ▼   │
        ┌─────────────┐         ┌──────────────┐
        │  Read Cache │         │  Redis +     │  Atomic claim via Lua
        │  (Redis)    │         │  Lua scripts │  for hot-burst campaigns.
        │  + Bloom    │         │              │  Single-digit ms.
        │  filter     │         └──────┬───────┘
        └─────────────┘                │
                                       ▼
                              ┌──────────────┐
                              │  Postgres    │  Source of truth.
                              │              │  Tables:
                              │              │   campaigns
                              │              │   codes
                              │              │   redemptions
                              │              │   redemption_attempts
                              └──────┬───────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  Kafka       │  Topic: coupon.redeemed
                              │              │  Consumers: cart, finance,
                              │              │  analytics, fraud
                              └──────────────┘
```

The gateway is the first line. Caps brute-force at, say, 10 attempts per user per minute and 30 per IP per minute. Bot patterns (no JS, suspicious UA) get stricter limits. Without this, an attacker enumerates the namespace in hours.

The read service validates a code without consuming it. That is the "apply coupon" button on the cart page. Reads come from cache and fall back to the DB on miss. Response includes the discount shape and any stacking restrictions.

The write service is the redeem path. For low-traffic campaigns it goes straight to Postgres. For hot-burst campaigns it goes through Redis with a Lua script and async-writes the redemption row to Postgres.

The read cache holds campaign metadata (discount type, expiry, stacking rules). A Bloom filter sits in front: if the prefix is not in the filter, return 404 immediately without hitting the DB. Stops scrapers from generating DB load.

The hot-burst Redis holds the counter or claimable pool for active campaigns. The Lua script makes "check, decrement, return" atomic in a single round trip. Sub-millisecond.

Postgres is the source of truth. Campaigns, codes, redemptions, attempts. The redemption table has a unique index on `(campaign_id, user_id)` for per-user single-use enforcement at the DB level even when Redis has already said yes.

Kafka carries `coupon.redeemed`. The cart applies the discount. Fraud correlates with other signals. Analytics aggregates. The coupon service does not care who consumes the event.

</details>

## Step 5: atomic single-use under burst

This is the question the interviewer is actually testing.

> *Ten thousand users hit BLACKFRI100 at the same instant. Only 1000 codes are available. How do you give one to each of the first 1000 without giving two to anyone and without skipping any of the 1000 slots?*

Take 10 minutes. Sketch at least two approaches. Compare them on: correctness under concurrency, latency, what happens if one node dies mid-claim, and how you survive Redis going down.

<details>
<summary><b>Reveal: three viable approaches</b></summary>

**Approach A: Postgres unique index, blocking insert.**

Pre-generate 1000 rows in a `codes` table, each marked `unused`. Redemption is an `UPDATE` with a conditional clause:

```sql
WITH claimed AS (
  UPDATE codes
  SET state = 'used', redeemed_by = $user_id, redeemed_at = NOW()
  WHERE code_id = (
    SELECT code_id FROM codes
    WHERE campaign_id = $campaign AND state = 'unused'
    ORDER BY code_id
    FOR UPDATE SKIP LOCKED
    LIMIT 1
  )
  RETURNING code_id
)
INSERT INTO redemptions (code_id, user_id, redeemed_at)
SELECT code_id, $user_id, NOW() FROM claimed
ON CONFLICT (code_id, user_id) DO NOTHING
RETURNING redemption_id;
```

`FOR UPDATE SKIP LOCKED` is the magic. Each concurrent transaction picks a *different* unused row. The 1001st request finds no unused row and returns nothing.

Per-user uniqueness: a unique index on `redemptions (campaign_id, user_id)` (or `(code_id, user_id)` depending on semantics) catches the case where the same user races against themselves.

Latency: 5 to 20ms per redeem under modest concurrency. Under 10,000 QPS on one row the row-level locking can queue up; you would not run this against a single hot row at burst-scale without something in front.

If Postgres dies after `UPDATE` commits but before the client sees the response, the user retries. The retry's `(campaign_id, user_id)` lookup finds the previous redemption and returns it idempotently. No double-claim.

**Approach B: Redis SETNX (or DECR) for the claim, Postgres for the record.**

The campaign has a counter in Redis: `campaign:blackfri:remaining = 1000`. Redeem does:

```lua
-- Atomic Lua script
local remaining = tonumber(redis.call("GET", KEYS[1]))
if remaining == nil or remaining <= 0 then
  return {err = "exhausted"}
end
local already = redis.call("SISMEMBER", KEYS[2], ARGV[1])
if already == 1 then
  return {err = "already_redeemed"}
end
redis.call("DECR", KEYS[1])
redis.call("SADD", KEYS[2], ARGV[1])
return {ok = redis.call("GET", KEYS[1])}
```

`KEYS[1]` is `campaign:blackfri:remaining`. `KEYS[2]` is `campaign:blackfri:users` (set of users who already redeemed). `ARGV[1]` is the user id. The Lua block is atomic; Redis runs it serially, no race.

Latency: under 1ms typical. Redis is single-threaded so a hot campaign's claims serialize on one core, which is fine for 10k QPS (Redis handles hundreds of thousands of simple ops per second).

After Redis says yes, asynchronously write the redemption to Postgres. The DB unique index `(campaign_id, user_id)` is a backstop: if Redis and Postgres ever disagree (Redis lost state, replay happened), the DB rejects duplicates.

The sad path: Redis crashes after DECR but before the async DB write. You have a counter decrement with no recorded redemption. The user paid nothing because the cart received the success, but no audit row exists. The fix is to persist the redemption synchronously via a small WAL before returning success. Acceptable cost is ~5ms extra.

**Approach C: Token bucket pre-issued.**

Pre-generate 1000 single-use tokens at campaign-creation time. Push them all into a Redis list: `campaign:blackfri:tokens`. Redeem does `LPOP campaign:blackfri:tokens`. If the list is empty, return exhausted. The popped token is the proof of claim.

Then validate the (user, token) at checkout: the user must present this token. Per-user uniqueness handled by tracking which user popped which token in a separate set.

Latency: sub-millisecond. List operations on Redis are O(1).

If Redis loses the list (failover with stale replica), tokens disappear. Mitigation: also store the token list in Postgres so it can be rebuilt. On rebuild, mark tokens that were already popped (you have a record because the redemption hit Postgres) and re-push the unpopped ones.

**Which one to pick.** Approach A alone is fine up to a few hundred QPS on the hot key. Below that threshold you do not need Redis at all. For a flash sale, the standard answer is B with A as backstop: Redis handles the burst, Postgres is the safety net, and the unique index in Postgres makes Redis state recoverable. Approach C is the right pick when the tokens are meaningful artifacts the user actually holds (gift card numbers, lottery tickets). Otherwise the counter approach is simpler.

The senior answer in one breath: "Approach B for the hot path, with the unique index from Approach A as the ground truth. Approach C if marketing wants the codes to be meaningful." One minute of clear reasoning wins the room.

</details>

## Step 6: abuse prevention

Two real scenarios that will happen on launch day.

**Scenario 1.** A user sets up a script that fires 50 redeem attempts per second, trying every code from `SAVE10` to `SAVE99`. Most fail. Some succeed. You see thousands of failures per minute on your dashboard.

**Scenario 2.** Marketing sends BLACKFRI100 to their newsletter at 9am. By 9:05am the code is posted on a deal-aggregation forum. By 9:10am users from outside your newsletter audience are redeeming it. The 1000 codes are gone in 30 seconds. The intended audience complains they did not get a chance.

Sketch defenses for each. Take 5 minutes per scenario.

<details>
<summary><b>Reveal: defenses</b></summary>

**Scenario 1: brute force.** Layer the defenses; no single one is enough.

The cheapest big win is per-user rate limiting. Authenticated users get 10 attempts per minute, 50 per hour. Anonymous gets 5 per minute per IP. Token bucket in Redis, 429 with `Retry-After` after the cap.

Add per-IP rate limiting for unauthenticated traffic. 30 attempts per minute per IP catches session-less requests.

The Bloom filter prefilter is the killer move. If the submitted code is not in the filter of issued codes, return "invalid code" without hitting the DB. Brute-force load never reaches your authoritative store. The filter is in-memory at the service; rebuilt on campaign create. A 0.1% false-positive rate is fine: a small fraction of guesses leak through to a real lookup.

After 5 consecutive failures from the same user or IP, double the cooldown. After 10, ban for an hour. After 20, ban for a day and flag for review. After 3 failed attempts in a minute, throw up a CAPTCHA before the next try. Slows automated tools to a crawl.

Device fingerprint as a signal: same browser fingerprint creating many accounts and each attempting codes is suspicious. Combine with rate limits; do not rely on fingerprint alone because it is bypassable.

**Scenario 2: code leakage beyond intended audience.** The fundamental problem: a shared code that should be private has been disclosed. There is no perfect defense once it has leaked. There are mitigations.

The cheapest big win for leakage is an audience filter at validate time. Codes carry an `audience_filter`: "must be a subscriber as of date X", "must be a customer of segment Y", "must have made a purchase in the last 90 days". Validate fails if the user does not match, even if the code is correct. The leaker can post the code but most leakers' audiences cannot use it.

Better, mail unique per-user codes instead of one shared BLACKFRI100. Even if a code leaks, only that one user's code is burnt. Worth the extra storage.

If you insist on a shared code, cap at a number that survives leakage. 1000 codes vanish in seconds when leaked; 100,000 buys more grace.

Velocity-based shutdown catches both bugs and leakage. If a campaign sees a 100x spike in redemption attempts in 1 minute relative to the expected rate, auto-pause and alert. Marketing reviews and either confirms the spike is legit or kills the campaign before all slots burn.

Honeypot codes are sneaky and cheap. Insert a fake code into the mailing that is not actually valid. Track who tries to use it. If a user redeems a real code *and* the honeypot, they are the leaker or someone downstream of the leaker. With unique-per-user mailings, each code is already a watermark; when it shows up on a forum, you know who leaked it.

The honest answer the interviewer wants: defense is layered. No single mitigation works for both abuse types. The validate-side audience filter is the cheapest big win for leakage; per-user rate limits and the Bloom filter are the cheapest big wins for brute force.

</details>

## Follow-up questions

Try answering each in 2 to 3 sentences before reading the solution.

1. **A user submits BLACKFRI100. Their request times out after Redis decrements the counter but before Postgres records the redemption. They retry. What does your system do?**

2. **The campaign has 1000 codes. After launch, 1003 redemptions are recorded in Postgres. How did this happen? How do you detect it and prevent it?**

3. **Stackable codes.** A cart has SAVE10 (10 percent off) and FREESHIP (free shipping). The user adds BLACKFRI100 (100 percent off). What does your validate endpoint return? Where does the stacking logic live?

4. **Refund flow.** An order with BLACKFRI100 is refunded the next day. Marketing wants the code released back into the pool so someone else can use it. Engineering hates this. What is the right answer?

5. **Expiration in the wrong time zone.** A code expires at "midnight on Dec 31". The user is in Tokyo. The code was issued in PST. What does the user see, what does the API return, and how do you avoid being yelled at on Twitter?

6. **A campaign has 10 million per-user codes pre-generated. Marketing realizes the discount amount is wrong. They want to update all 10M without invalidating any already-redeemed ones. Can you?**

7. **Multi-region deployment.** Your e-commerce site has US and EU regions. A US-issued code is redeemed against the EU site. How do you guarantee single-use across regions?

8. **Bloom filter says "code not present" but the code is actually present (false negative).** Wait, Bloom filters do not have false negatives. Explain why, and what kind of error they *do* have, and how that affects this design.

9. **Reusable code SAVE10 has been used 9999 times. Limit is 10000. Twenty users hit redeem simultaneously. How do you give it to exactly one of them and tell the other nineteen "limit reached"?**

10. **A code was generated, mailed to a user, but the user never redeems it before expiry. After expiry, can you reuse that code string for a new campaign? Why or why not?**

## Related problems

- **[Approval Management (011)](../011-approval-management/question.md)**. The audit trail, immutable record-keeping, and state machine patterns apply directly to the redemption log here.
- **[Shopping Cart (012)](../012-shopping-cart/question.md)**. The cart consumes coupon.redeemed events and applies discounts. Cart idempotency is the other side of redemption idempotency.
- **[Rate Limiter (004)](../004-rate-limiter/question.md)**. The per-user and per-IP rate limits in Step 6 are the standard algorithms. Pick one with intent.
- **[Distributed Cache (009)](../009-distributed-cache/question.md)**. The Redis layer here is the same caching layer; understand its eviction and replication story before depending on it for hot-burst correctness.
