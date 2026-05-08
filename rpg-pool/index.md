---
title: The RPG worker pool
nav_order: 11
has_children: false
---

# The RPG worker pool
{: .no_toc }

**Status:** Draft V1

The RPG worker pool is where K3S's domain logic lives. It's also where most of the production lines of code in this architecture end up — the PHP worker is small by design, but the RPG side has all the business rules, all the DB2 access, and all the orchestration. This chapter covers the production-shape patterns: how the batch initiator works, how worker jobs are sized and submitted, how they coordinate, how they fail, and how they recover.

The demo's `DEMOWRK` is a stripped-down version of what's described here. The shape is the same; the details are bigger.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The shape

A batch run involves three categories of RPG programs:

- **The batch initiator.** A short-lived program that reads candidate rows, populates `WORK_QUEUE`, submits worker jobs, and exits (or stays alive as a monitor).
- **The workers.** Long-running jobs that consume from `WORK_QUEUE`, process one row each, and exit when the queue empties.
- **Business logic programs (`*PRE`, `*POST`).** Called by workers. Owned by the customer's purchasing logic, not by the AI worker pattern.

The architectural commitment is that **business logic stays in the worker side, in RPG, in the customer's library**. The AI worker (PHP) is a transport adapter. Anything that requires knowing what a row means lives in RPG.

The flow:

```
User-initiated batch (interactive or scheduled)
    │
    ▼  CALL
Batch initiator (RPG, transient)
    ├─ creates batch_id
    ├─ creates batch metadata row
    ├─ reads candidate rows from operational tables
    ├─ for each: SNDDTAQ to WORK_QUEUE
    ├─ SBMJOBs N worker jobs
    └─ exits (or monitors)

Worker job × N (RPG, persistent for the batch)
    ├─ at startup: create reply queue
    ├─ loop:
    │   ├─ RCVDTAQ from WORK_QUEUE (timeout)
    │   ├─ if no message: exit loop
    │   ├─ read row, call PRE, build prompt
    │   ├─ SNDDTAQ to AI_OUT_QUEUE
    │   ├─ RCVDTAQ from own reply queue
    │   ├─ call POST, update row
    │   └─ continue
    ├─ at shutdown: delete reply queue, mark batch progress
    └─ exit
```

Three queues, three programs, one customer library. That's the production execution unit.

---

## Library and object layout

For each customer, in their library (e.g., `ACME_5DTA`):

