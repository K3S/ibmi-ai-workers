---
title: Operating in production
nav_order: 14
has_children: false
---

# Operating in production
{: .no_toc }

**Status:** Draft V2 V2 — speculative

Most "AI worker" tutorials stop before this chapter. That's also where most production failures live. Observability, monitoring, restart behavior, capacity planning, debugging — the boring infrastructure that determines whether the system survives contact with reality.

A note on the status: this chapter is the most speculative in the guide because, at the time of writing, K3S has not yet run this architecture in production. The patterns described here are what we believe will work based on the architecture and on general experience with similar systems. We'll revise this chapter aggressively after we've operated it for real.

This chapter assumes the RPG + PHP architecture from the K3S-shape path. Most of the patterns also apply to the pure-RPG path, with the obvious omission of the PHP-specific operational concerns.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Running PHP workers as long-lived jobs

In production, PHP workers are not started manually from QSH. They run as IBM i autostart jobs in a dedicated subsystem, the same way you'd manage any persistent background process on the platform. This section walks through the setup.

### The pieces

You need:

1. A user profile that owns the worker process
2. A directory in IFS containing the worker code
3. A CL program that launches PHP from PASE
4. A job description that points at the CL program
5. A subsystem with autostart job entries
6. An IPL startup hook so the subsystem starts automatically

Each piece is small. The whole setup is maybe 50 lines of CL.

### The user profile

The worker runs under a dedicated profile. Nobody logs in as this profile — it's a service account.

```
CRTUSRPRF USRPRF(K3SAIWRK)                                    +
          PASSWORD(*NONE)                                     +
          USRCLS(*USER)                                       +
          INLPGM(*NONE)                                       +
          INLMNU(*SIGNOFF)                                    +
          TEXT('K3S AI Worker process owner')                 +
          AUT(*EXCLUDE)
```

`PASSWORD(*NONE)` means no human can sign in as this profile. The subsystem starts jobs under it via the job description; that's the only way it gets used.

The profile needs the following authority:

- `*USE` on `/opt/k3s/ai-worker/` (read code from IFS)
- `*USE` on `K3SAI` library (read profile/key tables)
- `*CHANGE` on `K3SAI/USAGE_LOG` (write usage rows)
- `*USE` on `K3SAI/AIOUTQ` (consume AI requests)
- `*USE` on per-customer reply queues (send AI replies)

It deliberately does **not** have authority on customer operational tables. The PHP worker can't read or write customer data even if a bug tried to. The platform enforces multi-tenant isolation at the authority layer, not in application code.

### The IFS directory

```
/opt/k3s/ai-worker/
├── bin/
│   └── worker.php
├── src/
├── config/
├── composer.json
├── composer.lock
├── vendor/
└── .env
```

Owned by `K3SAIWRK`. Authority is `K3SAIWRK *RWX`, everyone else `*EXCLUDE`. The `.env` file in particular needs tight authority — if it holds secrets (and even worker-config-only env files often grow to hold them), it's a target.

### The CL launcher

The autostart job entry expects a CL command, but we want to run PHP from PASE. The bridge is a small CL program that sets up the environment and invokes PHP via QSH.

`AIWSTART.CLLE`:

```cl
PGM

/* PASE/PHP environment */
ADDENVVAR ENVVAR(PATH)                                          +
          VALUE('/QOpenSys/pkgs/bin:/usr/bin:/QOpenSys/usr/bin') +
          REPLACE(*YES)

ADDENVVAR ENVVAR(LIBPATH)                                       +
          VALUE('/QOpenSys/pkgs/lib')                           +
          REPLACE(*YES)

/* K3S AI worker configuration */
ADDENVVAR ENVVAR(K3SAI_INBOUND_LIB)   VALUE('K3SAI')   REPLACE(*YES)
ADDENVVAR ENVVAR(K3SAI_INBOUND_QUEUE) VALUE('AIOUTQ')  REPLACE(*YES)
ADDENVVAR ENVVAR(K3SAI_LOG_DEST)                                  +
          VALUE('/QIBM/UserData/K3SAI/log')                       +
          REPLACE(*YES)
ADDENVVAR ENVVAR(K3SAI_KEK_PATH)                                  +
          VALUE('/QIBM/UserData/K3SAI/kek/master.bin')            +
          REPLACE(*YES)
ADDENVVAR ENVVAR(K3SAI_TIMEOUT_MS)    VALUE('60000')   REPLACE(*YES)
ADDENVVAR ENVVAR(K3SAI_MAX_RETRIES)   VALUE('5')       REPLACE(*YES)

/* Run PHP via QSH */
QSH CMD('/QOpenSys/pkgs/bin/php /opt/k3s/ai-worker/bin/worker.php')

/* Reached only when PHP exits */
SNDPGMMSG MSG('K3S AI Worker exited; subsystem will restart.')

ENDPGM
```

