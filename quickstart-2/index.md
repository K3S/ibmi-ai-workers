---
title: Quickstart 2 — Five workers (RPG + PHP)
nav_order: 7
has_children: false
---

# Quickstart 2 — Five workers in parallel (RPG + PHP)
{: .no_toc }

**Status:** Draft V1 — code untested on Calvin yet

This chapter takes the demo from [Quickstart 1 (RPG + PHP)]({% link quickstart-1/index.md %}) and runs five RPG workers against the same five rows, in parallel. Wall-clock time drops from "five sequential round trips" to "one round trip happening five times at once."

The key insight: **parallelism falls out of the architecture, not from new code**. The PHP worker doesn't change. The contract doesn't change. Most of the RPG doesn't change. What changes is small enough that you can do it in half an hour.

If you've read [Quickstart 2 (RPG only)]({% link quickstart-2-rpg/index.md %}), you've already seen this same lesson once — five workers using SQL claiming to coordinate, no queue between them needed for V1. This chapter applies the same parallelism pattern to the RPG+PHP architecture, with one notable addition: **a `WORK_QUEUE` between the batch initiator and the workers**, because the K3S architecture commits to using queues as the work-distribution mechanism for production.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You've completed [Quickstart 1 (RPG + PHP)]({% link quickstart-1/index.md %}). The library, table, queue, and PHP worker are in place. The RPG and CL programs are compiled.

This chapter modifies three programs (`DEMOSTART`, `DEMOWRK`, and `DEMOPST`) and adds one new data queue (`WORK_QUEUE`). Everything else stays the same.

---

## What's different

Three changes:

**1. New queue: `WORK_QUEUE`.** A queue in `DEMOLIB` that distributes work across the RPG workers. Each message says "process this row." Workers pull from it independently; FIFO ordering means no two workers grab the same row.

**2. `DEMOSTART` changes significantly.** Now populates `WORK_QUEUE` with five messages, then submits five copies of `DEMOWRK` with different worker IDs.

**3. `DEMOWRK` changes significantly.** Takes a `worker_id` parameter, builds its reply queue name from that ID, reads work units from `WORK_QUEUE` instead of iterating `DEMO_INPUT` directly, and exits when the queue is empty.

What's unchanged:

- **`worker.php`** is unchanged. The PHP worker is shared infrastructure.
- **`DEMOPRE`** is unchanged. Pre-AI logic doesn't depend on which worker is calling it.
- **`DEMOPST`** has one small change: takes a `worker_id` parameter so each worker writes its own ID.
- **The data queue contract** is unchanged.
- **`AI_OUT_QUEUE`** is unchanged. All five workers write to the same queue; PHP serves any of them.

---

## The big picture for this chapter

```
        ┌────────────────────────────┐
        │   User runs DEMOSTART      │
        │                            │
        │   1. Populates WORK_QUEUE  │
        │      with row_ids 1-5      │
        │   2. SBMJOBs DEMOWRK × 5   │
        └──────────────┬─────────────┘
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
      ┌──────────┐         ┌──────────┐
      │WORK_QUEUE│         │SBMJOB ×5 │
      │ row=1    │         │worker=1  │
      │ row=2    │         │worker=2  │
      │ row=3    │         │worker=3  │
      │ row=4    │         │worker=4  │
      │ row=5    │         │worker=5  │
      └────┬─────┘         └────┬─────┘
           │                    │
           │       ┌────────────┘
           │       │
           ▼       ▼
   ┌──────────────────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐
   │   DEMOWRK #1     │  │ #2 │  │ #3 │  │ #4 │  │ #5 │
   │                  │  │    │  │    │  │    │  │    │
   │  RCV WORK_QUEUE  │  │... │  │... │  │... │  │... │
   │  call DEMOPRE    │  │    │  │    │  │    │  │    │
   │  send AI_OUT     │  │    │  │    │  │    │  │    │
   │  RCV RPLY_000001 │  │    │  │    │  │    │  │    │
   │  call DEMOPST    │  │    │  │    │  │    │  │    │
   └────────┬─────────┘  └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘
            │              │       │       │       │
            └──────────────┼───────┼───────┼───────┘
                           ▼       ▼       ▼
                   ┌────────────────────┐
                   │   AI_OUT_QUEUE     │  (in K3SAI)
                   └─────────┬──────────┘
                             │
                             ▼
                   ┌────────────────────┐
                   │    worker.php      │
                   └────────────────────┘
```

Five RPG workers, all running concurrently. Each has its own reply queue (`RPLY_000001` through `RPLY_000005`), so PHP responses route back to the worker that asked. The PHP worker is one process serving five concurrent conversations through Guzzle's connection pooling.

---

## Setup change: create WORK_QUEUE

