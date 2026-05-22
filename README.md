# System Design Practice Problems

> Interview-style system design questions with the ambiguity, follow-ups, and judgement calls of a real onsite. Not just "Design X — here's the answer" essays.

Most system design content reads like a textbook. Real interviews don't. The interviewer drops a vague prompt, you ask clarifying questions, you sketch, they push back, you redesign. This repo is built around that loop.

## What makes this different

Each problem has two files:

- **`question.md`** — the interview scene. A short prompt from the "interviewer", clarifying questions you should ask, capacity-estimate math you must do, a **partial architecture diagram you complete yourself**, then five to ten follow-up scenarios. Most steps have a `<details>` block you reveal only after attempting the step yourself.
- **`solution.md`** — the full deep dive. API design, data model, sharding strategy, read/write paths, cache invalidation, geo distribution, observability, trade-offs, the most common follow-up answers, and the mistakes weak candidates make.

You won't get the most out of this by reading the solution first. The point is to sit with the question, sketch the diagram, do the math, *then* compare.

## What's inside

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

If you're new to system design interviews, this order builds the skills progressively:

1. URL Shortener — read paths, caching, encoding choices
2. Rate Limiter — algorithms with state, distributed counters
3. Distributed Cache — consistent hashing, replication, eviction
4. Typeahead — trie data structure at scale, ranking
5. Notification System — fan-out, retry, deduplication
6. News Feed — push vs pull, hot users
7. Chat System — real-time delivery guarantees, presence
8. Ride Sharing — geospatial indexing, matching
9. Web Crawler — BFS at scale, politeness, dedup
10. Video Streaming — CDN, adaptive bitrate, storage tiers

## Companion blog

Each problem is also published on [amirulislamalmamun.com/practice/system-design/](https://amirulislamalmamun.com/practice/system-design/) with the question on top and the solution hidden behind a Reveal button. Use whichever surface you prefer.

## Contributing

If you have a system design question that genuinely shows up in interviews and isn't covered here, see [CONTRIBUTION.md](CONTRIBUTION.md). Quality bar is high: each problem must include the full question/solution split, with at least five follow-ups and one partial diagram for the candidate to complete.

## License

MIT. Use the problems freely for self-study, study groups, or interview prep classes. Attribution appreciated but not required.