Compile with `CRTBNDCL` into `K3SAI`.

A few notes:

`ADDENVVAR` is how you set environment variables that PASE picks up. The PHP `Config::loadFromEnv()` in your worker reads these.

`QSH CMD(...)` is the IBM i command for "run this in QSH (the PASE shell)." When PHP exits, the QSH command returns and the CL program continues to the `SNDPGMMSG`, then ends. The subsystem notices the autostart job ended and restarts it. That's your built-in supervision: a crashed worker comes back up automatically.

Putting environment variables in the CL launcher (rather than in `.env`) is a deliberate choice for worker-level config. Anything that's the same for every worker on the system goes here. Anything secret (API keys, encryption keys) goes through the key vault, not env vars.

### The job description

```
CRTJOBD JOBD(K3SAI/AIWRKJOBD)                                   +
        JOBQ(QSYS/QSYSNOMAX)                                    +
        TEXT('K3S AI Worker autostart job description')         +
        USER(K3SAIWRK)                                          +
        INLLIBL(K3SAI QGPL QTEMP)                              +
        OUTQ(*USRPRF)                                           +
        LOG(4 00 *NOLIST)                                       +
        LOGCLPGM(*YES)                                          +
        RQSDTA('CALL PGM(K3SAI/AIWSTART)')                     +
        AUT(*USE)
```

Notice the library list: `K3SAI QGPL QTEMP`. **No customer libraries.** PHP runs in the K3S admin context, qualifies any DB2 access explicitly (`K3SAI.AI_PROFILE`), and never relies on library list resolution to find tables. Multi-tenancy enforced at the library list level.

`USER(K3SAIWRK)` is what makes jobs run under that profile.

`RQSDTA('CALL PGM(K3SAI/AIWSTART)')` is what the job runs when it starts.

### The subsystem

```
CRTSBSD SBSD(K3SAI/AIWRK)                                       +
        POOLS((1 *BASE))                                        +
        TEXT('K3S AI Worker subsystem')                         +
        AUT(*USE)                                               +
        SGNDSPF(*NONE)                                          +
        MAXJOBS(20)
```

`POOLS((1 *BASE))` runs jobs in the system base storage pool. For high-volume production you may want a dedicated pool (`POOLS((1 *SHRPOOL1))` plus `CHGSHRPOOL`) to isolate AI workers from other batch traffic. Not necessary at V1 scale.

`MAXJOBS(20)` caps concurrent jobs. Sized to whatever you'll realistically need with headroom; raise via `CHGSBSD` later if you outgrow it.

### Autostart job entries

One per worker:

```
ADDAJE  SBSD(K3SAI/AIWRK)                                       +
        JOB(AIWORKER1)                                          +
        JOBD(K3SAI/AIWRKJOBD)

ADDAJE  SBSD(K3SAI/AIWRK)                                       +
        JOB(AIWORKER2)                                          +
        JOBD(K3SAI/AIWRKJOBD)

ADDAJE  SBSD(K3SAI/AIWRK)                                       +
        JOB(AIWORKER3)                                          +
        JOBD(K3SAI/AIWRKJOBD)

ADDAJE  SBSD(K3SAI/AIWRK)                                       +
        JOB(AIWORKER4)                                          +
        JOBD(K3SAI/AIWRKJOBD)
```

Each `ADDAJE` adds one autostart job. `STRSBS K3SAI/AIWRK` launches one job per entry. Four entries means four workers.

