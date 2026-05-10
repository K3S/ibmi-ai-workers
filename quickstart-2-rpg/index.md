---
title: Quickstart 2 — Five workers (RPG only)
nav_order: 4
has_children: false
---

# Quickstart 2 — Five workers in parallel (RPG only)
{: .no_toc }

**Status:** Draft V2

This chapter takes the demo from [Quickstart 1 (RPG only)]({% link quickstart-1-rpg/index.md %}) and runs five RPG workers concurrently against the same five rows. Wall-clock time drops from "five sequential round trips" to "one round trip happening five times at once."

The interesting part: **we don't introduce a queue.** The pattern uses SQL claiming on the operational table — workers atomically claim a row, process it, mark it complete, and move on. Multiple workers can run simultaneously without stepping on each other because DB2 row locks serialize the claims.

This is a well-trodden RPG pattern. Many shops have done this before with various flavors of long-running jobs and DB2-coordinated work distribution. The AI round trip is just what each worker does once it's claimed a row.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You've completed [Quickstart 1 (RPG only)]({% link quickstart-1-rpg/index.md %}). The library, table, and API key data area are in place. The four programs (`AIPRE`, `AICALL`, `AIPOST`, `DEMOWRK`) are compiled.

This chapter modifies two of those programs (`DEMOSTART` and `DEMOWRK`) and adds a `WORKER_ID` column to `DEMO_INPUT`. Everything else stays the same.

---

## What's different

Three changes:

**1. New columns on `DEMO_INPUT`: `WORKER_ID`, `CLAIMED_AT`.** Tracks which worker claimed and processed each row.

**2. `DEMOWRK` changes significantly.** Instead of cursor-iterating all unprocessed rows, it claims rows one at a time using an atomic `UPDATE`. It now takes a `worker_id` parameter so each instance can mark its rows distinctively.

**3. `DEMOSTART` changes significantly.** Submits five copies of `DEMOWRK`, each with a different `worker_id`.

`AIPRE`, `AICALL`, and `AIPOST` are unchanged. The AI round trip itself doesn't care how many workers are doing it.

---

## The big picture for this chapter

```
        ┌─────────────────────────────┐
        │     User runs DEMOSTART     │
        │                             │
        │     SBMJOB DEMOWRK × 5      │
        └──────────────┬──────────────┘
                       │
            ┌──────────┼──────────┬──────────┬──────────┐
            ▼          ▼          ▼          ▼          ▼
      ┌─────────┐  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
      │DEMOWRK 1│  │DEMOWRK2│ │DEMOWRK3│ │DEMOWRK4│ │DEMOWRK5│
      │         │  │        │ │        │ │        │ │        │
      │ claim   │  │ claim  │ │ claim  │ │ claim  │ │ claim  │
      │ AIPRE   │  │ AIPRE  │ │ AIPRE  │ │ AIPRE  │ │ AIPRE  │
      │ AICALL  │  │ AICALL │ │ AICALL │ │ AICALL │ │ AICALL │
      │ AIPOST  │  │ AIPOST │ │ AIPOST │ │ AIPOST │ │ AIPOST │
      │ loop    │  │ loop   │ │ loop   │ │ loop   │ │ loop   │
      └─────────┘  └────────┘ └────────┘ └────────┘ └────────┘
                       │
                       ▼ each calls Anthropic separately, in parallel
              ┌──────────────────┐
              │  api.anthropic   │
              └──────────────────┘
```

Five workers, all running concurrently, each holding their own HTTPS connection to Anthropic. The AI calls happen in parallel. With five rows and five workers, the demo finishes in approximately one round trip's worth of wall-clock time.

The coordination is done by DB2: each worker's claim is an atomic SQL update. No queues, no shared state outside the table itself. The platform does the synchronization for us.

---

## Schema change: add WORKER_ID and CLAIMED_AT

```sql
ALTER TABLE DEMOLIB/DEMO_INPUT 
  ADD COLUMN WORKER_ID INTEGER;

ALTER TABLE DEMOLIB/DEMO_INPUT 
  ADD COLUMN CLAIMED_AT TIMESTAMP;
```

