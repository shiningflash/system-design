## Solution: Design a News Feed (Twitter / Instagram)

### TL;DR

The news feed problem is fundamentally about **fan-out**. Naive "write everywhere" (push) breaks the moment one user has more than ~100k followers; naive "read everywhere" (pull) breaks the moment one user follows more than a few hundred accounts. The right answer is hybrid: push for most users, pull for celebrities, with the threshold tuned by measured system load.

The architecture has two parallel paths feeding one read path. The push path writes post_ids into precomputed per-follower timeline lists in Redis. The pull path serves celebrities by maintaining "recent posts" lists per author that followers query at read time. The Timeline Service merges both, ranks the candidates through an ML scoring service, and returns 50 posts.

The interesting engineering is in the asymmetries: how to detect a "celebrity" dynamically, how to keep timelines fresh without scrubbing 100M rows when a post is deleted, how to keep ranking flexible without precomputing scores, and how to handle the dozen edge cases (blocks, unfollows, new sign-ups) that each have their own subtle behavior.

### 1. Clarifying questions

Covered in question.md. The single most important question is **the heaviest user's follower count**. That number alone determines whether naive push works or you must go hybrid.

### 2. Capacity estimates

Reiterated from the question:

- 5800 posts/sec sustained, 17K peak.
- 35K timeline loads/sec sustained, 100K peak.
- 580K timeline writes/sec under naive push (already painful, before celebrities).
- 15TB for precomputed timelines.

The decisive observation: one celebrity post = 100M writes. Even one per second is too much for fan-out workers. Naive push is impossible.

### 3. API design

**Read timeline:**

```
GET /api/v1/timeline/home?cursor=<opaque>&limit=50
Authorization: Bearer <token>

Response (200):
{
  "posts": [
    { "id": "1234567890", "author": {...}, "content": "...", "created_at": "...", "likes": 42, "media": [...] },
    ...
  ],
  "next_cursor": "<opaque>"
}
```

The `cursor` is opaque. Internally it encodes (last_seen_post_id, last_seen_score, timestamp). Opaque so we can change the pagination scheme without breaking clients.

**Create post:**

```
POST /api/v1/posts
{
  "content": "Hello",
  "media_ids": ["mid1", "mid2"]    # uploaded separately via media service
}

Response (201):
{ "post_id": "1234567890", "created_at": "..." }
```

The 201 returns as soon as the post is durably stored. The fan-out happens asynchronously. The user sees their own post in their own feed immediately (because the client optimistically prepends it), and other users see it within a few seconds.

### 4. Data model

**Posts table** (sharded by post_id):

```sql
CREATE TABLE posts (
    post_id          BIGINT PRIMARY KEY,        -- Snowflake-style: ts + shard + seq
    author_id        BIGINT NOT NULL,
    content          TEXT NOT NULL,
    media_ids        JSONB,
    reply_to_post    BIGINT,
    repost_of_post   BIGINT,
    created_at       TIMESTAMPTZ NOT NULL,
    deleted_at       TIMESTAMPTZ,                -- soft delete
    visibility       SMALLINT NOT NULL DEFAULT 1 -- 1=public, 2=followers, 3=private
);
CREATE INDEX idx_author_created ON posts (author_id, created_at DESC) WHERE deleted_at IS NULL;
```

**Follows table** (sharded by follower_id):

```sql
CREATE TABLE follows (
    follower_id      BIGINT NOT NULL,
    followee_id      BIGINT NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_followee ON follows (followee_id);   -- reverse for fan-out
```

**Why duplicated indexes.** Reading "who do I follow?" is by follower_id (PK). Reading "who follows X?" for fan-out is by followee_id (secondary). The secondary index is colocated in the same shard, so reading it is fast inside the shard but a scatter-gather across shards if you want all of X's followers.

For celebrities, the scatter-gather is unbounded (100M followers split across all shards). Solution: maintain a separate denormalized `followers_by_followee` table sharded by followee_id. Now reading a celebrity's followers is single-shard.

**Timeline store** (Redis, sharded by user_id):

```
Key:   timeline:{user_id}
Value: sorted set of (post_id, score)   # score = posted_at, for sort
Trim:  keep only top 1000 entries per user
```

