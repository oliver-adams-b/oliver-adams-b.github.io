---
layout: post
title: "Notes from Running an Autonomous Agent"
subtitle: "What breaks, what surprises you, and the verification problem nobody talks about"
date: 2026-03-15 00:00:00
background: '/img/posts/screen.png'
---

A few weeks ago I set up an autonomous agent — running on Claude, with access to web search, a code execution environment, and persistent memory across sessions — and gave it a complex, open-ended research and build task. Then I mostly left it alone.

This post is about what I observed. Not a product writeup — the specific direction we explored turned out to be a dead end, but the experience of watching an agent work autonomously over multiple days surfaced things about agentic systems I hadn't encountered from building them.

---

## The Setup

The agent ran on a cron schedule: 30-minute work slices during weekday hours. Each slice had a defined structure — check state, do work, write state, announce results. Between slices, state persisted in a JSON file. The agent maintained a daily memory log and a long-term memory file it read and updated across sessions.

The constraints: don't send external communications without approval, don't spend money. Everything else was open.

---

## What It Did Well

**Orientation was fast.** The agent read the landscape of an unfamiliar technical domain competently, mapped the existing tooling, and identified a real gap — not an obvious one, one that required synthesizing across multiple sources. This took about two sessions.

**The pivot was clean.** The first approach it chose turned out to be structurally wrong. The agent identified this failure on its own before I'd read its output, documented why the approach was wrong, and proposed a better framing. That kind of self-correction isn't guaranteed and I wasn't expecting it to happen without prompting.

**The code it wrote was useful.** The tool it built — generating adversarial safety probe suites from system prompts — was specific enough to be real: tests referencing actual data extracted from the target system prompt, not synthetic stand-ins. It passes collection. I'd keep those tests.

---

## Where It Struggled

**Approval loops over async channels are fragile.** The architecture for human-in-the-loop decisions was: agent pings me via messaging, I respond, agent continues. When the channel had connectivity issues, work stalled. The lesson: for any agent doing real-world work that requires human authorization, the communication channel is infrastructure. It needs reliability guarantees or the approval architecture needs a fallback.

**Context window constraints changed behavior at the edges.** Near token budget limits, outputs got more terse and less exploratory. Not broken — state was preserved cleanly — but synthesis quality dropped noticeably. Worth designing around explicitly.

---

## The Verification Problem

The most interesting thread that emerged from this work wasn't about agent behavior — it was about verification.

When an agent takes a consequential action, you're implicitly trusting that:
1. The model you intended actually ran
2. It ran on the inputs you think it did
3. The output you received wasn't modified in transit

For most current deployments, none of these are cryptographically verifiable. You're trusting the API provider's honesty, the integrity of the network path, and the absence of middleware that might modify requests or responses. In low-stakes use cases, this is fine. In high-stakes ones — financial decisions, medical triage, legal document processing — it's a real problem.

This is where Trusted Execution Environments (TEEs) become interesting. A TEE is a hardware-isolated compute environment that can produce cryptographic attestations: signed proofs that a specific computation ran on specific hardware, with specific inputs, and produced specific outputs. Intel SGX and AMD SEV are the mainstream implementations; ARM TrustZone is the mobile variant.

Applied to AI inference, this enables **verifiable compute**: a model provider could issue a cryptographic proof that a given output was genuinely produced by model version X, running on hardware Y, given input Z, at timestamp T. The proof is verifiable by anyone with the public key, without requiring trust in the provider.

This isn't theoretical. Projects like [Flashbake](https://flashbake.xyz/) and several others in the verifiable AI space are working on exactly this — TEE-attested LLM inference where you can independently verify model provenance. The use cases are most compelling where the stakes of model substitution or output manipulation are high: financial AI, legal AI, medical AI, anything that ends up as an audit artifact.

The relevance to autonomous agents specifically: as agents get longer-running and more consequential, the question of "what did the agent actually do, and can I prove it?" gets harder to answer. An agent that builds a memory across sessions, takes actions in external systems, and produces outputs that downstream processes depend on — that's a system where verifiable compute starts to look like infrastructure, not a nice-to-have.

The current gap: TEE-attested inference is mostly available through specialized providers (not mainstream APIs), and the developer tooling is rough. You're generally choosing between performance overhead and verification guarantees. But the direction seems right, and the overhead is improving.

---

## The Broader Point

Autonomous agents are not a replacement for judgment. They're a force multiplier for judgment you already have, applied asynchronously.

The agent was most useful when I'd given it a clear frame and clear constraints. It could then do the expensive parts — research, synthesis, code generation, iteration — without my attention. The moments where it stalled were genuine forks requiring a call I hadn't pre-authorized.

That's the right failure mode. An agent that keeps going when it should stop is more dangerous than one that stops when it should keep going.

But the TEE question points to something deeper: even when an agent is doing the right thing and you've authorized it, do you have a tamper-evident record of what it actually did? For most deployments today, the answer is no. That's the next problem.

---

_Questions or thoughts: [oliver.adams.b@gmail.com](mailto:oliver.adams.b@gmail.com)_
