---
layout: post
title: "From Notebook to Production: Building an Enterprise Generative AI Platform"
subtitle: "What it actually takes to turn an LLM proof-of-concept into a system that organizations can rely on"
date: 2025-06-15 00:00:00
background: '/img/posts/screen.png'
---

Most generative AI projects start the same way: a notebook, an API key, and a working demo that impresses everyone in the room. The gap between that demo and a system that operates reliably inside an organization at scale is where the real engineering happens — and it's larger than most teams anticipate.

I spent the better part of 2024 and 2025 closing that gap at a large insurance technology company. This post is about what that actually looked like.

---

## The Starting Point

The project was claims processing automation: taking FSA reimbursement claims — unstructured documents, photos, receipts — and determining whether they qualified for reimbursement. The initial system was straightforward: a LangChain-based notebook that called an LLM with a structured prompt, parsed the output, and returned a decision.

It worked. In a demo environment, against clean test data, it got the decisions right. That was enough to get buy-in for taking it to production.

What wasn't visible in the demo: there was no error handling. No retry logic. No input validation. No monitoring. No way to audit decisions after the fact. The "system" was a Python script that assumed the input was well-formed and the API would always respond correctly.

These assumptions are reasonable in a notebook. They're disqualifying in production.

---

## Rearchitecting for Reality

The first thing I did was throw away the notebook and start over with production constraints as the design brief, not an afterthought.

**The service layer** became a FastAPI application with strict request and response validation through Pydantic. Every input that arrived at the API boundary was validated before it touched any business logic. This sounds tedious — and it is — but it's also the thing that makes debugging tractable when something breaks at 2am on a Sunday.

**The orchestration layer** moved from LangChain to LangGraph. The reason: LangGraph makes multi-step reasoning workflows explicit as a graph, which means you can reason about them, test them, and modify individual steps without risking side effects in adjacent ones. For a system that needed to ingest documents, retrieve context, reason over structured and unstructured data, and produce an auditable output, having that structure was essential.

**Evaluation** was the part that took longest to get right. Offline evaluation — ground truth datasets, controlled testing environments — was straightforward enough. The harder problem was online evaluation: how do you know the system is still performing correctly on the actual distribution of claims it's seeing in production, weeks after you deployed it?

I built a monitoring pipeline that compared model outputs against downstream outcomes (claims that were appealed, cases flagged for manual review, patterns in approval/rejection rates). This closed the feedback loop between production behavior and the next iteration of the system. Without it, "97% accuracy in testing" is a number that ages poorly.

**Infrastructure**: Docker containers on Kubernetes, Helm for deployment configuration, Terraform for infrastructure provisioning across Azure and AWS. GitHub Actions CI/CD with custom runners that mirrored the production environment. The goal was that no deployment happened manually, and no environment was meaningfully different from any other.

---

## The Broader Platform Problem

Once the claims system was in production, the next problem became visible: teams across the organization were independently building AI prototypes. Each team was making different infrastructure choices, building different deployment patterns, solving the same problems in different ways. The organization was accumulating AI technical debt faster than it was building AI capability.

The natural response was a platform — a standardized framework for building AI-backed services that encoded the patterns we'd learned from the claims system.

The platform centered on a cookiecutter repository template. When a team started a new AI service, they began from a template that already had the right structure: modular separation between platform logic and application logic, consistent patterns for configuration, logging, validation, and inference, Poetry for dependency management, a Makefile-based developer interface that ran identically locally and in CI.

The critical design decision was what to standardize and what to leave flexible. We standardized everything that teams were getting wrong in isolation — authentication, rate limiting, error handling, observability, deployment pipelines. We left flexible everything that was actually specific to the problem the team was solving — the model choice, the prompt structure, the business logic.

The result: teams that previously took six months to go from prototype to production were doing it in a week.

---

## What I Learned

A few things that weren't obvious until I'd done this:

**The evaluation problem is harder than the modeling problem.** Getting an LLM to produce correct outputs on a test set is the easy part. Knowing whether it's still producing correct outputs on the real distribution, six months later, after the upstream data has shifted in ways you didn't anticipate — that's the hard part. Build the evaluation infrastructure before you need it.

**Standardization has to earn its constraints.** Every layer of standardization in the platform is something a team has to live with. If those constraints don't buy real value — if a team is slower because of the platform, not faster — the platform will be worked around. The only way to prevent that is to make the standards genuinely useful, which means understanding what problems teams are actually solving.

**The organizational problem is at least as hard as the technical problem.** Getting an AI system to work is engineering. Getting an organization to trust it, to route real decisions through it, to update it when it degrades — that's something else. The technical work is the easier prerequisite.

---

The full system has been in production for over a year. The claims processing accuracy has held at 97%. The platform has enabled multiple teams across the organization to ship AI systems that would otherwise have taken quarters. 

That's the bar for "production AI" — not that it works in a demo, but that it keeps working when you're not watching it.

---

_For questions or to talk through any of this, reach out at [oliver.adams.b@gmail.com](mailto:oliver.adams.b@gmail.com)._
