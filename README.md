# System Design Practice Problems

> Interview-style system design questions with the ambiguity, follow-ups, and judgement calls of a real onsite.

Most system design content reads like a textbook. Real interviews don't. The interviewer drops a vague prompt, you ask clarifying questions, you sketch, they push back, you redesign. This repo is built around that loop.

## What makes this different

Each problem has two files:

- **`question.md`**. The interview scene. A short prompt from the "interviewer", clarifying questions you should ask, capacity-estimate math you must do, a **partial architecture diagram you complete yourself**, then five to ten follow-up scenarios. Most steps have a `<details>` block you reveal only after attempting the step yourself.
- **`solution.md`**. The full deep dive. API design, data model, sharding strategy, read/write paths, cache invalidation, geo distribution, observability, trade-offs, the most common follow-up answers, and the mistakes weak candidates make.

You won't get the most out of this by reading the solution first. The point is to sit with the question, sketch the diagram, do the math, *then* compare.

## What's inside

The set covers two skill levels. **Problems 1 to 10** are the classic Medium/Hard onsite questions. **Problems 11 to 20** are entry-level questions that show up in junior interviews and walk the design from 10 users to 1 million users explicitly.

| # | Problem | Category | Difficulty |
|---|---------|----------|------------|
| 1 | Design a URL Shortener | Basics | Medium |
| 2 | Design a News Feed (Twitter) | Social & Feeds | Hard |
| 3 | Design a Chat System (WhatsApp) | Real-Time Communication | Hard |
| 4 | Design a Rate Limiter | Reliability | Medium |
| 5 | Design Typeahead / Autocomplete | Search | Medium |
| 6 | Design YouTube / Netflix | Streaming | Hard |
| 7 | Design Uber / Lyft | Geospatial | Hard |
| 8 | Design a Web Crawler | Distributed Systems | Hard |
| 9 | Design a Distributed Cache | Distributed Systems | Medium |
| 10 | Design a Notification System | Messaging | Medium |
| 11 | Design an Approval Management Service | Workflow | Easy |
| 12 | Design a Shopping Cart Service | Session State | Easy |
| 13 | Design a Todo List with Sharing and Collaboration | Collaboration | Easy |
| 14 | Design a Coupon Code Redemption System | Concurrency | Easy |
| 15 | Design a Comment System (Threaded + Voted + Moderated) | User Content | Easy |
| 16 | Design a Load Balancer | Networking | Easy |
| 17 | Design a Read-Heavy System (Patterns Walkthrough) | Patterns | Easy |
| 18 | Design a Write-Heavy System (Patterns Walkthrough) | Patterns | Easy |
| 19 | Design a Help Desk Ticketing System (Zendesk basic) | Workflow | Easy |
| 20 | Design a File Upload & Share Service (Dropbox-lite) | Storage | Easy |

See [PROBLEMS.md](PROBLEMS.md) for the live index (generated from each problem's frontmatter).

## How to use this repo

1. Pick a problem. Don't read the solution.
2. Open `question.md`. Read the scene.
3. Spend 5 minutes asking yourself clarifying questions before peeking at the suggested list.
4. Do the back-of-envelope math by hand.
5. Sketch the architecture on paper (or in a text file). Fill in the placeholders in the partial diagram.
6. Work through the follow-ups. Write rough answers.
7. Now open `solution.md`. Compare. Where did you stop too early? Where did you over-engineer?

A 45-minute onsite covers maybe 60% of a single problem here. Treat each as a 90-minute self-paced session.

## Repo layout

```
problems/
  001-url-shortener/
    question.md      <- interview scene + try-it-yourself prompts
    solution.md      <- deep walkthrough + follow-up answers
  002-news-feed/
    ...
scripts/
  build_index.py     <- regenerates PROBLEMS.md from frontmatter
PROBLEMS.md          <- generated index
```

## Frontmatter contract

Each `question.md` starts with:

```yaml
---
id: 1
title: Design a URL Shortener
category: Basics
topics: [encoding, kv store, caching, sharding]
difficulty: Medium
solution: solution.md
---
```

This contract is what makes the index page on my blog auto-update when I push new problems here. If you fork the repo, keep the contract intact so any tooling around it keeps working.

## Reading order

If you're new to system design interviews, start with the entry-level set (11 to 20) to build the basic vocabulary. Then move to the medium/hard set (1 to 10).

**Entry-level path** (problems 11 to 20):

1. Load Balancer (16): what an LB actually does
2. Read-Heavy System (17): caching tiers and replicas
3. Write-Heavy System (18): buffers, batches, queues
4. Shopping Cart (12): session state and persistence
5. Todo List with Sharing (13): real-time collab basics
6. Comment System (15): nested data and moderation
7. Coupon Redemption (14): uniqueness under load
8. Help Desk Ticketing (19): workflow + SLA
9. Approval Management (11): the canonical workflow engine (flagship)
10. File Upload & Share (20): chunked uploads and tiered storage

**Medium/hard path** (problems 1 to 10):

1. URL Shortener (1): read paths, caching, encoding choices
2. Rate Limiter (4): algorithms with state, distributed counters
3. Distributed Cache (9): consistent hashing, replication, eviction
4. Typeahead (5): trie data structure at scale, ranking
5. Notification System (10): fan-out, retry, deduplication
6. News Feed (2): push vs pull, hot users
7. Chat System (3): real-time delivery guarantees, presence
8. Ride Sharing (7): geospatial indexing, matching
9. Web Crawler (8): BFS at scale, politeness, dedup
10. Video Streaming (6): CDN, adaptive bitrate, storage tiers

## Companion blog

Each problem is also published on [amirulislamalmamun.com/practice/system-design/](https://amirulislamalmamun.com/practice/system-design/) with the question on top and the solution hidden behind a Reveal button. Use whichever surface you prefer.

## Contributing

If you have a system design question that genuinely shows up in interviews and isn't covered here, see [CONTRIBUTION.md](CONTRIBUTION.md). Quality bar is high: each problem must include the full question/solution split, with at least five follow-ups and one partial diagram for the candidate to complete.

## License

MIT. Use the problems freely for self-study, study groups, or interview prep classes. Attribution appreciated but not required.