For cold users (no activity in 30 days), the Redis entry is evicted; the timeline must be rebuilt on next access. We accept this trade-off because cold users are >50% of the user base and we cannot keep 300M timelines hot.

### 5. The core decision: hybrid fan-out

#### Pure push

When user A posts: enqueue N tasks (one per follower). Each worker writes the post_id into one follower's timeline list.

```python
def on_post(post):
    followers = follow_index.get_followers(post.author_id)  # may be huge
    for follower_id in followers:
        timeline_writer.enqueue(follower_id, post.id)
```

This works for small follower counts. For 1M followers, this is 1M task messages on the queue, and 1M timeline writes. At 5800 posts/sec, this is fine for normal users. But:

- One celebrity post = 100M tasks. At 17K posts/sec peak, even just 1% of posts being from celebrities is 170 celebrity posts/sec × 100M = 17B writes/sec. Impossible.

#### Pure pull

When user B opens timeline: read recent posts from every account they follow, merge, rank.

```python
def get_timeline(user_id):
    followees = follow_index.get_followees(user_id)
    candidates = []
    for f in followees:
        candidates.extend(post_index.recent_posts(f, limit=20))
    return rank(candidates)[:50]
```

Read latency is O(followees). For someone following 5000 accounts, this is 5000 fan-in reads. Even at 5ms each, sequential is 25 seconds. Parallel: 5000 concurrent requests against the post index per timeline load. Multiply by 35K timeline loads/sec, millions of fan-in requests per second.

Doesn't work either.

#### Hybrid: push for the long tail, pull for celebrities

```python
CELEBRITY_THRESHOLD = 1_000_000   # tunable

def on_post(post):
    follower_count = follow_index.follower_count(post.author_id)
    if follower_count <= CELEBRITY_THRESHOLD:
        # Push path
        for follower_id in follow_index.get_followers(post.author_id):
            timeline_writer.enqueue(follower_id, post.id)
    # else: celebrity. Do nothing. The pull path picks them up.

def get_timeline(user_id):
    # Push side: precomputed list
    pushed_ids = timeline_store.get_recent(user_id, limit=200)
    
    # Pull side: celebrity authors this user follows
    celeb_authors = follow_index.get_celebrities_followed(user_id)  # cached
    pulled_ids = []
    for author in celeb_authors:
        pulled_ids.extend(author_timeline.recent(author, limit=20))
    
    # Merge and rank
    candidates = pushed_ids + pulled_ids
    return rank_and_pick(candidates, 50)
```

The two paths feed one merge step. Most users have only a handful of celebrity follows, so the pull side is cheap. Most posts are from non-celebrities, so the push side is bounded.

The dynamic threshold: instead of a hard 1M, you measure system load. If fan-out queue depth grows, raise the threshold (more authors treated as celebs). If pull-side latency grows, lower it. The threshold is per-author, set by a background job that looks at follower_count and post_rate.

### 6. Architecture (detailed)

