# IBM i AI Workers

A pattern for calling LLMs at scale from RPG batch jobs on IBM i, using PHP as the transport layer and data queues as the coordination point.

This guide is descriptive: it documents the architecture we built at King III Solutions Inc. (K3S) for processing AI-driven purchasing checks across our customer base. It's prescriptive in flavor: where we made a choice for reasons that generalize, we say so, and we recommend it. Where we made a choice that's specific to K3S, we say that too, and leave the alternative open to you.

It is being written **as the system is being built**. Some chapters are detailed; some are stubs; some don't exist yet. The status of each chapter is marked at the top of the page. This is deliberate — we'd rather publish in motion than wait until everything is settled and risk losing the reasoning behind the decisions.

## Who this is for

You'll get the most out of this if:

- You're an IBM i developer or architect, comfortable with RPG, DB2, and SBMJOB.
- You've made at least one synchronous LLM call from RPG already — perhaps using `SYSTOOLS.HTTPGETCLOB` or the integrated HTTP APIs. You've seen a single round trip work, and you're now wondering how to do it for real, in production, at volume.
- You're not satisfied with "call AI once per row, sequentially." You need concurrency, multi-tenancy, retries, observability, and cost control.
- You're willing to operate across two languages — RPG and PHP — and want to understand where the boundary belongs and why.

If you're earlier in the journey — for example, you've never called an AI service from IBM i at all — this guide will probably get ahead of you. Start with one of the excellent single-call tutorials from the IBM i community (Seiden Group, Liam Allan, Scott Klement, Profound Logic) and come back when you want to scale it.

## What this guide covers

- **Architecture.** Why RPG owns business logic, why PHP owns transport, why data queues sit between them, and what each side does and doesn't know.
- **The contract.** The exact message format passed between RPG and PHP across the data queues — the "API" between the two halves of the system.
- **The PHP worker.** A long-lived CLI process that pulls from a queue, calls an AI provider, and writes results back. What it does, what it deliberately doesn't, and why it stays small.
- **The RPG worker pool.** How a batch initiator submits N parallel RPG worker jobs, how each worker pulls from a shared queue, how they coordinate completion, and how they handle failure.
- **Multi-tenancy.** How library list awareness lets one shared PHP worker serve all customers without knowing about any of them, and how per-customer AI profiles are resolved at runtime.
- **AI provider concerns.** Key custody for customers who bring their own API key (BYOK), rate limit fairness when customers share an account (hosted), retry and backoff middleware, and provider abstraction for swapping between Anthropic, OpenAI, and on-prem models.
- **Operations.** Observability, usage logging, cost telemetry, monitoring, restart behavior, and what production failure modes actually look like.

## What this guide doesn't cover

- **Prompt engineering for purchasing.** The specific prompts K3S uses for purchasing-exception checks are part of our product. The mechanics of getting a prompt to an AI and back are general; the contents of those prompts are not.
- **Choosing an AI provider.** We don't compare Anthropic to OpenAI to others. Use the one that fits your needs.
- **IBM i fundamentals.** We assume working knowledge of RPG, DB2, library lists, SBMJOB, and PASE. We'll explain anything that's unusual *in this context*, but we won't teach the platform.
- **The basics of LLMs.** We treat them as black boxes that take a prompt and return a response. If you want to understand transformers, look elsewhere.

## Why we're publishing this

Three honest reasons:

1. **Writing forces clarity.** Decisions made in our heads stay fuzzy. Decisions written down for strangers have to be specific. The guide is, first, how K3S thinks carefully about its own architecture.
2. **The IBM i community needs this.** A lot of strong material exists on calling AI services once from RPG. Very little exists on doing it at scale, multi-tenant, in production. We're filling a gap that's real.
3. **It's how we hand off internally.** K3S engineers who join after this is built need to learn the system. The same artifact that helps a stranger on the internet helps a new hire at K3S.

We're not publishing because the architecture is finished, perfect, or universally applicable. It's none of those things. We're publishing because the reasoning behind our choices is, we think, more useful than the choices themselves.

## Status

This guide is in active build. As of May 7, 2026, the published chapters are:

- [ ] Introduction
- [ ] Foundations
- [ ] Architecture overview
- [ ] The data queue contract
- [ ] The PHP worker
- [ ] The RPG worker pool
- [ ] Cross-cutting concerns
- [ ] Multi-tenancy
- [ ] AI provider concerns
- [ ] Operating in production
- [ ] Reference

Check the box, link the page, and update the date as each chapter lands.

## Contributing, questions, corrections

If something is wrong, unclear, or missing context that would have helped you, please open an issue. We'd rather hear about it than not.

If you've built something similar in your shop and want to compare notes, we'd genuinely like that. Reach out via issues, or via the K3S contact channels at [k3s.com](https://k3s.com).

## License

The prose of this guide is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). You're free to share, adapt, translate, and build on this material — including for commercial purposes — as long as you provide attribution to K3S and link back to this guide.

Code samples in this guide are licensed under the [MIT License](https://opensource.org/licenses/MIT). Use them in your own projects, proprietary or open, with no further obligation.

See `LICENSE` and `LICENSE-CODE` in the repository root for full text.

---

*Maintained by the team at [King III Solutions Inc.](https://k3s.com). Published at [ibmi-ai-workers.k3s.com](https://ibmi-ai-workers.k3s.com).*