| Object | Type | Purpose |
|:---|:---|:---|
| `WORK_QUEUE` | `*DTAQ` | Per-batch work distribution |
| `RPLY_*` | `*DTAQ` | Per-worker reply queues, transient |
| `AI_BATCH` | `*FILE` | Batch metadata table |
| `AIBATSTRT` | `*PGM` | Batch initiator |
| `AIWORKER` | `*PGM` | Worker loop |
| `AIPRE` | `*PGM` or service program | Pre-AI logic (customer's domain) |
| `AIPOST` | `*PGM` or service program | Post-AI logic (customer's domain) |
| `AIBATEND` | `*PGM` | Batch completion handler |

Operational tables (the ones being processed) live in their normal places. The pattern doesn't move them.

The batch metadata table looks something like:

```sql
CREATE TABLE ACME_5DTA.AI_BATCH (
    BATCH_ID         VARCHAR(20)  NOT NULL,
    BATCH_TYPE       VARCHAR(20)  NOT NULL,
    INITIATED_BY     VARCHAR(10)  NOT NULL,
    INITIATED_AT     TIMESTAMP    NOT NULL,
    STARTED_AT       TIMESTAMP,
    COMPLETED_AT     TIMESTAMP,
    STATUS           VARCHAR(20)  NOT NULL,    -- queued, running, complete, failed
    TOTAL_UNITS      INTEGER      NOT NULL,
    PROCESSED_UNITS  INTEGER      DEFAULT 0,
    FAILED_UNITS     INTEGER      DEFAULT 0,
    WORKER_COUNT     INTEGER      NOT NULL,
    NOTES            VARCHAR(2000),
    PRIMARY KEY (BATCH_ID)
);
```

Each batch gets one row. Workers update `PROCESSED_UNITS` and `FAILED_UNITS` as they go. The last worker to exit flips `STATUS` to complete.

---

## The batch initiator

`AIBATSTRT` runs interactively or from a scheduler. It's small — most of its work is dispatching, not processing.

The structure:

```
1. Parse parameters (batch type, optional row filter)
2. Generate a batch_id (timestamp + sequence)
3. Insert AI_BATCH row with STATUS = 'queued', TOTAL_UNITS = 0
4. Open a cursor over candidate rows from operational tables
5. For each row:
     a. SNDDTAQ {row_id, batch_id, ...} to WORK_QUEUE
     b. Increment in-memory counter
6. Update AI_BATCH set TOTAL_UNITS = counter, STATUS = 'running', STARTED_AT = now
7. Submit N worker jobs via SBMJOB, passing batch_id and worker_id
8. Exit (or stay alive as a monitor)
```

A few important properties:

**It runs in the user's library list.** The candidate rows live in the customer's operational tables, and the cursor reads them directly. No qualified table names; library list does its job.

**It populates the queue *before* submitting workers.** This means workers always have something to consume when they start. If we submitted workers first and populated the queue second, the first few workers might exit immediately on a timeout, thinking the batch is empty.

**It writes the AI_BATCH row early.** This gives ops a record that something started, even if the batch crashes before workers run. Status of `'queued'` distinguishes "haven't actually launched workers yet" from `'running'`.

**The work unit message is small.** Just enough to identify the row. The worker reads the full row data when it dequeues.

Sketch of the loop, as RPG:

```rpg
exec sql declare candidates cursor for
  select PO_LINE_ID
    from ACME_5DTA.PO_LINES
   where REVIEW_STATUS = 'PENDING'
     and CREATED_AT >= :sinceTimestamp
   order by PRIORITY desc, PO_LINE_ID asc;

exec sql open candidates;

unitCount = 0;
dou eofUnits;
  exec sql fetch candidates into :poLineId;
  if sqlcode <> 0;
    eofUnits = *on;
    leave;
  endif;

  // Build work unit JSON
  workUnit.row_id   = poLineId;
  workUnit.batch_id = batchId;
  data-gen workUnit %data(workMessage : 'noprefix=workUnit_') 
                    %gen('YAJL/YAJLDTAGEN');

  // Send to WORK_QUEUE
  exec sql call qsys2.send_data_queue(
    MESSAGE_DATA       => :workMessage,
    DATA_QUEUE         => 'WORK_QUEUE',
    DATA_QUEUE_LIBRARY => :customerLib
  );

  unitCount += 1;
enddo;

exec sql close candidates;

// Update batch metadata with the count
exec sql 
  update ACME_5DTA.AI_BATCH 
     set TOTAL_UNITS = :unitCount,
         STATUS      = 'running',
         STARTED_AT  = current_timestamp
   where BATCH_ID    = :batchId;

// Submit workers
for workerNum = 1 to workerCount;
  sbmCmd = 'SBMJOB CMD(CALL PGM(' + %trim(customerLib) + '/AIWORKER) +
                       PARM(''' + %trim(batchId) + ''' ''' + 
                       %editc(workerNum : 'X') + ''')) +
            JOB(AIWRK' + %editc(workerNum : 'X') + ') +
            JOBQ(QBATCH) +
            INLLIBL(' + %trim(customerLib) + ' K3SAI QGPL QTEMP) +
            LOG(4 00 *NOLIST)';
  QCMDEXC(sbmCmd : 500);
endfor;
```

The `for` loop submitting workers is straightforward. Some shops prefer to use a job description that wraps the parameters; that's a stylistic choice.

---

## The worker

The worker is the long-running RPG job. Each one processes many rows, sequentially (one at a time within itself), but multiple workers run concurrently.

Production worker structure:

```rpg
**FREE
ctl-opt dftactgrp(*no) actgrp('AIWORKER')
        option(*srcstmt: *nodebugio: *nounref);

dcl-pi *n;
  pInBatchId  char(20) const;
  pInWorkerId char(10) const;
end-pi;

// === External procedures ===
dcl-pr AIPRE varchar(8000) extproc('AIPRE');
  rowId int(20) const;
end-pr;

dcl-pr AIPOST extproc('AIPOST');
  rowId        int(20)       const;
  responseJson varchar(8000) const;
  workerId     int(10)       const;
  outResult    varchar(20);   // 'SUCCESS', 'FAIL', 'RETRY'
end-pr;

// ... QCMDEXC, structures, etc

// === Setup ===
workerId   = %int(%trim(pInWorkerId));
batchId    = %trim(pInBatchId);
replyQueue = 'RPLY_' + buildWorkerSuffix(workerId);

createReplyQueue(replyQueue);
markWorkerStarted(batchId, workerId);

// === Main loop ===
dou done;
  // Pull from WORK_QUEUE with timeout
  workMessage = receiveWorkUnit('WORK_QUEUE', customerLib, 5);
  if workMessage = '';
    done = *on;
    leave;
  endif;

  data-into workUnit %data(workMessage) %parser('YAJL/YAJLINTO');

  monitor;
    processRow(workUnit.row_id);
    incrementProcessed(batchId);
  on-error;
    incrementFailed(batchId);
    logRowFailure(workUnit.row_id : %errno : %errmsg);
  endmon;
enddo;

// === Shutdown ===
deleteReplyQueue(replyQueue);
markWorkerStopped(batchId, workerId);
maybeMarkBatchComplete(batchId);

*inlr = *on;
return;

// =====================================================
// processRow — the round trip for one row
// =====================================================
dcl-proc processRow;
  dcl-pi *n;
    rowId int(20) const;
  end-pi;

  prompt = AIPRE(rowId);
  buildAiRequest(rowId : prompt : request);
  data-gen request %data(requestJson : 'noprefix=request_') 
                   %gen('YAJL/YAJLDTAGEN');

  exec sql call qsys2.send_data_queue(
    MESSAGE_DATA       => :requestJson,
    DATA_QUEUE         => 'AIOUTQ',
    DATA_QUEUE_LIBRARY => 'K3SAI'
  );

  responseJson = receiveReply(replyQueue : customerLib : 60);
  if responseJson = '';
    AIPOST(rowId : '{"status":"timeout"}' : workerId : result);
    return;
  endif;

  AIPOST(rowId : responseJson : workerId : result);

  if result = 'RETRY';
    requeueWorkUnit(rowId : batchId);
  endif;
end-proc;
```

Subroutines that aren't shown (`receiveWorkUnit`, `incrementProcessed`, `maybeMarkBatchComplete`, etc.) are short SQL operations against `AI_BATCH` and the queues. Their structure is straightforward; what matters is that the worker delegates them to named subprocedures so the main loop is readable.

A few production-shaped properties worth noting:

**The `MONITOR` block.** Per-row failures don't crash the worker. If `AIPRE` throws, if the queue send fails, if `AIPOST` throws, the worker logs the failure and moves on to the next row. One bad row doesn't stall a 10,000-row batch.

**`incrementProcessed` and `incrementFailed`.** These are atomic SQL `UPDATE ... SET counter = counter + 1` statements. Multiple workers updating the same row is fine because IBM i row locks serialize the increments.

**`maybeMarkBatchComplete`.** Called by every worker at exit. Reads `AI_BATCH`, checks if `PROCESSED_UNITS + FAILED_UNITS = TOTAL_UNITS`, and if so, flips status to complete and fires whatever notification you want. Last worker out turns off the lights.

**`AIPOST` returns a result code.** This is the new part vs. the demo. Production `AIPOST` may decide a row needs to be retried (transient failure, ambiguous AI response). It returns `'RETRY'` and the worker re-queues the row. The decision belongs in business logic, not in the worker plumbing.

---

## Sizing the pool

How many workers do you submit per batch? The honest answer is "whatever makes the batch finish in a reasonable time without wedging the system."

The constraints, in order of how often they bind:

**1. AI provider concurrency.** Most cloud providers cap concurrent in-flight requests. Anthropic's Tier 4 is on the order of hundreds; on-prem Ollama on a single L4 might be 4. The PHP side handles fanout via Guzzle's pool, but the *demand* you can put on it is bounded by what RPG generates. If you have 50 RPG workers each waiting on one AI call, you have at most 50 concurrent calls.

**2. Subsystem MAXJOBS.** The subsystem you submit workers to has a `MAXJOBS` value. Default is often modest; if you submit beyond it, jobs queue. For batch workloads this usually isn't binding (50 workers fits comfortably in `QBATCH`'s defaults), but worth checking with `WRKSBSD QBATCH` before scaling up.

**3. Memory pool.** Each RPG worker is ~30-50 MB resident plus its DB2 connection plus its activation group state. 50 workers × 50 MB = 2.5 GB. Multiply across concurrent batches (multi-customer concurrency) and the number can get real. The subsystem's pool needs headroom or you'll thrash.

**4. DB2 contention.** If all your workers are updating the same operational table, you can hit row-lock contention even at modest worker counts. Less a "sizing" concern and more a "watch your access patterns" one.

A reasonable starting point for a single-customer batch: **N = 20**. Enough parallelism to feel fast; small enough to stay well within all the constraints. Tune from there based on actual measurements.

A note on tuning: don't measure with a 5-row batch. Measure with a 1,000-row batch. Small batches don't show the steady-state behavior — they're dominated by startup costs.

---

## Multi-batch and multi-customer concurrency

The pattern naturally supports multiple concurrent batches, both within a customer and across customers.

**Within a customer**: each batch has its own `batch_id`, its own work units in `WORK_QUEUE`, and its own set of workers. Multiple batches running concurrently means more workers running concurrently and `WORK_QUEUE` contains units from multiple batches mixed FIFO.

**Across customers**: each customer has its own `WORK_QUEUE` in their own library. Workers in customer A pull from `ACME_5DTA/WORK_QUEUE`; workers in customer B pull from `BARCO_5DTA/WORK_QUEUE`. They never share work queues. They *do* share `AI_OUT_QUEUE` in the K3S admin library — the PHP worker doesn't care which customer a request came from, it just serves it.

The subsystem can host workers from multiple customers simultaneously. There's no architectural reason to give each customer its own subsystem; one shared subsystem with enough capacity is fine. The customers are isolated by library list, not by subsystem.

---

## Failure modes and recovery

What goes wrong, and what happens.

**A worker job crashes mid-row.** The reply queue still exists (the cleanup CL didn't run). The work unit was already RCVDTAQ'd from `WORK_QUEUE`, so it's gone. The row has no result. Recovery: the daily orphaned-queue cleanup job notices the reply queue belongs to a job that no longer exists and deletes it. The row is left in `PENDING` and will get picked up by the next batch run.

This is acceptable for V1. A more sophisticated version uses two-phase queue receive (peek, process, then commit removal), so a crashed worker leaves the work unit on the queue for another worker to pick up. Worth implementing later; not needed for V1.

**The PHP worker crashes mid-call.** PHP's signal handler ensures it sends the in-flight reply before exiting. Worst case: the PHP autostart job restarts immediately and the next request is served. The RPG worker's request that was in flight when PHP died gets a timeout (60s default) and `AIPOST` records it as failed.

**The AI provider has an outage.** PHP retries with exponential backoff up to 5 times. If the outage is short, the request eventually succeeds. If it's long, the retries exhaust and the worker returns a `PROVIDER_ERROR`. RPG records the failure. Production `AIPOST` may flag the row for manual review or for retry in a later batch.

**A poison message hits the PHP worker.** Some malformed JSON, some prompt that triggers an infinite loop in the parser. The worker catches it, sends an `INVALID_REQUEST` error reply, logs the failure, continues. The bad row is marked errored on the RPG side.

**The IBM i reboots mid-batch.** Workers die. Batch metadata says `STATUS = 'running'` with no progress. Recovery is manual: ops looks at unfinished batches via `STATUS = 'running'` and `STARTED_AT < some-threshold`, decides whether to mark them failed or to resubmit them. A "resume" RPG program that re-queues `PENDING` rows for unfinished batches is straightforward to write.

**A batch finishes, but `WORK_QUEUE` still has messages.** Means workers exited early or the initiator over-counted. Diagnostic: `STATUS = 'complete'` but `PROCESSED_UNITS + FAILED_UNITS < TOTAL_UNITS`. Means something left work undone. Worth alerting on.

---

## Monitoring a running batch

Two queries worth having handy.

**Batch progress:**

```sql
SELECT BATCH_ID, STATUS, TOTAL_UNITS, 
       PROCESSED_UNITS + FAILED_UNITS AS DONE,
       (PROCESSED_UNITS + FAILED_UNITS) * 100 / NULLIF(TOTAL_UNITS, 0) AS PCT,
       FAILED_UNITS,
       TIMESTAMPDIFF(2, CHAR(CURRENT_TIMESTAMP - STARTED_AT)) AS SECONDS_RUNNING
  FROM ACME_5DTA.AI_BATCH
 WHERE STATUS = 'running'
 ORDER BY STARTED_AT DESC;
```

**Worker activity:**

```sql
SELECT JOB_NAME, JOB_STATUS, FUNCTION, RUN_PRIORITY,
       CPU_TIME, ELAPSED_INTERACTION_COUNT
  FROM TABLE(QSYS2.ACTIVE_JOB_INFO(
    JOB_NAME_FILTER => 'AIWRK*',
    DETAILED_INFO   => 'ALL'
  ));
```

For real-time queue depths:

```sql
SELECT MESSAGES_ON_QUEUE FROM TABLE(QSYS2.DATA_QUEUE_INFO(
  DATA_QUEUE => 'WORK_QUEUE',
  DATA_QUEUE_LIBRARY => 'ACME_5DTA'
));
```

A queue depth that *grows* during a batch means workers can't keep up with the initiator. A queue depth that *holds* steady means workers are matching the rate. A queue depth that drops means the initiator is done and workers are catching up.

---

## What's deliberately not here in V1

- **Two-phase commit on queue receive.** Crashed workers lose their in-flight work unit until the row is re-picked-up by a future batch. Adding peek-and-commit means crashes lose nothing. Future enhancement.
- **Priority queues.** All work units in `WORK_QUEUE` are FIFO. Production may want some rows to jump to the front (high-priority customer, time-sensitive PO). Switching `WORK_QUEUE` to keyed FIFO with a priority key is the standard fix; not in V1.
- **Cross-batch deduplication.** Two batches running concurrently might both try to process the same row. V1 doesn't prevent this; the second `AIPOST` to update the row wins. Production may want a row-claiming mechanism.
- **Adaptive worker count.** Right now N is fixed when the batch starts. If the AI is slow today, more workers won't help (they all stall on AI). If it's fast, fewer workers might be enough. Adaptive sizing based on observed throughput is doable, not in V1.
- **A "drain" command.** No clean way to ask running workers to finish current rows and not pick up new ones. Useful for graceful upgrades. Not in V1.

---

## Open for discussion

V1 calls in this chapter that need real-shop calibration:

- **The `customerLib` resolution pattern.** I show it as a runtime variable, but where does the worker get it from? Job library list? Parameter? Looking up by user profile? K3S has a convention for this; this chapter should reflect it.
- **Whether `AIPRE`/`AIPOST` are bound modules in a service program or separate `*PGM` objects.** Bound is faster but tightens coupling; separate is looser. RPG team's call.
- **Default worker count of 20.** Pure guess. Should be measured.
- **Whether batches stay in the customer's library or get copied to a shared K3S admin library for cross-customer ops visibility.** Multi-customer ops dashboards are easier with the latter; data sovereignty is cleaner with the former.
- **The `result` enum from `AIPOST` (SUCCESS, FAIL, RETRY).** Three values feels right but might be too few or too many. Open to tweaking.

---

[Next: Multi-tenancy]({% link multi-tenancy/index.md %}) →