`CLAIMED_AT` lets us distinguish "claimed but not yet processed" from "completed." Useful for recovery if a worker crashes mid-row.

We're also adding an index that the claim query needs:

```sql
CREATE INDEX DEMOLIB/DEMO_INPUT_PEND
       ON DEMOLIB/DEMO_INPUT (PROCESSED_AT, ROW_ID)
       WHERE PROCESSED_AT IS NULL;
```

A partial index, only including unprocessed rows. Tiny, fast.

---

## Reset the table

If you ran Quickstart 1, your table has processed rows. Clear them:

```sql
UPDATE DEMOLIB/DEMO_INPUT 
   SET PROCESSED_AT    = NULL,
       WORKER_ID       = NULL,
       CLAIMED_AT      = NULL,
       AI_VERDICT      = NULL,
       AI_ACTUAL_SUM   = NULL,
       AI_RESPONSE_RAW = NULL;
```

---

## Changes to programs

### `AIPRE`, `AICALL`, `AIPOST` — unchanged

Keep them as compiled.

### `DEMOWRK` — the big change

Now takes `worker_id` as a parameter. Loops on SQL claiming instead of cursor iteration.

```rpg
**FREE
ctl-opt dftactgrp(*no) actgrp('DEMOWRK')
        option(*srcstmt: *nodebugio: *nounref);

dcl-pi *n;
  pInWorkerId char(10) const;
end-pi;

dcl-pr AIPRE varchar(2000) extproc('AIPRE');
  rowId int(10) const;
end-pr;

dcl-pr AICALL varchar(8000) extproc('AICALL');
  prompt varchar(2000) const;
end-pr;

dcl-pr AIPOST extproc('AIPOST');
  rowId       int(10)        const;
  responseRaw varchar(8000)  const;
end-pr;

dcl-s workerId    int(10);
dcl-s rowId       int(10);
dcl-s prompt      varchar(2000);
dcl-s responseRaw varchar(8000);
dcl-s done        ind inz(*off);

workerId = %int(%trim(pInWorkerId));

dou done;

  // Atomic claim: find the next unclaimed row and mark it ours.
  rowId = 0;
  exec sql
    select ROW_ID into :rowId
      from final table (
        update DEMOLIB/DEMO_INPUT
           set WORKER_ID  = :workerId,
               CLAIMED_AT = current_timestamp
         where ROW_ID = (
           select ROW_ID
             from DEMOLIB/DEMO_INPUT
            where PROCESSED_AT is null
              and CLAIMED_AT is null
            order by ROW_ID
            fetch first 1 row only
            for update
         )
        )
        as claimed;

  if sqlcode <> 0 or rowId = 0;
    done = *on;
    leave;
  endif;

  monitor;
    prompt      = AIPRE(rowId);
    responseRaw = AICALL(prompt);
    AIPOST(rowId : responseRaw);
  on-error;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = 'WORKER_ERROR',
             PROCESSED_AT    = current_timestamp
       where ROW_ID = :rowId;
  endmon;

enddo;

*inlr = *on;
return;
```

The interesting part is the claim query. From the inside out:

**Inner SELECT** finds the next unclaimed, unprocessed row by `ROW_ID` order. The `FOR UPDATE` clause acquires a row lock.

**Outer UPDATE** sets `WORKER_ID` and `CLAIMED_AT`.

**Outer SELECT FROM FINAL TABLE** returns the `ROW_ID` we just claimed back to RPG.

The whole thing is one statement, atomic. Two workers running this simultaneously will serialize on the row lock. No two workers ever claim the same row.

The `MONITOR` block guards against per-row failures. If anything throws, the worker logs `WORKER_ERROR` and moves on.

When no rows remain unclaimed, the inner SELECT returns nothing, the UPDATE affects no rows, and the worker exits.

### `DEMOSTART` — submit five workers

