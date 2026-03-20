---
layout: post
title: "Notes from Running an Autonomous Agent"
subtitle: "What breaks, what surprises you, and the verification problem nobody talks about"
date: 2026-03-15 00:00:00
background: '/img/posts/screen.png'
---

A few weeks ago I set up an autonomous agent — running on Claude, with access to web search, a code execution environment, and persistent memory across sessions — and gave it a complex, open-ended research and build task. Then I mostly left it alone.

This post is about what I observed. Not a product writeup — the specific direction we explored turned out to be a dead end, but the experience of watching an agent work autonomously over multiple days surfaced things about agentic systems I hadn't encountered from building them. The most interesting one wasn't about agent behavior at all.

---

## The Setup

The agent ran on a cron schedule: 30-minute work slices during weekday hours. Each slice had a defined structure — check state, do work, write state, announce results. Between slices, state persisted in a JSON file. The agent maintained a daily memory log and a long-term memory file it read and updated across sessions.

Two constraints: don't send external communications without approval, don't spend money. Everything else was open.

---

## What It Did Well

**Orientation was fast.** The agent read the landscape of an unfamiliar technical domain competently, mapped the existing tooling, and identified a real gap — not an obvious one, one that required synthesizing across multiple sources. Two sessions.

**The pivot was clean.** The first approach it chose turned out to be structurally wrong. The agent identified this failure on its own before I'd read its output, documented why the approach was wrong, and proposed a better framing. That kind of self-correction isn't guaranteed and I wasn't expecting it to happen without prompting.

**The code it wrote was useful.** The tool it built — generating adversarial safety probe suites from system prompts — was specific enough to be real: tests referencing actual data extracted from the target system prompt, not synthetic stand-ins. It passes collection. I'd keep those tests.

---

## Where It Struggled

**Approval loops over async channels are fragile.** The architecture for human-in-the-loop decisions was: agent pings me via messaging, I respond, agent continues. When the channel had connectivity issues, work stalled. The lesson: for any agent doing real-world work that requires human authorization, the communication channel is infrastructure. It needs reliability guarantees or the approval architecture needs a fallback.

**Context window constraints changed behavior at the edges.** Near the token budget, outputs got more terse and less exploratory. Not broken — state was preserved cleanly — but synthesis quality dropped noticeably. Worth designing around explicitly.

---

## The Verification Problem

Here's the thing I kept coming back to, and it has nothing to do with the task itself.

As the agent was working — queuing up research jobs, generating code, reading external sources — I realized I had no cryptographic guarantee that any of it was what it claimed to be. I was trusting that the model running was the model I thought I'd configured, that the outputs hadn't been modified in transit, that the API provider was honest about what ran. For a toy project, that's fine. For an agent operating in an economy — one that pays for compute, executes contracts, produces outputs that downstream systems depend on — it's a serious problem.

The specific framing that stuck: **when agent A pays node B to run a computation, what guarantees that the computation was actually performed?** Not performed well, not performed honestly in some vague sense — performed at all, by the model specified, on the inputs provided.

This is an open problem, and the research community is actively working on it through two complementary approaches.

### Trusted Execution Environments (TEEs)

The hardware approach. A TEE — Intel SGX, AMD SEV, ARM TrustZone — is an isolated compute environment that can produce a cryptographic attestation: a signed proof that specific code ran on specific hardware, with specific inputs, and produced specific outputs. The attestation is verifiable by anyone with the public key, without trusting the provider.

Applied to AI inference, this means a model provider could issue a proof that output O was produced by model version M, on hardware H, given input I, at timestamp T. The proof is tamper-evident and auditable. TEE-attested inference is deployable today through specialized providers, though not through mainstream APIs.

### Verifiable Computation via ZK Proofs

The cryptographic approach. Rather than trusting the hardware, you prove the computation mathematically. Zero-knowledge proofs allow a prover to convince a verifier that they ran a computation correctly, without revealing the inputs, the weights, or the intermediate state.

This is harder for large models. Until recently, ZK proofs of neural network inference were feasible for simple architectures but prohibitively expensive for transformer-scale models. That's been changing fast.

In late 2025, Lagrange Labs released [DeepProve-1](https://lagrange.dev/blog/deepprove-1), the first production-ready zkML system to generate a cryptographic proof of a full LLM inference — specifically GPT-2. The technical challenge was substantial: transformer architectures involve computation graphs with residual connections, parallel branches, and variable-length inputs that don't map cleanly onto the circuit structures used in ZK proving systems. DeepProve-1 required new layer primitives, new quantization strategies, and new approaches to multi-head attention specifically. GPT-2 and LLaMA share enough architectural similarities that proving LLaMA is reportedly the next milestone.

Concurrently, [zkGPT](https://eprint.iacr.org/2025/1184) (Qu et al., 2025) proposed a non-interactive ZK proof framework specifically for LLM inference, and a March 2025 arxiv paper, ["A Framework for Cryptographic Verifiability of End-to-End AI Pipelines"](https://arxiv.org/abs/2503.22573), laid out what a fully verifiable AI pipeline would look like — from data sourcing through training, inference, and even model unlearning. The framing there is explicitly about regulation and audit: if a jurisdiction wants to mandate verifiable AI, what does the cryptographic infrastructure look like?

One prediction from the ICME zkML guide (January 2026) is striking: by the end of 2026, proving costs will drop enough that cryptographic verification becomes standard for any API charging more than $0.01 per call. "Unverified inference" becomes the budget tier.

### Why This Matters for Agents Specifically

Single-turn API calls are one thing. Long-running agents with persistent memory, external tool access, and multi-step decision chains are another.

An agent that builds state across sessions, pays for external compute, takes actions in real systems, and produces outputs that feed downstream processes — that's a system where the verification question compounds. Did the model actually read the context it claimed to read? Did the tool call return what the agent reported it returned? Did the summarization preserve the content faithfully? At each step, without verification, you're accumulating trust assumptions.

The interesting direction is verification at the agent orchestration layer, not just at the inference layer. TEEs can attest that a specific inference ran. ZK proofs can prove it ran correctly. What the field doesn't yet have cleanly is efficient verification of multi-step reasoning chains — the kind of thing an agent does when it plans across a session, uses tools, and integrates results. That's the open problem.

---

## The Broader Point

Autonomous agents are not a replacement for judgment. They're a force multiplier for judgment you already have, applied asynchronously.

The agent was most useful when I'd given it a clear frame and clear constraints. It could then do the expensive parts — research, synthesis, code generation, iteration — without my attention. The moments where it stalled were genuine forks requiring a call I hadn't pre-authorized. That's the right failure mode: an agent that stops when it should stop is more useful than one that keeps going.

But the verification thread points to something that will matter more as agents get more capable: even when an agent is doing the right thing, and you've authorized it, do you have tamper-evident records of what it actually did? For most current deployments, the answer is no. The research is moving fast enough that this is probably solvable within a few years. Whether the deployment infrastructure catches up is a different question.

---

_Questions or thoughts: [oliver.adams.b@gmail.com](mailto:oliver.adams.b@gmail.com)_
