---
title: Home
layout: default
nav_order: 1
description: "A pattern for calling LLMs at scale from RPG batch jobs on IBM i, using PHP as the transport layer and data queues as the coordination point."
permalink: /
---

# IBM i AI Workers
{: .fs-9 }

A pattern for calling LLMs at scale from RPG batch jobs on IBM i, using PHP as the transport layer and data queues as the coordination point.
{: .fs-6 .fw-300 }

[Get started](#start-here){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/K3S/ibmi-ai-workers){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What this is

This guide documents the architecture King III Solutions Inc. (K3S) built for processing AI-driven purchasing checks across our customer base — running on IBM i, integrating with cloud and on-premises LLMs, multi-tenant from day one, and designed to scale into the tens of thousands of checks per batch.

It's **descriptive**: it tells the K3S story honestly, including the decisions that turned out to be wrong.

It's **prescriptive in flavor**: where a choice generalizes beyond K3S, we say so and we recommend it.

It's **published as it's built**. Some chapters are detailed. Some are stubs. Some don't exist yet. We'd rather publish in motion and let you see the reasoning unfold than wait until everything is settled and lose the thinking that produced it.

## Who this is for

You'll get the most out of this guide if:

- You're an IBM i developer or architect, comfortable with RPG, DB2, and SBMJOB.
- You've made at least one synchronous LLM call from RPG already — perhaps using `SYSTOOLS.HTTPGETCLOB` or the integrated HTTP APIs.
- You're not satisfied with calling AI once per row, sequentially. You need concurrency, multi-tenancy, retries, observability, and cost control.
- You're willing to operate across two languages — RPG and PHP — and want to understand where the boundary belongs and why.

If you've never called an LLM from IBM i at all, this guide will get ahead of you. Start with one of the excellent single-call tutorials from the IBM i community ([Seiden Group](https://www.seidengroup.com), [Liam Allan](https://worksofliam.com), [Scott Klement](https://www.scottklement.com), [Profound Logic](https://www.profoundlogic.com)) and come back when you want to scale it.

If you're an IBM i shop just starting your modern-RPG journey, you may also want our companion site [**RPG Tutorial**](https://rpgtutorial.k3s.com) — a free, open-source guide to modern RPGLE and CL development with VS Code.

## What you'll learn

- **Architecture.** Why RPG owns business logic, why PHP owns transport, why data queues sit between them, and what each side does and doesn't know.
- **The contract.** The exact message format passed between RPG and PHP across data queues — the API between the two halves of the system.
- **The PHP worker.** A long-lived CLI process that pulls from a queue, calls an AI provider, and writes results back. What it does, what it deliberately doesn't, and why it stays small.
- **The RPG worker pool.** How a batch initiator submits N parallel RPG worker jobs, how each worker pulls from a shared queue, how they coordinate completion, and how they handle failure.
- **Multi-tenancy.** How library list awareness lets one shared PHP worker serve all customers without knowing about any of them.
- **AI provider concerns.** Key custody for customers who bring their own API key (BYOK), rate limit fairness when customers share an account, retry and backoff middleware, and provider abstraction.
- **Operations.** Observability, usage logging, cost telemetry, monitoring, restart behavior, and what production failure modes actually look like.

## Start here

If you're new to the guide, read in this order:

1. [**Foundations**]({% link foundations/index.md %}) — what you need to know and have running before this guide is useful.
2. [**Quickstart — One worker**]({% link quickstart-1/index.md %}) — see the architecture work end-to-end on your own system.
3. [**Quickstart — Five workers**]({% link quickstart-2/index.md %}) — see the architecture's parallelism for free, with no code changes.
4. [**Architecture overview**]({% link architecture/index.md %}) — now that you've seen it work, here's why it's shaped this way.
5. [**The data queue contract**]({% link contract/index.md %}) — the API between the two halves. The most important chapter to get right.

After that, the chapters can be read in any order based on what you're building next.

## Status

Last updated **May 7, 2026**.

| Chapter | Status |
|:---|:---|
| [Foundations]({% link foundations/index.md %}) | V1 |
| [Quickstart — One worker]({% link quickstart-1/index.md %}) | V1 |
| [Quickstart — Five workers]({% link quickstart-2/index.md %}) | V1 |
| [Architecture overview]({% link architecture/index.md %}) | V1 |
| [The data queue contract]({% link contract/index.md %}) | V1 |
| [The PHP worker]({% link php-worker/index.md %}) | V1 |
| [The RPG worker pool]({% link rpg-pool/index.md %}) | V1 |
| [Multi-tenancy]({% link multi-tenancy/index.md %}) | V1 |
| [AI provider concerns]({% link providers/index.md %}) | V1 |
| [Operating in production]({% link operations/index.md %}) | V1 |
| [Reference]({% link reference/index.md %}) | V1 |

## Why we're publishing this

Three honest reasons:

1. **Writing forces clarity.** Decisions made in our heads stay fuzzy. Decisions written down for strangers have to be specific.
2. **The IBM i community needs this.** A lot of strong material exists on calling AI services once from RPG. Very little exists on doing it at scale, multi-tenant, in production. We're filling a gap that's real.
3. **It's how we hand off internally.** K3S engineers who join after this is built need to learn the system. The same artifact that helps a stranger on the internet helps a new hire at K3S.

We're not publishing because the architecture is finished, perfect, or universally applicable. It's none of those things. We're publishing because the reasoning behind our choices is, we think, more useful than the choices themselves.

## About K3S

[King III Solutions Inc.](https://k3s.com) (K3S) is a B2B supply chain replenishment software company, founded in 1990 and based in Marietta, Georgia. K3S serves mid-market distributors across North America with purchasing optimization software running on IBM i.

This guide is one of K3S's open-source contributions to the IBM i community. Our other public site, [RPG Tutorial](https://rpgtutorial.k3s.com), teaches modern RPGLE and CL development.

## License

The prose of this guide is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). Code samples are licensed under the [MIT License](https://opensource.org/licenses/MIT).