Adjusting worker count is `RMVAJE` and `ADDAJE`, then end and restart the subsystem.

### IPL startup

Edit your IPL startup program (`QSYS/QSTRUP` or whatever your shop uses) to include:

```cl
STRSBS SBSD(K3SAI/AIWRK)
```

After the next IPL — or after running it manually once — workers run continuously, restarting themselves if they crash, restarting the whole subsystem at every IPL.

### Sizing the worker count

V1 starting point: **4 workers.** Each holds up to 30 in-flight AI calls via Guzzle's connection pool, so 120 concurrent calls total. At ~1 second average AI latency, that's roughly 120 calls/second steady-state — enough for early platform scale.

The constraints on worker count, in order of how often they bind:

1. **AI provider concurrency.** Cap at provider's RPM/TPM divided by per-worker throughput. Anthropic Tier 4 is on the order of hundreds of RPM; with 30-in-flight per worker at ~1s each, you saturate Tier 4 around 4-8 workers. More than that is wasted.

2. **Subsystem capacity.** `MAXJOBS(20)` accommodates 8-16 workers comfortably with headroom for transient spikes. Past that, raise the cap.

3. **Memory pool.** Each worker is ~50-80 MB resident plus its DB2 connection plus Guzzle's pool. 8 workers ≈ 600 MB. Make sure your `*BASE` pool has the headroom or move workers to a dedicated pool.

4. **Variance smoothing.** One worker is a single point of failure. Two is the minimum for HA. Beyond that, more workers smooth latency variance — when one is mid-call, others are picking up new work.

Tune by measuring `AIOUTQ` queue depth during peak load. Growing depth means workers can't keep up. Steady or decreasing means you're keeping pace.

---

## Operating the workers

### Starting

```
STRSBS SBSD(K3SAI/AIWRK)
```

All AJE entries fire. Within seconds, `WRKACTJOB SBS(AIWRK)` shows them all running.

### Stopping gracefully

```
ENDSBS SBS(AIWRK) OPTION(*CNTRLD) DELAY(60)
```

Sends a controlled-end signal to each worker. The PHP signal handler catches it, finishes the current AI call, sends the reply, and exits cleanly. The subsystem ends within the 60-second window. In-flight requests are not lost.

### Stopping immediately (rare)

```
ENDSBS SBS(AIWRK) OPTION(*IMMED)
```

Workers killed mid-request. In-flight AI calls become orphaned (they may complete on the provider side but the response is lost). RPG workers waiting on replies time out and mark their rows as failed. Avoid except in emergencies.

### Updating worker code

The deploy procedure:

1. Push new code to `/opt/k3s/ai-worker/`.
2. `composer install --no-dev` if dependencies changed.
3. Verify with `php -l bin/worker.php` (lint check) and a quick smoke test in QSH.
4. `ENDSBS SBS(AIWRK) OPTION(*CNTRLD) DELAY(60)` — workers drain.
5. `STRSBS SBSD(K3SAI/AIWRK)` — workers come back up running new code.

Total downtime for the AI service: ~90 seconds, most of which is the controlled drain. RPG workers that submit requests during the gap see their requests sit on `AIOUTQ` and get serviced when workers come back up. No data is lost.

For a true zero-downtime deploy, you'd run two pools of workers and shift traffic between them. Worth doing if downtime ever becomes user-visible. Not necessary at V1 scale.

### Scaling the pool

Adding a worker:

```
ADDAJE  SBSD(K3SAI/AIWRK)                                       +
        JOB(AIWORKER5)                                          +
        JOBD(K3SAI/AIWRKJOBD)

STRSBSJOB SBS(K3SAI/AIWRK) JOB(AIWORKER5)
```

(Or end and restart the subsystem to start all AJEs cleanly.)

Removing a worker: `ENDJOB JOB(AIWORKER5) OPTION(*CNTRLD) DELAY(60)`, then `RMVAJE` if permanent.

### What "running" looks like

`WRKACTJOB SBS(AIWRK)` during normal operation:

```
Subsystem/Job  User       Type  Status   Function
AIWRK          K3SAIWRK   SBS   ACTIVE
  AIWORKER1    K3SAIWRK   ASJ   DEQW     QSQDQRCV
  AIWORKER2    K3SAIWRK   ASJ   DEQW     QSQDQRCV
  AIWORKER3    K3SAIWRK   ASJ   DEQW     QSQDQRCV
  AIWORKER4    K3SAIWRK   ASJ   DEQW     QSQDQRCV
```

`DEQW` (dequeue wait) is the expected idle status — workers are blocking on `AIOUTQ`, waiting for the next message. CPU near zero. `QSQDQRCV` is the function waiting on a data queue receive.

When a request arrives, one worker briefly transitions to `RUN` while it processes. After replying, it goes back to `DEQW`.

If you ever see all workers in `RUN` simultaneously and `AIOUTQ` queue depth growing, the pool is saturated. Add workers, or look for what's flooding the system.

If a worker shows `MSGW` (message wait — usually unhandled error), it's stuck. Check its joblog.

---

## What you need to see, daily

Three numbers should be visible at all times during normal operation:

1. **Are the PHP workers running?** Active job count under `AIWRK`, restarts in the last hour.
2. **What's the queue depth on `K3SAI/AIOUTQ`?** Should be near zero in steady state. Sustained growth means demand exceeds capacity.
3. **What's the error rate?** Percentage of requests in the last hour returning non-success status. Should be under 1% normally.

A simple dashboard with these three numbers, refreshed every minute, catches most operational problems before users notice.

A more thorough dashboard adds:

- Per-customer request counts in the last hour
- Latency p50/p95/p99 for the last hour
- Token consumption rate (tokens/second across all customers)
- Active batch count (rows with `STATUS = 'running'` in any `AI_BATCH` table)
- Outstanding batch units (sum of `TOTAL_UNITS - PROCESSED_UNITS - FAILED_UNITS`)
- Cost rate ($/hour, computed from `USAGE_LOG`)

These are SQL queries against `K3SAI.USAGE_LOG` and IBM i system services. A web dashboard pulling them every 30 seconds is enough.

---

## Logs

Four log streams. Different purposes, different volumes, different storage.

| Stream | Purpose | Volume | Retention |
|:---|:---|:---|:---|
| `K3SAI.USAGE_LOG` | Per-call billing/debugging source of truth | High (every call) | Long-term per data policy |
| Worker stderr/joblog | Operational events (start, stop, errors) | Low | Per joblog policy |
| `/QIBM/UserData/K3SAI/log/` | Structured JSON application log | Medium | 30 days |
| RPG batch joblogs | RPG worker debugging | Low | Per joblog policy |

The structured worker log file is the most actionable for an ops engineer. JSON one-liners, greppable from QSH:

```json
{"ts":"2026-05-07T14:22:18.123Z","pid":12345,"level":"info","event":"call_complete","request_id":"550e8400","customer":"ACME","provider":"anthropic","model":"claude-sonnet-4-5","tokens_in":487,"tokens_out":2,"latency_ms":842,"status":"success"}
```

Rotate daily, retain 30 days. A simple shell script wired into a CL job handles rotation:

```sh
# /QOpenSys/usr/local/bin/k3s-rotate-logs.sh
DIR=/QIBM/UserData/K3SAI/log
TODAY=$(date +%Y-%m-%d)
cd $DIR
find . -name "worker-*.log" -mtime +0 -exec mv {} {}.$TODAY \;
find . -name "worker-*.log.*" -mtime +30 -delete
```

---

## Monitoring queries

Concrete SQL you'll run a lot. Save these as views or as queries in your dashboard tool.

### System health right now

Queue depth:

```sql
SELECT MESSAGES_ON_QUEUE, MAX_LENGTH, MAX_MESSAGES
  FROM TABLE(QSYS2.DATA_QUEUE_INFO(
    DATA_QUEUE         => 'AIOUTQ',
    DATA_QUEUE_LIBRARY => 'K3SAI'
  ));
```

Active worker jobs:

```sql
SELECT JOB_NAME, JOB_STATUS, ELAPSED_CPU_PERCENTAGE, RUN_PRIORITY
  FROM TABLE(QSYS2.ACTIVE_JOB_INFO(
    SUBSYSTEM_LIST_FILTER => 'AIWRK',
    DETAILED_INFO         => 'WORK'
  ));
```

Recent errors:

```sql
SELECT REQUEST_ID, CUSTOMER, PROVIDER, STATUS, ERROR_CODE, LOGGED_AT
  FROM K3SAI.USAGE_LOG
 WHERE STATUS != 'success'
   AND LOGGED_AT >= CURRENT TIMESTAMP - 1 HOUR
 ORDER BY LOGGED_AT DESC
 FETCH FIRST 50 ROWS ONLY;
```

### Throughput in the last hour

```sql
SELECT COUNT(*) AS REQUESTS,
       COUNT(*) / 60 AS REQUESTS_PER_MIN,
       AVG(LATENCY_MS) AS AVG_LATENCY_MS,
       PERCENTILE_DISC(0.5)  WITHIN GROUP (ORDER BY LATENCY_MS) AS P50_MS,
       PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY LATENCY_MS) AS P95_MS,
       PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY LATENCY_MS) AS P99_MS
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT TIMESTAMP - 1 HOUR
   AND STATUS = 'success';
```

### Per-customer activity today

```sql
SELECT CUSTOMER,
       COUNT(*) AS REQUESTS,
       SUM(TOKENS_IN + TOKENS_OUT) AS TOTAL_TOKENS,
       SUM(COST_BASIS_USD) AS TOTAL_COST_USD,
       100.0 * SUM(CASE WHEN STATUS = 'success' THEN 1 ELSE 0 END) / COUNT(*) AS SUCCESS_PCT
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT DATE
 GROUP BY CUSTOMER
 ORDER BY TOTAL_COST_USD DESC;
```

### Active batches

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

(Repeat per customer library or use a UNION across customers.)

### Cost burn rate

```sql
SELECT CUSTOMER, MODEL, PROVIDER,
       SUM(COST_BASIS_USD) AS COST_LAST_HOUR,
       COUNT(*) AS CALLS
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT TIMESTAMP - 1 HOUR
   AND COST_BASIS_USD > 0
 GROUP BY CUSTOMER, MODEL, PROVIDER
 ORDER BY COST_LAST_HOUR DESC;
```

If a customer's cost suddenly spikes 10x, that's worth investigating. Probably a runaway batch or a bug in their `AIPRE` building enormous prompts.

---

## Common failure modes

### Workers all in DEQW, queue growing

