---
title: Architecture overview
nav_order: 8
has_children: false
---

# Architecture overview
{: .no_toc }

**Status:** Draft

This chapter walks the whole stack on one page. The point is to make the boundary between RPG and PHP visible — and to defend it. Every later chapter is depth on a piece you'll see here first.

If you only read three chapters of this guide, this is the second of them. (The first is [Foundations]({% link foundations/index.md %}); the third is [The data queue contract]({% link contract/index.md %}).)

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The shape, in one paragraph

A user starts a batch from your application. RPG creates a batch ID, fans out a small number of long-running RPG worker jobs, and writes the work to be done onto a data queue. Each RPG worker pulls a row at a time, runs all the business logic that happens *before* the AI call (data lookups, prompt assembly), and hands the prompt to a separate, shared PHP worker via a second data queue. The PHP worker calls the AI, gets the response, and hands it back through a reply queue. The RPG worker takes the response, runs all the business logic that happens *after* the AI call (parsing, rule application, DB updates), marks the row reviewed, and pulls the next one. The PHP worker is small, transport-only, and shared across every customer. The RPG workers contain all the domain logic and run inside the customer's library list. This split is the whole architecture.

## Topology

The system has two long-lived components and several transient ones. Here's what runs where.

```
                  ┌───────────────────────────────────────────┐
                  │ Application (Web Interface, green-screen, │
                  │ whatever user clicks "run AI batch"       │
                  └────────────────┬──────────────────────────┘
                                   │
                                   ▼  CALL
                  ┌───────────────────────────────────────────┐
                  │ Batch initiator (RPG, transient)          │
                  │   runs in customer library list           │
                  │   creates batch_id                        │
                  │   writes work units to WORK_QUEUE         │
                  │   SBMJOBs N RPG worker jobs               │
                  │   exits (or monitors)                     │
                  └────────────────┬──────────────────────────┘
                                   │
                       ┌───────────┴────────────┐
                       ▼                        ▼
                ┌──────────────┐         ┌──────────────┐
                │ WORK_QUEUE   │         │ RESULT_TBL   │
                │ (in customer │         │ (in customer │
                │  library)    │         │  library)    │
                └──────┬───────┘         └──────┬───────┘
                       │                        ▲
        ┌──────────────┼──────────────┐         │
        ▼              ▼              ▼         │
   ┌─────────┐   ┌─────────┐    ┌─────────┐    │
   │ RPG #1  │   │ RPG #2  │    │ RPG #N  │    │
   │ worker  │   │ worker  │    │ worker  │    │
   │         │   │         │    │         │    │
   │ pre-AI  │   │ pre-AI  │    │ pre-AI  │    │
   │ logic   │   │ logic   │    │ logic   │    │
   │ post-AI │   │ post-AI │    │ post-AI │    │
   │ logic   │   │ logic   │    │ logic   │────┘
   └────┬────┘   └────┬────┘    └────┬────┘
        │             │              │
        └─────────────┼──────────────┘
                      ▼
              ┌────────────────┐
              │ AI_OUT_QUEUE   │  (shared K3S library, all customers)
              └────────┬───────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐    ┌─────────┐
   │ PHP #1  │   │ PHP #2  │    │ PHP #M  │
   │ worker  │   │ worker  │    │ worker  │
   │         │   │         │    │         │
   │ Guzzle  │   │ Guzzle  │    │ Guzzle  │
   │ pool,   │   │ pool,   │    │ pool,   │
   │ ~10 in  │   │ ~10 in  │    │ ~10 in  │
   │ flight  │   │ flight  │    │ flight  │
   └────┬────┘   └────┬────┘    └────┬────┘
        │             │              │
        └─────────────┼──────────────┘
                      ▼
              ┌────────────────┐
              │  AI provider   │  (Anthropic, OpenAI, on-prem)
              └────────────────┘

         Reply queues (one per RPG worker, in customer library)
         carry responses back from PHP to the RPG worker that asked.
```

A few things worth noting in that diagram:

- **WORK_QUEUE and RESULT_TBL live in the customer's library.** They're per-customer. So is each RPG worker's reply queue.
- **AI_OUT_QUEUE lives in a shared K3S admin library.** It's the same queue for every customer. PHP workers don't need to know which customer a request came from to do their job.
- **PHP workers are persistent.** They start once (typically as autostart jobs) and run forever, idle when there's no work. They are not started per batch and not started per customer.
- **RPG workers are per-batch.** A batch initiator submits N of them when work begins, and they exit when WORK_QUEUE drains.

## The boundary, and why

The most important decision in this architecture is where to draw the line between RPG and PHP. We draw it at transport.

**RPG owns:**

- Reading rows from the operational tables that drive the batch.
- Running business-logic APIs to assemble the data that goes into the prompt.
- Building the prompt itself.
- Parsing the AI response.
- Applying business rules to determine final disposition.
- Writing dispositions back to the operational tables.
- Marking rows as reviewed.

**PHP owns:**

- Receiving a prompt over a queue.
- Resolving which AI provider, which model, and which API key to use.
- Calling the AI provider over HTTPS.
- Handling retries, rate limits, and provider-specific quirks.
- Logging usage (tokens, latency, cost).
- Returning the raw response over a queue.