```
                                Client (mobile, web)
                                       │
                                       ▼
                                ┌──────────────┐
                                │  API Gateway │  Auth, rate limit, request shaping
                                └──┬───────────┘
                          read     │      write (post)
                                   │
                  ┌────────────────┴────────────────┐
                  │                                 │
                  ▼                                 ▼
          ┌───────────────┐                ┌────────────────┐
          │   Timeline    │                │     Post       │
          │   Service     │                │    Service     │
          │  (stateless)  │                │  (stateless)   │
          └──┬────────┬───┘                └────┬───────────┘
             │        │                          │
             │        │ pull celebs              │ append-only writes
             │        ▼                          ▼
             │  ┌──────────────┐          ┌──────────────────┐
             │  │ Per-author   │          │  Posts DB         │
             │  │ recent-posts │          │  (Cassandra /     │
             │  │ cache (Redis)│◄─────────┤   Postgres        │
             │  │              │  CDC     │   sharded by      │
             │  └──────────────┘          │   post_id)        │
             │                            └────┬─────────────┘
             │                                 │
             │                                 │ CDC / outbox
             │                                 ▼
             │                          ┌──────────────────┐
             │                          │  posts.created   │  Kafka topic
             │                          │  topic           │
             │                          └────┬─────────────┘
             │                               │
             │                               ▼
             │                          ┌──────────────────┐
             │                          │ Fan-out          │  Consumes posts.created.
             │                          │ Dispatcher       │  Looks up author's
             │                          │ (Kafka consumer) │  follower count.
             │                          └────┬─────────────┘  Celebrity? skip.
             │                               │                Else: emit per-follower
             │                               ▼                tasks.
             │                          ┌──────────────────┐
             │                          │ timeline.write   │  Kafka topic
             │                          │ topic            │
             │                          │ (one msg/follower)│
             │                          └────┬─────────────┘
             │                               │
             ▼                               ▼
   ┌─────────────────┐               ┌──────────────────┐
   │  Timeline Store │◄──────────────│  Fan-out Workers │  Stateless pool.
   │  (Redis sharded │   writes      │  (Kafka consumer)│  Each task: write
   │   by user_id)   │               └──────────────────┘  one post_id into
   │  Sorted sets    │                                     one user's timeline
   └────────┬────────┘                                     sorted set.
            │
            │ trim to top 1000
            │ spill to KV for cold users
            ▼
   ┌─────────────────┐
   │  Cold-timeline  │   Cassandra. Holds older entries
   │  KV store       │   for users not in active hot set.
   └─────────────────┘

   Ranking (called by Timeline Service):
   ┌─────────────────┐
   │  Ranking Service│   Stateless. Calls Feature Store +
   │                 │   ML model. Returns scores for
   │                 │   candidate post_ids.
   └─────────────────┘
```

Why each component:

- **Separate Post Service and Timeline Service.** Different scaling characteristics. Posts are append-only; timelines are read-heavy with hot-spots.
- **Cassandra (or Postgres-sharded) for posts.** Posts are write-once, mostly read by ID, occasionally listed by author. Cassandra wins on write throughput; Postgres-sharded gives stronger consistency. Either works.
- **Redis sorted sets for hot timelines.** Sorted by score (created_at). O(log N) insert. Trim with ZREMRANGEBYRANK to keep top 1000.
- **Cassandra for cold timelines.** When a user goes inactive for 7 days, we move their Redis entry to Cassandra. When they come back, the read populates Redis again. Saves Redis memory.
- **Kafka as the spine.** posts.created → fan-out decisions → timeline.write tasks. Decouples write rate from fan-out worker capacity. If workers fall behind, queue depth grows but no data is lost; we backfill.

### 7. Read and write paths

**Write path (POST /posts)**

1. Client → API GW → Post Service.
2. Post Service validates content, runs through moderation pre-flight (NSFW image check is async, but obvious abuse like banned keywords is sync).
3. Insert into Posts DB. Use a Snowflake-style post_id so it's globally unique without coordination and sortable by time.
4. Insert is durable in the primary shard. Reply 201 to client now (~80ms).
5. CDC stream from Posts DB writes the post event to `posts.created` Kafka topic.
6. Fan-out Dispatcher consumes `posts.created`. Looks up author's follower count from a cached count (refreshed every 5 min from the follows table).
7. If author is non-celebrity:
   - Stream the author's follower list from `followers_by_followee` (cursored).
   - For each follower, emit a small message to `timeline.write` topic: `{follower_id, post_id, score=created_at}`.
   - This emits in batches of 10k followers per Kafka message to amortize overhead.
8. If author is celebrity:
   - Write post_id to the author's "recent posts" cache (the pull-path target).
   - Do not emit fan-out tasks.
9. Fan-out Workers consume `timeline.write`. For each task, ZADD into the follower's Redis timeline sorted set, then ZREMRANGEBYRANK to keep top 1000.

End-to-end latency from post submit to follower's timeline (for non-celeb): 1-5 seconds at steady state. Spikes if fan-out queue depth grows.

**Read path (GET /timeline/home)**

