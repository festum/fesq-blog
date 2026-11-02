---
title: How We Keep Secrets Out of Our Agents' Context
description: A key an agent has read is a key that can leak, even out of "encrypted" reasoning. Five walls we run today, the gaps we haven't closed, and why policy beats begging.
pubDate: 2026-11-01
tags: [security, agents, engineering]
draft: false
---

In August 2026 a research team published a trick that should make anyone running agents sit up. Providers hide a model's chain of thought by returning it as an encrypted block that the client stores and sends back on the next request.

The blocks turned out to be interchangeable across sessions, users, and models within one provider's ecosystem. Feed a strong model's block to the provider's weaker, less guarded sibling and it transcribes the hidden reasoning in plaintext. The authors scraped 315,320 such blocks from public repositories and recovered 367 PII artifacts and 182 credentials, including 62 API keys, 33 passwords, and 30 personal emails. Some of that data existed only inside the encrypted reasoning, so the person who shared the log could not have seen it before publishing ([paper](https://arxiv.org/abs/2608.09867)).

Encryption kept the blocks private from casual readers. It never protected them from the vendor's own model. That is the working definition of our problem: anything that entered an agent's context can come back out, through the model, through logs, through a replayed block. The only reliable way to stop a leak is to make sure the secret was never there.

## Begging is not a defense

