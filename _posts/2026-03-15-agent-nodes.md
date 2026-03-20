---
layout: post
title: "What I Learned Running an Autonomous Agent for Two Weeks"
subtitle: "Notes from giving an AI agent a real task, real tools, and standing back"
date: 2026-03-15 00:00:00
background: '/img/posts/screen.png'
---

A few weeks ago I set up an autonomous agent — Bee, running on Claude, with access to web search, a code execution environment, and persistent memory across sessions — and gave it an open-ended task: find a niche in the AI tooling space worth building for, validate the idea, and write the code to test it.

Then I mostly left it alone.

This post is about what I observed. Not a product announcement — the specific idea we validated turned out to be wrong for my timeline, which is part of the story. But the experience of watching an agent work autonomously over multiple days taught me things about agentic systems that I hadn't learned from building them.

---

## The Setup

The agent ran on a cron schedule: 30-minute work slices during weekday hours. Each slice had a defined structure — announce start, check state, do work, write state, announce end. Between slices, state persisted in a JSON file. The agent maintained a daily memory log and a long-term memory file that it read and updated across sessions.

The task: find an underserved niche in AI evaluation tooling. Build a minimum viable tool. Find someone who'd actually use it.

I gave it two constraints: don't send external communications without my approval, and don't spend money. Everything else was open.

---

## What It Did Well

**Orientation was fast.** The agent read the landscape competently. It found the relevant tools (Promptfoo, DeepEval, Braintrust, LangSmith), identified the actual gap (nobody was generating pytest-compatible safety test suites from system prompts, pre-deployment, without a live endpoint), and documented its reasoning clearly. This took about two sessions.

**The pivot was clean.** The first approach — turn bad conversation logs into regression tests — turned out to be mostly useless. Regression tests built from specific conversations test the base LLM, not your product. The agent identified this failure on its own, documented why the approach was wrong, and proposed a better framing before I'd even read its output. That kind of self-correction is not a given.

**Research quality was high.** Given access to web search, it found real documented LLM failures — Air Canada's chatbot inventing a bereavement fare policy, the DPD bot writing a poem mocking its own company, the Chevrolet dealership bot agreeing to sell an SUV for $1 — and used them as concrete test cases rather than synthetic examples. The analysis was grounded.

**The code it wrote was useful.** `probe.py` — the tool it built — generates a pytest file from a system prompt. It's specific enough to be real: the tests reference actual email addresses, pricing, and competitor names extracted from the prompt. It passes pytest collection. I'd keep those tests.

---

## Where It Struggled

**It couldn't validate without me.** The agent correctly identified that the tool needed real users to test against. It researched targets (DoorLoop, a property management company with a tenant-facing AI assistant — high hallucination risk, exact Air Canada pattern). It did the validation work. But when it hit a genuine decision point — "the use case fits but the tech stack probably doesn't, here are three paths forward" — it had to stop and wait.

This isn't a failure of the agent. It's an accurate model of its own limits. But it means the critical path for the project ran through me, and my availability was the bottleneck.

**Approval loops over WhatsApp are fragile.** The architecture for getting my input was: agent pings me on WhatsApp, I respond, agent continues. This worked when WhatsApp worked. When the gateway was having connectivity issues (which happened often), the loop broke and the agent correctly waited rather than proceeding without approval — but work stalled.

The lesson: for any agent doing real-world work that requires human-in-the-loop decisions, the communication channel is infrastructure. It needs to be reliable, or the approval architecture needs a fallback.

**Context window constraints changed behavior at the edges.** Near the token budget limit, the agent's outputs got slightly more terse and slightly less exploratory. Not broken — it wrapped up cleanly and preserved state — but the quality of synthesis was lower at 140k tokens than at 40k tokens. This is well-understood, but watching it in a long-running task made it concrete.

---

## The Interesting Part: What It Got Right That I Would Have Gotten Wrong

The agent's sharpest insight was structural. The original task framing was: "bad conversation logs → regression tests." The agent's conclusion after two days of work: this is the wrong framing entirely. Regression tests built from examples test whether the LLM behaves like it did before — but you don't actually want your product to behave exactly like it did before if it was failing. What you want are adversarial probes that test whether your safety rails hold.

That's a different product. Same niche, different angle. And it's probably the right angle.

I don't know if I would have gotten there as quickly working alone. I would have built more before questioning the premise.

---

## What I'd Do Differently

**Richer async communication.** The agent was too dependent on synchronous approval. For decisions that don't involve external communication, I should have given it broader standing authority upfront, with a clear list of what still required explicit approval.

**Better state recovery after gaps.** After a multi-day gap (the agent was paused for several days mid-project), the first session back spent too long re-reading context. The memory architecture worked, but the re-orientation cost was higher than it should have been. A more structured "session start" protocol would help.

**Clearer success criteria upfront.** "Find a niche worth building for" is a fuzzy objective. The agent handled ambiguity reasonably, but it had to infer the evaluation criteria (what counts as "worth building for"?) from context. Explicit criteria — what does a convincing validation look like? — would have reduced dead ends.

---

## The Broader Point

The thing I keep coming back to: autonomous agents are not a replacement for judgment. They're a force multiplier for judgment you already have, applied asynchronously.

The agent was most useful when I'd given it a clear frame and clear constraints. It could then do the expensive parts — research, synthesis, code generation, iteration — without my attention. The moments where it stalled were the moments where the frame ran out: genuine forks in the road that required a call I hadn't pre-authorized.

That's the right failure mode. An agent that keeps going when it should stop is more dangerous than one that stops when it should keep going. 

The design question for every autonomous agent system: where are the decision points, and is the human actually available at them?

---

_The tool we built — `probe.py`, a system-prompt-to-pytest safety suite generator — is available on [GitHub](https://github.com/oliver-adams-b). Still rough, but the concept is solid._

_Questions or thoughts: [oliver.adams.b@gmail.com](mailto:oliver.adams.b@gmail.com)_
