---
title: Quickstart 1 — One worker (RPG only)
nav_order: 3
has_children: false
---

# Quickstart 1 — One worker, one round trip (RPG only)
{: .no_toc }

**Status:** Draft V1 — code untested on Calvin yet

This chapter walks you through the smallest end-to-end version of the architecture, with one important constraint: **everything is RPG.** No PHP. No second toolchain. Your IBM i talks directly to the AI provider over HTTPS using built-in SQL services.

The goal is a working AI round trip in your team's primary language, in roughly thirty minutes. After this and [Quickstart 2 (RPG only)]({% link quickstart-2-rpg/index.md %}), you'll have a complete picture of how AI integrates into RPG without introducing any new layers. That's enough for many shops. The chapters that follow ([Why PHP for the delivery layer]({% link why-php/index.md %}), then the RPG+PHP versions of the same demos) make the case for adding PHP when scale demands it — but you can stop here if pure RPG meets your needs.

Everything you need is in this chapter. Copy the SQL, CL, and RPG into source members on your IBM i, compile, configure, and run.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What you'll do

By the end of this chapter you will have:

- Created a library and a table on your IBM i.
- Seeded the table with five rows of "is this math correct?" claims.
- Compiled four RPG/CL programs that read the rows, build prompts, call Anthropic directly via HTTP, parse the responses, and write results back.
- Run the demo end-to-end and seen the AI verify five mathematical claims.
- A working baseline you can come back to whenever something breaks in your real implementation.

The demo is deliberately tiny. It's the AI round trip stripped to its skeleton, all in RPG. Every piece you see here corresponds to something larger in production: `AIPRE` is your real prompt-building business logic, `AIPOST` is your real response-handling logic, `AICALL` is the HTTP call layer that gets more sophisticated when you start handling retries and rate limits. The architecture stays the same as it grows; the parts get bigger.

## Prerequisites

