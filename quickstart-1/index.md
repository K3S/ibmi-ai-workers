---
title: Quickstart 1 — One worker (RPG + PHP)
nav_order: 6
has_children: false
---

# Quickstart 1 — One worker, one round trip (RPG + PHP)
{: .no_toc }

**Status:** Draft V1 — code untested on Calvin yet

This is the same demo as [Quickstart 1 (RPG only)]({% link quickstart-1-rpg/index.md %}), but with PHP added as the delivery layer between RPG and the AI provider. If you haven't read the pure-RPG version and the [Why PHP]({% link why-php/index.md %}) chapter that follows it, do those first — this chapter assumes you've seen the pure-RPG round trip work and have read the case for adding PHP.

The demo achieves the same end result as the pure-RPG version: five rows get processed, the AI verifies five mathematical claims, results land in DB2. What's different is *how* the AI call happens. RPG hands the request to PHP via a data queue; PHP calls Anthropic; PHP sends the response back through another data queue; RPG processes the result. Same logical operation, different transport.

Everything you need is in this chapter. Copy the SQL, CL, RPG, and PHP into source members on your IBM i, compile, configure, and run. The corresponding source files are also in the repo's [`/demo/`](https://github.com/K3S/ibmi-ai-workers/tree/main/demo) folder.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What you'll do

By the end of this chapter you will have:

- Created a library, a table, and a data queue on your IBM i.
- Seeded the table with five rows of "is this math correct?" claims.
- Written and started a PHP worker that listens on the AI request queue, calls Anthropic, and replies.
- Compiled three RPG programs and one CL program that read the rows, build prompts, send them through the queue, parse the responses, and write results back.
- Run the demo end-to-end and seen the AI verify five mathematical claims.
- A working baseline you can come back to whenever something breaks in your real implementation.

This is the K3S-style architecture in miniature. The PHP worker here is the same shape as the production worker, just smaller. The data queue contract is exactly the [V1 contract]({% link contract/index.md %}). What you build here scales to the production version by adding profile resolution, retry middleware, and key custody — none of which change the fundamental shape.

## Prerequisites

