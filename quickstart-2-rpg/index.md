---
title: Quickstart 2 — Five workers (RPG only)
nav_order: 4
has_children: false
---

# Quickstart 2 — Five workers in parallel (RPG only)
{: .no_toc }

**Status:** Draft V1 — code untested on Calvin yet

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

**1. New column on `DEMO_INPUT`: `WORKER_ID`.** Tracks which worker claimed and processed each row. Unique per worker, so we can see the parallelism happening.

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

## Schema change: add WORKER_ID

```sql
ALTER TABLE DEMOLIB/DEMO_INPUT 
  ADD COLUMN WORKER_ID INTEGER;

ALTER TABLE DEMOLIB/DEMO_INPUT 
  ADD COLUMN CLAIMED_AT TIMESTAMP;
```

`CLAIMED_AT` lets us distinguish "claimed but not yet processed" from "completed." Useful for recovery if a worker crashes mid-row (we'll cover this below).

We're also adding an index that the claim query needs:

```sql
CREATE INDEX DEMOLIB/DEMO_INPUT_PEND
       ON DEMOLIB/DEMO_INPUT (PROCESSED_AT, ROW_ID)
       WHERE PROCESSED_AT IS NULL;
```

This is a partial index — it only includes rows where `PROCESSED_AT IS NULL`. Tiny, fast, exactly what the claim query needs.

---

## Reset the table

If you ran Quickstart 1, your table has processed rows. Clear them and add the new columns:

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

Keep them as compiled. They're the same procedures.

### `DEMOWRK` — the big change

Now takes `worker_id` as a parameter. Loops on SQL claiming instead of cursor iteration.

```rpg
**FREE
ctl-opt dftactgrp(*no) actgrp('DEMOWRK')
        option(*srcstmt: *nodebugio: *nounref);

// === Entry parameters ===
dcl-pi *n;
  pInWorkerId char(10) const;
end-pi;

// === External procedure prototypes ===
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

// === Variables ===
dcl-s workerId    int(10);
dcl-s rowId       int(10);
dcl-s prompt      varchar(2000);
dcl-s responseRaw varchar(8000);
dcl-s done        ind inz(*off);

// Convert worker ID
workerId = %int(%trim(pInWorkerId));

// === Main loop: claim a row, process it, repeat ===
dou done;

  // Atomic claim: find an unprocessed, unclaimed row and mark it ours.
  // Using UPDATE with a subquery + RETURNING gives us "claim and read" 
  // in one operation.
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

  // No row claimed? we're done.
  if sqlcode <> 0 or rowId = 0;
    done = *on;
    leave;
  endif;

  // Process the row
  monitor;
    prompt      = AIPRE(rowId);
    responseRaw = AICALL(prompt);
    AIPOST(rowId : responseRaw);
  on-error;
    // Per-row failure: log and continue
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

The interesting part is the claim query. Walk through it from the inside out:

**Inner SELECT** finds the next unclaimed, unprocessed row by `ROW_ID` order. The `FOR UPDATE` clause acquires a row lock. While this lock is held, no other worker can read this row in their inner SELECT.

**Outer UPDATE** sets `WORKER_ID` and `CLAIMED_AT`. The lock is held across the update.

**Outer SELECT FROM FINAL TABLE** returns the `ROW_ID` we just claimed back to RPG. This is DB2's syntax for "do this update and tell me what you changed."

The whole thing is one statement, one transaction, atomic. Two workers running this simultaneously will serialize on the row lock — one claims, the other waits, then claims a different row. No two workers ever claim the same row.

The `MONITOR` block guards against per-row failures. If `AICALL` throws (network error, bad JSON), the worker logs `WORKER_ERROR` to that row's `AI_RESPONSE_RAW` and continues to claim the next row. One bad row doesn't stall the worker.

When no rows remain unclaimed, the inner SELECT returns nothing, the UPDATE affects no rows, the outer SELECT returns SQLCODE 100 (no row), and the worker exits.

### `DEMOSTART` — submit five workers

```cl
PGM

/* Submit 5 workers, each with a different worker_id */
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

Five `SBMJOB` calls. Each worker gets its own `worker_id`. The `INLLIBL` ensures every worker has `DEMOLIB` on its library list.

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

Make sure all rows are reset (empty `PROCESSED_AT`, `WORKER_ID`, etc.). Then:

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

Five workers running concurrently. Each is in the middle of an HTTPS call to Anthropic, with its own TLS connection.

After ~1-2 seconds (one AI round trip each, in parallel), they all finish. Run `WRKACTJOB` again — they're gone.

---

## Verifying parallelism

```sql
SELECT ROW_ID, NUM_A, NUM_B, CLAIMED_SUM, 
       AI_VERDICT, AI_ACTUAL_SUM, 
       WORKER_ID, CLAIMED_AT, PROCESSED_AT
  FROM DEMOLIB/DEMO_INPUT
  ORDER BY ROW_ID;
```

You should see all five rows processed, each by a different worker:

```
ROW_ID NUM_A NUM_B CLAIMED AI_VERDICT AI_SUM WORKER CLAIMED_AT          PROCESSED_AT
   1     1     1      2      Y         2      3    14.22.18.105        14.22.19.024
   2     3     4      8      N         7      1    14.22.18.110        14.22.19.117
   3    10    15     25      Y        25      5    14.22.18.118        14.22.19.087
   4     7     7     13      N        14      2    14.22.18.122        14.22.19.234
   5   100   200    300      Y       300      4    14.22.18.131        14.22.19.301
```

Things to look for:

- **`WORKER_ID` is a different value for each row.** Confirms five separate workers each grabbed and processed work. The specific assignment depends on which worker won each claim race.
- **`CLAIMED_AT` timestamps cluster within ~50ms of each other.** All workers tried to claim simultaneously; DB2's row locks serialized the claims.
- **`PROCESSED_AT` timestamps are roughly 1 second after `CLAIMED_AT`.** That's the AI call duration. They're parallel — all five rows finished within a small window.
- **`AI_VERDICT` and `AI_ACTUAL_SUM` are correct.** Same as Quickstart 1.

If one worker_id is missing entirely (no row has `WORKER_ID = 4`), worker 4 lost every claim race because the others moved faster. With only 5 rows and 5 workers, this can happen — there are exactly enough rows for everyone, but only if the timing works out. Try seeding 20 rows instead of 5 if you want more robust evidence of distribution.

---

## Wall-clock comparison

Reset the table, then run Quickstart 1's `DEMOSTART` (the one-worker version) and time it. Then reset and run Quickstart 2's. With 5 rows:

- Quickstart 1: ~5 seconds (5 sequential AI calls × 1s each)
- Quickstart 2: ~1.5 seconds (5 parallel AI calls finishing roughly together, plus job startup)

The 3-4x speedup is the parallelism dividend. With more rows, the difference grows: at 100 rows, Quickstart 1 takes ~100 seconds and Quickstart 2 takes ~20 seconds (5x speedup, limited by Anthropic's per-call latency).

---

## What about crashed workers?

If a worker crashes mid-row, the row is left with `WORKER_ID` set, `CLAIMED_AT` set, but `PROCESSED_AT` null. No other worker will claim it — it's already claimed.

Recovery is a periodic job that finds these orphaned claims and resets them:

```sql
UPDATE DEMOLIB/DEMO_INPUT
   SET WORKER_ID  = NULL,
       CLAIMED_AT = NULL
 WHERE PROCESSED_AT IS NULL
   AND CLAIMED_AT IS NOT NULL
   AND CLAIMED_AT < CURRENT TIMESTAMP - 10 MINUTES;
```

Run this from a CL job on a schedule (every 5-10 minutes). Any claim older than 10 minutes is presumed stuck and gets unclaimed for someone else to retry. The threshold should be longer than your worst-case AI call latency including retries.

For the demo we don't need this. For production it's essential.

---

## What you've learned

The interesting thing about this chapter isn't what was added. It's what *didn't* change:

- `AIPRE`, `AICALL`, `AIPOST` — all unchanged. The AI round trip is the same.
- The setup — same library, same table (with two new columns), same data area for the API key.
- The architecture — workers do the same work, just five at once.

What changed:

- One CL program (more SBMJOBs).
- One RPG worker (claim instead of cursor; takes a worker_id).
- Schema (added two columns and an index).

That's the whole delta. Parallelism without queues, without coordination code, without any new mechanisms — just DB2 row locks and the SBMJOB facility.

---

## Why this works

Three properties of IBM i make this pattern straightforward:

**1. SBMJOB is a built-in fan-out mechanism.** No external job manager needed. The subsystem handles workers as first-class citizens.

**2. DB2 row locks serialize the claim atomically.** The `UPDATE ... FOR UPDATE` pattern is well-defined and well-tested. Two workers can never claim the same row.

**3. Long-lived RPG jobs are cheap once started.** The startup cost is paid once per worker, not once per row. Five workers processing 1000 rows pay the startup cost five times, not 1000 times.

These aren't tricks; they're the platform's normal operations applied to AI work.

---

## What's still not in this demo

- **Connection reuse for HTTPS.** Each call still opens a fresh TLS connection. With 5 workers × many rows, this adds up. The `QSYS2.HTTP_POST` SQL service doesn't expose connection pooling at the RPG level.
- **Retry logic.** A 429 from Anthropic fails the row. Production retries with backoff.
- **Per-customer profiles.** Hardcoded model and key.
- **Cross-batch coordination.** Two batches running concurrently would have workers from each claiming each other's rows. For the demo, only one batch runs at a time.
- **Provider abstraction.** Switching providers means rewriting the call layer.
- **Rate limit fairness across customers.** No throttling.

---

## Cleanup

```
DLTLIB LIB(DEMOLIB)
```

---

## Where to go from here

You've now seen the architecture's bones in pure RPG: a worker pool, parallelism via SBMJOB, work distribution via SQL claiming, AI round trips via SQL HTTP services. For many shops, this is enough.

For K3S — and for shops that anticipate the kind of throughput where pure RPG starts straining — there's another step. The next chapter, [Why PHP for the delivery layer]({% link why-php/index.md %}), explains what changes when you introduce PHP between RPG and the AI provider, why we did it at K3S, and when you should consider doing it too.

---

[Next: Why PHP for the delivery layer]({% link why-php/index.md %}) →