Notice what's *not* on the PHP side: any knowledge of what a row means, what a prompt is for, or what to do with the response. The PHP worker can serve a purchasing-exception batch, a vendor-description-classification batch, and a sales-forecast-summarization batch without changing a line of code. It doesn't know which is which.

There are six reasons this division is right, and each one matters in production.

### 1. Reusability

The PHP worker has no domain assumptions. Today it's powering one use case. Next quarter you'll think of a second. The year after, a third. If the worker only knows "prompt in, response out," every new use case is *new RPG and a new prompt template*, not a new worker. If the worker knew about purchasing-exception checks specifically, you'd be forking it for every new feature.

### 2. Tenant isolation

Business logic in RPG runs in the customer's job, with the customer's library list, against the customer's tables, with the customer's authority model. That's the IBM i isolation story you already trust. The moment PHP starts reading and writing customer operational tables, you've punched a hole in that — PHP needs authority across all customer libraries, and one bug can splatter customer A's data into customer B's tables. With the boundary at transport, the worker never touches operational data. It can't make a tenancy mistake because it doesn't know what tenancy is.

### 3. Where your team's expertise actually lives

If your team is like most IBM i shops, RPG is the language you've been writing fluently for decades. PHP is the new muscle. Putting domain logic in the place where you already have deep pattern-matching, and putting *only* the new transport mechanism in the new language, is the lower-risk distribution of work. The hard parts of the system live where the experienced developers can debug them.

### 4. Testability and replay

When something goes sideways in production — and it will — "the AI gave us a weird response on line 47 of batch 12" is something you want to be able to replay without re-running the whole batch. If the worker just transports, the prompt for line 47 is sitting in the queue (or its log). You can re-fire it manually, compare responses, debug. If the worker also reads and writes operational data, replay is much harder because the side effects are tangled with the call.

### 5. Backpressure and pacing live in the queue

If the AI is slow or rate-limited, the AI_OUT_QUEUE grows. RPG keeps producing at its own pace; PHP consumes as fast as it can. They're decoupled. If RPG were calling PHP synchronously per row instead, RPG sits and waits — and one slow customer's batch could block others, or you'd accumulate hundreds of half-finished RPG jobs holding open resources.

### 6. Failure isolation

PHP worker dies? RPG keeps producing into the queue. Restart the worker, it picks up where the queue is. RPG dies? Worker drains the queue and idles. Either side can be restarted independently. If they were tangled together via direct calls, a worker hiccup means RPG hiccups.

---

## The lifecycle of a batch

Here's what actually happens, chronologically, when a user kicks off a batch of 10,000 rows.

**At system startup, once.** Some autostart job (or a CL you run manually after IPL) launches M PHP workers via SBMJOB. They start, connect to the AI provider, wait on AI_OUT_QUEUE. They never exit. If one crashes, an autostart monitor restarts it.

**User clicks "Run AI batch" in the application.** The application calls a CL program in the customer's library, passing the batch parameters. The CL submits the batch initiator RPG job.

**Batch initiator runs (transient, seconds to minutes).** It generates a batch_id. It creates a per-batch result table or writes batch metadata. It reads the candidate rows from the operational tables and writes one small message per row (just `batch_id` and `line_id`) to WORK_QUEUE. It SBMJOBs N RPG worker jobs, passing each one a worker_id and the batch_id. Then it either exits or stays alive as a monitor — your choice.

**N RPG workers run in parallel.** Each worker creates its own private reply queue at startup. Then it loops:

1. RCVDTAQ from WORK_QUEUE, with timeout. Get a `{batch_id, line_id}` work unit.
2. Read the full row from the operational tables. Run business-logic APIs to assemble context. Build the prompt.
3. SNDDTAQ to AI_OUT_QUEUE with `{customer, profile_ref, prompt, reply_queue_name}`.
4. RCVDTAQ from its own reply queue, blocking until response arrives.
5. Parse the response. Apply business rules. Update operational tables. Mark row reviewed.
6. Loop.

When WORK_QUEUE returns nothing on RCVDTAQ (timeout), the worker checks "is the batch done?" and exits if so.

**M PHP workers, in parallel, doing transport.** Each PHP worker loops:

1. RCVDTAQ from AI_OUT_QUEUE. Get a request.
2. Look up the AI profile referenced in the message (which provider, which model, which key).
3. Add the request to a Guzzle pool that's keeping ~10 calls in flight at any given time.
4. When the response comes back, log usage and SNDDTAQ to the named reply queue.

PHP workers never read from WORK_QUEUE, never touch operational tables, never know which customer a request came from beyond the profile reference.

**Batch completes.** Last RPG worker out flips the batch to "complete" and fires whatever notification you want. PHP workers are still running, still idle, still waiting for the next batch from this customer or any other.

---

## Why long-lived PHP workers

A natural-feeling alternative to long-lived PHP workers is to spawn one PHP process per AI call: RPG fires a `system()` call to a PHP script, the script does its thing, exits, RPG continues. This works for prototypes. It does not scale. Three reasons:

**Process startup is expensive on PASE.** A fresh PHP process on IBM i takes hundreds of milliseconds to a second to start — interpreter init, autoloader scan, configuration load. Multiplied across 10,000 rows, that's hours of pure overhead.

**TLS handshakes are expensive.** Each new PHP process opens fresh TCP connections to the AI provider, doing the full TLS handshake every time. That's another ~100ms per call. Long-lived workers reuse connections via Guzzle's connection pool — handshake once, reuse for the rest of the day.

**SBMJOB has its own cost.** Even if PHP startup were free, submitting an IBM i job to run it isn't. Job initiation, memory pool allocation, and joblog setup are all real costs.

The same argument applies to the RPG side, by the way. RPG workers are long-lived for the duration of the batch — one job processes many rows in a loop, not one job per row.

---

## Where parallelism lives

There are two parallelism dimensions, and they're independent.

**N — RPG worker count.** This is your throughput dial. More RPG workers means more concurrent rows in flight. Bounded by your subsystem's MAXJOBS, your memory pool, and how much DB2 contention your operational tables tolerate. 50 is a reasonable starting point for most batches. 200 is plausible. 2000 probably isn't.

**M — PHP worker count, and per-worker pool size.** PHP workers exist to keep the AI saturated. If a single PHP worker can hold 10 calls in flight via Guzzle pool, and your N RPG workers can produce up to N concurrent requests, you need M × pool_size ≥ N to avoid AI_OUT_QUEUE building up under steady state. Usually M = 4 to 8 is plenty.

The math we recommend starting with: N = 50, M = 5, pool_size = 12. That gives you 60 PHP-side concurrent capacity for 50 RPG-side maximum demand, with margin. Tune from there based on what your AI provider and your IBM i actually do under load.

The natural backpressure is the elegant part: RPG workers are self-throttling. Each one can only be processing one row at a time, and it's waiting for its own AI response before moving on. So the system never produces more concurrent AI calls than there are RPG workers. You don't need a separate rate-limiter for that ceiling — it's structural.

The vendor rate limit (Anthropic RPM/TPM, OpenAI tier limits, etc.) is a different concern. It belongs in the PHP worker, because that's where the actual API calls happen. We cover it in [AI provider concerns]({% link providers/index.md %}).

---

## Where backpressure lives

Three queues, three different jobs:

- **WORK_QUEUE** holds the *batch's* depth. If RPG workers are slow to consume — perhaps because the AI is slow today — work piles up here. That's fine. The queue is the buffer.
- **AI_OUT_QUEUE** holds the *system's* depth across all batches and all customers. Steady-state, this stays near zero because PHP capacity exceeds RPG demand. If it grows, it means PHP is the bottleneck — which usually means the AI provider is rate-limiting you or having an outage.
- **Reply queues** never have meaningful depth. Each one has at most one message in flight at a time (the response to whatever the worker last asked).

Watching queue depths is your primary observability tool. WORK_QUEUE depth tells you batch progress. AI_OUT_QUEUE depth tells you system health. Both are easy to monitor with `WRKDTAQ` or via SQL against `QSYS2.DATA_QUEUE_INFO`.

---

## What's deliberately not in this chapter

- **Per-customer AI profiles.** How a customer's choice of provider, model, and API key gets resolved at runtime. Covered in [Multi-tenancy]({% link multi-tenancy/index.md %}) and [AI provider concerns]({% link providers/index.md %}).
- **Provider abstraction.** How the worker swaps between Anthropic, OpenAI, and on-premises providers without business logic changes. Covered in [AI provider concerns]({% link providers/index.md %}).
- **The exact contract between RPG and PHP.** Field-by-field message formats, required vs. optional, error message shapes. Covered in [The data queue contract]({% link contract/index.md %}).
- **Failure modes in detail.** What happens when a poison message arrives, when the AI provider is down, when a worker wedges. Covered in [Operating in production]({% link operations/index.md %}).

The architecture above is the spine. Every later chapter hangs a specific concern off this skeleton.

---

## The decisions reflected here

Six architectural choices that this chapter quietly commits to:

1. **PHP runs on the IBM i in PASE**, not on a separate Linux box. The integration cost of a separate server isn't worth it when PHP runs natively on the platform.
2. **The PHP worker is shared across all customers**, not installed per-customer. Configuration varies by customer; code does not.
3. **The AI worker is a separate application from your existing API** (Replenish R7 in our case). Different lifecycle, different invocation surface, different dependency churn.
4. **Data queues are the language boundary**, not direct calls or HTTP. Asynchronous, durable, native to IBM i, and they decouple lifecycles cleanly.
5. **Long-lived workers on both sides**, not spawn-per-request. Startup costs would dominate at any meaningful volume.
6. **Business logic stays in RPG.** Transport stays in PHP. The boundary is the architecture.

Each of these is defensible, and each is reversible if you find evidence that you should have chosen differently. We've tried to give the reasoning so that if you make a different call, you do it knowing what you're trading.

---

[Next: The data queue contract]({% link contract/index.md %}) →