PHP workers are blocked on the queue (waiting for messages), but the queue depth is growing (messages aren't being received).

Possible causes:
- Workers crashed and the subsystem hasn't restarted them yet (rare; usually fast).
- DB2 connection in PHP died and `RECEIVE_DATA_QUEUE` is failing silently. Check worker logs.
- The RPG side is generating requests faster than PHP can process. Add workers.

Diagnostic:

```sql
SELECT JOB_NAME, JOB_STATUS, FUNCTION
  FROM TABLE(QSYS2.ACTIVE_JOB_INFO(SUBSYSTEM_LIST_FILTER => 'AIWRK'));
```

If all workers show `DEQW QSQDQRCV`, they're healthy and waiting. Queue growing means demand exceeds capacity. If any show something else (`MSGW`, `RUN` with no progress), look at that job's joblog.

### Workers all in RUN, never returning to DEQW

Workers are stuck mid-call. Possible causes:
- AI provider is hung/very slow. Check provider status page.
- Network outage between IBM i and provider. Try `curl` from QSH.
- A bug in the worker is causing infinite loops or deadlocks.

Diagnostic: check worker stderr/joblog and the structured log file. Look for the most recent `event: call_start` without a matching `event: call_complete`.

If it's a provider hang and you can't wait it out: `ENDSBS *IMMED` and restart. In-flight requests are lost but new requests get served.

### High error rate from one provider

Most calls failing with the same error code. Possible causes:
- Provider outage (404s, 503s).
- Authentication broken (401s) — key revoked or rotated incorrectly.
- Rate limit exceeded (429s) — your account is over.

Diagnostic:

```sql
SELECT PROVIDER, ERROR_CODE, COUNT(*) AS COUNT
  FROM K3SAI.USAGE_LOG
 WHERE STATUS != 'success'
   AND LOGGED_AT >= CURRENT TIMESTAMP - 15 MINUTES
 GROUP BY PROVIDER, ERROR_CODE
 ORDER BY COUNT DESC;
```

If 401: re-check your key in `KEY_VAULT`, run a manual `curl` test with that key.
If 429: check your plan, your concurrency limits, your token bucket settings.
If 503/504: probably provider-side, ride it out (retry middleware does this automatically).

### One customer's batch is starving others

ACME's huge batch fills `AIOUTQ` faster than other customers' requests can get in. Other customers see slow response times.

Diagnostic:

```sql
SELECT CUSTOMER, COUNT(*)
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT TIMESTAMP - 5 MINUTES
 GROUP BY CUSTOMER;
```

If one customer dominates: rate limit fairness (covered in [providers chapter]({% link providers/index.md %})) needs to engage. Check that the token-bucket logic is actually running. If `AIOUTQ` is FIFO without per-customer throttling, this happens easily.

V1 may not have full fairness logic. If so, the workaround is operational: cap individual customer batch sizes, or stagger their schedules so they don't all run at once.

### Crashed RPG worker leaves orphaned reply queue

Worker process died, didn't run its cleanup. Reply queue still exists, no job is reading it.

Diagnostic:

```sql
SELECT OBJECT_NAME, OBJECT_LIBRARY, OBJECT_OWNER
  FROM TABLE(QSYS2.OBJECT_STATISTICS('*ALLUSR', '*DTAQ'))
 WHERE OBJECT_NAME LIKE 'RPLY_%'
   AND DAYS(CURRENT_DATE) - DAYS(CHANGE_TIMESTAMP) > 1;
```

Reply queues with no associated active job, more than a day old, are orphaned. A daily cleanup CL job:

```cl
PGM
DCL VAR(&LIB) TYPE(*CHAR) LEN(10)
DCL VAR(&OBJ) TYPE(*CHAR) LEN(10)

/* Find orphaned reply queues and delete them */
RUNSQL SQL('DELETE FROM TABLE(QSYS2.OBJECT_STATISTICS(''*ALLUSR'', ''*DTAQ'')) +
           WHERE OBJECT_NAME LIKE ''RPLY_%'' +
             AND DAYS(CURRENT_DATE) - DAYS(CHANGE_TIMESTAMP) > 1') +
       COMMIT(*NONE)
ENDPGM
```

Schedule via `ADDJOBSCDE` to run nightly.

### Memory growth in PHP workers

A worker should run for days without growing significantly. If a worker's resident memory grows monotonically, you have a leak.

Diagnostic: `WRKACTJOB SBS(AIWRK)`, look at memory column. Or:

```sql
SELECT JOB_NAME, ELAPSED_TIME, TEMP_STORAGE_USED
  FROM TABLE(QSYS2.ACTIVE_JOB_INFO(SUBSYSTEM_LIST_FILTER => 'AIWRK'));
```

Workers that have been running for hours and using hundreds of MB are suspect. Mitigation in the short term: restart the subsystem on a daily schedule (controlled-end at off-hours). Long-term: find and fix the leak. Common culprits: unbounded caches, accumulating log buffers, library bugs.

A "self-recycle" pattern is also fine: have each worker count its requests served, and exit cleanly after N (e.g., 10,000). The subsystem restarts it. This is a defensible production pattern even without a known leak.

---

## Backup and recovery

What needs to be backed up:

- **`K3SAI.AI_PROFILE`** — customer profiles. Rebuilding by hand is painful.
- **`K3SAI.KEY_VAULT`** — encrypted keys. Useless without the KEK, but back up both together (they're useless separately).
- **`K3SAI.USAGE_LOG`** — billing source of truth. Critical to retain.
- **The KEK file** at `/QIBM/UserData/K3SAI/kek/`. Back up to secure offline storage.
- **The PHP code** at `/opt/k3s/ai-worker/`. Source-controlled, but a snapshot of the running version is useful.

What doesn't need to be backed up:

- **`K3SAI.AIOUTQ`** and per-worker reply queues. Transient. Rebuild on the fly.
- Worker joblogs and process logs. Useful for forensics, not for recovery.

What's hard to recover and worth thinking about:

- Per-customer data inside their libraries (operational tables, batch metadata) — backup is the customer's K3S data backup, not the AI worker concern. But know that the AI worker's reply queues only matter while a batch is in flight.

---

## Disaster scenarios

### IBM i unplanned reboot mid-batch

What happens:
- All in-flight RPG worker jobs die.
- All in-flight PHP processes die.
- `AIOUTQ` may have unprocessed messages (depending on whether they were `RECEIVE`d before crash).
- `AI_BATCH` rows show `STATUS = 'running'` with stale `STARTED_AT`.
- Reply queues for crashed workers are now orphaned.

Recovery, in order:
1. After IPL, the `K3SAIWRK` subsystem restarts automatically. PHP workers come back up.
2. Run a "find stale running batches" query: `STATUS = 'running'` and `STARTED_AT > 1 hour ago`.
3. Decide per batch: mark failed, or resume. A "resume" RPG that re-queues `PENDING` rows for unfinished batches is straightforward.
4. Run the orphaned-queue cleanup to remove stale reply queues.

### KEK file corrupted/lost

Without the KEK, you can't decrypt any DEK in `KEY_VAULT`, which means you can't decrypt any customer key. The system can't make AI calls for any BYOK customer. Hosted customers using the K3S key may still work if the K3S key is in a different KEK (which it should be — see [providers]({% link providers/index.md %})).

Recovery:
1. Restore KEK from secure backup.
2. Verify by decrypting one DEK successfully.
3. If KEK truly lost beyond recovery: contact each BYOK customer to provide a new key. Their old encrypted keys are now garbage and need to be replaced.

This is why KEK backup is the single most important operational artifact in this system.

### Anthropic (or other provider) revokes K3S's key

For hosted-tier customers using K3S's account: their AI calls fail with 401 until you generate a new key, install it, and update `KEY_VAULT`. BYOK customers are unaffected — their keys are theirs.

Recovery:
1. Generate new K3S API key in the provider console.
2. Add to `KEY_VAULT` as a new version, mark old one revoked.
3. Update profiles pointing at the K3S key to point at the new version.
4. Restart workers (so the profile cache picks up new keys).

If revocation came as a surprise (not a planned rotation): figure out *why*. Probably abuse detection (a bug caused a token spike, the provider auto-revoked). Address the root cause before the new key gets revoked too.

---

## What's deliberately not in this V1 chapter

- **Distributed tracing.** Per-request traces across RPG → queue → PHP → AI provider. Useful at scale, not built-in at V1.
- **Alerting integrations.** PagerDuty, Slack, email — these depend on what your shop uses. Worth wiring up; out of scope here.
- **Cost forecasting.** Historical usage extrapolated to predict next month's bill. Easy to build from `USAGE_LOG` but not part of the worker.
- **Multi-region or HA across LPARs.** This chapter assumes one IBM i system.
- **Capacity planning models.** "How many workers will we need at 1000 customers?" — answerable from measurement, not in this chapter.

---

## Open for discussion

V1 calls in this chapter that need real-shop calibration once we run this in production:

- **Worker count default of 4.** Pure guess. Should be measured under realistic load.
- **`MAXJOBS(20)`.** Arbitrary. Tune based on actual peak concurrency.
- **Subsystem name `AIWRK`.** Could be `K3SAIWRK` or whatever K3S conventions prefer.
- **30-day log retention.** Could be longer or shorter depending on debugging vs. storage tradeoffs.
- **Self-recycle vs. infinite-running workers.** A worker that exits after 10,000 requests is more robust to slow leaks. Worth deciding deliberately.
- **Whether RPG workers also run in a dedicated subsystem.** Currently I've assumed they go to `QBATCH` like normal batch jobs. A dedicated `AIRPG` subsystem isolates them from other batch traffic. May be worth doing.

---

[Next: Reference]({% link reference/index.md %}) →