```cl
PGM

SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('1'))         +
       JOB(DEMOWRK1) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB QGPL QTEMP)                      +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('2'))         +
       JOB(DEMOWRK2) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB QGPL QTEMP)                      +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('3'))         +
       JOB(DEMOWRK3) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB QGPL QTEMP)                      +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('4'))         +
       JOB(DEMOWRK4) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB QGPL QTEMP)                      +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('5'))         +
       JOB(DEMOWRK5) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB QGPL QTEMP)                      +
       LOG(4 00 *NOLIST)

SNDPGMMSG MSG('Submitted 5 workers.')

ENDPGM
```

---

## Recompile

```
CRTBNDRPG PGM(DEMOLIB/DEMOWRK)    SRCFILE(DEMOLIB/QRPGLESRC) +
          BNDSRVPGM(DEMOLIB/AILOGIC YAJL/YAJL) +
          REPLACE(*YES)

CRTBNDCL  PGM(DEMOLIB/DEMOSTART)  SRCFILE(DEMOLIB/QCLLESRC) +
          REPLACE(*YES)
```

`AIPRE`, `AICALL`, and `AIPOST` don't need recompilation.

---

## Running

Make sure all rows are reset. Then:

```
CALL DEMOLIB/DEMOSTART
```

You'll see:

```
Submitted 5 workers.
```

Within a couple of seconds, run `WRKACTJOB SBS(QBATCH)`:

```
Subsystem/Job  User  Number  Type  Status   Function
QBATCH         K4    ...     SBS   ACTIVE
  DEMOWRK1     K4    ...     BCH   RUN
  DEMOWRK2     K4    ...     BCH   RUN
  DEMOWRK3     K4    ...     BCH   RUN
  DEMOWRK4     K4    ...     BCH   RUN
  DEMOWRK5     K4    ...     BCH   RUN
```

Five workers running concurrently.

After ~1-2 seconds, they all finish.

---

## Verifying parallelism

```sql
SELECT ROW_ID, NUM_A, NUM_B, CLAIMED_SUM, 
       AI_VERDICT, AI_ACTUAL_SUM, 
       WORKER_ID, CLAIMED_AT, PROCESSED_AT
  FROM DEMOLIB/DEMO_INPUT
  ORDER BY ROW_ID;
```

Each row should have a different `WORKER_ID` (assignment depends on which worker won each claim race). Timestamps should cluster within ~1 second.

---

## Wall-clock comparison

- Quickstart 1: ~5 seconds (5 sequential AI calls)
- Quickstart 2: ~1.5 seconds (5 parallel AI calls)

The 3-4x speedup is the parallelism dividend. With more rows the speedup grows.

---

## What about crashed workers?

If a worker crashes mid-row, the row is left with `WORKER_ID` set, `CLAIMED_AT` set, but `PROCESSED_AT` null. No other worker will claim it.

Recovery is a periodic job:

```sql
UPDATE DEMOLIB/DEMO_INPUT
   SET WORKER_ID  = NULL,
       CLAIMED_AT = NULL
 WHERE PROCESSED_AT IS NULL
   AND CLAIMED_AT IS NOT NULL
   AND CLAIMED_AT < CURRENT TIMESTAMP - 10 MINUTES;
```

Run this on a schedule. Any claim older than 10 minutes is presumed stuck and gets unclaimed for retry. The threshold should be longer than your worst-case AI call latency including retries.

For the demo we don't need this. For production it's essential.

---

## Cleanup

```
DLTLIB LIB(DEMOLIB)
```

---

## Where to go from here

You've now seen the architecture's bones in pure RPG: a worker pool, parallelism via SBMJOB, work distribution via SQL claiming, AI round trips via SQL HTTP services. For many shops, this is enough.

For K3S — and for shops that anticipate the kind of throughput where pure RPG starts straining — there's another step. The next chapter, [Why PHP for the delivery layer]({% link why-php/index.md %}), explains what changes when you introduce PHP between RPG and the AI provider.

---

[Next: Why PHP for the delivery layer]({% link why-php/index.md %}) →