Temptation arrives first: write a rule into the system prompt that says never reveal credentials. Simon Willison named that instinct in 2023. The first solution people try, he said, is what he likes to call "prompt begging" ([talk transcript](https://simonwillison.net/2023/may/2/prompt-injection-explained/)). Attackers get to place their content after yours, and no instruction at the top survives an unlimited bag of tricks applied below it.

His later framing explains why the risk compounds. The lethal trifecta is three legs: access to private data, exposure to untrusted content, and a channel to the outside. Remove any one leg and the attack dies ([lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)). A secret sitting in context is the first leg, handed over before the attack starts.

Research agrees with the pessimism. Nasr et al. took twelve published prompt-injection defenses and attacked each one adaptively, tuning the attack to the defense instead of reusing a fixed payload. Most fell above 90% attack success rate, while the original papers had reported near-zero ([USENIX Security '26](https://www.usenix.org/conference/usenixsecurity26/presentation/nasr)). A system prompt is a request to the model, and the model is the component under attack. Requests are not access control.

So the defense has to live somewhere deterministic, where a wrong answer from the model cannot change the outcome. Our version starts with one law: the secret never enters the context window. The model handles intent and placeholders; a layer it cannot talk to supplies the values at the last moment. Five walls hold that law up.

## Five walls around the context

**Wall one: the gateway holds the keys.** Every consumer of our LLM gateway authenticates with its own virtual key. The database carries ten of those against eighteen upstream provider keys, and the agents only ever see their own virtual key. An upstream credential exists in exactly one place, the gateway's config store, and nowhere in any agent's reach. Revoking a compromised agent is one row, not a provider-wide key rotation. Routing follows the same rule: a fixed priority chain in configuration decides where each request goes, so the model never gets a vote in which backend receives its output.

**Wall two: the harness says no in code.** Our agent harness carries a blocklist with no override switch, covering commands like a recursive wipe of the root filesystem or a fork bomb. On top of it sit user-defined deny rules, and those apply before any "run without asking" mode, so a yolo flag cannot lift them. Certain paths never take a write at all. SSH keys, cloud credentials, sudoers, `.env` files, and the caches where password managers keep decrypted entries all sit on that list. Even the optional write sandbox, which pins every file write to a project directory, cannot override these blocks. For jobs that run with nobody watching, the default answer to a dangerous command is deny: scheduled jobs, one-shot queries, and unattended sessions all fail closed. The coding executor has its own version of the same ladder, an approval mode per tool where deny always wins, plus ordered command patterns where the first match decides. Two gaps in that wall are honest work in progress, more below.

**Wall three: the environment gets scrubbed.** Credentials leak sideways as often as they leak forward. MCP server subprocesses receive a filtered environment with only eight basics plus XDG variables through; every API key, token, and secret is stripped before the process starts. The same stripping runs for scripted code execution, so a generated script that calls `printenv` finds no credential worth printing. Error messages coming back from MCP tools get sanitized into a single redacted marker before the model ever reads them. Forwarding a variable into a container stays opt-in by name, because anything you forward can be read and shipped onward by whatever process happens to run inside that container.

**Wall four: exits get redacted.** Secret detection runs on our own config, and it is switched on. Conversation state and task files carry a standing rule against storing tokens, passwords, or keys at all, which keeps a leak from laundering itself into persistent memory. The pipeline that publishes this blog redacts credentials before the first search query, not before publication, because a secret that reached a search engine cannot be recalled.

**Wall five: humans keep the expensive gates.** Four boundaries in our shared rules file stop a run and ask for authorization: anything irreversible or outbound, any change to data models or schemas, any large mutually exclusive architecture fork, and any write to an external service. When a headless run hits one, it halts with a single line saying what it needs instead of guessing. The gate behaves like a German Bürgeramt (the citizen office where you queue for paperwork): no form, no service, no matter how urgent your story is. Guessing is how secrets leak, so the halt is deliberate.

![Cross-section diagram: five concentric walls around a central context window. Wall one is the gateway holding provider keys, wall two the harness deny floor, wall three environment scrubbing, wall four exit redaction, wall five human gates. An arrow labeled "secret" stops at wall one.](/images/secrets-out-of-context/five-walls.svg)

*The secret's path stops at the outer wall; everything inside the circle works with intent and placeholders instead of values.*

## The gaps we haven't closed

Honest accounting says three things are missing.

Our coding executor has no deny patterns configured yet. The ordered command rules exist in the product and ship documented, and our config simply hasn't got them, so today that layer rests on the harness walls and the human gates rather than on its own. A deny rule that catches a vault dump command, for example `bw get item` split into steps that print intermediate values, is the obvious first entry.

The website blocklist sits disabled too. With it off, nothing stops a web or browser tool from fetching a public hostile domain; the private-network guard still refuses internal addresses, but that protects the LAN, not the destination. The config key exists and is set to false, which makes this the cheapest gap to close.

And the vault workflow still needs its sealed wrapper. Bitwarden illustrates why the wrapper matters: the management API has no partial-update endpoint, edits are a full-document replace, and a plain item fetch returns the entire record including the password. The safe pattern is a pipeline where the fetch, the edit, and the write-back run as one command and the plaintext lives only inside that command's pipe. What we have today is process discipline, not enforcement. The missing piece is a trusted wrapper that returns success or failure to the model and nothing else, backed by a deny rule so that even a split-and-print attempt hits a wall. Blind injection is the name for this: the model emits a command with a placeholder, the wrapper supplies the value, and the output is sanitized on the way back.

![Flow diagram: the agent emits a command containing a placeholder, a trusted wrapper fetches and edits inside one pipeline while the vault holds the record, and only a sanitized "ok / failed" returns to the model. A dashed red path showing an attempt to split the pipeline and print values is marked denied before execution.](/images/secrets-out-of-context/blind-injection.svg)

*The wrapper keeps plaintext inside one box; anything that tries to split the pipe dies at the deny rule.*

A single rule ties the layers together: a credential that has entered any context is treated as compromised. The response is revoke and rotate first, cleanup second. The block replay research is the reason the rule is unconditional. You cannot audit your way back from a secret that a weaker model may already have transcribed.

## What exists off the shelf

Nobody has to build the walls from scratch, and nobody should expect a single product to hold all five. A Sophos assessment this year is the sober summary: no production-grade implementation of sealed tool endpoints exists yet, the CaMeL design has no production deployment, and the mainstream coding harnesses have not adopted the dual-model pattern ([Sophos blog](https://www.sophos.com/en-us/blog/inside-the-lethal-trifecta-blast-radius-reduction-in-ai-agent-deployments)).

These projects split by which wall they reinforce. For deterministic gating there is [Helio](https://github.com/gethelio/helio), a governance proxy that sits between an unmodified agent and its tools, evaluating YAML policy on every tool call, and [Chock](https://github.com/open-coder-ai/chock), which compiles committed policy into hooks and CI gates so the rules travel with the repository. [cc-safety-net](https://github.com/kenryu42/cc-safety-net) parses commands before execution to block destructive operations and credential reads, and [Cedar for agents](https://github.com/cedar-policy/cedar-for-agents) applies Cedar, a formally specified policy language, to tool authorization for agents. For key isolation, [Infisical](https://github.com/infisical/infisical) ships an agent vault where a forward proxy holds the credentials and attaches them on the way out, so the agent never holds them. The curated [awesome-ai-agent-governance](https://github.com/systempromptio/awesome-ai-agent-governance) list gathers the rest in one place.

One public example shows why any of this matters beyond theory. A popular coding agent stores full prompts and responses in object storage when you authenticate with its official token path, with no opt-out, while a direct bring-your-own-key connection to the provider skips the storage entirely ([issue #14281](https://github.com/anomalyco/opencode/issues/14281)). The maintainers' position is that this covers free-tier models and is documented. Whatever you think of that position, the lesson transfers: know which path your traffic takes and who keeps a copy.

## The ladder

Four rungs, in the order the secret would travel. It never enters, because keys live behind the gateway and models handle placeholders. It cannot get in from the side, because commands, paths, and environments are filtered by code that does not negotiate. It cannot get out, because exits are redacted and state files refuse to hold it. And if it somehow does get out, the response is already decided: revoke and rotate before anything else.

Every wall is boring on its own. A virtual key, a deny list, a stripped environment, a redaction pass, a human gate. Together they separate an agent that reads a secret from an agent that merely acts on one.