1. Client → API GW → Timeline Service.
2. Timeline Service fetches:
   a. `push_ids`: top 200 post_ids from Redis sorted set for this user. ~5ms.
   b. `celeb_authors`: list of celebrity accounts this user follows. Cached separately, ~2ms.
   c. For each celeb author, fetch their last 20 post_ids from per-author cache. Parallel. ~10ms for 50 celebs.
3. Merge candidates: ~500 post_ids total (200 push + 300 pull from 50 celebs).
4. Hydrate features for ranking. Batch fetch from Feature Store. ~20ms.
5. Send candidates + features to Ranking Service. Returns scores. ~30ms.
6. Apply diversity rules (no two consecutive posts from same author; ensure mix of types).
7. Pick top 50.
8. Hydrate full content: batch fetch from Post Service. ~20ms.
9. Return 50 posts.

Total: ~90ms P50, ~200ms P99. The hydration step dominates.

### 8. Scaling

#### a. Timeline cache

- Sharded by user_id, hash partitioned across ~64 Redis shards.
- Hot users: in Redis. Cold users: spilled to Cassandra.
- Hot/cold determined by last activity within 7 days. Background job rotates.
- Total hot users: estimate 30M (10% of DAU). 30M × 1000 entries × 20 bytes = 600GB. Across 64 shards = 10GB per shard. Fits.

#### b. Posts DB

- Sharded by post_id hash. Snowflake IDs are sortable, but we shard by hash to avoid hot writes on recent IDs.
- 500M posts/day × 365 × 5 years = ~1T posts. At ~500 bytes each = ~500TB. 64 shards × 8TB each. Fits.
- Read replicas in each region. P99 replication lag: 1 second.

#### c. Follows graph

- Sharded by follower_id for the forward lookup.
- Separately sharded by followee_id (denormalized) for fan-out.
- Bidirectional consistency is eventually consistent: a follow is written to forward table first, then async to reverse. A 1-second window where you follow X but X doesn't yet have you in their follower list. This is OK because fan-out tolerates an out-of-date follower list (the next post will catch you up).

#### d. Fan-out workers

- Auto-scale on Kafka consumer lag.
- Workers do ZADD + ZREMRANGEBYRANK in one pipeline. ~1-2ms per task.
- One worker pod handles ~5K timeline writes/sec. We need ~580K/sec sustained → ~120 pods. Peak ~400 pods.

#### e. Hot-key on the read path

Some celebrities are followed by everyone (the "Elon problem" inverted: many users following one celeb means many reads of their recent-posts cache). The same hot-key mitigations as URL shortener apply:

- Read replicas of the celeb cache key.
- In-process LRU on Timeline Service.
- Stale-while-revalidate on celeb caches (60-second TTL with background refresh).

### 9. Reliability

- **Posts DB shard failure.** Posts for that shard are unavailable. Fan-out for posts in that shard is paused (Kafka consumer for that shard stops). When shard recovers, consumer resumes from offset. No data loss.
- **Timeline cache shard failure.** Reads for users in that shard fall through to Cassandra (cold path), slower but correct. Active users see degraded performance until shard recovers.
- **Fan-out worker failure.** No effect; another worker picks up the task. Idempotent: ZADD with the same (member, score) twice is a no-op.
- **Ranking service failure.** Fall back to chronological ordering of the candidate set. Quality drops, but feed still works. This degradation is invisible to most users.
- **Regional failure.** Global LB routes traffic to other regions. The home region's stale write data takes 1-2 minutes to fully replicate. Affected users see a slightly stale feed for that window.

### 10. Observability

| Metric | Why |
|--------|-----|
| `timeline.read.p99` by region | Headline SLO |
| `timeline.candidate_count.p50` | If <100, ranking quality drops |
| `fanout.queue_depth` | Leading indicator of stale timelines |
| `fanout.write_lag_p99` (time from post → timeline write) | Should be under 5s |
| `fanout.celebrity_threshold` | Auto-tuned; alert if extreme |
| `ranking.latency_p99` | Should be under 50ms |
| `cache.hit_rate` (timeline + post + feature stores) | Cascade of cache misses = bad day |
| `post.creation_rate` | Sudden drop indicates auth problems |
| `block.events_per_sec` and resulting cache invalidations | Spikes during attacks |

