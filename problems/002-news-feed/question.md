---
id: 2
title: Design a News Feed (Twitter / Instagram)
category: Social & Feeds
topics: [fan-out, timeline, hot users, ranking, caching]
difficulty: Hard
solution: solution.md
---

## Scene

The interviewer used to work on Twitter's home timeline team. They open with:

> *Design Twitter. Specifically the home timeline: the chronological-ish list of posts from people you follow, plus some "you might like" injection. Walk me through it.*

Then they sit back. The prompt looks like "design a feed," but what they actually want to hear about is fan-out. Miss that and the rest of your 40 minutes drawing boxes doesn't add up to much.

## Step 1: clarify before you design

Take 5 minutes. What do you ask? Don't read further until you have at least six questions written down.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. Scale. "Daily active users? Average follower count? Heaviest user's follower count? Posts per day?" Twitter-like answers: 300M DAU, median 100 followers, max user 100M followers, 500M posts/day. That one number (the heaviest user) tilts the whole design.
2. Timeline semantics. "Strict chronological, or ranked? If ranked, by what signals?" Modern Twitter, Instagram, and Facebook are all ranked. The architecture for a ranked timeline is meaningfully different because there is an ML scoring layer on the read path.
3. Latency target. "What is the P99 for loading the home timeline?" Sub-200ms is the bar for a feed app to feel responsive.
4. Read/write ratio. "Roughly how many timeline loads per post?" Heavily read-skewed. 100:1 read-to-write is typical. This is what justifies precomputed timelines.
5. Freshness. "How fresh must the timeline be? Can a post take 5 seconds to appear in followers' timelines?" The usual answer is "a few seconds is fine, but a user's own post must appear instantly in their own feed."
6. Media in posts. "Are we showing images and video inline, or just text?" If media, there is a separate CDN and thumbnail pipeline. In this round we focus on the feed itself, but I'd mention it.
7. Notifications. "Is the notification system part of this design, or separate?" Separate is the right answer. The feed is the home timeline; notifications are a different stream.
8. Personalization scope. "Just people I follow, or also recommended posts from accounts I don't follow?" Adds an injection step in the read path.

If you only asked "how many users," you missed the most important question: the heaviest user's follower count. That number alone decides whether you can push or have to pull for some users.

</details>

## Step 2: capacity estimates

Inputs from the interviewer:

- 300M DAU
- Median user: 100 followers
- P99 user: 1M followers
- Top user: 100M followers
- 500M posts per day
- Each user loads timeline ~10x/day
- Timeline shows the latest 50 posts

Compute:

1. Posts per second (sustained, peak)
2. Timeline loads per second (sustained, peak)
3. Fan-out volume per second under naive push (every post writes to every follower's timeline)
4. Storage for precomputed timelines, assuming 1000 posts kept per user
5. Why naive push breaks for a celebrity user

<details>
<summary><b>Reveal: the math</b></summary>

Posts per second.
500M / 86400 ≈ 5800 posts/sec sustained. Peak 3x → **17K posts/sec**.

Timeline loads per second.
300M × 10 / 86400 ≈ 35K loads/sec sustained, peak ~100K loads/sec.

Naive fan-out volume.
On average each post fans out to 100 followers. 5800 posts/sec × 100 = **580K timeline writes/sec sustained**, ~2M peak. Doable, but distribution matters.

Storage for precomputed timelines.
300M users × 1000 posts each × 50 bytes per timeline entry (just post_id + author_id + ts) = **15TB**. Sharded, fits.

Where naive push breaks.
One tweet from Elon (100M followers) is 100M writes for a single post. At peak we'd see ~17K of these per second in the worst case, generating 1.7 trillion writes/sec. Even one celebrity post per second is 100M writes for that single event. The fan-out worker queue gets buried, replication lag explodes, the rest of the system suffers.

The insight: the heaviest user breaks naive push. The design has to treat them differently.

</details>

## Step 3: the central design choice. Push, pull, or hybrid

This is the heart of the question. Before drawing anything, decide your approach. Take 10 minutes to think through:

- Push (fan-out on write). When user A posts, copy the post_id into every follower's timeline cache.
- Pull (fan-out on read). When user B opens their timeline, query every account they follow and merge.
- Hybrid. Push for users with < N followers, pull for celebrities.

For each, ask: what is the latency to read? To write? What breaks at scale? And what about a user who follows 5000 accounts?

<details>
<summary><b>Reveal: comparison</b></summary>

| Approach | Read latency | Write cost | Falls apart when |
|----------|--------------|------------|------------------|
| Push | ~10ms (precomputed) | O(followers) per post | A celebrity posts. 100M writes per post crushes the system. |
| Pull | O(following) per read. ~500ms for active users. | O(1) per post (one write). | A heavy follower (someone following 5000 accounts) opens their app. 5000 fan-in queries. |
| Hybrid | ~10ms for normal users; slightly higher for users who follow celebrities. | Bounded fan-out (push only for authors below a threshold). | Boundary effects: how do you decide who is a celebrity? Edge cases around the threshold. |

I'd go hybrid:

- Push for normal users posting (median 100 followers). Cheap.
- Pull for celebrities posting (anyone with > 1M followers). Their posts are not pushed; instead, when a follower opens their timeline, we merge in any recent celebrity posts from accounts they follow.
- The detection threshold is dynamic, not just 1M. You may set it lower for users who tweet frequently (high fan-out × high volume = same load) or higher for users who rarely post.

The exact threshold is tunable. Measure system load and adjust.

</details>

## Step 4: sketch the high-level architecture

Fill in the four `[ ? ]` placeholders. Hint: the timeline service is what serves reads; you need somewhere to store the precomputed timelines; you need a worker pool to do the push fan-out; you need a way to merge in celebrity posts.

```
       Client (mobile, web)
              │
              ▼
       ┌─────────────┐
       │   API GW    │
       └──┬───────┬──┘
          │       │
   read   │       │  write (post)
          │       │
          ▼       ▼
   ┌──────────┐ ┌──────────┐
   │   [ ? ]  │ │   Post   │  (stores posts in canonical store)
   │ (read)   │ │  Service │
   └────┬─────┘ └────┬─────┘
        │            │
        │            ▼
        │       ┌────────────────┐
        │       │   [ ? ]        │  (gets the post and dispatches fan-out)
        │       └────────┬───────┘
        │                │
        ▼                ▼
   ┌──────────┐   ┌──────────┐
   │   [ ? ]  │◄──┤   [ ? ]   │  (workers that write into per-follower timeline lists)
   └──────────┘   └──────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
       Client (mobile, web)
              │
              ▼
       ┌─────────────┐
       │   API GW    │
       └──┬───────┬──┘
          │       │
   read   │       │  write (post)
          │       │
          ▼       ▼
   ┌──────────┐ ┌──────────┐
   │ Timeline │ │   Post   │  Canonical post store.
   │ Service  │ │  Service │  Cassandra or Postgres sharded by post_id.
   │ (reads)  │ └────┬─────┘  Source of truth for post content.
   └────┬─────┘      │
        │            ▼
        │      ┌───────────────┐
        │      │  Fan-out      │  Reads new post from Post Service,
        │      │  Dispatcher   │  decides push vs pull per follower group,
        │      │  (Kafka cons.)│  emits per-follower tasks.
        │      └─────┬─────────┘
        │            │
        │            ▼
        │      ┌──────────────────┐
        │      │  Fan-out         │  Pool of stateless workers.
        │      │  Workers (N pods)│  Each task writes one timeline entry.
        │      └─────┬────────────┘
        │            │
        ▼            ▼
   ┌────────────────────────────┐
   │   Timeline Store (Redis    │  Precomputed per-user list of recent
   │   + spillover to KV / Cass)│  post_ids. Capped at ~1000 per user.
   │   Sharded by user_id.      │  Hot data in Redis, cold in KV.
   └────────────────────────────┘

   On read, Timeline Service does:
     1. Fetch precomputed list of post_ids from Timeline Store.
     2. For each celebrity that this user follows, pull recent posts (separate path).
     3. Merge + rerank.
     4. Hydrate post_ids → full post content via Post Service.
     5. Return.
```

Component responsibilities:

- Post Service. Owns the canonical post record. Returns full content given a post_id. Sharded by post_id hash. Read-replicated.
- Fan-out Dispatcher. Consumes the `posts.created` Kafka topic. Looks up the author's follower count. Under threshold, emit N "push to timeline" tasks. Over threshold, do nothing (the pull path handles celebrities).
- Fan-out Workers. Stateless pool. Each consumes one timeline-write task and writes a post_id into the target follower's timeline list. Auto-scales with queue depth.
- Timeline Store. Per-user sorted list of post_ids. Hot users (active in the last week) in Redis; cold users in Cassandra. Tiered.
- Timeline Service. The read path. Fetches the user's precomputed list, merges in celebrity content from the pull path, hydrates and reranks, returns.

</details>

## Step 5: handling the heavy follower (a user who follows 5000 accounts)

In a push-only world, this user's timeline has 5000 sources writing to it. Fine. In hybrid (push + pull-for-celebs), the same user might follow 50 celebrities. Every timeline load does 50 parallel reads. Is that OK? How do you make it OK?

<details>
<summary><b>Reveal: heavy-follower handling</b></summary>

50 parallel reads per timeline load is OK if:

1. Each read is fast (under 20ms): query a per-author "recent posts" list, which is itself cached.
2. The reads run in parallel, not sequentially: orchestrate them in the Timeline Service.
3. The merged result is cached: if user X loads their timeline at T1 and T1+30 seconds, the second load reuses most of T1's result.

The pattern: each celebrity has a "recent 50 posts" list cached in Redis. Pulling from 50 such lists is 50 hits to the same Redis cluster, served from memory, under 20ms total in parallel. That is much cheaper than push, which would have written 100M entries for that celebrity's last post (including writes for users who will never see that timeline today).

The key insight: pull is expensive per active user but bounded; push is cheap per active user but unbounded for celebrities. Hybrid takes the best of both.

</details>

## Step 6: ranking (timeline is not strict chronological)

Modern timelines are not chronological. Posts are ranked by an ML model: predicted engagement, recency decay, diversity penalties. Where does ranking live in your architecture? Should the precomputed timeline already be ranked, or is ranking on the read path?

<details>
<summary><b>Reveal: ranking placement</b></summary>

Ranking lives on the read path, not in the precompute.

Why:
- The model changes often (the ML team ships new versions weekly). If you ranked at write time, you'd have to recompute every user's stored timeline whenever the model changed.
- Ranking inputs are user-state dependent: how recently the viewer interacted with this author, what the viewer is doing right now (which page came before this load), and so on. You can't capture that at write time.
- Ranking runs on a candidate set of ~200 posts to pick 50. Doing that at read time, after the candidate set is assembled, is bounded and cheap.

Pipeline:
1. Candidate generation: take the top 200 most recent post_ids from the user's timeline list (push path) plus recent celebrity posts (pull path).
2. Feature lookup: fetch features (author engagement signals, viewer history) for the candidates. Heavily cached.
3. Score: send the 200 candidates through the ML scoring service. ~50ms.
4. Pick top 50, apply diversity rules ("no more than 2 posts from the same author in a row"), return.

The score model is a separate service. The Timeline Service doesn't know how it works; it just sends candidates and gets scores back. That separation matters, because the ranking team owns the model lifecycle.

</details>

## Step 7: read the full solution

You have the core: fan-out, hybrid celebrity handling, ranking placement. The solution covers database choices, multi-region replication, the user "block" feature (which complicates fan-out), and the day-2 problems you only discover after launch.

## Follow-up questions

Try answering each in 3 to 4 sentences before reading the solution.

1. A user blocks another user. How do you make sure the blocked user's old posts no longer appear in the blocker's timeline? Delete from the precomputed timeline, or filter at read time?
2. A user unfollows someone. Do you remove that author's posts from their existing timeline immediately, or let them age out naturally?
3. A user deletes a post. It might be sitting in millions of precomputed timelines. How do you handle that, given that scrubbing 100M timeline lists is expensive?
4. A new user signs up and follows 50 accounts immediately. Their timeline is empty until those accounts post. How do you bootstrap it?
5. Cold users. A user who has not opened the app for 30 days has stale timeline data. Do you keep precomputing for them?
6. Backfill for new follow. User A follows user B. Does A's timeline retroactively get B's last few posts?
7. Real-time updates (a new post appears while you're scrolling). Push the new post over WebSocket, or just refresh on pull-to-refresh? Trade-offs.
8. Pagination beyond the first 50. How do you paginate? Cursor on what? What if posts have been deleted since the cursor was issued?
9. You see in observability that one fan-out worker is processing 100x the load of others. Diagnose.
10. Your CEO wants to add "see posts from people you don't follow but might like" injection at position 5, 15, 25 of every feed. Where does this go in the pipeline? What is the latency cost?

## Related problems

- [Chat System (003)](../003-chat-system/question.md). The same delivery and fan-out problem applies to chat (DMs are 1-to-1 fan-out instead of 1-to-many, but the patterns rhyme).
- [Notification System (010)](../010-notification-system/question.md). The notification pipeline shares the fan-out worker pool pattern and the celebrity problem.
- [Distributed Cache (009)](../009-distributed-cache/question.md). The timeline store leans hard on Redis; you should know its limits.
- [Typeahead (005)](../005-typeahead-autocomplete/question.md). Both this problem and search have the "ranked candidate generation" two-stage architecture.
