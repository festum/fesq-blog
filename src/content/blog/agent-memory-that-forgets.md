---
title: Designing Agent Memory That Forgets
description: What a memory layer should keep, and what it must drop.
pubDate: 2026-10-15
tags: [agents, engineering]
---

An agent that remembers everything eventually does nothing well. Unbounded memory turns retrieval into noise, and a system that recalls every prior interaction starts answering questions that were never asked. Forgetting is not a failure of the memory layer — it is the feature that makes the rest of it useful.

The design question is not how much to store, but what earns a place. A memory layer exists to change future behavior: a preference that shapes the next response, a fact that prevents a repeated mistake, a decision whose context will matter again. Anything that fails that test is history, not memory, and history belongs in logs.

## What we keep

- Facts that demonstrably change the agent's next action.
- Decisions and their reasons, so the rationale survives the person who made it.
- Corrections, weighted heavily — being told "not that" twice is expensive.
- Sources, so a remembered claim can be re-checked instead of trusted.

Everything else decays on purpose. Retention windows, staleness scoring, and explicit eviction are ordinary engineering, not special machinery. The discipline is accepting that a smaller, sharper memory outperforms an exhaustive one — and that re-deriving a dropped fact is almost always cheaper than carrying it forever.