Alerts:
- Page on: timeline P99 > 500ms for 5min; fan-out lag > 30s; ranking failure rate > 1%.
- Ticket on: celebrity threshold change events (we want to know when the load profile shifts).

### 11. Follow-up answers

**1. User blocks another user.**

Two options:
- **Scrub the precomputed timeline.** Find all entries from the blocked user in the blocker's timeline sorted set, remove them. Cheap for individual blocks (O(1000) lookup in one sorted set), but does nothing for celebrity posts (which are in the pull path).
- **Filter at read time.** Maintain a `blocked_users:{user_id}` set in Redis. On every timeline read, filter candidates against this set. Slightly more expensive per read, but uniformly correct across push and pull paths.

Recommendation: **filter at read time**. The set lookup is O(1) per candidate; for 500 candidates it's 0.5ms. Eliminates the bug class where a block partially-but-not-fully clears history.

For posts the blocker wrote *replying to* the blocked user (visibility chains): also apply the filter to any reply_to_post resolution. Subtle but important.

**2. User unfollows someone.**

The user's precomputed timeline already contains the unfollowed user's posts. Two options:
- **Lazy:** let those entries age out as new posts push them down (within ~1 day for active users).
- **Eager:** scan the timeline and remove entries from that author.

Eager scrub is annoying in Redis sorted sets (you have to read all 1000 entries, find ones from that author, ZREM them). Costs ~10ms per unfollow.

Most products do **lazy** for unfollow. It's not user-visible (the unfollowed person's posts age off the timeline quickly anyway). The same logic applies to "I'm muting" features.

**3. Post deletion when post is in 100M timelines.**

You cannot scrub 100M timeline entries. Instead:

- Mark post as `deleted_at` in Posts DB.
- On read path: when hydrating post_ids → content, skip entries where `deleted_at IS NOT NULL`. The deletion is filtered out invisibly.
- Sorted sets eventually shed deleted entries via the natural trim-to-1000.

This is one of the reasons we hydrate post_ids → content at read time rather than storing content in the timeline cache. Lazy filtering at hydration time is essentially free.

**4. New user signs up and follows 50 accounts.**

Their timeline is empty. Generate it once:

- For each of the 50 followees, fetch their recent 20 posts. Parallel.
- Merge, take top 200, write to the new user's Redis timeline sorted set.

This takes ~100ms. Run it during signup completion as part of the welcome flow. From the user's perspective the timeline "appears" by the time they finish onboarding.

For users who follow celebrities at signup: those go through the pull path on first timeline load (no special handling needed; celebs are never in the precomputed list).

**5. Cold users.**

A user inactive for 30 days. Do we keep precomputing for them?

No. After 7 days of inactivity:
- Stop fan-out writes to that user's Redis sorted set. The dispatcher checks "is this user warm?" and skips cold ones.
- Move the user's Redis sorted set to Cassandra as backup.
- When they return: detect on login, schedule a "rebuild timeline" task that reads from their followees and reconstructs the sorted set.

This saves significant Redis memory and fan-out work. ~50% of follows are to/from cold users at any moment.

**6. Backfill for new follow.**

User A follows user B. Should A's timeline retroactively include B's last few posts?

Twitter says yes (you see ~3 recent posts from a newly-followed account immediately). It feels right; without it the new follow feels broken.

Implementation: on follow, fetch B's last ~10 posts and ZADD them to A's timeline (with appropriate scores). Done in the follow request's response path; takes ~10ms.

Edge case: B is a celebrity. They are not in A's precomputed timeline by design. The first timeline read pulls B's recent posts through the celebrity pull path, so no special handling needed.

**7. Real-time updates (new post while scrolling).**

Two approaches:
- **Pull-to-refresh.** Simple. User pulls down, client fetches `timeline/home?since=<last_seen_score>`.
- **WebSocket push.** Server pushes new post_ids to active clients. More work, but feels modern.

For most apps, **pull-to-refresh** is the right answer. WebSockets are mostly used for real-time notification badges ("3 new posts!") not for the feed itself. The notification badge is a much smaller channel.

**8. Pagination beyond first 50.**

