---
id: 15
title: Design a Comment System (Threaded + Voted + Moderated)
category: User Content
topics: [nested data, voting, moderation, soft delete, ranking]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer puts down their coffee and says:

> *"We run a news site. Every article has a comment section. People reply to each other, they upvote and downvote, mods remove spam, sometimes a comment goes viral and gets 5000 replies. Design the comment system. Think Disqus, HackerNews, Reddit-style."*

This looks like a CRUD problem. It is not. Comments are the smallest possible piece of user content but they have every hard property packed in: nested data (a comment that replies to a reply to a reply), heavy reads with bursty writes (everyone refreshes when something gets popular), voting with hot-row contention, soft-delete that has to preserve thread structure, ranking algorithms beyond simple chronological order, and a moderation pipeline that has to keep up with abuse.

Candidates who jump to "comments table with parent_id, done" miss every interesting question. The interesting questions are: how do you fetch a 5000-node thread without 5000 database round-trips, how do you count 1000 upvotes in 5 seconds without a hot row, how do you delete a comment that has 200 replies under it without orphaning them, and how does the moderation pipeline decide what is spam without humans reviewing everything.

We walk this from a 10-article blog to a viral site with 1M comments per day. At each stage something breaks.

## Step 1: clarify before you design

Take 5 to 7 minutes. Eight questions minimum.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Depth limit.** "Can a reply to a reply to a reply go on forever, or is there a cap?" Reddit caps at ~10 visible levels. HackerNews has no hard cap but indents stop after 8. Without a cap, a recursive tree can produce huge fetch latency and abusive nesting (one user replying to themselves 200 times deep).
2. **Edit history.** "Can users edit their comments? Forever or only for a window? Do we keep the history?" 5-minute edit window is the common pattern. Permanent editability with no history breaks accountability ("that's not what they said"). HackerNews lets you edit for ~2 hours.
3. **Voting model.** "Upvote-only (like Facebook reactions) or up + down? Anonymous votes or named? Can a user change their vote?" Down-votes change moderation incentives and require dedup logic.
4. **Sort orders.** "What sort orders does the UI need? Newest, oldest, top, controversial, 'best'?" Each one has different storage and caching needs. "Best" (Reddit's confidence-bound formula) is expensive to compute live.
5. **Moderation model.** "Auto-detect spam? Human moderators? User reports? All three? Pre-publish review or post-publish takedown?" Pre-publish (every comment waits for a mod) is a different system from post-publish (every comment is live immediately, mods react).
6. **Delete semantics.** "When a user deletes a comment with 50 replies, what happens? Tombstone? Hard delete? Children orphaned?" Almost always: soft delete with tombstone so the thread structure survives.
7. **Read vs write ratio.** "How many reads per write? Are comments cached?" News article comments are read 100x to 1000x more than they are written. Caching the rendered tree dominates the architecture.
8. **Real-time updates.** "When someone posts a new reply, does my screen update live, or only on refresh?" Real-time (WebSocket fan-out) is a different design from refresh-driven (poll).
9. **Auth model.** "Anonymous comments allowed? Pseudonymous? Required login?" Anonymous changes the abuse story dramatically.
10. **Notifications.** "When someone replies to me, do I get notified?" This is usually a separate service consuming events, not built into the comment system itself.

If you walk in asking only "how many comments per day" you have lost the interesting design space. Depth limit, sort order, and moderation model are the three that change the architecture the most.

</details>

## Step 2: capacity estimates

The interviewer hands you two scenarios, a small blog and a viral site.

**Small blog:**
- 10 articles published per day
- 100 comments per day total across all articles
- Each article live for ~7 days of meaningful traffic
- Average comment: 200 bytes

**Viral site:**
- 100,000 articles in the active set
- 1M comments per day across all articles
- Top 1% of articles get 80% of comments (so ~1000 articles see 800k comments per day)
- Each comment: 300 bytes average, but some long-form replies hit 5KB
- Read-to-write ratio: 1000:1

Compute for the viral site:

1. Writes per second sustained and peak
2. Reads per second sustained
3. Storage for one year of comments
4. Working set: hottest 1000 articles, how much data
5. Peak comments per article per second on the single hottest article in a viral moment

<details>
<summary><b>Reveal: the math at both scales</b></summary>

**Small blog:**

- 100 comments/day = ~1 comment per 15 minutes. Trivial.
- Reads at 1000:1 = 100k page loads per day = ~1 per second. A laptop handles this.
- Storage: 100 × 365 × 200 bytes = ~7MB per year. Nothing.
- Total system: one Postgres, no cache, single app instance. You ship it in a weekend.

**Viral site:**

- 1M comments/day / 86400 = ~12 writes/sec sustained. Peak ~3 to 5x: ~40 to 60 writes/sec.
- Reads at 1000:1 = 1B reads/day = ~12,000 reads/sec sustained, peak ~40k reads/sec.
- Storage: 1M × 365 × 300 bytes = ~110GB/year for comments alone. Plus votes, flags, edits: round to ~250GB/year.
- Hottest 1000 articles, 800 comments each on average over their lifetime = 800k comments × 300 bytes = ~240MB. Fits in Redis easily.
- A single viral moment: one article gets posted to the front page, 10k concurrent users, 50 comments per second on that one article, 1000 votes per second. This is the hot-spot you must handle.

**Key insights:**

- Steady-state load is small. The system is dominated by **read amplification** (1000:1) and by **burst behavior** (viral moments produce 100x sustained load on a single row).
- Storage is small enough that you do not need to think about sharding for capacity. You think about sharding for *blast radius* and to isolate hot articles from cold ones.
- The single hardest number: **40k reads/sec at peak** for one article's comment tree. A naive recursive query against Postgres for every page load would crater the DB. The cache layer is mandatory.
- Vote throughput on a hot comment can hit 1000 votes/second. That is a hot-row problem at the DB level.

</details>

## Step 3: storing nested data

This is the design decision that frames the entire system. You have a tree of comments. How do you store it in a relational database such that you can:

- Insert a new comment cheaply (the hot write path)
- Fetch an entire thread for an article cheaply (the hot read path)
- Delete a subtree cheaply (rare but happens)
- Move a subtree (very rare, almost never needed for comments)

Four classical approaches. Before peeking, sketch the pros and cons of each in your own words.

<details>
<summary><b>Reveal: the four approaches compared</b></summary>

**1. Adjacency list (parent_id pointer).**

```sql
CREATE TABLE comments (
    comment_id  BIGINT PRIMARY KEY,
    article_id  BIGINT NOT NULL,
    parent_id   BIGINT,                    -- NULL = top-level
    body        TEXT,
    created_at  TIMESTAMPTZ
);
CREATE INDEX idx_parent ON comments(parent_id);
CREATE INDEX idx_article ON comments(article_id);
```

- **Insert:** trivial. `INSERT INTO comments (..., parent_id) VALUES (..., 42)`.
- **Fetch subtree:** recursive CTE (`WITH RECURSIVE`). Postgres handles it but the query plan is a loop with a join per level. For a thread with 5000 comments and depth 8, this is 8 joins. Workable but not free.
- **Move subtree:** trivial. Update one parent_id.
- **The classic problem:** "give me the full thread for article X" is a recursive query, not a simple `WHERE article_id = X`.

**2. Materialized path.**

```sql
CREATE TABLE comments (
    comment_id  BIGINT PRIMARY KEY,
    article_id  BIGINT NOT NULL,
    path        TEXT NOT NULL,             -- "/123/456/789" = ancestors
    depth       INT NOT NULL,
    body        TEXT,
    created_at  TIMESTAMPTZ
);
CREATE INDEX idx_path ON comments(article_id, path);
```

- The `path` column encodes the full ancestor chain. A reply has path = parent's path + "/" + self.id.
- **Insert:** mostly trivial. You need the parent's path before you can compute your own. One extra SELECT or one cleverly written INSERT.
- **Fetch subtree:** `WHERE article_id = X AND path LIKE '/123/%'` is fast with a prefix index. One query, no recursion.
- **Fetch entire article's tree:** `WHERE article_id = X` (any path). One query.
- **Move subtree:** expensive. You have to rewrite the path of every descendant. Rare for comments so this is fine.
- **Sort children of a node:** `WHERE path LIKE '/123/%' AND depth = parent.depth + 1` plus an ORDER BY.

**3. Nested set (left/right values).**

```sql
CREATE TABLE comments (
    comment_id  BIGINT PRIMARY KEY,
    article_id  BIGINT NOT NULL,
    lft         INT NOT NULL,
    rgt         INT NOT NULL,
    body        TEXT,
    created_at  TIMESTAMPTZ
);
```

- Each node gets a left and right number such that descendants have left/right values between the parent's.
- **Fetch subtree:** very fast. `WHERE lft BETWEEN parent.lft AND parent.rgt`.
- **Insert:** brutally expensive at scale. Inserting a comment requires updating the lft/rgt of every comment to its "right" in the tree, which can be thousands of rows. Concurrent inserts contend on every row.
- Used in academic examples; almost no production comment system picks this. Comment workloads are insert-heavy and nested set is the wrong shape.

**4. Closure table.**

```sql
CREATE TABLE comments (
    comment_id  BIGINT PRIMARY KEY,
    article_id  BIGINT NOT NULL,
    body        TEXT,
    created_at  TIMESTAMPTZ
);

CREATE TABLE comment_ancestry (
    ancestor_id   BIGINT NOT NULL,
    descendant_id BIGINT NOT NULL,
    depth         INT NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id)
);
```

- Separate table holds every ancestor-descendant pair for every node.
- **Fetch subtree:** `WHERE ancestor_id = X` on the ancestry table, join back to comments.
- **Insert:** must insert one ancestry row per ancestor. A comment at depth 5 produces 5 ancestry rows.
- **Storage:** O(total tree depth) extra rows. For a 5000-comment thread averaging depth 4, that's 20,000 extra rows.
- **Flexible.** Closure tables make moves easy and let you ask "how deep is the subtree" cheaply.
- Used by Stack Overflow internally for some hierarchies. Solid but heavy on write amplification.

**Comparison table:**

| Approach | Insert | Fetch subtree | Fetch full article | Move | Storage overhead |
|----------|--------|---------------|--------------------|------|------------------|
| Adjacency list | Cheap | Recursive CTE | Single query, then assemble in app | Cheap | None |
| Materialized path | Cheap (one extra SELECT) | Single query, prefix scan | Single query | Expensive | One string per row |
| Nested set | Very expensive | Single range query | Single query | Very expensive | None |
| Closure table | O(depth) writes | Single join | Single join | Cheap to moderate | One row per ancestor pair |

**Recommendation for a comment system: adjacency list + materialized path hybrid.**

- Keep `parent_id` (adjacency list) because it is the natural shape for inserts and for "show me the parent of this comment."
- Also keep `path` (materialized path) for the hot read query "give me everything under this subtree" without recursion.
- Both are derived from each other so they cannot drift if you populate `path` from `parent_id` on insert.

The redundancy is cheap. You eat one TEXT column per row (~50 bytes average for paths under 10 levels) in exchange for never running a recursive CTE on the hot read path.

</details>

## Step 4: sketch the architecture

Fill in the placeholders. Each marks a major component.

```
                Client (web, mobile, embed iframe)
                              │
                              ▼
                    ┌──────────────────┐
                    │   [ ? ]          │  (auth, rate limit, abuse heuristics)
                    └────────┬─────────┘
                             │
        post comment / vote  │     load comments
                             │
              ┌──────────────┼──────────────┐
              │                             │
              ▼                             ▼
        ┌─────────────┐               ┌──────────────┐
        │  [ ? ]      │               │   [ ? ]      │  (rendered comment
        │ (validates, │               │              │   trees, hot articles
        │  persists,  │               │              │   served from memory)
        │  emits      │               │              │
        │  events)    │               │              │
        └──┬─────┬────┘               └──────┬───────┘
           │     │                           │
           │     │                           │
           │     ▼                           ▼
           │  ┌──────────────────┐    ┌──────────────────┐
           │  │  [ ? ]           │    │  Read Replica    │
           │  │ (vote counts     │    │  (Postgres)      │
           │  │  in-memory,      │    │                  │
           │  │  flushed to DB)  │    └──────────────────┘
           │  └──────────────────┘
           │
           ▼
        ┌──────────────────┐
        │  Comments DB     │  (source of truth: comments,
        │  (Postgres)      │   votes, flags, edits, moderation)
        └────────┬─────────┘
                 │
                 │ async (Kafka)
                 ▼
        ┌──────────────────┐
        │  [ ? ]           │  (spam/toxicity classifier,
        │                  │   user report queue, manual
        │                  │   moderation tools)
        └──────────────────┘
```

<details>
<summary><b>Reveal: complete architecture</b></summary>

```
                Client (web, mobile, embed iframe)
                              │
                              ▼
                    ┌──────────────────┐
                    │   API Gateway    │  (auth, per-user rate limit,
                    │   + WAF          │   simple bot detection)
                    └────────┬─────────┘
                             │
        post comment / vote  │     load comments
                             │
              ┌──────────────┼──────────────┐
              │                             │
              ▼                             ▼
        ┌─────────────┐               ┌──────────────┐
        │  Comment    │               │  Read Cache  │
        │  Service    │               │  (Redis +    │
        │ (validates, │               │   CDN edge)  │
        │  persists,  │               │  rendered    │
        │  emits      │               │  trees per   │
        │  events)    │               │  article     │
        └──┬─────┬────┘               └──────┬───────┘
           │     │                           │
           │     │                           │
           │     ▼                           ▼
           │  ┌──────────────────┐    ┌──────────────────┐
           │  │  Vote Aggregator │    │  Read Replica    │
           │  │  (Redis INCR,    │    │  (Postgres)      │
           │  │  batch-flush to  │    │                  │
           │  │  DB every 5s)    │    └──────────────────┘
           │  └────────┬─────────┘
           │           │
           ▼           ▼
        ┌──────────────────┐
        │  Comments DB     │  (source of truth: comments,
        │  (Postgres,      │   votes, flags, edits,
        │   primary)       │   moderation_queue)
        └────────┬─────────┘
                 │
                 │ async (Kafka topic: comment.events)
                 ▼
        ┌──────────────────┐
        │  Moderation      │  (auto: profanity + ML toxicity +
        │  Pipeline        │   link detector. Manual: human queue.
        │                  │   Reactive: user report intake.)
        └──────────────────┘

        Other consumers of the event stream:
            - Notification service ("someone replied to you")
            - Analytics ("which articles drive engagement")
            - Search index (Elasticsearch for comment search)
```

Component responsibilities:

- **API Gateway + WAF.** First defense. Per-user rate limit (10 comments per minute, 100 votes per minute). Drop obvious bots (UA strings, missing headers, known bad IPs).
- **Comment Service.** Stateless. Validates body length, runs synchronous spam pre-check against bloom filter, writes to Postgres, emits `comment.created` to Kafka. Owns the write path for comments, votes, flags.
- **Read Cache.** Rendered comment trees per article, keyed by `(article_id, sort_order)`. Invalidated on writes that affect that article. CDN sits in front for popular articles.
- **Vote Aggregator.** Critical for hot comments. Votes hit Redis (atomic INCR) immediately, return success to the user. A background job batches the deltas and flushes to Postgres every 5 seconds.
- **Comments DB.** Source of truth. Sharded by `article_id` once you outgrow a single primary.
- **Read replica.** Serves the comment tree for cache misses. Multiple replicas; reads do not block on writes.
- **Moderation pipeline.** Async. Consumes every new comment from Kafka. Runs classifiers; based on confidence, auto-approves, queues for human review, or auto-removes. User-reported comments enter the same queue with higher priority.

</details>

## Step 5: vote aggregation under contention

A comment goes viral. 1000 users upvote it within 5 seconds. Your naive design is:

```sql
UPDATE comments SET score = score + 1 WHERE comment_id = 42;
INSERT INTO votes (user_id, comment_id, value) VALUES (?, 42, 1);
```

What happens? Why is this a problem? Design something that handles 1000 votes/sec on a single comment without the database melting.

<details>
<summary><b>Reveal: hot-row problem and the fix</b></summary>

**What happens with the naive design:**

Every UPDATE on the same row takes a row-level lock. The 1000 concurrent UPDATE statements serialize behind that lock. Postgres handles them, but:

- Each UPDATE waits for the previous one. Latency on every voter's request becomes the queue depth × per-update time.
- The transaction log fills with 1000 row-version entries for the same row.
- Other reads against that comment are also waiting on the lock if you use `SELECT FOR UPDATE` anywhere.
- Replication lag grows because the primary is busy.
- The DB CPU spikes for one row, while everything else slows down.

This is the **hot row problem**. The DB is doing work proportional to vote volume on a single key.

**The fix has three parts:**

**1. Decouple vote write from score update.**

The user's click does not need to update the DB row directly. It needs:
- Their vote recorded (so we can dedup).
- The score eventually correct.

Path:
- `POST /comments/42/vote {value: 1}` → Vote Aggregator.
- Aggregator: `INCRBY vote_delta:42 1` in Redis (constant-time, in-memory).
- Aggregator: writes the per-user vote to a `votes_pending` Redis hash (`HSET user:7:votes 42 1`) for dedup.
- Returns 200 to the user in ~5ms.

**2. Batch-flush to the database.**

A background worker runs every 5 seconds:
- Reads `vote_delta:*` keys.
- For each comment_id: `UPDATE comments SET score = score + <delta> WHERE comment_id = ?`. One UPDATE per comment, not per vote.
- Drains `votes_pending` and bulk-inserts into the `votes` table.
- Resets the delta keys.

1000 votes in 5 seconds become **1 DB update** instead of 1000. The hot row sees one write every 5 seconds.

**3. Dedup at the Redis layer.**

A user clicks Upvote twice. Or changes from Up to Down. The Redis hash holds the user's current vote on each comment. On submit:
- `HGET user:7:votes 42` to see existing value.
- Compute delta: if existing was 0 and new is 1, delta = +1. If existing was -1 and new is 1, delta = +2.
- `HSET user:7:votes 42 1` to record current state.
- `INCRBY vote_delta:42 <delta>`.

The delta path is what flushes to the DB. The user-vote map is what prevents double-counting.

**Trade-offs:**

- **Eventual consistency on score display.** The displayed score may lag the actual count by up to 5 seconds. Acceptable for a comment system; most users do not notice.
- **Data loss window if Redis dies.** Up to 5 seconds of votes could be lost. Mitigations: Redis AOF + replica, or write-ahead a vote-event to Kafka before Redis (then the worker reads from Kafka, not Redis, and Kafka is durable). For a comment system the 5s window is usually fine.
- **Score never goes negative more than briefly.** If a comment gets mass-downvoted and the delta flush is delayed, the cached score might show old value. Same trade-off as above.

This pattern is the standard answer for "high write rate on a single counter." It works for vote counts, view counts, like counts, anything that aggregates.

</details>

## Step 6: moderation pipeline

Comments get spam, NSFW content, hate speech, off-topic noise. The interviewer asks: "Walk me through how a comment gets moderated, from posted to either visible-to-everyone or removed."

Three input signals: automated detection at post time, user reports after the fact, manual scanning by mods. Sketch the pipeline and the states a comment moves through.

<details>
<summary><b>Reveal: moderation pipeline</b></summary>

**States a comment can be in:**

| State | Visible to | How it got here |
|-------|-----------|-----------------|
| `published` | Everyone | Posted and passed auto-checks |
| `pending_review` | Author only | Auto-checker flagged with medium confidence |
| `shadow_banned` | Author only | Author was flagged as bad actor; comment looks live to them but no one else |
| `removed_auto` | No one (tombstone for thread structure) | Auto-checker high-confidence spam/abuse |
| `removed_manual` | No one | Human mod removed |
| `removed_self` | No one | Author deleted |

**The pipeline:**

```
                    POST /comments
                          │
                          ▼
              ┌─────────────────────────┐
              │  Synchronous pre-check  │
              │  (must finish in <50ms) │
              │  - Length / format      │
              │  - URL allowlist        │
              │  - Bloom filter of      │
              │    banned phrases       │
              │  - Author reputation    │
              └────────────┬────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        clean │     suspect│            │ banned-phrase
              ▼            ▼            ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ published│  │ pending_ │  │ removed_ │
        │          │  │ review   │  │ auto     │
        └────┬─────┘  └────┬─────┘  └──────────┘
             │             │
             │             │
             │             ▼
             │       ┌──────────────────┐
             │       │  Human moderator │
             │       │  queue. Mod      │
             │       │  approves or     │
             │       │  removes.        │
             │       └──────────────────┘
             │
             └──────► Async: emit comment.created to Kafka
                              │
              ┌───────────────┼──────────────┐
              │               │              │
              ▼               ▼              ▼
        ┌──────────┐  ┌──────────────┐  ┌──────────────┐
        │ ML       │  │ Link scanner │  │ Notification │
        │ toxicity │  │ (phishing,   │  │ ("X replied  │
        │ classifier│ │  malware)    │  │  to you")    │
        └────┬─────┘  └──────┬───────┘  └──────────────┘
             │               │
             │ high confidence flag
             ▼               ▼
        ┌──────────────────────────────┐
        │  Mark comment for review     │
        │  (state → pending_review or  │
        │   removed_auto)              │
        └──────────────────────────────┘

        User-reported comments enter the human queue directly,
        sorted by report count + author reputation + content score.
```

**Synchronous pre-check is the only thing on the hot path.** It must finish in ~50ms because it blocks the user's POST. So it does cheap things: length checks, regex against a maintained banned-phrase list (loaded into a bloom filter for speed), reputation lookup ("is this user already shadow-banned"), simple URL allowlist.

**Async checks are slower and more expensive.** ML toxicity models take 100-500ms per inference. Link scanning involves an external API. These run after the comment is already visible, and if they flag it, the comment's state is updated (and cache invalidated). The user sees their comment briefly, then if it gets removed, sees a "your comment was removed" notice.

**User reports** are the third input. A "report" button on each comment lets users flag content. Reports queue into the same human-review queue, sorted by: report count, the reporter's own reputation (heavy users carry more weight), the content's auto-toxicity score, the author's reputation.

**Confidence-based routing.**

The auto-checker returns a confidence score 0.0 to 1.0. Thresholds:
- `< 0.2`: auto-publish, no flag.
- `0.2 to 0.7`: publish but queue for human review.
- `0.7 to 0.95`: hold in `pending_review`, only visible to author until a human approves.
- `> 0.95`: auto-remove, do not show to anyone.

The exact thresholds get tuned over time. False-positive rate (good comments wrongly removed) and false-negative rate (bad comments wrongly published) trade off against each other.

**Why not pre-publish review for everything.**

Two reasons:
1. Latency. Users expect comments to appear immediately.
2. Volume. At 1M comments/day, even 1 second per comment of human time would require ~10k mod hours per day. Not viable.

Post-publish with fast async takedown is the standard for high-volume sites. Pre-publish review is reserved for content from low-reputation users or articles flagged as sensitive.

**The shadow ban.**

A user posts spam repeatedly. You ban them, they create a new account. Instead, shadow-ban: their comments look published to them but invisible to everyone else. They spend energy posting comments that no one sees. Less likely to spawn alt accounts because the system gives them no signal.

The implementation is a per-user flag. On rendering the tree, if the requesting user is the shadow-banned user, include their comments. Otherwise filter them out. The author sees their full thread; everyone else sees a hole.

</details>

## Step 7: incomplete render flow

Walk through what happens when a user loads `/articles/news-of-the-day`. The article is hot, gets 10k loads per minute. Sketch the read path.

<details>
<summary><b>Reveal: read path with cache tiering</b></summary>

```
        GET /articles/news-of-the-day
                  │
                  ▼
        ┌───────────────────┐
        │   CDN edge        │
        │   (Cloudflare,    │
        │    Fastly)        │
        └────────┬──────────┘
                 │ miss
                 ▼
        ┌───────────────────┐
        │  Read Cache       │  key: article:42:tree:sort=hot
        │  (Redis)          │  value: rendered JSON of full
        │                   │  comment tree
        └────────┬──────────┘
                 │ miss
                 ▼
        ┌───────────────────┐
        │  Read Service     │
        │  - SELECT all     │
        │    comments       │
        │    WHERE          │
        │    article_id=42  │
        │  - Build tree in  │
        │    memory         │
        │  - Apply sort     │
        │  - Render JSON    │
        │  - Cache for 60s  │
        └────────┬──────────┘
                 │
                 ▼
        ┌───────────────────┐
        │  Read Replica     │
        │  (Postgres)       │
        └───────────────────┘
```

Cache key includes the sort order because the same article has different trees for "newest" vs "hot" vs "top." TTL is short (60s) because new comments arrive frequently. Writes invalidate the cache for the affected article.

For very hot articles, the CDN holds the rendered JSON. 99% of reads never touch your origin. The CDN refreshes from origin every 60 seconds.

For cold articles, every read goes to the read replica. Worst case a cold article has 0 comments and the response is trivially small.

</details>

## Follow-up questions

Try to answer each in 2 to 4 sentences before reading the solution.

1. **A comment with 200 replies under it gets deleted by its author.** What happens to the replies? Walk through the data and the UI.

2. **A user spams 1000 comments in 10 seconds via a script.** Where does this get caught? How do you avoid blocking a legitimate user who posts 5 comments in a minute during a hot discussion?

3. **Edit window vs immutable history.** A user edits their comment 3 hours after posting. The original said something they want to walk back. Should other users see "(edited)"? Should they be able to see the original? What about for moderation purposes?

4. **The "hot" sort algorithm.** Define Reddit's "hot" ranking formula. Why does it decay with time? What happens if you just sort by score?

5. **A new comment is posted. Your cached tree is now stale.** Do you invalidate the entire cache key, do partial updates, or accept staleness? What is the trade-off?

6. **A user reports a comment as spam. 50 other users report the same comment within 5 minutes.** Do you wait for a human, or do you auto-hide it? Where does the threshold come from?

7. **Real-time updates.** Someone wants the comment count to update live on the article page, and replies to appear without refresh. Sketch the WebSocket fan-out without melting the server when an article gets 10k concurrent viewers.

8. **Pagination on a 5000-comment thread.** You cannot send all 5000 to the client. What is your paging strategy and how do you handle "load more replies" when a child has 80 sub-replies?

9. **A comment thread suddenly attracts brigading from another site.** Sudden flood of accounts with no prior activity, all downvoting one comment. How do you detect this and what do you do?

10. **GDPR delete.** A user requests deletion of all their comments. They have 4000 comments going back 5 years, many of which have replies underneath. What happens?

## Related problems

- **[Approval Management (011)](../011-approval-management/question.md)**, the moderation queue is a workflow engine with the same state-machine plus role-routing patterns; the per-mod queue parallels the per-approver dashboard.
- **[Todo List Sharing (013)](../013-todo-list-sharing/question.md)**, the soft-delete-with-tombstone pattern shows up in any system where deletes must preserve structure.
- **[Notification System (010)](../010-notification-system/question.md)**, replies and mentions fan out through this exact notification pipeline; the comment system emits events, the notification system delivers.
- **[Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md)**, the vote aggregation and Kafka-first write pattern are textbook examples from this problem area.
