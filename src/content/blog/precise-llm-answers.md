---
title: How We Made Our LLM Agents Answer Precisely
description: Four parallel memory searches, timestamps on every fact, banks per agent, and a Thursday-night cleanup ritual. The precision is scheduled, and here is the bill.
pubDate: 2026-11-01
tags: [agents, engineering, memory]
draft: false
---

At the end of October I asked one of our agents a question with a clock in it. What broke on the free-tier routing path last month, and how was it fixed? The answer came back with receipts: dates, the retry budget that was raised, the probe evidence, the fix. No confabulated timeline, no "around that time somewhere".

It knew what I meant by October because I told it October, and because everything it stores carries time. That precision is the only reason an agent's memory is worth having. This post is how we got there: one box, fifteen containers, the meter running, and most of the levers transfer to any stack.

```
recall(
  bank:  "hermes-s0",
  query: "free-tier provider API error",
  temporal_window: { start: "2026-10-01", end: "2026-11-01" }
)
→  "502s traced to provider retries; retry budget raised to 300s,
    timeouts synced across the stack; 10/10 probes green after"
    [cited: observation dated 2026-10-07, plus 37 more in-window hits]
```

## Four searches, one answer

The memory layer is [Hindsight](https://github.com/vectorize-io/hindsight), self-hosted, MIT licensed. When an agent asks a question, recall runs four retrieval arms in parallel, not one:

- **Semantic**: the query is embedded and matched against stored vectors.
- **Keyword**: classic full-text search, which saves you the day someone types an exact error code.
- **Graph**: entities in memory are linked; recall walks a few hops out from anything the query touches. Traversal budgets are tiered by query complexity (the lowest tier is a hundred nodes, the high tier a thousand).
- **Temporal**: the arm that fires when the question carries a clock. More below.

Four ranked lists fuse with reciprocal rank fusion, then a local cross-encoder reranks the survivors against the raw query. The reranker runs on our CPU, not in someone's cloud.

![Diagram: recall() fans out into four parallel retrieval arms (semantic, keyword, graph, temporal), then fuses the ranked lists and reranks locally with a cross-encoder. The read path spends 4.8K tokens across 181 calls and no LLM.](/images/precise-llm-answers/recall-fan.svg)

*Four retrieval arms run in parallel and never touch an LLM; only embedding lookups and a local reranker spend anything, and they spend pennies.*

What I care about most stays the same: **recall makes zero LLM calls**. In a measured two-day window our read path was 181 recall operations, made up of 94 embedding lookups and 87 local cross-encoder scores, and 4.8K tokens of embedding spend in total. Reads are nearly free, which is exactly what you want from a component an agent hits on every turn.

## Asking with a clock

Most agent memory fails the same way. You ask what happened last month, you get an answer about last week.

Recency was the only time signal the store had. Hindsight takes time in two ways. Put it in the query itself ("in October", "after the migration") and the query analyzer parses the phrase. Or, when you already know the range, pass an explicit window:

```json
{ "temporal_window": { "start": "2026-10-01", "end": "2026-11-01" } }
```

Relative phrasing ("the last two weeks") anchors to a `query_timestamp`, so a scheduled job and an interactive session can ask the same question and mean different weeks.

Three details are the difference between a time filter and a time *ranking*:

**The window ranks, it does not filter.** Memories inside the window surface higher; the semantic, keyword, and graph arms keep searching outside it. This is deliberate. A hard filter throws away the older fact that actually explains the October incident, and "last month" is a hint the ranking can weigh. When you do need an exhaustive bounded read, the list API takes `start_date` and `end_date` over a choice of timestamps (created, updated, or when the event was *mentioned* or *occurred*), and that one really does cut.

**The window is split into time buckets.** The range is divided into eight segments, and the strongest match from each segment enters the candidate pool first, so a question about October reads the whole month instead of the three loudest days. Selection inside a bucket prefers relevance over recency: the newest thing is not automatically the truest thing. An earlier implementation kept the fifty most recent in-window memories, which on a bank with 660,000 rows degraded into a thirty-second full scan; the current path caps the work at a pool of sixty candidates and ten entry points per fact type.

**Time proximity is a nudge.** Matches near the center of the window get a small boost, matches at the edges a 10% penalty, and non-temporal queries score neutral, so adding a window never silently rewrites an unrelated question.

Bounds are inclusive, a naive datetime means UTC, and a reversed window is a 422 rather than a silent empty result. Small details, but they are why I trust the component to run unattended.

## The write path

Retrieval is half the story. The other half is what gets written down, because precise questions against sloppy facts still return sloppy answers.

**Retain** turns a session into facts. Text is chunked at roughly 3,000 characters and each chunk gets one LLM extraction call, which is where nearly all of the write cost lives. The write itself is a three-phase pipeline: entity resolution (the fuzzy matching that knows "Alice" and "Alice Chen" are one person, which needs its own database connection so it can do vector lookups) runs first, then a single transaction writes the facts and their links, then display data is attached best-effort afterward. Temporal links are capped at twenty per fact, so a frequently discussed entity cannot drown its own timeline. Re-ingesting unchanged text skips the LLM entirely.

**Consolidation** runs after retain, in rounds. Each round takes up to a hundred unconsolidated memories, and a bank over that limit yields its worker slot and re-queues, so one busy agent cannot starve the others. Every memory in the round gets a cheap database-only recall to find its related observations, then the batch goes to the LLM as sub-batches, one call each, and the output is a set of create, update, and delete instructions on observations. A failed sub-batch is bisected and retried instead of dropped, and every batch checkpoints.

Out of that comes **observations**: deduplicated beliefs (near-duplicates merge at cosine similarity 0.97) that carry exact quotes back to the facts they came from and a proof count of how many independent facts support them. Observations are refined rather than overwritten, with history kept, and they are freshness-aware: when new facts have not been consolidated yet, the system knows its observations are stale and checks them against the raw layer before answering.

Above that sits **mental models**: standing answers, precomputed. Answering a question climbs a ladder, cheapest first: mental models, then observations, then raw facts. Mental models are rebuilt in the background only when their own scope changed, and fetching one is a plain database read. Reflect, the query-time reasoning pass, is capped at twelve LLM calls, 100K context tokens, and five minutes of wall time, so a hard question stays hard-capped.

By the vendor's own profiling, over 80% of consolidation wall-clock time is LLM time. Everything else, the dedup, the linking, the ladder, is what it costs to keep the LLM portion small.

![Diagram of the write path: retain feeds consolidation rounds, which produce observations, which precompute mental models; below, the read ladder climbs mental model to observations to raw facts; a bar chart shows the two-day bill (consolidation 4.83M tokens, mental-model refresh 3.49M, retain 422K, recall 4.8K with zero LLM calls).](/images/precise-llm-answers/write-path.svg)

*Every layer up the write path is paid; the read ladder climbs the cheapest layer first, which is why recall sits at the bottom of the bill.*

## Five banks and a Thursday ritual

All of this lives in banks, and the split is the most consequential decision in the stack.

We run five, one per agent runtime:

- `hermes-s0`: the main supervisor's bank, 6,239 facts.
- `claude-ppro`: the coding agent's, 5,028.
- `pi-s0`: the batch executor's, 2,222.
- `hermes-s2` and `hermes-p14s`: two more supervisors, 1,650 and 1,618.

A bank is a hard boundary. Recall never crosses one. Think of the paper dividers in a lunchbox (便當, the Taiwanese and Japanese lunchbox): the dividers are the point, because the sauce stays off the rice. One agent's context leakage is the failure mode that makes agents confidently, fluently wrong. A supervisor's half-finished plan and a coder's finished fix look equally authoritative once they are flattened into the same vector space.

Our split follows the runtime instead of the topic, because a bank should answer for one brain. Within that constraint, scope gets sharper three ways. Units carry tags, and tags decide who may read a given mental model. `observation_scopes` let the same raw facts grow *different* observations for different consumers, which is how one incident produces an executive summary for the humans and a detailed timeline for the on-call agent without either polluting the other. When knowledge genuinely must cross agents, the answer is a shared bank with tags, not open doors between neighbors.

Which brings me to the maintenance nobody advertises. Every Thursday at 18:00 a script runs what we call the cleanup ritual, and it does not wait for anything to be broken. It deletes marker and probe documents left by smoke tests. It deletes self-loops in the link graph (there are a hundred sitting there right now, accumulated since last Thursday) and reversed duplicate semantic links (two hundred twenty-six pairs, again since last Thursday; temporal and causal edges are left alone, because for those the direction *is* the information). It runs VACUUM, escalating to the table-rewriting VACUUM FULL only when a table's heap has grown to three times its live data, and it reindexes the link table concurrently only once the index passes 200MB (ours is at 65MB, so it has been politely skipping).

Why schedule it: a self-loop returns the seed to itself, and a reversed twin doubles a neighborhood that was never real. Both are noise edges, and graph traversal reads noise edges with total confidence. Precision decays quietly in any store that accumulates links, and the only fix that works is a calendar entry. It is waste separation on a schedule, the way a German apartment block does it: the pickup day decides when the cardboard leaves your hallway, not how full the bin feels that morning. Below the weekly pass, a fifteen-minute job prunes stale operational rows with a thirty-day retention.

## What it costs

Here is a measured two-day window from our own gateway logs, 2,177 model calls:

- **Consolidation**: 1,400 calls, 4.83M tokens.
- **Mental-model refresh**: 482 calls, 3.49M tokens.
- **Retain**: 104 calls, 422K tokens.
- **Recall**: 181 calls, 4.8K tokens, and none of them an LLM.

Reads are three orders of magnitude cheaper than writes, and that ratio is the design. Writing is expensive because writing is where accuracy is bought. Our own tuning leans into it: consolidation runs a cheap flash-class model, in small sub-batches, with a single parallel worker, because it runs forever in the background where latency does not matter.

Those same numbers through the hosted service would be a line item: Hindsight Cloud lists retain at $10 per million tokens, recall at $0.75, and storage at $0.25 per million tokens per month once memories pass thirty days. Self-hosted, the license is zero and the bill is the tokens, which cross a gateway we own, onto models we chose. The database is 921MB today.

Against the alternatives, everyone else is betting on something different. Mem0 bets on extracted facts plus vectors, clean and simple. Zep bets on a temporal knowledge graph, hosted-first, while Letta hands the model its own memory files to manage. LangMem is a store for LangGraph; Cognee, a graph-and-vector ETL pipeline. Hindsight's bet is the layered write path you just read, evidence chains all the way up, and a read path that never calls an LLM. Its benchmark page, vendor-run and worth that discount, currently shows 94.6% on LongMemEval S and 92% on LoComo 10. The third-party comparison at [aiagentmemory.org](https://aiagentmemory.org/articles/llm-memory-comparison/) is a fairer place to shop.

## The frame around the memory

A memory system is a component. What makes it answer precisely at 3 a.m. without a human in the loop is the frame.

In front sits [Bifrost](https://github.com/maximhq/bifrost), an open-source AI gateway, doing two jobs. It issues a virtual key per consumer, one per agent, so revoking a runaway agent is a config change and usage is attributable per key. It also runs a priority chain of routing rules: an embedding fallback at priority 1000, the memory-write route at 999, normal traffic at 256 and 255. Each rule fans out to weighted targets, and the weights are the economics made explicit: normal traffic splits 40/40/10/10 across providers, while memory writes send 30% to a flash-class model and 70% to a free tier that a small in-house proxy serves, so the paid providers keep their quota.

For humans, the door is Open WebUI, fronted by a self-hosted SearXNG for search and a page-reader for ingestion, with memory functions wired to the same bank system. Every agent on the box reads one shared rules file, symlinked into place: ten sections covering verification triggers, a solution ladder that forces the cheapest fix first, and four gates (irreversible actions, data-model changes, expensive forks, external writes) that a headless run is required to stop at with a single line saying exactly what authorization it needs. A control signal in the environment decides who may push and who may not. The rules are the banks' twin: banks bound what memory an agent can reach, rules bound what an agent can do with it.

## Precision is scheduled

I keep coming back to that October question, because nobody was clever when it was answered.

Four searches fanned out in parallel, one of them over time. The facts landed in a bank that belongs to one brain, with quotes back to their sources. A round of consolidation folded them into observations, capped at twelve LLM calls and five minutes. A Thursday script swept the noise edges out of the graph. And a gateway that has never met me decided which model was cheap enough for the job.

If any of this transfers to your stack, five levers did the work:
- Ask with a time window; 'recently' means whatever the listener hears.
- Store quotes and proof counts, so an answer can cite the fact behind it.
- Keep one bank per agent; cross-contaminated context is fluent and wrong.
- Sweep noise edges on a schedule, because precision decays silently.
- Make reads cheap and writes thorough; accuracy is bought on the write path.

The past is never dead, as Gavin Stevens tells Temple Drake in *Requiem for a Nunn*. A memory system takes the line literally: nothing is deleted until a retention policy says so, nothing is trusted without a quote, and every Thursday evening the past gets its index rebuilt. That is what it takes. The answers are precise because someone put the precision on a calendar.

## Sources

- [Hindsight: how consolidation scales](https://hindsight.vectorize.io/blog/2026/05/08/how-hindsight-scales): round sizes, LLM-bound wall clock, reflect ceilings, temporal arm budgets (vendor).
- [Hindsight retrieval documentation](https://hindsight.vectorize.io/developer/retrieval): four-arm retrieval, temporal scoring, rank-not-filter semantics.
- [Hindsight recall API](https://hindsight.vectorize.io/developer/api/recall): `temporal_window`, `query_timestamp`, query limits.
- [Hindsight list memories API](https://hindsight.vectorize.io/developer/api/memories): true time filtering and timestamp choice.
- [Pull request #3678](https://github.com/vectorize-io/hindsight/pull/3678): temporal window bounds, UTC handling, reversed-window 422.
- [Hindsight retain documentation](https://hindsight.vectorize.io/developer/retain): chunking, entity resolution, fact types.
- [Commit 914ba796](https://github.com/vectorize-io/hindsight/commit/914ba796539b76f48269b537012c597c66587e2a): the three-phase write pipeline and the twenty-temporal-link cap.
- [Hindsight observations documentation](https://hindsight.vectorize.io/developer/observations): proof counts, refinement, dedup threshold, observation scopes.
- [Hindsight mental models documentation](https://hindsight.vectorize.io/developer/mental-models): the read ladder and precomputation.
- [Hindsight benchmarks](https://benchmarks.hindsight.vectorize.io/): vendor-run scores (LongMemEval S 94.6%, LoComo 10 92%).
- [Hindsight Cloud pricing](https://vectorize.io/pricing): retain, recall, and storage list prices.
- [LLM memory systems comparison](https://aiagentmemory.org/articles/llm-memory-comparison/): third-party table across Mem0, Zep, Letta, LangMem, Cognee, and others.
- [Bifrost](https://github.com/maximhq/bifrost): open-source AI gateway, Apache 2.0.
- ["The past is never dead"](https://en.wikipedia.org/wiki/Requiem_for_a_Nun): attribution, Gavin Stevens, Act I, Scene III, *Requiem for a Nunn* (1951).