Cursor on (score, post_id). Score = post creation timestamp. Tie-breaking by post_id (deterministic) for posts at exactly the same ms.

`GET /timeline?cursor=<score>:<post_id>&limit=50` returns posts strictly below that cursor.

If the post at the cursor has since been deleted: that's fine; cursor is a position, not a reference. The deletion just means one fewer result in the page.

The deeper problem: pagination beyond ~500 posts deep is rare and we don't promise stable order. If the user scrolls 1000 posts deep, ranking may have shifted in ways the cursor cannot represent. Accept this; nobody scrolls that deep.

**9. One fan-out worker processing 100x normal load.**

Diagnosis steps:
1. **Check if it's a partition/key issue.** Each worker consumes from a Kafka partition. If one partition has hot-key data (one author with disproportionate follower count being in that partition), it gets uneven load. Re-partitioning by `(post_id, follower_id_hash)` rather than just post_id distributes celebs across partitions.
2. **Check for a duplicate worker.** Two pods consuming the same partition due to a Kafka consumer group bug? Look for duplicate writes in the timeline store.
3. **Check the task content.** Is one celebrity below the threshold but with a high follower count (just under 1M)? Their fan-out fills one worker.
4. **Adjust the celebrity threshold.** If a user with 800k followers is in the push path and overwhelming a partition, lower the threshold to put them in pull.

The senior answer goes through all four. The mid-level answer only mentions consumer group rebalancing.

**10. "Posts you might like" injection.**

CEO request: inject recommendations at positions 5, 15, 25 of every feed.

Where: in the Timeline Service, after ranking but before returning. Two new steps:

1. **Candidate generation.** A separate `RecommendedPostsService` returns ~20 candidates per request, based on user interests (computed offline) and recent trending posts.
2. **Injection.** Take the top 50 ranked organic posts, weave in 3 recommendations at fixed positions, return.

Latency cost: one extra ~30ms call to RecommendedPostsService per timeline read. To hide it, parallelize with the existing fetch (don't make it sequential).

Quality risk: recommendations are usually lower-quality than followed accounts. Need to measure if dwell time or engagement drops on injected positions. A/B test before rolling out.

This is also where you can run experiments for advertising: position 5/15/25 are perfect "slots" for sponsored content. Productizing this is half the value of the architecture.

### 12. Trade-offs and what a senior would mention

- **Why not store ranked timelines.** Ranking is on the read path. Reranking precomputed timelines on every model change would be a massive batch job; on read it's a candidate-set scoring (200 items) instead.
- **Why hybrid not adaptive per-request.** Could we decide push vs pull per timeline read instead of per author? Yes, but the cache locality benefit is small and the system complexity is large. Static-per-author (with dynamic threshold) is simpler.
- **Why a separate Posts table and a separate Timelines.** Could we just keep posts in the timeline cache? No: posts are 500 bytes (with content); timeline entries are 20 bytes (just post_id). Storing posts in 100M timelines bloats memory by 25x.
- **What I would revisit at 10x scale.**
  - Federated timeline store: per-region timeline shards with cross-region replication only for celebrities followed across regions.
  - Streaming ranking: continuously update each user's "candidate set" as new posts arrive rather than building it on every read.
  - Move ranking onto edge servers for sub-100ms global P99.

### 13. Common interview mistakes

- **"Just push to every follower."** Doesn't survive five seconds of celebrity math.
- **"Just pull at read time."** Doesn't survive a user who follows 5000 people.
- **No mention of ranking.** Modern feeds are not chronological. If you describe a chronological feed, the interviewer assumes you don't know what feeds look like in 2025.
- **Storing post content in timelines.** Bloats by 25x. Store post_ids and hydrate.
- **Ignoring deletes.** What happens when a post in 100M timelines is deleted? "Lazy filter at hydration" is the answer.
- **No mention of blocks/mutes.** These are real product features that the architecture must accommodate.
- **Sequential reads in fan-in.** If you describe pull as "loop over followees and fetch their posts," you'll get a 30-second latency. Parallel matters.

If you can hit 8 of these, you are interviewing at staff level. Most candidates miss the hot-key, ranking placement, and post-deletion questions.
