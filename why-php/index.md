---
title: Why PHP for the delivery layer
nav_order: 5
has_children: false
---

# Why PHP for the delivery layer
{: .no_toc }

**Status:** Draft V1

You've now built and run the architecture in pure RPG. It works. For many use cases it's the right answer. But this chapter is about what happens when pure RPG starts straining, and what we do about it.

K3S landed on PHP as the delivery layer between RPG and the AI provider. That decision wasn't obvious. We're an IBM i shop with deep RPG expertise. Adding PHP introduces a second toolchain, a second language for our team to maintain, and a queue boundary that wouldn't otherwise exist. The benefits had to outweigh those costs, or we'd have stayed pure RPG.

This chapter walks through the math, the operational reality, and the architectural reasoning that pushed us toward PHP. It's the inflection point of the guide: before this chapter, the work is RPG-only; after this chapter, PHP enters the picture.

If pure RPG meets your needs, you can stop here. The next two chapters ([Quickstart 1 (RPG + PHP)]({% link quickstart-1/index.md %}) and [Quickstart 2 (RPG + PHP)]({% link quickstart-2/index.md %})) revisit the same demos with PHP added. The chapter after those returns to architecture and theory for everyone.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What pure RPG does well

Before making the case for PHP, an honest accounting of what pure RPG does *right*. This isn't backhanded — these are real strengths that any decision to add PHP has to justify trading away.

**One language, one toolchain.** Your team already writes RPG. The build pipeline already compiles RPG. The deployment story is the standard `CRTBNDRPG`. Adding PHP introduces Composer, autoloading, a vendor directory, and a new mental model for code organization. None of that is hard, but it's all overhead.

**No language boundary to maintain.** When everything is RPG, you don't need a contract specifying what crosses between systems. You don't need to think about CCSID conversion at the boundary, JSON encoding, or queue formats. The data structures stay native.

**Lower operational footprint per request.** Each RPG call is one job, one DB2 connection, one HTTPS call. There's no second layer to monitor, restart, or fail.

**Familiar debugging.** When something breaks, your team reaches for `STRDBG`, joblogs, and standard IBM i tools. Adding PHP means PHP stack traces, Composer dependency issues, and PASE-specific quirks become part of the debugging surface.

**Direct to AI.** No proxy. The HTTPS call goes from your IBM i straight to the provider. One layer of network, one layer of code. Simpler to reason about.

If your AI workload fits within what pure RPG can comfortably handle, these benefits are real. The PHP layer is a tax you'd be paying for capabilities you don't need.

---

## Where pure RPG starts straining

Three things bend pure RPG toward "not enough" as scale grows. They compound.

### 1. TLS handshake overhead per call

Every call to `QSYS2.HTTP_POST` (or `SYSTOOLS.HTTPPOSTCLOB`) opens a fresh TLS connection. The SQL HTTP services don't expose connection pooling — you can't say "reuse the connection from the last call." Each call pays:

- DNS resolution: ~10-50ms
- TCP connection: ~30-100ms (network-dependent)
- TLS handshake including certificate validation: ~100-200ms
- HTTP request and response: actual AI time

Total non-AI overhead per call: **~150-350ms**, every call.

For one call, this is invisible. For five calls in a demo, it's noticeable but acceptable. For 10,000 calls in a batch, it's **25-60 minutes of pure handshake overhead** spread across however many workers you have.

You can mitigate this somewhat by running more workers in parallel, but the *total system overhead* doesn't go down — you just pay it across more concurrent jobs. Each worker's HTTPS connection is still cold every call.

A long-lived process holding open a Guzzle pool reuses connections. After the first call, subsequent calls skip the handshake entirely. Same per-call cost as the AI itself plus a few milliseconds. That's the first thing PHP gives you.

### 2. Concurrency-per-process is bounded by RPG's call model

In RPG, each worker job does one HTTP call at a time. To call AI for 100 rows simultaneously, you need 100 RPG workers, each holding its own HTTPS connection, each consuming an SBMJOB slot.

That's not a problem for moderate scale. 20 workers, 50 workers — your IBM i handles them fine. But:

- 200 workers means 200 active jobs, 200 HTTPS connections, 200 instances of TLS context, 200 entries in `WRKACTJOB`.
- 500 workers starts pressing against `MAXJOBS`, memory pool sizing, and DB2 connection limits.

A PHP worker using Guzzle's connection pool can hold 30+ HTTP calls in flight from a single process. Five PHP workers handle 150 in-flight calls; eight handle 240. The IBM i sees five or eight jobs, not 150 or 240.

(In practice, we recommend starting with a pool size of 10-12 per worker and scaling up after measurement shows the pool is the bottleneck, not the AI provider's rate limit. The 30+ number is a ceiling on what's achievable, not the recommended starting point. See [Architecture overview]({% link architecture/index.md %}#sizing-the-system) for the full sizing math.)

The math:

| | Pure RPG (1 call per worker) | RPG + PHP (30 calls per PHP worker) |
|:---|:---|:---|
| 50 in-flight AI calls | 50 RPG jobs | ~10 RPG jobs + 2 PHP processes |
| 150 in-flight AI calls | 150 RPG jobs | ~30 RPG jobs + 5 PHP processes |
| 300 in-flight AI calls | 300 RPG jobs | ~60 RPG jobs + 10 PHP processes |

(The "RPG jobs" in the PHP column are smaller — they're waiting on a queue reply, not holding HTTPS connections, so they're lighter.)

For K3S targeting platform scale (potentially thousands of concurrent users across many customers), the IBM i resource budget for "AI workers" gets uncomfortably large in pure RPG. PHP keeps the footprint small.

### 3. Provider variability becomes harder to manage

Your demo called Anthropic. What happens when:

- Some customers want OpenAI? Different endpoint, different auth, different request shape, different response shape.
- Some customers want on-premises Ollama? Different protocol entirely.
- A customer brings their own API key (BYOK) and you need to encrypt it at rest?
- You hit Anthropic's rate limit and need to retry with exponential backoff?
- You want to track tokens per call for billing?

In pure RPG, each of these becomes new code in your worker. The `AICALL` program grows. It needs branches per provider, encrypted key storage, retry middleware, usage logging. By the time you're done, you've written the equivalent of a small HTTP framework.

In PHP, each of these is a known pattern. Provider abstractions are a class hierarchy. Retry is Guzzle middleware. Encryption is libsodium. Token tracking is parsing the response. The PHP ecosystem has mature solutions to these problems that RPG would either reimplement or do without.

This isn't "PHP is better than RPG." It's "PHP has been used heavily for this exact kind of HTTP-client-with-middleware work, and the libraries reflect that." RPG can do it; the libraries just aren't there.

---

## The throughput math, made concrete

Here's a worked example to make the difference tangible.

**Scenario:** 300 K3S customers each running a 1,000-row PE check batch in the same hour. Total: 300,000 AI calls. Each call averages 1,000ms of provider time.

Required sustained throughput: 300,000 / 3,600 = **83 calls per second**.

To sustain 83 calls/sec with ~1s latency, you need **~100 in-flight calls** at any moment (with some headroom for variance).

**Pure RPG:**

- 100 RPG workers, each doing one HTTP call at a time.
- Each worker holds one job, one DB2 connection, one HTTPS connection, ~30-50MB of memory.
- Total: 100 active RPG jobs across the AI workload, maybe ~5GB of RPG worker memory, 100 entries in WRKACTJOB.

This is workable but heavy. Your subsystem's `MAXJOBS` needs to be 100+. Your memory pool needs to fit 100 worker processes. Every worker pays full TLS handshake on every call, so the per-call latency is closer to 1.2-1.4s than 1s, which means you actually need ~120-140 workers to hit 83 calls/sec.

**RPG + PHP:**

- ~30 RPG workers, each holding a queue reply (mostly idle, waiting on PHP).
- 4 PHP workers, each holding ~30 calls in flight via Guzzle pool.
- PHP reuses TLS connections, so per-call latency stays at ~1s.

Total: ~30 RPG jobs (each lightweight, mostly waiting), 4 PHP processes, maybe ~1.5GB of total worker memory. The IBM i sees ~34 active jobs at peak.

Same throughput. Roughly **3-4x less IBM i resource consumption.**

If you have headroom, this doesn't matter. If you're sharing Calvin with night batch processing, customer-facing R7 traffic, and other production workloads, that headroom matters.

---

## What PHP gives you, specifically

A list of capabilities the PHP layer adds that pure RPG doesn't provide naturally:

**1. Connection pooling.** Guzzle keeps TLS connections open across requests. Eliminates per-call handshake cost.

**2. Async fan-out per process.** One PHP process can have many HTTP calls in flight via `GuzzleHttp\Pool` — typically 10-12 to start, with room to scale to 30+ after measurement. Each call is independent; the process services whichever responds first. RPG's per-job model can't do this.

**3. Provider abstraction.** A `ProviderInterface` with implementations for Anthropic, OpenAI, Ollama, and future providers. Adding a new provider is one new class, not changes scattered through worker code.

**4. Retry middleware.** Exponential backoff with jitter, configurable retry decisions, automatic 429/503 handling. All implemented as Guzzle middleware. Three lines of configuration.

**5. Encrypted key storage.** libsodium for AES-256-GCM. Envelope encryption for per-customer keys. Cleanly separated from request logic.

**6. JSON parsing flexibility.** PHP's `json_decode` is more forgiving than YAJL when the AI returns slightly malformed responses. Easier to handle the inevitable edge cases.

**7. Rate limiting across customers.** Token bucket logic for per-customer fairness when customers share a provider account.

**8. Usage logging.** Per-call token counts, latencies, costs — straightforward to write to a usage log table.

**9. Future flexibility.** Streaming responses, batch APIs, function calling, multi-modal inputs — these are easier to add to a PHP-based layer when providers support them.

Many of these can be done in RPG. None of them are *as easy* in RPG.

---

## What PHP costs you

Equally honest: here's what introducing PHP costs.

**Another language for your team.** If your team is RPG-only, every line of PHP is a line they can't easily read or modify. Hiring becomes "we need someone who knows IBM i AND PHP." Onboarding takes longer.

**A second deployment story.** PHP code lives in IFS, not in libraries. You install Composer dependencies, manage `vendor/` directories, run a long-lived PASE process. None of this is exotic, but it's not how RPG shops operate by default.

**A queue boundary to maintain.** The data queue between RPG and PHP is an API. It needs a contract, versioning, and discipline. Changes to the contract require coordinated changes on both sides.

**Operational complexity.** Now you're monitoring two things — RPG workers and PHP workers. Two places for things to go wrong. Two log streams. Two restart procedures.

**Debugging across a boundary.** When something fails, the failure could be on either side. You learn to read PHP stack traces alongside RPG joblogs.

**Initial investment.** Building the PHP worker, the provider abstraction, the queue contract, the encryption — none of it is rocket science, but it's real work. Probably 2-4 weeks of focused engineering before you have something production-ready.

These costs are real and worth being clear-eyed about. K3S accepted them because the throughput and architectural benefits outweighed them. Your shop may decide differently.

---

## When pure RPG is enough

If you're in any of these situations, pure RPG is probably the right call:

**Modest AI volume.** Hundreds of calls per batch, a few batches per day. Pure RPG handles this comfortably. The PHP layer is over-engineered for this scale.

**Single provider, single key.** One Anthropic key, one model, one shop. No need for provider abstraction or per-customer keys. The complexity PHP solves doesn't exist in your environment.

**No PHP capacity on your team.** If introducing PHP means hiring or extensively training, the operational cost outweighs the throughput benefit unless you really need the throughput.

**No multi-tenancy.** If you're an internal IT shop running AI for one company (yourselves), the multi-tenant patterns PHP enables aren't useful. Pure RPG is simpler.

**Greenfield experimentation.** If you're still figuring out whether AI is useful for your shop, build it in pure RPG first. You can always add PHP later when you know what you actually need.

There's no shame in pure RPG. It's a legitimate choice for many shops. The platforms that genuinely need PHP are the platforms running AI at scale for many customers.

---

## When PHP earns its place

If you're in any of these situations, the PHP layer probably earns its cost:

**High-throughput, multi-tenant.** ISVs running AI on behalf of many customers. K3S serving distributors. Anyone where the AI workload has many simultaneous users.

**Multiple providers.** Some customers want Anthropic, some OpenAI, some on-prem. The provider abstraction in PHP is significantly cleaner than equivalent branching in RPG.

**BYOK or hosted-tier billing.** When you're encrypting customer keys, tracking per-customer usage, or enforcing per-customer rate limits, PHP's library ecosystem is much better suited.

**Long-running batches that strain IBM i job capacity.** When pure RPG would require hundreds of concurrent jobs and you don't want that pressure on your subsystem.

**A team with PHP capability already.** If you're already running PHP somewhere in your stack (web apps, REST APIs), the marginal cost of another PHP process is small.

**Anticipated growth.** Even if you don't need PHP today, if you can see the throughput curve heading up, building the PHP layer once is cheaper than migrating later.

---

## How K3S thought about this

Some honest context, for the audience that the guide primarily serves: the K3S team itself.

We considered staying pure RPG. The argument was strong: our team is RPG-fluent, our customers are IBM i shops, our existing R7 infrastructure is already in PHP/Mezzio so we have PHP capacity but it's a separate domain (web frontends, not AI workers). Adding PHP to the AI worker meant building something genuinely new in a language we use elsewhere.

What pushed us toward PHP:

**1. Our largest opportunity's potential scale.** A small initial pilot growing toward tens of thousands of stores changes the throughput conversation entirely. Pure RPG at peak load would mean hundreds of concurrent IBM i jobs just for AI, on top of the night batch processing window we're already managing.

**2. Multi-tenancy is core to what K3S does.** We're an ISV serving many distributors concurrently. The provider abstraction, BYOK key custody, and per-customer rate-limit fairness aren't theoretical concerns for us — they're table stakes.

**3. We have PHP capacity to draw on.** Our R7 stack is PHP-based. We have engineers who can read and write PHP fluently. Hiring for "IBM i + PHP" is something we already do.

**4. The architecture wants to keep growing.** This isn't the only AI feature we'll build. The provider abstraction, the queue contract, the worker pattern — all of these set up the next AI feature to be cheaper to add. Investing once, reusing many times.

If we were a smaller shop with less AI ambition, we'd probably have stayed pure RPG. The choice is genuinely shop-specific.

---

## The path forward in this guide

If you decide to stay pure RPG: stop here. The architecture you've already built in [Quickstart 1 (RPG only)]({% link quickstart-1-rpg/index.md %}) and [Quickstart 2 (RPG only)]({% link quickstart-2-rpg/index.md %}) is enough. The remaining chapters discuss patterns that mostly assume PHP; they may still be useful, but they're optimized for the platform pattern.

If you decide to add PHP: continue to [Quickstart 1 (RPG + PHP)]({% link quickstart-1/index.md %}). It's the same demo as the pure-RPG version, with PHP added as the delivery layer. You'll see exactly what changes, what doesn't, and what the boundary looks like in practice.

The chapters after that ([Architecture overview]({% link architecture/index.md %}) onward) make the most sense once you've seen both versions. They explain the design decisions, the contract, the production-shape patterns, and the operational concerns that emerge when you're running this for real.

---

## Open for discussion

V1 calls in this chapter that are worth revisiting:

- **The "scale where PHP earns its place" threshold.** I've placed it roughly at "platform-scale ISV with multi-tenant concerns and high concurrency." That's K3S-shaped. Other shops may find different thresholds.
- **Whether pure RPG with HTTP keep-alive (if SQL services someday support it) would change the calculus.** If the SQL HTTP services someday expose connection reuse, the TLS handshake argument weakens significantly. Worth watching IBM i SQL service updates.
- **The framing of the chapter itself.** Does the case feel honest, or does it feel like marketing for PHP? Worth pressure-testing with readers who didn't make the same choice K3S did.

---

[Next: Quickstart 1 — One worker (RPG + PHP)]({% link quickstart-1/index.md %}) →