```
CRTDTAQ DTAQ(DEMOLIB/WORK_QUEUE) +
        TYPE(*STD) +
        MAXLEN(500) +
        SEQ(*FIFO) +
        FORCE(*NO) +
        AUT(*USE) +
        CCSID(1208) +
        TEXT('Demo work distribution queue')
```

Smaller `MAXLEN` than `AI_OUT_QUEUE` because work unit messages are tiny.

---

## Changes to programs

### `DEMOPRE` — unchanged

Skip.

### `DEMOPST` — add worker_id parameter

```rpg
**FREE
ctl-opt nomain;

dcl-proc DEMOPST export;
  dcl-pi *n;
    inRowId        int(10)        const;
    inResponseJson varchar(4000)  const;
    inWorkerId     int(10)        const;
  end-pi;

  dcl-ds reply qualified;
    status     varchar(20);
    response   varchar(2000);
    request_id varchar(36);
  end-ds;

  dcl-ds aiResult qualified;
    correct    ind;
    actual_sum int(10);
  end-ds;

  dcl-s verdict     char(1);
  dcl-s rawResponse varchar(2000);

  data-into reply %data(inResponseJson) %parser('YAJL/YAJLINTO');

  if reply.status <> 'success';
    rawResponse = 'ERROR: ' + reply.status;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp,
             WORKER_ID       = :inWorkerId
       where ROW_ID = :inRowId;
    return;
  endif;

  monitor;
    data-into aiResult %data(reply.response) %parser('YAJL/YAJLINTO');
  on-error;
    rawResponse = reply.response;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp,
             WORKER_ID       = :inWorkerId
       where ROW_ID = :inRowId;
    return;
  endmon;

  if aiResult.correct;
    verdict = 'Y';
  else;
    verdict = 'N';
  endif;

  rawResponse = reply.response;

  exec sql
    update DEMOLIB/DEMO_INPUT
       set AI_VERDICT      = :verdict,
           AI_ACTUAL_SUM   = :aiResult.actual_sum,
           AI_RESPONSE_RAW = :rawResponse,
           PROCESSED_AT    = current_timestamp,
           WORKER_ID       = :inWorkerId
     where ROW_ID = :inRowId;
end-proc;
```

Three lines of SQL changed: `WORKER_ID = :inWorkerId` instead of `WORKER_ID = 1`.

### `DEMOWRK` — accept worker_id, read from WORK_QUEUE

```rpg
**FREE
ctl-opt dftactgrp(*no) actgrp('DEMOWRK')
        option(*srcstmt: *nodebugio: *nounref);

dcl-pi *n;
  pInWorkerId char(10) const;
end-pi;

dcl-pr DEMOPRE varchar(2000) extproc('DEMOPRE');
  rowId int(10) const;
end-pr;

dcl-pr DEMOPST extproc('DEMOPST');
  rowId        int(10)       const;
  responseJson varchar(4000) const;
  workerId     int(10)       const;
end-pr;

dcl-pr QCMDEXC extpgm('QCMDEXC');
  command   char(2000) const;
  cmdLength packed(15:5) const;
end-pr;

dcl-s workerId    int(10);
dcl-s rowId       int(10);
dcl-s prompt      varchar(2000);
dcl-s requestJson varchar(4000);
dcl-s responseJson varchar(4000);
dcl-s workMessage varchar(500);
dcl-s requestId   varchar(36);
dcl-s replyQueue  char(15);
dcl-s replyLib    char(10) inz('DEMOLIB');
dcl-s crtCmd      char(500);
dcl-s dltCmd      char(100);
dcl-s done        ind inz(*off);

dcl-ds request qualified;
  version    varchar(10);
  request_id varchar(36);
  customer   varchar(10);
  profile_ref varchar(20);
  dcl-ds reply_queue;
    library varchar(10);
    name    varchar(15);
  end-ds;
  prompt     varchar(2000);
  max_tokens int(10);
  temperature packed(3:1);
  dcl-ds metadata;
    row_id   int(10);
    batch_id varchar(20);
  end-ds;
end-ds;

dcl-ds workUnit qualified;
  row_id   int(10);
  batch_id varchar(20);
end-ds;

workerId   = %int(%trim(pInWorkerId));
replyQueue = 'RPLY_' + %char(workerId);

crtCmd = 'CRTDTAQ DTAQ(DEMOLIB/' + %trim(replyQueue) + ') +
          TYPE(*STD) MAXLEN(2000000) +
          SEQ(*FIFO) CCSID(1208) +
          AUT(*USE) +
          TEXT(''Demo reply queue worker ' + %char(workerId) + ''')';
QCMDEXC(crtCmd : 500);

dou done;

  exec sql
    select MESSAGE_DATA
      into :workMessage
      from table(qsys2.receive_data_queue(
        DATA_QUEUE         => 'WORK_QUEUE',
        DATA_QUEUE_LIBRARY => 'DEMOLIB',
        WAIT_TIME          => 5,
        REMOVE_MESSAGE     => 'YES'
      ));

  if sqlcode <> 0 or workMessage = '';
    done = *on;
    leave;
  endif;

  data-into workUnit %data(workMessage) %parser('YAJL/YAJLINTO');
  rowId = workUnit.row_id;

  prompt = DEMOPRE(rowId);

  exec sql
    values systools.generate_uuid()
      into :requestId;

  request.version           = '1.0';
  request.request_id        = requestId;
  request.customer          = 'DEMO';
  request.profile_ref       = 'DEMO_DEFAULT';
  request.reply_queue.library = %trim(replyLib);
  request.reply_queue.name    = %trim(replyQueue);
  request.prompt            = prompt;
  request.max_tokens        = 50;
  request.temperature       = 0;
  request.metadata.row_id   = rowId;
  request.metadata.batch_id = workUnit.batch_id;

  data-gen request
           %data(requestJson : 'noprefix=request_')
           %gen('YAJL/YAJLDTAGEN');

  exec sql
    call qsys2.send_data_queue(
      MESSAGE_DATA       => :requestJson,
      DATA_QUEUE         => 'AIOUTQ',
      DATA_QUEUE_LIBRARY => 'K3SAI'
    );

  exec sql
    select MESSAGE_DATA
      into :responseJson
      from table(qsys2.receive_data_queue(
        DATA_QUEUE         => :replyQueue,
        DATA_QUEUE_LIBRARY => :replyLib,
        WAIT_TIME          => 60,
        REMOVE_MESSAGE     => 'YES'
      ));

  if sqlcode = 0 and responseJson <> '';
    DEMOPST(rowId : responseJson : workerId);
  else;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = 'TIMEOUT',
             PROCESSED_AT    = current_timestamp,
             WORKER_ID       = :workerId
       where ROW_ID = :rowId;
  endif;

enddo;

dltCmd = 'DLTDTAQ DTAQ(DEMOLIB/' + %trim(replyQueue) + ')';
QCMDEXC(dltCmd : 100);

*inlr = *on;
return;
```

