# Contributing

Thanks for your interest. The quality bar for problems here is intentionally high. A new problem should make a candidate noticeably better prepared, not just add to the count.

## Before you propose a new problem

Ask:

1. **Does it test a distinct design skill?** "Design Instagram" overlaps almost entirely with News Feed + Video Streaming. Skip it.
2. **Has it actually been asked?** If you cannot point to two or more candidates being asked this in the last year, it is probably too niche.
3. **Does it have at least one non-obvious trade-off?** Problems that have one clear right answer are not interesting interview questions.

## Structure of a problem folder

```
problems/NNN-slug/
  question.md
  solution.md
```

Use the next sequential 3-digit ID. Slug should be the title lowercased, words joined with hyphens, prepositions and articles dropped.

## question.md

Front matter (exact field names — the build script depends on them):

```yaml
---
id: 11
title: Design X
category: <one of the existing categories, or propose a new one>
topics: [tag1, tag2, tag3, tag4, tag5]
difficulty: Medium    # one of Easy, Medium, Hard
solution: solution.md
---
```

Body sections (in this order):

1. **Scene** — one paragraph. Where you are, who the interviewer is, what they just said. Sets tone.
2. **The prompt** — verbatim what the interviewer would say. 2 to 5 sentences. Deliberately under-specified.
3. **Step 1: Clarifying questions** — invite the reader to spend 5 minutes listing questions, then reveal a suggested set in a `<details>` block.
4. **Step 2: Capacity estimates** — give the inputs (DAU, qps, payload size), ask the reader to compute 4 to 6 outputs, reveal the math.
5. **Step 3: Sketch the high-level architecture** — provide an intentionally incomplete ASCII diagram with `[ ? ]` placeholders. Reveal the complete diagram in a `<details>` block.
6. **Step 4: API design** — list the endpoints to design. Reveal the reference API.
7. **Step 5 onwards** — problem-specific design steps. Sharding strategy, caching, real-time path, etc.
8. **Follow-up questions** — 5 to 10 numbered questions. Do NOT answer them here; answers belong in solution.md.
9. **Related problems** — link to 2 to 4 other problems in this repo that share patterns.

Use plain HTML `<details><summary>` blocks for reveal sections. Do not use any framework-specific syntax.

## solution.md

The deep walkthrough. Roughly 800 to 2000 lines. Required sections:

1. **TL;DR** — three paragraphs.
2. **Clarifying questions** — list the ones you would ask and why each matters.
3. **Capacity estimates** — full math, not just answers.
4. **API design** — full spec including status codes and error cases.
5. **Data model** — schema with reasoning for every column and index choice.
6. **Core algorithm** — the central design decision (encoding, fan-out, indexing, etc.) with at least two alternatives compared.
7. **High-level architecture** — diagram + component walkthrough.
8. **Read and write paths** — what happens on a single request.
9. **Scaling** — caching, sharding, replication, CDN, geographic distribution. Sub-sections for each.
10. **Reliability** — failure modes, retries, what stays consistent under partition.
11. **Observability** — metrics, logs, alerts that should be in place from day one.
12. **Follow-up answers** — one section per follow-up question from question.md, with the answer a senior candidate would give.
13. **Trade-offs and what a senior would mention** — architectural calls, business considerations, what you'd revisit at 10x scale.
14. **Common interview mistakes** — what weaker answers do wrong.

## Style guide

- Sentences over bullet points where the idea is connected.
- Bullet points for genuinely parallel options or trade-off lists.
- ASCII diagrams over Mermaid where the diagram is a flow.
- Mermaid for state machines and sequence diagrams.
- No marketing words: "robust", "comprehensive", "seamless", "leverage". Just say what it does.
- Show your numbers. Order-of-magnitude is fine, but show the working.
- Acknowledge what you do not know. "I would benchmark this before committing" is a senior-candidate sentence.

## Workflow

1. Fork and branch.
2. Add your `problems/NNN-slug/` folder with both files.
3. Run `python3 scripts/build_index.py` to refresh PROBLEMS.md.
4. Open a PR. Include in the PR body: the source of the question (which company/interviewer pattern), and one example of a clarifying question that catches weaker candidates off guard.

## Review

PRs are reviewed on three criteria:

1. **Substance** — do the follow-up answers go past surface level?
2. **Honesty** — does it admit trade-offs, or pretend the answer is obvious?
3. **Interview realism** — would a real interviewer recognize this as something they would ask?

If the PR clears all three, it ships.
