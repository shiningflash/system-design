---
id: 15
title: Design a Comment System (Threaded + Voted + Moderated)
category: User Content
topics: [nested data, voting, moderation, soft delete, ranking]
difficulty: Easy
solution: solution.md
---

## Scene

The interviewer puts down their coffee.

> *"We run a news site. Every article has a comment section. People reply to each other, they upvote and downvote, mods remove spam, sometimes a comment goes viral and gets 5000 replies. Design the comment system. Think Disqus, HackerNews, Reddit-style."*

This looks like CRUD. It is not. Comments are the smallest piece of user content you can imagine and they pack in every hard property: nested data (a reply to a reply to a reply), heavy reads with bursty writes, voting with hot-row contention, soft delete that has to keep thread structure intact, ranking that goes beyond chronological, and a moderation pipeline that has to keep up with abuse.

Candidates who jump to "comments table with parent_id, done" skip every interesting question. The interesting ones: how do you fetch a 5000-node thread without 5000 database round trips, how do you count 1000 upvotes in 5 seconds without a hot row, how do you delete a comment with 200 replies under it without orphaning them, and how does the moderation pipeline decide what is spam without a human looking at every comment.

We walk this from a 10-article blog to a viral site doing 1M comments a day. At each stage something breaks and that is what we fix.

## Step 1: clarify before you design

Take 5 to 7 minutes. Eight questions minimum.

<details>
<summary><b>Reveal: questions a strong candidate asks</b></summary>