The shape is the same loop: receive work, build prompt, send to AI, wait for reply, process. What changed: "receive work" now means pulling from `WORK_QUEUE`, and the reply queue name is parameterized.

### `DEMOSTART` — populate WORK_QUEUE, submit 5 workers

```cl
PGM

DCL VAR(&BATCH_ID) TYPE(*CHAR) LEN(20)
DCL VAR(&MSG)      TYPE(*CHAR) LEN(100)

RTVSYSVAL  SYSVAL(QDATETIME) RTNVAR(&BATCH_ID)

/* Populate WORK_QUEUE with rows */
RUNSQL SQL('INSERT INTO TABLE(QSYS2.SEND_DATA_QUEUE_INFO( +
            DATA_QUEUE => ''WORK_QUEUE'', +
            DATA_QUEUE_LIBRARY => ''DEMOLIB'')) +
            SELECT ''{"row_id":'' || +
                   CAST(ROW_ID AS VARCHAR(10)) || +
                   '',"batch_id":"DEMO_BATCH_2"}'' +
            FROM DEMOLIB/DEMO_INPUT +
            WHERE PROCESSED_AT IS NULL +
            ORDER BY ROW_ID') +
       COMMIT(*NONE)

/* Submit 5 workers */
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('1'))         +
       JOB(DEMOWRK1) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB K3SAI QGPL QTEMP)                +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('2'))         +
       JOB(DEMOWRK2) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB K3SAI QGPL QTEMP)                +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('3'))         +
       JOB(DEMOWRK3) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB K3SAI QGPL QTEMP)                +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('4'))         +
       JOB(DEMOWRK4) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB K3SAI QGPL QTEMP)                +
       LOG(4 00 *NOLIST)
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK) PARM('5'))         +
       JOB(DEMOWRK5) JOBQ(QBATCH)                       +
       INLLIBL(DEMOLIB K3SAI QGPL QTEMP)                +
       LOG(4 00 *NOLIST)

CHGVAR VAR(&MSG) VALUE('Submitted 5 workers; 5 work units queued.')
SNDPGMMSG MSG(&MSG)

ENDPGM
```

The `RUNSQL` to populate `WORK_QUEUE` uses `QSYS2.SEND_DATA_QUEUE_INFO` as a target of `INSERT` — a way to send queue messages from SQL. If your IBM i version doesn't support this pattern, the alternative is calling `QSYS2.SEND_DATA_QUEUE` from a small RPG initialization program.

### `worker.php` — unchanged