You should be able to answer "yes" to all the items in the [Foundations ready check]({% link foundations/index.md %}#ready-check). Specifically for this chapter:

- IBM i system you can deploy to, with authority to create libraries, tables, and compile programs.
- Outbound HTTPS to `api.anthropic.com` works from your IBM i.
- An Anthropic API key.
- YAJL installed for RPG JSON parsing (`DATA-INTO`/`DATA-GEN` parsers). Most modern IBM i shops have it; if not, it's a free download from Scott Klement.
- IBM i 7.4 or newer for `QSYS2.HTTP_POST_VERBOSE` and other HTTP SQL services. Older versions can use `SYSTOOLS.HTTPPOSTCLOB` with adjustments noted in the troubleshooting section.

The demo creates one library (`DEMOLIB`) and one table (`DEMOLIB/DEMO_INPUT`). Cleanup at the end of the chapter removes the library entirely.

---

## The big picture for this demo

```
                ┌───────────────────────────┐
                │    User runs DEMOSTART    │
                └─────────────┬─────────────┘
                              │ SBMJOB
                              ▼
                ┌───────────────────────────┐
                │   DEMOWRK (RPG worker)    │
                │                           │
                │   For each unprocessed    │
                │   row in DEMO_INPUT:      │
                │     call AIPRE   ─────────┼──► reads row, builds prompt
                │     call AICALL  ─────────┼──► HTTPS to Anthropic
                │     call AIPOST  ─────────┼──► parses response, writes row
                └───────────────────────────┘
```

Five rows. One worker. Five round trips, sequentially. The worker does everything: orchestration, HTTP, prompt construction, response parsing, database updates. It's all in RPG.

In [Quickstart 2 (RPG only)]({% link quickstart-2-rpg/index.md %}) we add parallelism by running multiple workers concurrently. The shape stays the same — just N workers instead of 1.

---

## Setup

### Step 1: Create the library

```
CRTLIB LIB(DEMOLIB) TEXT('AI Workers RPG-only Demo')
```

### Step 2: Create the input table

Run this from STRSQL, ACS Run SQL Scripts, or `RUNSQLSTM`:

```sql
CREATE TABLE DEMOLIB/DEMO_INPUT (
    ROW_ID            INTEGER       NOT NULL,
    NUM_A             INTEGER       NOT NULL,
    NUM_B             INTEGER       NOT NULL,
    CLAIMED_SUM       INTEGER       NOT NULL,
    EXPECTED_CORRECT  CHAR(1)       NOT NULL,
    AI_VERDICT        CHAR(1),
    AI_ACTUAL_SUM     INTEGER,
    AI_RESPONSE_RAW   VARCHAR(4000),
    PROCESSED_AT      TIMESTAMP,
    PRIMARY KEY (ROW_ID)
);

LABEL ON TABLE DEMOLIB/DEMO_INPUT IS 'AI Demo Input Rows';
```

### Step 3: Seed the table

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

Three correct claims, two wrong. The AI should agree with the correct ones and disagree with the wrong ones.

### Step 4: Set up an environment variable for the API key

The worker needs to read the Anthropic API key at runtime. The cleanest way to do this in RPG without baking the key into source code is to store it in a data area. Create one:

```
CRTDTAARA DTAARA(DEMOLIB/AIKEY) +
          TYPE(*CHAR) +
          LEN(200) +
          TEXT('Anthropic API key for demo') +
          AUT(*EXCLUDE)
```

Then set its value (replace with your actual key):

```
CHGDTAARA DTAARA(DEMOLIB/AIKEY) VALUE('sk-ant-YOUR-KEY-HERE')
```

Restrict authority to only the user that will run the demo:

```
GRTOBJAUT OBJ(DEMOLIB/AIKEY) OBJTYPE(*DTAARA) USER(YOURUSER) AUT(*USE)
```

For production, key custody is significantly more involved (encryption at rest, per-customer keys). See [AI provider concerns]({% link providers/index.md %}). For the demo, a tightly-restricted data area is enough.

---

## The RPG and CL programs

Four programs. Compile each into `DEMOLIB`.

### `AIPRE` — pre-AI logic

Reads a row from `DEMO_INPUT` and builds the prompt string. In production this is where significant business logic lives (vendor lookups, demand history, policy text). For the demo it formats three numbers into a sentence.

```rpg
**FREE
ctl-opt nomain;

dcl-proc AIPRE export;
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

Identical in spirit to the RPG+PHP version. The pre-AI logic doesn't care how the AI gets called.

### `AICALL` — the HTTP call to Anthropic

This is the program that's *new* in the pure-RPG version. It takes a prompt, calls Anthropic via SQL HTTP services, and returns the raw response.

```rpg
**FREE
ctl-opt nomain;

dcl-proc AICALL export;
  dcl-pi *n varchar(8000);
    inPrompt varchar(2000) const;
  end-pi;

  dcl-s requestJson varchar(4000);
  dcl-s responseJson varchar(8000);
  dcl-s apiKey varchar(200);
  dcl-s headers varchar(1000);

  // Request structure for DATA-GEN
  dcl-ds request qualified;
    model varchar(50);
    max_tokens int(10);
    temperature packed(3:1);
    dcl-ds messages dim(1);
      role varchar(20);
      content varchar(2000);
    end-ds;
  end-ds;

  // Get API key from data area
  in apiKey datarea('AIKEY');
  apiKey = %trimr(apiKey);

  // Build request structure
  request.model = 'claude-sonnet-4-5';
  request.max_tokens = 50;
  request.temperature = 0;
  request.messages(1).role = 'user';
  request.messages(1).content = inPrompt;

  // Generate request JSON
  data-gen request 
           %data(requestJson : 'noprefix=request_') 
           %gen('YAJL/YAJLDTAGEN');

  // Build headers
  headers = '<httpHeader>' +
            '<header name="x-api-key" value="' + %trim(apiKey) + '"/>' +
            '<header name="anthropic-version" value="2023-06-01"/>' +
            '<header name="content-type" value="application/json"/>' +
            '</httpHeader>';

  // Make the HTTPS POST call
  exec sql 
    set :responseJson = 
      QSYS2.HTTP_POST(
        URL => 'https://api.anthropic.com/v1/messages',
        BODY => :requestJson,
        HEADER => :headers
      );

  return responseJson;
end-proc;
```

A few notes:

The `QSYS2.HTTP_POST` SQL service does the HTTPS call. It's part of IBM i 7.4 and later. Returns the response body as a CLOB.

The headers are passed as XML, which is the format `QSYS2.HTTP_POST` expects. Older versions of IBM i may use `SYSTOOLS.HTTPPOSTCLOB` with slightly different syntax.

The request JSON is built with `DATA-GEN` and YAJL — same pattern as the RPG+PHP version. The request structure includes the full Anthropic Messages API shape.

The API key is read from the data area at the top of the procedure. For production, this would be more sophisticated — per-customer key resolution, decryption, audit logging. For the demo, one shared key works.

**TLS handshake overhead.** Each call to `AICALL` opens a fresh TLS connection to Anthropic. That's roughly 100-300ms of overhead per call, on top of the AI's response time. For a five-row demo this is invisible. For 10,000-row batches it matters — and it's the main reason [the PHP transport layer]({% link why-php/index.md %}) exists.

### `AIPOST` — post-AI logic

Takes the row ID and the raw response from Anthropic. Parses out the AI's text response, parses *that* (because the AI's response is itself JSON), and updates the row.

```rpg
**FREE
ctl-opt nomain;

dcl-proc AIPOST export;
  dcl-pi *n;
    inRowId        int(10)        const;
    inResponseRaw  varchar(8000)  const;
  end-pi;

  // Anthropic API response structure
  dcl-ds anthropicResp qualified;
    dcl-ds content dim(10);
      type varchar(20);
      text varchar(2000);
    end-ds;
    dcl-ds usage;
      input_tokens  int(10);
      output_tokens int(10);
    end-ds;
    stop_reason varchar(50);
  end-ds;

  // The AI's inner JSON response
  dcl-ds aiResult qualified;
    correct    ind;
    actual_sum int(10);
  end-ds;

  dcl-s aiText      varchar(2000);
  dcl-s verdict     char(1);
  dcl-s rawResponse varchar(2000);

  // Parse Anthropic's response envelope
  monitor;
    data-into anthropicResp 
              %data(inResponseRaw) 
              %parser('YAJL/YAJLINTO' : 'allow_extra=yes');
  on-error;
    rawResponse = 'PARSE_ERROR: ' + %subst(inResponseRaw : 1 : 500);
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp
       where ROW_ID = :inRowId;
    return;
  endmon;

  // Extract the AI's text from the first content block
  if %elem(anthropicResp.content) >= 1 and 
     anthropicResp.content(1).type = 'text';
    aiText = anthropicResp.content(1).text;
  else;
    rawResponse = 'NO_CONTENT';
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp
       where ROW_ID = :inRowId;
    return;
  endif;

  // Parse the AI's structured response
  monitor;
    data-into aiResult 
              %data(aiText) 
              %parser('YAJL/YAJLINTO');
  on-error;
    rawResponse = aiText;
    exec sql
      update DEMOLIB/DEMO_INPUT
         set AI_RESPONSE_RAW = :rawResponse,
             PROCESSED_AT    = current_timestamp
       where ROW_ID = :inRowId;
    return;
  endmon;

  // Happy path
  if aiResult.correct;
    verdict = 'Y';
  else;
    verdict = 'N';
  endif;

  rawResponse = aiText;

  exec sql
    update DEMOLIB/DEMO_INPUT
       set AI_VERDICT      = :verdict,
           AI_ACTUAL_SUM   = :aiResult.actual_sum,
           AI_RESPONSE_RAW = :rawResponse,
           PROCESSED_AT    = current_timestamp
     where ROW_ID = :inRowId;
end-proc;
```

This program is meaningfully more complex than its RPG+PHP counterpart. The reason: in the RPG+PHP version, PHP normalized the response shape before sending it back. Here, RPG receives Anthropic's raw response and has to navigate it directly: first the API envelope (`content[0].text`), then the AI's inner JSON.

That's the cost of pure RPG — every program that calls AI has to handle provider-specific response shapes itself, or you build a `AICALL` wrapper that normalizes them. We're doing the simpler thing for the demo.

### `DEMOWRK` — the worker

Loops over unprocessed rows. For each: calls `AIPRE`, calls `AICALL`, calls `AIPOST`. No queues, no parallelism — just a sequential cursor.

```rpg
**FREE
ctl-opt dftactgrp(*no) actgrp('DEMOWRK')
        option(*srcstmt: *nodebugio: *nounref);

// External procedure prototypes
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

// Variables
dcl-s rowId       int(10);
dcl-s prompt      varchar(2000);
dcl-s responseRaw varchar(8000);
dcl-s eof         ind inz(*off);

// === Main loop ===
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

  prompt      = AIPRE(rowId);
  responseRaw = AICALL(prompt);
  AIPOST(rowId : responseRaw);
enddo;

exec sql close workCursor;

*inlr = *on;
return;
```

This is the simplest version of the worker possible. Cursor over candidate rows, three procedure calls per row, done. In [Quickstart 2 (RPG only)]({% link quickstart-2-rpg/index.md %}) we add multiple workers and replace the cursor with SQL claiming so they don't step on each other.

### `DEMOSTART` — the entry point

```cl
PGM

/* Submit the worker to batch */
SBMJOB CMD(CALL PGM(DEMOLIB/DEMOWRK))            +
       JOB(DEMOWRK1)                              +
       JOBQ(QBATCH)                               +
       INLLIBL(DEMOLIB QGPL QTEMP)                +
       LOG(4 00 *NOLIST)

SNDPGMMSG MSG('Demo worker submitted as DEMOWRK1.')

ENDPGM
```

Same as the RPG+PHP version. The CL doesn't change — only what `DEMOWRK` does internally.

### Compile

From a 5250 command line, with `DEMOLIB` in your library list:

```
CRTRPGMOD MODULE(DEMOLIB/AIPRE)   SRCFILE(DEMOLIB/QRPGLESRC)
CRTRPGMOD MODULE(DEMOLIB/AICALL)  SRCFILE(DEMOLIB/QRPGLESRC)
CRTRPGMOD MODULE(DEMOLIB/AIPOST)  SRCFILE(DEMOLIB/QRPGLESRC)

CRTSRVPGM SRVPGM(DEMOLIB/AILOGIC) +
          MODULE(DEMOLIB/AIPRE DEMOLIB/AICALL DEMOLIB/AIPOST) +
          EXPORT(*ALL)

CRTBNDRPG PGM(DEMOLIB/DEMOWRK)    SRCFILE(DEMOLIB/QRPGLESRC) +
          BNDSRVPGM(DEMOLIB/AILOGIC YAJL/YAJL)

CRTBNDCL  PGM(DEMOLIB/DEMOSTART)  SRCFILE(DEMOLIB/QCLLESRC)
```

If you don't have `DEMOLIB/QRPGLESRC` and `DEMOLIB/QCLLESRC` source files yet:

```
CRTSRCPF FILE(DEMOLIB/QRPGLESRC) RCDLEN(112)
CRTSRCPF FILE(DEMOLIB/QCLLESRC)  RCDLEN(80)
```

---

## Running the demo

From a 5250 session:

```
CALL DEMOLIB/DEMOSTART
```

Output:

```
Demo worker submitted as DEMOWRK1.
```

The worker is now processing rows in the background. With one worker doing five rows sequentially, total time is roughly 5 × Anthropic's response time. Expect 5-10 seconds.

You can watch the worker by checking its joblog:

```
WRKSPLF SELECT(*ALL DEMOWRK1)
```

Or by checking active jobs:

```
WRKACTJOB SBS(QBATCH)
```

You should see `DEMOWRK1` running for a few seconds, then complete.

---

## Verifying results

```sql
SELECT ROW_ID, NUM_A, NUM_B, CLAIMED_SUM, 
       EXPECTED_CORRECT, AI_VERDICT, AI_ACTUAL_SUM,
       PROCESSED_AT
  FROM DEMOLIB/DEMO_INPUT
  ORDER BY ROW_ID;
```

You should see all five rows filled in:

```
ROW_ID NUM_A NUM_B CLAIMED EXPECTED AI_VERDICT AI_ACTUAL PROCESSED_AT
   1     1     1      2      Y         Y          2     2026-05-07-...
   2     3     4      8      N         N          7     2026-05-07-...
   3    10    15     25      Y         Y         25     2026-05-07-...
   4     7     7     13      N         N         14     2026-05-07-...
   5   100   200    300      Y         Y        300     2026-05-07-...
```

What you're looking for:

- **All five rows have `PROCESSED_AT` filled in.** If not, the worker errored. Check the joblog of `DEMOWRK1` — likely an HTTP failure or a parse error.
- **`AI_VERDICT` matches `EXPECTED_CORRECT` for all five.** If the AI got addition wrong, that's a model issue, not a demo bug.
- **`AI_ACTUAL_SUM` is the correct math.** Should be 2, 7, 25, 14, 300.

If you want to see the AI's raw responses:

```sql
SELECT ROW_ID, AI_RESPONSE_RAW 
  FROM DEMOLIB/DEMO_INPUT 
  ORDER BY ROW_ID;
```

You should see clean JSON like `{"correct": true, "actual_sum": 2}` for each row.

---

## Re-running

`DEMOWRK` only processes rows where `PROCESSED_AT IS NULL`. To run again:

```sql
UPDATE DEMOLIB/DEMO_INPUT 
   SET PROCESSED_AT    = NULL,
       AI_VERDICT      = NULL,
       AI_ACTUAL_SUM   = NULL,
       AI_RESPONSE_RAW = NULL;
```

Then `CALL DEMOLIB/DEMOSTART` again.

---

## Cleanup

```
DLTLIB LIB(DEMOLIB)
```

This removes the table, the data area (and the API key with it), the programs, and the source. Nothing else lingers on the system.

---

## What you've built

A working AI round trip in pure RPG. No external transport layer, no second toolchain. The `QSYS2.HTTP_POST` SQL service does the HTTPS work; YAJL handles the JSON; modern free-format RPG ties it together.

This is enough for many production use cases. If your AI workload is modest — hundreds of calls per batch, a few batches per day — pure RPG handles it cleanly.

---

## What's deliberately not in this demo

- **Parallelism.** One row at a time. Five rows takes 5x the per-call latency. We add parallelism in [Quickstart 2 (RPG only)]({% link quickstart-2-rpg/index.md %}).
- **Connection reuse.** Each `AICALL` opens a fresh TLS connection to Anthropic. ~200ms of handshake overhead per call. Real but acceptable for this scale.
- **Retry logic.** A 429 from Anthropic fails the call. Production would back off and retry.
- **Per-customer profiles.** Hardcoded model and key in `AICALL`. Production resolves these per-customer.
- **Rate limit handling.** No throttling.
- **Provider abstraction.** The code is hardcoded for Anthropic's API shape. Switching to OpenAI means rewriting `AICALL` and `AIPOST`.
- **Streaming responses.** The whole response is collected before parsing.

These limitations accumulate. By the time you need all of them, you're considering whether to build them in RPG or to introduce PHP as the transport layer. That's the conversation in [Why PHP for the delivery layer]({% link why-php/index.md %}).

---

## Switching to a different provider

If you'd rather use OpenAI or another provider, three things change:

- The endpoint URL in `AICALL` (e.g., `https://api.openai.com/v1/chat/completions`).
- The auth header format (`Authorization: Bearer ...` instead of `x-api-key`).
- The response parsing in `AIPOST` (OpenAI's response shape is different from Anthropic's).

The structural work is the same in any provider. Each just has its own protocol weirdness.

---

[Next: Quickstart 2 — Five workers (RPG only)]({% link quickstart-2-rpg/index.md %}) →
