---
title: What SWE-agent and Anthropic's Agent Guidelines Changed in Our Rules
description: Every rule in our open rule set traces to SWE-agent's interface research or Anthropic's agent guidelines; what they changed in how our two agents run unattended.
pubDate: 2026-11-02
tags: [agents, engineering, research]
draft: false
---

## The rule that replaced a failure

Every line of our rules file replaced a failure we hit in daily multi-agent use. The costliest failure class was check-pass claims with no output behind them: an agent announcing that tests passed when nothing had been run. The reasoning was fine; the interface could not tell a claim from a result, so runs ended green on work nobody did.

We wrote the failures down. The result is [festum/ai-rules](https://github.com/festum/ai-rules), an MIT-licensed rule set that every coding agent we run reads from a single file: `system/rules.md`, 162 lines across ten sections. One setup script creates the links, and on the machine writing this post, nine editor and CLI config paths resolve in one symlink hop to that file while the controller's contract is the tenth. Edit the file once, and every tool picks it up next session. Long procedures live in skills, which keeps the always-on contract short instead of a manual, and a local overlay file holds private preferences on top of the shared rules without merging into them.

![Diagram: one rules file on the left, a single symlink hop fanning out to ten targets on the right, nine editor and CLI config paths and one controller contract.](/images/what-swe-agent-and-anthropics-agent-guidelines-changed-in-our-rules/rules-fan.svg)

*One file, one hop: nine coding tools and one controller read the same contract.*

## An interface, not an instruction sheet

SWE-bench handed language models 2,294 real GitHub issues across 12 Python repositories and asked for patches. The best model at the time, Claude 2, resolved 1.96% of them ([paper](https://arxiv.org/abs/2310.06770)). The models were part of the problem, and the interface was the rest of it: agents were dropped into a raw Linux shell built for humans.

SWE-agent tested that hypothesis directly. It replaced the shell with an agent-computer interface: a small action set, guardrails against common mistakes, and concise feedback after every command. With GPT-4 Turbo it resolved 12.47% of the benchmark against a previous best of 3.8%, and an ablation on SWE-bench Lite put it 10.7 percentage points above the same model working in a plain shell ([paper](https://arxiv.org/abs/2405.15793)). Its file viewer opens a window of at most 100 lines at a time, on the theory that a bounded window beats a wall of text for a model the same way it beats one for a person. The interface also manages history, so the model works from current state instead of an unbounded transcript, which is the budget problem our output caps solve.

Anthropic reached the same result from the tool side. Building their SWE-bench agent, they spent more time optimizing tools than the overall prompt: relative file paths tripped the model, and requiring absolute paths fixed it. Their instruction to tool builders borrows from the factory floor: poka-yoke your tools, a Japanese term for mistake-proofing, the fixture that makes the wrong assembly impossible rather than merely unlikely ([guidelines](https://www.anthropic.com/research/building-effective-agents)).

Section 9 of the rules file is that research in five lines: batch independent reads, read files in bounded sections of at most 100 lines, truncate command output at the same limit, cap listings at 50 results, and honor explicit time and cost ceilings. The 100-line viewer in SWE-agent has a direct descendant here. The sentence we kept from Anthropic is their rule of thumb for anyone building an agent: "plan to invest just as much effort in creating good agent-computer interfaces (ACI)".

## The check the agent can run

Without a check the agent can run, done is whatever looks done, and the human becomes the verification loop that notices each mistake. That is the pattern Anthropic names the trust-then-verify gap, a plausible implementation that hides missing edge cases ([best practices](https://code.claude.com/docs/en/best-practices)). The same guide tells the agent to show evidence instead of asserting success, and separates the phases, explore first, then plan, then implement, then commit, because code written before the problem is understood solves the wrong thing.

Three sections of the rules file carry that advice. Section 1.4 sets the verification trigger: extra checks run when executable behavior changes, when material risk is concrete, when uncertainty affects a decision, or when someone explicitly asks. Tool use, a git repo, prose edits, and completed commands are not triggers, so a clear low-risk operation ends on its own result instead of a round invented to fill a report. Section 10 requires a pass claim to rest on output observed in the same session, and accepts a reliable operation result for routine actions. Section 5 defines done as required checks passing with nothing restated from the prompt.

The pruning advice applies to the rules themselves. For every line in a CLAUDE.md, the guide asks whether removing it would cause mistakes, and cuts the line if not. The rules file failed that test once and needed two trimming passes; the latest rewrite brought it down to 162 lines of telegraph prose. A rule nobody reads enforces nothing.

## Two agents, one contract

Anthropic sorts agentic systems into workflows and agents, then names the patterns between them: orchestrator-workers, where a lead agent decomposes and delegates, and evaluator-optimizer, where one model produces while another critiques in a loop. Their research system runs the first pattern, beat a single agent by 90.2% on an internal eval, and costs about 15 times the tokens of a chat. The same write-up notes that coding tasks offer fewer genuinely parallel subtasks than research does ([post](https://www.anthropic.com/engineering/multi-agent-research-system)).

We took those patterns for the separation rather than the parallelism. [Hermes](https://hermes-agent.nousresearch.com/docs) is the controller: it owns scope, gates, shared state, and acceptance of evidence. [omp](https://github.com/can1357/oh-my-pi) is the executor: it plans, implements, runs the checks, and reports back. The agent that does the work never grades it, which is 選手兼裁判 (the player also refereeing), the arrangement no league allows. Plan, implement, and verify still run as one sequence, because that is the order the Claude guide prescribes. At a gate the executor stops on one BLOCKED line naming the reason and the authorization it needs, and the controller resumes that same session with the missing fact instead of starting over.

Self-correction research says loops need edges. Self-Refine has a model critique and revise its own output until iteration stops paying ([paper](https://arxiv.org/abs/2303.17651)); Reflexion stores verbal reflections in an episodic memory and leaves the weights alone ([paper](https://arxiv.org/abs/2303.11366)). Both gain from iteration, and both can iterate forever. Our verify round is Anthropic's evaluator-optimizer pattern with a ceiling attached: a fresh pass reads the diff against the acceptance criteria and ends on one line, CLEAN or ISSUES. Two unsuccessful fixes for one failure force a backtrack and a different approach, and plan critique stops when material findings are resolved instead of after a quota of clean rounds.

Continuity comes from the same place GSD's state records come from: a STATE.md per task that keeps decisions, blockers, and verification results in files that outlive any single conversation ([repo](https://github.com/chudeemeke/get-stuff-done)). Every agent and session reads and writes that one file, so resuming means reading state, not rebuilding it. Questions and one-off lookups never create state; the file exists only for work that spans turns.

![Diagram: two panels connected by an arrow loop. The controller panel lists scope, gates, shared state, and evidence acceptance; the executor panel lists plan, implement, verify, report; four gate markers sit on the return path.](/images/what-swe-agent-and-anthropics-agent-guidelines-changed-in-our-rules/controller-executor-loop.svg)

*The controller holds gates, state, and evidence; the executor holds the plan and the checks. Neither side crosses.*

## What the rules bought us

Each of the six failure classes in the repository's README has a rule behind it: check-pass claims without evidence, re-diagnosis loops, repetition under stress, behavior reversed on a ticket's say-so, rule bloat, and drift between plan and code. Every one of them happened first. The rule exists because the failure did, and the rules file gets revised on a fixed cadence, so each version replaces the last.

The rules file is always-on, so every line pays rent in each session's context, which is the other reason it stays at 162. What survived the cuts: long procedures belong in skills, session memory is not state, wording decays while structure holds, and one concept keeps one name. Rules and the skills they dispatch to live in the same repository, so a rule and its workflow ship in one commit.

Unattended runs stop at four boundaries: irreversible or outbound actions, schema changes, high-cost architecture forks, and writes to external services. The executor resolves everything else itself, which is what lets a dispatch run unattended and still respect the gates.

What changed: a failure now has one place to go. It becomes a line in a public file, the line reaches every tool on the next session, and the next run inherits the fix. The whole mechanism costs 162 lines.