Nothing to do. Same PHP worker as Quickstart 1 (RPG + PHP).

---

## Recompile

```
CRTRPGMOD MODULE(DEMOLIB/DEMOPST)  SRCFILE(DEMOLIB/QRPGLESRC)

UPDSRVPGM SRVPGM(DEMOLIB/DEMOLOGIC) +
          MODULE(DEMOLIB/DEMOPST)

CRTBNDRPG PGM(DEMOLIB/DEMOWRK)     SRCFILE(DEMOLIB/QRPGLESRC) +
          BNDSRVPGM(DEMOLIB/DEMOLOGIC YAJL/YAJL) +
          REPLACE(*YES)

CRTBNDCL  PGM(DEMOLIB/DEMOSTART)   SRCFILE(DEMOLIB/QCLLESRC) +
          REPLACE(*YES)
```

---

## Running it

Reset the rows from the previous run:

```sql
UPDATE DEMOLIB/DEMO_INPUT 
   SET PROCESSED_AT    = NULL,
       WORKER_ID       = NULL,
       AI_VERDICT      = NULL,
       AI_ACTUAL_SUM   = NULL,
       AI_RESPONSE_RAW = NULL;
```

Make sure `worker.php` is still running with `ANTHROPIC_API_KEY` exported.

```
CALL DEMOLIB/DEMOSTART
```

Run `WRKACTJOB` and you should see `DEMOWRK1` through `DEMOWRK5` running concurrently. The PHP worker output shows five round trips in rapid succession, no longer in sequential order — they're all in flight at once.

Total wall-clock time: approximately one AI call's worth, ~1-2 seconds.

---

## Verifying parallelism

```sql
SELECT ROW_ID, NUM_A, NUM_B, CLAIMED_SUM, 
       AI_VERDICT, AI_ACTUAL_SUM, 
       WORKER_ID, PROCESSED_AT
  FROM DEMOLIB/DEMO_INPUT
  ORDER BY ROW_ID;
```

Each row should have a different `WORKER_ID` (1-5), and `PROCESSED_AT` timestamps should cluster within ~1 second of each other.

---

## Wall-clock comparison

- Quickstart 1 (RPG + PHP): ~5 seconds for 5 rows
- Quickstart 2 (RPG + PHP): ~1.5 seconds for 5 rows

The parallelism dividend is ~3-4x. With more rows it scales further, bounded by AI provider rate limits and IBM i resource limits.

---

## Optional: a second PHP worker

The PHP worker can handle five concurrent calls fine via Guzzle pool, but you can prove the architecture's symmetry by starting a second one:

```
cd /home/yourprofile/aidemo
export ANTHROPIC_API_KEY="sk-ant-..."
php worker.php
```

Now two PHP workers listen on `K3SAI/AIOUTQ`. FIFO ordering ensures one of them gets each message. M PHP workers × N RPG workers, both scaling independently, both connected only by the queue contract.

---

## What you've learned

What didn't change:

- The PHP worker.
- The data queue contract.
- The AI provider.
- `DEMOPRE`.
- The architectural shape.

What did change:

- One CL program (populate queue, submit 5 jobs).
- One RPG worker (read from queue, accept worker_id).
- One small `DEMOPST` change.

That's it. Parallelism wasn't engineered — it was already there, latent, and we just had to invoke it.

---

## What's still not in this V1 demo

- Per-customer AI profiles (hardcoded in worker)
- Multi-tenant library list resolution
- Rate limiting and retry policy
- Backpressure and queue depth monitoring
- Worker pool sizing
- Structured logging and usage tracking

Each is covered in later chapters.

---

## Cleanup

```
DLTDTAQ DTAQ(DEMOLIB/WORK_QUEUE)
DLTLIB  LIB(DEMOLIB)
DLTLIB  LIB(K3SAI)
```

Stop the PHP worker(s) with Ctrl-C.

---

## Where to go from here

You've now seen the full architecture working in pure RPG and in RPG + PHP. The chapters that follow explain the design decisions, the contract, and the production-shape patterns:

- [Architecture overview]({% link architecture/index.md %}) — why this shape
- [The data queue contract]({% link contract/index.md %}) — the API between the two halves
- [The PHP worker]({% link php-worker/index.md %}) — what the production worker looks like
- [The RPG worker pool]({% link rpg-pool/index.md %}) — production patterns for the RPG side
- [Multi-tenancy]({% link multi-tenancy/index.md %}) — one shared worker, many customers
- [AI provider concerns]({% link providers/index.md %}) — keys, rates, providers, costs
- [Operating in production]({% link operations/index.md %}) — observability, monitoring, debugging

You can read in any order; they all build on the demo you just ran.

---

[Next: Architecture overview]({% link architecture/index.md %}) →