1. **Depth limit.** Can a reply to a reply to a reply go on forever, or is there a cap? Reddit caps visible nesting around 10 levels. HackerNews has no hard cap but indents stop after 8. Without a cap, recursive trees produce horrible fetch latency and let one user reply to themselves 200 times deep.
2. **Edit history.** Can users edit their comments? Forever, or only for a window? Do we keep history? The 5-minute edit window is the common pattern. Permanent edits with no history break accountability ("that's not what they said"). HackerNews lets you edit for about 2 hours.
3. **Voting model.** Upvote-only like Facebook reactions, or up plus down? Anonymous votes or named? Can a user change their vote? Downvotes change moderation incentives and need dedup logic.
4. **Sort orders.** What does the UI need? Newest, oldest, top, controversial, "best"? Each has different storage and caching needs. "Best" (Reddit's confidence-bound formula) is expensive to compute live.
5. **Moderation model.** Auto-detect spam? Human mods? User reports? All three? Pre-publish review or post-publish takedown? Pre-publish (every comment waits for a mod) is a different system from post-publish (every comment goes live immediately, mods react).
6. **Delete semantics.** When a user deletes a comment that has 50 replies, what happens? Tombstone, hard delete, children orphaned? Almost always: soft delete with tombstone so the thread survives.
7. **Read vs write ratio.** How many reads per write? Are comments cached? News article comments get read 100x to 1000x more than they get written. Caching the rendered tree dominates the architecture.
8. **Real-time updates.** When someone posts a reply, does my screen update live, or only on refresh? Real-time (WebSocket fan-out) is a different design from poll-driven.
9. **Auth.** Anonymous comments allowed? Pseudonymous? Required login? Anonymous changes the abuse story.
10. **Notifications.** When someone replies to me, do I get pinged? Usually a separate service consuming events, not built into the comment system itself.

If you walk in asking only "how many comments per day" you have lost the interesting design space. Depth limit, sort order, and moderation model are the three that change the architecture the most.

</details>

## Step 2: capacity estimates

Two scenarios. A small blog and a viral site.

**Small blog:**
- 10 articles published per day
- 100 comments per day across all articles
- Each article live for about 7 days of meaningful traffic
- Average comment: 200 bytes

**Viral site:**
- 100,000 articles in the active set
- 1M comments per day
- Top 1% of articles get 80% of comments (~1000 articles see 800k comments per day)
- 300 bytes per comment on average, long-form replies up to 5KB
- Read-to-write ratio: 1000:1

Compute for the viral site: writes per second sustained and peak, reads per second sustained, storage for one year, working set for the hottest 1000 articles, peak comments per article per second on the hottest article in a viral moment.

<details>
<summary><b>Reveal: the math at both scales</b></summary>

**Small blog:**

100 comments/day is roughly one comment per 15 minutes. Trivial. Reads at 1000:1 land at one per second. A laptop runs this. Storage: 100 × 365 × 200 bytes = ~7MB per year. Nothing. The whole system is one Postgres, no cache, single app instance. Ship it in a weekend.

**Viral site:**

1M / 86400 = ~12 writes/sec sustained. Peak 3-5x: ~40 to 60 writes/sec. Reads at 1000:1 = ~12,000 reads/sec sustained, ~40k reads/sec peak. Storage: 1M × 365 × 300 bytes = ~110GB/year for comment bodies, ~250GB/year once you add votes, flags, and edits.

The hot working set (top 1000 articles, ~800 comments each, ~300 bytes) is ~240MB. Fits in Redis easily. A single viral moment puts 50 comments/sec on one article and 1000 votes/sec on the top comment. That hot spot is what you design around.

The interesting numbers are not the totals. Steady-state load is small. The system is dominated by *read amplification* (1000:1) and by *burst behavior* (one viral comment producing 100x sustained load on one row).

Storage is small enough that you do not need to shard for capacity. You shard for blast radius and to keep hot articles from poisoning cold ones.

The hardest number is the 40k reads/sec at peak for one article's comment tree. A naive recursive query against Postgres on every page load would crater the DB. The cache layer is not optional.

Vote throughput on a hot comment can hit 1000 votes/second. That is a hot-row problem at the DB level and needs its own fix.

</details>

## Step 3: storing nested data

This is the design decision that frames the whole system. You have a tree of comments. How do you store it in a relational database such that you can insert a new comment cheaply (the hot write path), fetch an entire thread for an article cheaply (the hot read path), delete a subtree cheaply (rare but happens), and move a subtree (very rare for comments).

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

Inserts are trivial. Fetching a subtree means a recursive CTE (`WITH RECURSIVE`). Postgres handles it but the plan is a loop with a join per level. A 5000-comment thread at depth 8 means 8 joins. Workable, not free. Moving a subtree is one parent_id update. The classic pain: "give me the full thread for article X" is a recursive query, not a simple `WHERE article_id = X`.

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

The `path` column encodes the full ancestor chain. A reply has path = parent's path + "/" + self.id. Inserts need one extra SELECT to read the parent's path. Fetching a subtree is `WHERE article_id = X AND path LIKE '/123/%'` with a prefix index. One query, no recursion. Fetching the article's whole tree is `WHERE article_id = X`. One query. Moving a subtree is expensive because you rewrite the path of every descendant, but for comments you almost never do that.

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

Each node gets a left and right number such that descendants sit between the parent's. Fetching a subtree is `WHERE lft BETWEEN parent.lft AND parent.rgt`. Very fast for reads. Inserts are brutal: every comment to the right in the tree needs its lft/rgt updated, which can be thousands of rows, and concurrent inserts contend on every one. Used in academic examples. Almost no production comment system picks it. Comment workloads are insert-heavy and nested set is the wrong shape.

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

A separate table holds every ancestor-descendant pair. Fetching a subtree is a `WHERE ancestor_id = X` on the ancestry table joined back to comments. Inserts add one ancestry row per ancestor. A comment at depth 5 produces 5 ancestry rows. Storage is O(total tree depth) extra rows. For a 5000-comment thread averaging depth 4, that's 20k extra rows. Closure tables make moves easy and let you ask "how deep is the subtree" cheaply. Stack Overflow uses them internally for some hierarchies. Solid, but heavy on write amplification.

| Approach | Insert | Fetch subtree | Fetch full article | Move | Storage overhead |
|----------|--------|---------------|--------------------|------|------------------|
| Adjacency list | Cheap | Recursive CTE | Single query, assemble in app | Cheap | None |
| Materialized path | Cheap (one extra SELECT) | Single prefix scan | Single query | Expensive | One string per row |
| Nested set | Very expensive | Single range query | Single query | Very expensive | None |
| Closure table | O(depth) writes | Single join | Single join | Cheap | One row per ancestor pair |

The recommendation for a comment system is adjacency list plus materialized path together. Keep `parent_id` because it is the natural shape for inserts and for "show me the parent of this comment." Also keep `path` for the hot read query "give me everything under this subtree" without recursion. Both are derived from each other, so they cannot drift if you populate `path` from `parent_id` at insert time.

The redundancy is cheap. You eat one TEXT column per row (about 50 bytes average for paths under 10 levels) in exchange for never running a recursive CTE on the hot read path.

</details>

## Step 4: sketch the architecture

Fill in the placeholders. Each one is a major component.

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

A few things worth knowing about each piece. The API Gateway is the first defense. Per-user rate limit (10 comments per minute, 100 votes per minute) plus a WAF for obvious bot traffic. The Comment Service is stateless: validates body length, runs a synchronous spam pre-check against a bloom filter, writes to Postgres, emits `comment.created` to Kafka. The Read Cache holds rendered comment trees keyed by `(article_id, sort_order)`, invalidated on writes for that article, with the CDN sitting in front for popular articles.

The Vote Aggregator is what saves you on hot comments. Votes hit Redis (atomic INCR) immediately and the user gets a success right away. A background job batches the deltas and flushes to Postgres every 5 seconds.

The Comments DB is the source of truth, sharded by `article_id` once you outgrow one primary. Read replicas serve the cache misses. The moderation pipeline runs off Kafka: every new comment runs through classifiers, and the confidence score determines whether it auto-publishes, queues for human review, or gets auto-removed. User-reported comments enter the same queue with higher priority.

</details>

## Step 5: vote aggregation under contention

A comment goes viral. 1000 users upvote it in 5 seconds. Your naive design:

```sql
UPDATE comments SET score = score + 1 WHERE comment_id = 42;
INSERT INTO votes (user_id, comment_id, value) VALUES (?, 42, 1);
```

What happens? Why is this a problem? Design something that handles 1000 votes/sec on a single comment without melting the database.

<details>
<summary><b>Reveal: hot-row problem and the fix</b></summary>

Every UPDATE on the same row takes a row-level lock. The 1000 concurrent UPDATEs serialize behind that lock. Postgres still handles them, but every voter's request waits for the queue depth times the per-update time. The WAL fills with 1000 row-version entries for the same row. Other reads waiting on `SELECT FOR UPDATE` block too. Replication lag grows because the primary is busy. DB CPU spikes on one row while everything else slows down. That is the hot-row problem in one paragraph.

The fix has three parts.

**Decouple vote write from score update.** The user's click does not need to update the DB row directly. It needs the vote recorded (for dedup) and the score eventually correct. So `POST /comments/42/vote {value: 1}` goes to the Vote Aggregator. The aggregator runs `INCRBY vote_delta:42 1` in Redis (constant-time, in-memory) and writes the per-user vote to a `votes_pending` Redis hash for dedup. Returns 200 to the user in ~5ms.

**Batch-flush to the database.** A background worker runs every 5 seconds. It reads the `vote_delta:*` keys, and for each comment_id runs one `UPDATE comments SET score = score + <delta> WHERE comment_id = ?`. One UPDATE per comment, not per vote. It drains the `votes_pending` hash and bulk-inserts into the `votes` table. Resets the delta keys.

1000 votes in 5 seconds become *one DB update* instead of 1000. The hot row sees one write every 5 seconds.

**Dedup at the Redis layer.** A user clicks Upvote twice. Or changes from Up to Down. The Redis hash holds the user's current vote on each comment. On submit, `HGET user:7:votes 42` reads existing; compute delta (if existing was 0 and new is 1, delta = +1; if existing was -1 and new is 1, delta = +2); `HSET user:7:votes 42 1`; `INCRBY vote_delta:42 <delta>`. The delta path flushes to the DB. The user-vote map prevents double-counting.

Trade-offs. The displayed score lags actual count by up to 5 seconds. Acceptable; most users do not notice. Up to 5 seconds of votes could be lost if Redis dies before flush. Mitigate with Redis AOF + replica, or write-ahead the vote event to Kafka before Redis (the worker then reads from Kafka, which is durable). For most comment systems the 5s window is fine.

This pattern is the standard answer for "high write rate on a single counter." It works for vote counts, view counts, like counts, anything that aggregates.

</details>

## Step 6: moderation pipeline

Comments get spam, NSFW content, hate speech, off-topic noise. The interviewer asks: walk me through how a comment gets moderated, from posted to either visible-to-everyone or removed.

Three input signals: automated detection at post time, user reports after the fact, manual scanning by mods. Sketch the pipeline and the states a comment moves through.

<details>
<summary><b>Reveal: moderation pipeline</b></summary>

The states a comment can be in:

| State | Visible to | How it got here |
|-------|-----------|-----------------|
| `published` | Everyone | Posted and passed auto-checks |
| `pending_review` | Author only | Auto-checker flagged with medium confidence |
| `shadow_banned` | Author only | Author flagged as bad actor; comment looks live to them, invisible to everyone else |
| `removed_auto` | No one (tombstone for thread structure) | Auto-checker high-confidence spam/abuse |
| `removed_manual` | No one | Human mod removed |
| `removed_self` | No one | Author deleted |

The pipeline:

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

The synchronous pre-check is the only thing on the hot path. It must finish in ~50ms because it blocks the user's POST. So it does cheap things: length checks, a regex/bloom-filter pass against a maintained banned-phrase list, reputation lookup ("is this user already shadow-banned"), simple URL allowlist.

Async checks are slower and more expensive. ML toxicity models take 100-500ms per inference. Link scanning involves an external API. These run after the comment is already visible, and if they flag it, the comment's state updates (and the cache invalidates). The user sees their comment briefly, then if it gets removed, sees a "your comment was removed" notice.

User reports are the third input. A "report" button on each comment lets users flag content. Reports queue into the same human-review tool, sorted by report count, the reporter's reputation (heavy users carry more weight), content's auto-toxicity score, and the author's reputation.

The auto-checker returns a confidence score 0.0 to 1.0. Thresholds: under 0.2, auto-publish. 0.2 to 0.7, publish but queue for human review. 0.7 to 0.95, hold in `pending_review`, only visible to author until a human approves. Over 0.95, auto-remove. The exact numbers get tuned over time, trading false-positive rate against false-negative rate.

Why not pre-publish review for everything? Two reasons. Latency: users expect comments to appear immediately. Volume: at 1M comments/day, even one second of human attention per comment would need ~10k mod hours per day. Not viable. Post-publish with fast async takedown is the standard for high-volume sites. Pre-publish review is reserved for low-reputation users or articles flagged as sensitive.

The shadow ban is the clever bit. A user posts spam repeatedly. Ban them, they create a new account. Instead, shadow-ban: their comments look published to them but invisible to everyone else. They spend energy posting comments nobody sees, less likely to spawn alts because the system gives them no signal. Per-user flag in the DB; on rendering, if the requesting user is the shadow-banned user, include their comments. Otherwise filter them out.

</details>

## Step 7: render flow

A user loads `/articles/news-of-the-day`. The article is hot, gets 10k loads per minute. Sketch the read path.

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

For cold articles, every read goes to the read replica. Worst case a cold article has zero comments and the response is trivially small.

</details>

## Follow-up questions

Try to answer each in 2 to 4 sentences before reading the solution.

1. A comment with 200 replies under it gets deleted by its author. What happens to the replies? Walk through the data and the UI.

2. A user spams 1000 comments in 10 seconds via a script. Where does this get caught? How do you avoid blocking a legitimate user who posts 5 comments in a minute during a hot discussion?

3. Edit window vs immutable history. A user edits their comment 3 hours after posting. The original said something they want to walk back. Should other users see "(edited)"? Should they be able to see the original? What about for moderation?

4. The "hot" sort algorithm. Define Reddit's "hot" ranking. Why does it decay with time? What happens if you sort by score alone?

5. A new comment is posted. Your cached tree is now stale. Do you invalidate the whole cache key, do partial updates, or accept staleness? What is the trade-off?

6. A user reports a comment as spam. 50 others report the same comment within 5 minutes. Do you wait for a human, or auto-hide it? Where does the threshold come from?

7. Real-time updates. Someone wants the comment count to update live on the article page and replies to appear without refresh. Sketch the WebSocket fan-out without melting the server when an article hits 10k concurrent viewers.

8. Pagination on a 5000-comment thread. You cannot send all 5000 to the client. What is your paging strategy and how do you handle "load more replies" when a child has 80 sub-replies?

9. A comment thread suddenly attracts brigading from another site. Sudden flood of accounts with no prior activity, all downvoting one comment. How do you detect this and what do you do?

10. GDPR delete. A user requests deletion of all their comments. They have 4000 comments going back 5 years, many with replies underneath. What happens?

## Related problems

- [Approval Management (011)](../011-approval-management/question.md). The moderation queue is a workflow engine with the same state-machine plus role-routing patterns; the per-mod queue parallels the per-approver dashboard.
- [Todo List Sharing (013)](../013-todo-list-sharing/question.md). The soft-delete-with-tombstone pattern shows up in any system where deletes must preserve structure.
- [Notification System (010)](../010-notification-system/question.md). Replies and mentions fan out through this exact notification pipeline; the comment system emits events, the notification system delivers.
- [Write-Heavy System Patterns (018)](../018-write-heavy-patterns/question.md). The vote aggregation and Kafka-first write pattern are textbook examples from this problem area.