You should be able to answer "yes" to all eight items in the [Foundations ready check]({% link foundations/index.md %}#ready-check). Specifically:

- IBM i system you can deploy to, with authority to create libraries, tables, queues, and compile programs.
- PHP 8.1+ runs from QSH with `ext-curl`, `ext-ibm_db2`, and Composer.
- Outbound HTTPS to `api.anthropic.com` works from your IBM i.
- An Anthropic API key.
- YAJL installed for RPG JSON parsing.
- Comfortable enough with RPG and CL to compile programs from source.

The demo creates two libraries (`DEMOLIB` and `K3SAI`) and one table (`DEMOLIB/DEMO_INPUT`). Cleanup at the end removes both libraries entirely.

---

## The big picture for this demo

```
                ┌───────────────────────┐
                │  User runs DEMOSTART  │
                └───────────┬───────────┘
                            │ SBMJOB
                            ▼
                ┌───────────────────────┐
                │     DEMOWRK (RPG)     │
                │                       │
                │  Loop over rows:      │
                │   call DEMOPRE        │ ───► builds prompt from DEMO_INPUT row
                │   send AI request     │
                │   wait for reply      │
                │   call DEMOPST        │ ───► writes verdict to DEMO_INPUT
                └───────────┬───────────┘
                            │ JSON message
                            ▼
                ┌───────────────────────┐
                │   AI_OUT_QUEUE        │  in K3SAI library
                └───────────┬───────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │    worker.php         │
                │                       │
                │  Pull request         │
                │  Call Anthropic       │
                │  Send reply           │
                └───────────┬───────────┘
                            │ JSON message
                            ▼
                ┌───────────────────────┐
                │   RPLY_000001         │  in DEMOLIB
                │   (created by         │
                │    DEMOWRK)           │
                └───────────────────────┘
```

Compare this to the pure-RPG version: the new piece is the PHP worker in the middle, sitting between RPG and Anthropic. The data queues (`AI_OUT_QUEUE` and `RPLY_*`) are the language boundary. Everything else is structurally similar to what you've already seen.

For this V1 demo we're skipping the architecture's separate `WORK_QUEUE` (the queue that distributes work to multiple RPG workers) — we have one worker and it just iterates the table. The [Quickstart 2 chapter]({% link quickstart-2/index.md %}) introduces `WORK_QUEUE` when we go from one worker to five.

---

## Setup

### Step 1: Create the libraries

```
CRTLIB LIB(DEMOLIB) TEXT('AI Workers Demo - customer library')
CRTLIB LIB(K3SAI) TEXT('AI Workers Demo - shared admin library')
```

`DEMOLIB` is our pretend customer library. `K3SAI` is the shared admin library where the AI request queue lives.

### Step 2: Create the input table

```sql
CREATE TABLE DEMOLIB/DEMO_INPUT (
    ROW_ID            INTEGER       NOT NULL,
    NUM_A             INTEGER       NOT NULL,
    NUM_B             INTEGER       NOT NULL,
    CLAIMED_SUM       INTEGER       NOT NULL,
    EXPECTED_CORRECT  CHAR(1)       NOT NULL,
    WORKER_ID         INTEGER,
    AI_VERDICT        CHAR(1),
    AI_ACTUAL_SUM     INTEGER,
    AI_RESPONSE_RAW   VARCHAR(2000),
    PROCESSED_AT      TIMESTAMP,
    PRIMARY KEY (ROW_ID)
);

LABEL ON TABLE DEMOLIB/DEMO_INPUT IS 'AI Demo Input Rows';
```

### Step 3: Create the AI request queue

```
CRTDTAQ DTAQ(K3SAI/AIOUTQ) +
        TYPE(*STD) +
        MAXLEN(2000000) +
        SEQ(*FIFO) +
        FORCE(*NO) +
        AUT(*USE) +
        CCSID(1208) +
        TEXT('K3S AI Worker - inbound request queue')
```

`CCSID(1208)` is critical — data queues store bytes, and we're putting UTF-8 JSON on this queue. If the CCSID is wrong, JSON parsing on either end will mysteriously fail on any non-ASCII character.

We're not creating the reply queue yet. The RPG worker creates and destroys its own reply queue at job start and end.

### Step 4: Seed the input table

```sql
INSERT INTO DEMOLIB/DEMO_INPUT 
    (ROW_ID, NUM_A, NUM_B, CLAIMED_SUM, EXPECTED_CORRECT) 
VALUES
    (1,   1,   1,   2, 'Y'),
    (2,   3,   4,   8, 'N'),
    (3,  10,  15,  25, 'Y'),
    (4,   7,   7,  13, 'N'),
    (5, 100, 200, 300, 'Y');
```

---

## The PHP worker

The PHP worker is a CLI script that runs continuously. It listens on `K3SAI/AIOUTQ`, pulls each request as it arrives, calls Anthropic, and sends the response to whatever reply queue the request named.

We'll set it up in `/home/yourprofile/aidemo/`. Adjust the path to suit your system.

### Directory structure

```
/home/yourprofile/aidemo/
├── composer.json
├── worker.php
└── vendor/         (created by composer install)
```

### `composer.json`

```json
{
    "name": "k3s/ibmi-ai-workers-demo",
    "description": "Demo PHP worker for the IBM i AI Workers guide",
    "license": "MIT",
    "require": {
        "php": ">=8.1",
        "guzzlehttp/guzzle": "^7.8"
    }
}
```

Just one dependency: Guzzle for HTTPS. Data queue access goes through `ibm_db2` (PHP extension, not a Composer package) using IBM i SQL services.

### `worker.php`

```php
<?php
declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

use GuzzleHttp\Client;
use GuzzleHttp\Exception\GuzzleException;

// === Configuration ===
$apiKey    = getenv('ANTHROPIC_API_KEY') ?: exit("ANTHROPIC_API_KEY env var not set\n");
$model     = 'claude-sonnet-4-5';
$queueLib  = 'K3SAI';
$queueName = 'AIOUTQ';

// === DB2 connection (used for data queue SQL services) ===
$conn = db2_connect('*LOCAL', '', '');
if (!$conn) {
    exit("DB2 connect failed: " . db2_conn_errormsg() . "\n");
}

// === HTTP client ===
$http = new Client([
    'base_uri' => 'https://api.anthropic.com',
    'timeout'  => 60.0,
    'headers'  => [
        'x-api-key'         => $apiKey,
        'anthropic-version' => '2023-06-01',
        'content-type'      => 'application/json',
    ],
]);

echo "[worker] Started. Listening on {$queueLib}/{$queueName}\n";
echo "[worker] Press Ctrl-C to stop.\n";

while (true) {
    $request = receiveRequest($conn, $queueLib, $queueName);
    if ($request === null) {
        continue;
    }

    $start = microtime(true);
    $reply = callAi($http, $model, $request);
    $reply['latency_ms'] = (int) round((microtime(true) - $start) * 1000);
    $reply['metadata']   = $request['metadata'] ?? new stdClass();

    sendReply($conn, $request['reply_queue'], $reply);

    echo sprintf(
        "[worker] %s row=%s status=%s %dms\n",
        substr($request['request_id'] ?? '?', 0, 8),
        $request['metadata']['row_id'] ?? '?',
        $reply['status'],
        $reply['latency_ms']
    );
}

function receiveRequest($conn, string $lib, string $name): ?array
{
    $sql = "SELECT MESSAGE_DATA FROM TABLE(QSYS2.RECEIVE_DATA_QUEUE(
                DATA_QUEUE         => ?,
                DATA_QUEUE_LIBRARY => ?,
                WAIT_TIME          => 30,
                REMOVE_MESSAGE     => 'YES'
            ))";

    $stmt = db2_prepare($conn, $sql);
    db2_execute($stmt, [$name, $lib]);
    $row = db2_fetch_assoc($stmt);

    if (!$row || empty($row['MESSAGE_DATA'])) {
        return null;
    }

    try {
        return json_decode($row['MESSAGE_DATA'], true, 512, JSON_THROW_ON_ERROR);
    } catch (JsonException $e) {
        echo "[worker] Bad JSON received: " . $e->getMessage() . "\n";
        return null;
    }
}

function callAi(Client $http, string $defaultModel, array $request): array
{
    $base = [
        'version'    => '1.0',
        'request_id' => $request['request_id'] ?? 'unknown',
    ];

    try {
        $response = $http->post('/v1/messages', [
            'json' => [
                'model'       => $request['model_override'] ?? $defaultModel,
                'max_tokens'  => $request['max_tokens']     ?? 1024,
                'temperature' => $request['temperature']    ?? 0.0,
                'messages'    => [
                    ['role' => 'user', 'content' => $request['prompt']],
                ],
            ],
        ]);

        $body    = json_decode((string) $response->getBody(), true);
        $content = $body['content'][0]['text'] ?? '';

        return $base + [
            'status'        => 'success',
            'response'      => $content,
            'model_used'    => $body['model']                  ?? $defaultModel,
            'tokens_in'     => $body['usage']['input_tokens']  ?? 0,
            'tokens_out'    => $body['usage']['output_tokens'] ?? 0,
            'finish_reason' => $body['stop_reason']            ?? 'unknown',
        ];
    } catch (GuzzleException $e) {
        return $base + [
            'status'        => 'error',
            'error_code'    => 'PROVIDER_ERROR',
            'error_message' => $e->getMessage(),
            'attempts'      => 1,
        ];
    }
}

function sendReply($conn, array $replyQueue, array $reply): void
{
    $sql = "CALL QSYS2.SEND_DATA_QUEUE(
                MESSAGE_DATA       => ?,
                DATA_QUEUE         => ?,
                DATA_QUEUE_LIBRARY => ?
            )";
    $stmt = db2_prepare($conn, $sql);
    db2_execute($stmt, [
        json_encode($reply),
        $replyQueue['name'],
        $replyQueue['library'],
    ]);
}
```

What's worth noticing:

The data queue access uses SQL services — `QSYS2.RECEIVE_DATA_QUEUE` and `QSYS2.SEND_DATA_QUEUE`. PHP just talks DB2; DB2 talks to the queues.

The `WAIT_TIME => 30` makes PHP block for up to 30 seconds waiting for a message. If nothing arrives, it returns empty and we loop. The worker doesn't burn CPU when idle.

Error handling is minimal — a Guzzle exception becomes a `PROVIDER_ERROR` reply. Production handles 429s with retry, validates request fields, logs structured data.

The reply structure exactly matches the [V1 contract]({% link contract/index.md %}).

### Install dependencies

```
cd /home/yourprofile/aidemo
composer install
```

### Set the API key

```
export ANTHROPIC_API_KEY="sk-ant-..."
```

### Start the worker

```
php worker.php
```

You should see:

```
[worker] Started. Listening on K3SAI/AIOUTQ
[worker] Press Ctrl-C to stop.
```

The worker is now waiting for requests. Leave this terminal open.

---

## The RPG and CL programs

Three RPG programs and one CL program. Compile each into `DEMOLIB`.

### `DEMOPRE` — pre-AI logic

Same as the pure-RPG version. Reads a row, builds the prompt.

```rpg
**FREE
ctl-opt nomain;

dcl-proc DEMOPRE export;
  dcl-pi *n varchar(2000);
    inRowId int(10) const;
  end-pi;

  dcl-s prompt    varchar(2000);
  dcl-s numA      int(10);
  dcl-s numB      int(10);
  dcl-s claimed   int(10);

  exec sql 
    select NUM_A, NUM_B, CLAIMED_SUM
      into :numA, :numB, :claimed
      from DEMOLIB/DEMO_INPUT
     where ROW_ID = :inRowId;

  prompt = 'You are a math verifier. A user claims that ' +
           %char(numA) + ' + ' + %char(numB) + ' = ' + 
           %char(claimed) + '. ' +
           'Verify the claim. Respond with JSON only, in exactly ' +
           'this format: ' +
           '{"correct": true|false, "actual_sum": <integer>}. ' +
           'Do not include any other text or code fences.';

  return prompt;
end-proc;
```

### `DEMOPST` — post-AI logic

Different from the pure-RPG version because the response shape is different. Here the response is the V1 contract envelope (PHP wrapped Anthropic's response in our standard format), so we parse the envelope first, then the AI's inner JSON.

```rpg
**FREE
ctl-opt nomain;

dcl-proc DEMOPST export;
  dcl-pi *n;
    inRowId       int(10)        const;
    inResponseJson varchar(4000) const;
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

  dcl-s verdict char(1);
  dcl-s rawResponse varchar(2000);

  // Parse the outer reply (PHP -> RPG envelope)
  data-into reply %data(inResponseJson) %parser('YAJL/YAJLINTO');

  if reply.status <> 'success';
    rawResponse = 'ERROR: ' + reply.status;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp,
             WORKER_ID       = 1
       where ROW_ID = :inRowId;
    return;
  endif;

  // Parse the AI's inner JSON response
  monitor;
    data-into aiResult %data(reply.response) %parser('YAJL/YAJLINTO');
  on-error;
    rawResponse = reply.response;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp,
             WORKER_ID       = 1
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
           WORKER_ID       = 1
     where ROW_ID = :inRowId;
end-proc;
```

Notice this version is *simpler* than the pure-RPG version. PHP did the work of parsing Anthropic's API envelope and exposing just the AI's text in a normalized field. Our RPG only has to deal with the contract response, not the provider-specific shape.

### `DEMOWRK` — the worker

The orchestrator. Loops over unprocessed rows, calls `DEMOPRE`, sends to AI_OUT_QUEUE, waits for reply, calls `DEMOPST`. Creates and tears down its own reply queue.

```rpg
**FREE
ctl-opt dftactgrp(*no) actgrp('DEMOWRK')
        option(*srcstmt: *nodebugio: *nounref);

dcl-pr DEMOPRE varchar(2000) extproc('DEMOPRE');
  rowId int(10) const;
end-pr;

dcl-pr DEMOPST extproc('DEMOPST');
  rowId        int(10)       const;
  responseJson varchar(4000) const;
end-pr;

dcl-pr QCMDEXC extpgm('QCMDEXC');
  command   char(2000) const;
  cmdLength packed(15:5) const;
end-pr;

dcl-s rowId       int(10);
dcl-s prompt      varchar(2000);
dcl-s requestJson varchar(4000);
dcl-s responseJson varchar(4000);
dcl-s requestId   varchar(36);
dcl-s replyQueue  char(10) inz('RPLY_000001');
dcl-s replyLib    char(10) inz('DEMOLIB');
dcl-s eof         ind inz(*off);

dcl-ds request qualified;
  version    varchar(10);
  request_id varchar(36);
  customer   varchar(10);
  profile_ref varchar(20);
  dcl-ds reply_queue;
    library varchar(10);
    name    varchar(10);
  end-ds;
  prompt     varchar(2000);
  max_tokens int(10);
  temperature packed(3:1);
  dcl-ds metadata;
    row_id   int(10);
    batch_id varchar(20);
  end-ds;
end-ds;

QCMDEXC('CRTDTAQ DTAQ(DEMOLIB/RPLY_000001) +
         TYPE(*STD) MAXLEN(2000000) +
         SEQ(*FIFO) CCSID(1208) +
         AUT(*USE) +
         TEXT(''Demo reply queue worker 1'')' : 200);

exec sql declare workCursor cursor for
  select ROW_ID 
    from DEMOLIB/DEMO_INPUT
   where PROCESSED_AT is null
   order by ROW_ID;

exec sql open workCursor;

dou eof;
  exec sql fetch workCursor into :rowId;
  if sqlcode <> 0;
    eof = *on;
    leave;
  endif;

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
  request.metadata.batch_id = 'DEMO_BATCH';

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
    DEMOPST(rowId : responseJson);
  else;
    exec sql 
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = 'TIMEOUT',
             PROCESSED_AT    = current_timestamp,
             WORKER_ID       = 1
       where ROW_ID = :rowId;
  endif;

enddo;

exec sql close workCursor;

QCMDEXC('DLTDTAQ DTAQ(DEMOLIB/RPLY_000001)' : 36);

*inlr = *on;
return;
```

Compare to the pure-RPG `DEMOWRK`: this version doesn't call AICALL directly. Instead it builds a JSON request, sends it to a queue, and waits for the response. The HTTP work has moved to PHP.

### `DEMOSTART` — the entry point

```cl
PGM

SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK))            +
       JOB(DEMOWRK1)                              +
       JOBQ(QBATCH)                               +
       INLLIBL(DEMOLIB K3SAI QGPL QTEMP)         +
       LOG(4 00 *NOLIST)

SNDPGMMSG MSG('Demo worker submitted as DEMOWRK1.')

ENDPGM
```

Note `INLLIBL` includes both `DEMOLIB` and `K3SAI` so the worker can reach the AI_OUT_QUEUE.

### Compile

```
CRTRPGMOD MODULE(DEMOLIB/DEMOPRE)  SRCFILE(DEMOLIB/QRPGLESRC)
CRTRPGMOD MODULE(DEMOLIB/DEMOPST)  SRCFILE(DEMOLIB/QRPGLESRC)

CRTSRVPGM SRVPGM(DEMOLIB/DEMOLOGIC) +
          MODULE(DEMOLIB/DEMOPRE DEMOLIB/DEMOPST) +
          EXPORT(*ALL)

CRTBNDRPG PGM(DEMOLIB/DEMOWRK)     SRCFILE(DEMOLIB/QRPGLESRC) +
          BNDSRVPGM(DEMOLIB/DEMOLOGIC YAJL/YAJL)

CRTBNDCL  PGM(DEMOLIB/DEMOSTART)   SRCFILE(DEMOLIB/QCLLESRC)
```

---

## Running the demo

You need two terminals open:

**Terminal A — QSH, PHP worker:**

```
cd /home/yourprofile/aidemo
export ANTHROPIC_API_KEY="sk-ant-..."
php worker.php
```

**Terminal B — 5250 session:**

```
CALL DEMOLIB/DEMOSTART
```

You should see in Terminal B:

```
Demo worker submitted as DEMOWRK1.
```

Then in Terminal A, within a few seconds:

```
[worker] 550e8400 row=1 status=success 842ms
[worker] 9b1deb4d row=2 status=success 766ms
[worker] 38a8f2ed row=3 status=success 891ms
[worker] 5a7e9a91 row=4 status=success 803ms
[worker] f47ac10b row=5 status=success 912ms
```

Total wall-clock time is approximately 5 × per-call latency. With Anthropic's typical response time of ~1 second, expect 4-6 seconds total.

---

## Verifying results

```sql
SELECT ROW_ID, NUM_A, NUM_B, CLAIMED_SUM, 
       EXPECTED_CORRECT, AI_VERDICT, AI_ACTUAL_SUM,
       PROCESSED_AT, WORKER_ID
  FROM DEMOLIB/DEMO_INPUT
  ORDER BY ROW_ID;
```

You should see all five rows processed, with `AI_VERDICT` matching `EXPECTED_CORRECT` for all five.

---

## What's different from the pure-RPG version

What changed:

- New library (`K3SAI`) for the shared AI queue.
- New data queue (`AIOUTQ`) and per-worker reply queue (`RPLY_000001`).
- A PHP worker process running in QSH.
- `DEMOWRK` builds a JSON request and exchanges messages with PHP, instead of calling AI directly.
- `DEMOPST` parses a normalized contract response, not Anthropic's raw API envelope.

What didn't change:

- The shape of the demo (one CL → one RPG worker → process rows → write results).
- `DEMOPRE`'s logic. Pre-AI work is unchanged.
- The schema of `DEMO_INPUT`.
- The seed data.

What you've gained that wasn't in the pure-RPG version:

- A clean response shape (PHP normalized Anthropic's API).
- A natural seam where rate limiting, retry, key custody, and provider abstraction can live (in PHP).
- A path to higher throughput when you scale up — see [Quickstart 2 (RPG + PHP)]({% link quickstart-2/index.md %}).

What you've taken on:

- A second toolchain (PHP, Composer).
- A queue boundary to maintain.
- A second process to monitor and restart.
- Slight increase in operational footprint.

For a five-row demo, neither set is meaningful. At production scale, both matter — and that's the calculus the [Why PHP]({% link why-php/index.md %}) chapter walked through.

---

## Cleanup

```
DLTLIB LIB(DEMOLIB)
DLTLIB LIB(K3SAI)
```

Stop the PHP worker with Ctrl-C.

---

## What's deliberately not in this V1 demo

Same list as the pure-RPG version, plus:

- **Production-shape PHP worker code.** The PHP we're running is ~130 lines. The production version is structured into classes (Provider, ProfileResolver, Logger, etc.). See [The PHP worker]({% link php-worker/index.md %}) for the production shape.
- **Full V1 contract validation.** The PHP worker assumes well-formed requests. Production validates aggressively.
- **AI profile resolution.** Provider, model, and key are hardcoded. Production looks them up per `profile_ref`.

These are all topics covered in later chapters.

---

[Next: Quickstart 2 — Five workers (RPG + PHP)]({% link quickstart-2/index.md %}) →
