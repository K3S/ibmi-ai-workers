---
title: Reference
nav_order: 15
has_children: false
---

# Reference
{: .no_toc }

**Status:** Draft V1

Cheat sheets and quick-lookup material for things you'll need to find fast: the V1 contract message formats, common SQL queries, IBM i commands, error codes, environment variables, and the mental shortcuts that take a while to internalize but are obvious once you do.

This chapter is meant for skimming, not reading top to bottom. Bookmark it.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## V1 contract — request shape

JSON sent from RPG to PHP via `K3SAI/AIOUTQ`. UTF-8 (CCSID 1208).

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "customer": "ACME",
  "profile_ref": "ACME_DEFAULT",
  "reply_queue": {
    "library": "ACME_5DTA",
    "name": "RPLY_000001"
  },
  "prompt": "...",
  "system_prompt": "(optional)",
  "max_tokens": 1024,
  "temperature": 0.0,
  "model_override": "(optional, overrides profile)",
  "timeout_ms": 60000,
  "metadata": {
    "row_id": 12345,
    "batch_id": "BATCH_2026_05_07_001"
  }
}
```

| Field | Type | Required | Notes |
|:---|:---|:---|:---|
| `version` | string | yes | Currently `"1.0"` |
| `request_id` | string | yes | UUID, unique per request |
| `customer` | string | yes | Customer code, e.g. `"ACME"` |
| `profile_ref` | string | yes | Profile name from `K3SAI.AI_PROFILE` |
| `reply_queue.library` | string | yes | Where to send reply |
| `reply_queue.name` | string | yes | Reply queue name |
| `prompt` | string | yes | The actual user prompt |
| `system_prompt` | string | no | Provider-specific system prompt |
| `max_tokens` | integer | no | Default per profile, usually 1024 |
| `temperature` | number | no | 0.0-1.0, default 0.0 |
| `model_override` | string | no | Override profile's default model |
| `timeout_ms` | integer | no | Per-call timeout, default 60000 |
| `metadata` | object | no | Free-form, echoed back in reply |

---

## V1 contract — reply shape

JSON sent from PHP to the named reply queue.

### Success

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "success",
  "response": "...",
  "model_used": "claude-sonnet-4-5",
  "tokens_in": 487,
  "tokens_out": 312,
  "latency_ms": 842,
  "finish_reason": "end_turn",
  "metadata": { "row_id": 12345, "batch_id": "..." }
}
```

### Error

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "error",
  "error_code": "RATE_LIMITED",
  "error_message": "Anthropic rate limited after 5 attempts",
  "attempts": 5,
  "metadata": { "row_id": 12345, "batch_id": "..." }
}
```

| `status` | When |
|:---|:---|
| `success` | Provider returned 2xx and response parsed |
| `error` | All retries exhausted or non-retryable failure |

| `error_code` | Meaning |
|:---|:---|
| `RATE_LIMITED` | Provider returned 429 after retries exhausted |
| `PROVIDER_AUTH` | 401/403 — key invalid |
| `PROVIDER_ERROR` | 5xx after retries exhausted |
| `TIMEOUT` | Client-side timeout, provider didn't respond |
| `INVALID_REQUEST` | Provider returned 400 — malformed request |
| `PROFILE_NOT_FOUND` | `profile_ref` doesn't resolve |
| `INTERNAL` | Worker crashed/bug |

---

## Key SQL — usage and monitoring

### Queue depth

```sql
SELECT MESSAGES_ON_QUEUE
  FROM TABLE(QSYS2.DATA_QUEUE_INFO(
    DATA_QUEUE         => 'AIOUTQ',
    DATA_QUEUE_LIBRARY => 'K3SAI'
  ));
```

### Active workers

```sql
SELECT JOB_NAME, JOB_STATUS, ELAPSED_CPU_PERCENTAGE
  FROM TABLE(QSYS2.ACTIVE_JOB_INFO(SUBSYSTEM_LIST_FILTER => 'AIWRK'));
```

### Recent calls (last hour)

```sql
SELECT REQUEST_ID, CUSTOMER, PROVIDER, MODEL, STATUS, 
       LATENCY_MS, TOKENS_IN, TOKENS_OUT, COST_BASIS_USD, LOGGED_AT
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT TIMESTAMP - 1 HOUR
 ORDER BY LOGGED_AT DESC;
```

### Throughput in last hour

```sql
SELECT COUNT(*) AS REQUESTS,
       AVG(LATENCY_MS) AS AVG_MS,
       PERCENTILE_DISC(0.5)  WITHIN GROUP (ORDER BY LATENCY_MS) AS P50,
       PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY LATENCY_MS) AS P95,
       PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY LATENCY_MS) AS P99
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT TIMESTAMP - 1 HOUR
   AND STATUS = 'success';
```

### Cost today, by customer

```sql
SELECT CUSTOMER,
       COUNT(*) AS CALLS,
       SUM(TOKENS_IN + TOKENS_OUT) AS TOKENS,
       SUM(COST_BASIS_USD) AS COST_USD
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT DATE
 GROUP BY CUSTOMER
 ORDER BY COST_USD DESC;
```

### Error rate, last hour, by customer

```sql
SELECT CUSTOMER,
       COUNT(*) AS TOTAL,
       SUM(CASE WHEN STATUS != 'success' THEN 1 ELSE 0 END) AS ERRORS,
       100.0 * SUM(CASE WHEN STATUS != 'success' THEN 1 ELSE 0 END) / COUNT(*) AS ERROR_PCT
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT TIMESTAMP - 1 HOUR
 GROUP BY CUSTOMER
HAVING COUNT(*) > 10
 ORDER BY ERROR_PCT DESC;
```

### Active batches

```sql
SELECT BATCH_ID, STATUS, TOTAL_UNITS,
       PROCESSED_UNITS + FAILED_UNITS AS DONE,
       FAILED_UNITS,
       TIMESTAMPDIFF(2, CHAR(CURRENT_TIMESTAMP - STARTED_AT)) AS SECONDS_RUNNING
  FROM ACME_5DTA.AI_BATCH
 WHERE STATUS = 'running'
 ORDER BY STARTED_AT DESC;
```

(Repeat per customer library.)

### Sending a message to a queue from SQL

```sql
CALL QSYS2.SEND_DATA_QUEUE(
  MESSAGE_DATA       => '...',
  DATA_QUEUE         => 'AIOUTQ',
  DATA_QUEUE_LIBRARY => 'K3SAI'
);
```

### Receiving from a queue in SQL

```sql
SELECT MESSAGE_DATA
  FROM TABLE(QSYS2.RECEIVE_DATA_QUEUE(
    DATA_QUEUE         => 'AIOUTQ',
    DATA_QUEUE_LIBRARY => 'K3SAI',
    WAIT_TIME          => 30,
    REMOVE_MESSAGE     => 'YES'
  ));
```

`WAIT_TIME` is in seconds. `0` = no wait, `-1` = wait forever, positive number = wait that long.

---

## IBM i commands — operations cheat sheet

### Subsystem control

```
STRSBS SBSD(K3SAI/AIWRK)                       /* Start workers */
ENDSBS SBS(AIWRK) OPTION(*CNTRLD) DELAY(60)    /* Stop gracefully */
ENDSBS SBS(AIWRK) OPTION(*IMMED)               /* Stop hard */
WRKACTJOB SBS(AIWRK)                           /* See running workers */
```

### Job inspection

```
WRKSPLF SELECT(K3SAIWRK *ALL *ALL)             /* All worker joblogs */
WRKSBMJOB                                       /* Submitted jobs (RPG side) */
DSPJOBLOG JOB(123456/K3SAIWRK/AIWORKER1)       /* Specific job's log */
```

### Data queue management

```
CRTDTAQ DTAQ(K3SAI/AIOUTQ) TYPE(*STD)          +
        MAXLEN(2000000) SEQ(*FIFO)              +
        CCSID(1208) AUT(*USE)
DLTDTAQ DTAQ(K3SAI/AIOUTQ)
DSPOBJD OBJ(K3SAI/AIOUTQ) OBJTYPE(*DTAQ) DETAIL(*FULL)
WRKDTAQ DTAQ(K3SAI/*ALL)                       /* IBM i 7.4+ */
```

### Library and authority

```
GRTOBJAUT OBJ(K3SAI/AIOUTQ) OBJTYPE(*DTAQ)     +
          USER(K3SAIWRK) AUT(*USE)
RVKOBJAUT OBJ(K3SAI/AIOUTQ) OBJTYPE(*DTAQ)     +
          USER(SOMEUSER) AUT(*ALL)
DSPOBJAUT OBJ(K3SAI/AIOUTQ) OBJTYPE(*DTAQ)
```

---

## PHP environment variables

Read by `Config::loadFromEnv()` in the worker.

| Variable | Purpose | Default |
|:---|:---|:---|
| `K3SAI_INBOUND_LIB` | Library containing inbound queue | `K3SAI` |
| `K3SAI_INBOUND_QUEUE` | Inbound queue name | `AIOUTQ` |
| `K3SAI_LOG_DEST` | Log directory in IFS | stderr |
| `K3SAI_KEK_PATH` | Path to KEK file | `/QIBM/UserData/K3SAI/kek/master.bin` |
| `K3SAI_TIMEOUT_MS` | Per-call timeout | `60000` |
| `K3SAI_MAX_RETRIES` | Retry attempts before giving up | `5` |
| `K3SAI_USAGE_TABLE` | Where to write usage rows | `K3SAI.USAGE_LOG` |
| `K3SAI_RECYCLE_AFTER` | Self-recycle worker after N requests | unset (no recycle) |

---

## Object naming conventions

Reference for K3S libraries. Adjust for your shop.

| Library | Purpose | Examples |
|:---|:---|:---|
| `K3SAI` | Shared admin | `AIOUTQ`, `AI_PROFILE`, `KEY_VAULT`, `USAGE_LOG` |
| `<CUST>_5DTA` | Customer data | `WORK_QUEUE`, `RPLY_*`, `AI_BATCH`, `PO_LINES` |
| `<CUST>_5OBJ` | Customer programs | `AIBATSTRT`, `AIWORKER`, `AIPRE`, `AIPOST` |

| Object name pattern | Convention |
|:---|:---|
| `RPLY_<num>` | Per-worker reply queue, e.g. `RPLY_000001` |
| `AI_<name>` | AI-related shared object, e.g. `AIOUTQ`, `AI_PROFILE` |
| `AIWORKER<num>` | Worker job name, autostart |
| `AIBATSTRT` | Batch initiator program |
| `AIPRE` / `AIPOST` | Customer pre/post-AI logic |

---

## Provider quick reference

### Anthropic

| | |
|:---|:---|
| Endpoint | `https://api.anthropic.com/v1/messages` |
| Auth header | `x-api-key: sk-ant-...` |
| Required headers | `anthropic-version: 2023-06-01` |
| Body shape | `{model, max_tokens, messages, system?}` |
| Response content | `content[0].text` |
| Tokens | `usage.input_tokens`, `usage.output_tokens` |
| Stop reason | `stop_reason` |
| Common error codes | 401 (auth), 429 (rate), 529 (overloaded), 500 (general) |

### OpenAI

| | |
|:---|:---|
| Endpoint | `https://api.openai.com/v1/chat/completions` |
| Auth header | `Authorization: Bearer sk-...` |
| Body shape | `{model, messages: [{role, content}], max_tokens?, temperature?}` |
| Response content | `choices[0].message.content` |
| Tokens | `usage.prompt_tokens`, `usage.completion_tokens` |
| Stop reason | `choices[0].finish_reason` |
| Common error codes | 401 (auth), 429 (rate), 500 (general), 503 (overloaded) |

### Ollama (on-prem)

| | |
|:---|:---|
| Endpoint | `http://your-on-prem-host:11434/api/generate` (or `/api/chat`) |
| Auth | None (network-based trust) |
| Body shape | `{model, prompt, stream: false}` |
| Response content | `response` |
| Tokens | Inconsistent across versions |
| Stop reason | `done_reason` (in newer versions) |

---

## Rate card reference (current as of May 2026)

For cost calculation. Update when providers change pricing. Per million tokens, USD.

| Model | Input | Output |
|:---|---:|---:|
| `claude-sonnet-4-5` | $3.00 | $15.00 |
| `claude-opus-4-7` | $15.00 | $75.00 |
| `claude-haiku-4-5` | $0.80 | $4.00 |
| `gpt-4o` | $5.00 | $15.00 |
| `gpt-4o-mini` | $0.15 | $0.60 |
| `gpt-5` | (TBD) | (TBD) |
| Ollama on-prem | $0.00 | $0.00 |

K3S tracks this in PHP code (`CostCalculator` class). Could move to a DB2 table if updates become an admin operation.

---

## YAJL / DATA-INTO / DATA-GEN reminders

For RPG developers using JSON.

### Building JSON from a data structure

```rpg
data-gen myStruct
         %data(jsonOutput : 'noprefix=field_')
         %gen('YAJL/YAJLDTAGEN');
```

`noprefix=field_` strips the `field_` prefix from RPG names. Use it when your data structure subfields are named like `field_name`, `field_value` and you want JSON keys `name`, `value`.

### Parsing JSON into a data structure

```rpg
data-into myStruct
          %data(jsonInput)
          %parser('YAJL/YAJLINTO');
```

Useful options:
- `'allow_extra=yes'` — don't error on extra JSON fields not in the struct
- `'allow_missing=yes'` — don't error on missing JSON fields
- `'case=any'` — case-insensitive field matching

### Common gotchas

- **Booleans.** RPG `IND` type maps to JSON `true`/`false`. Don't use `CHAR(1)`.
- **Arrays.** Use `DIM(N)` on a sub-DS. The N must be the maximum; YAJL handles fewer items.
- **Optional fields.** Pre-initialize to defaults; YAJL leaves missing fields untouched.
- **CCSID.** YAJL works in UTF-8 by default. Make sure your queue is CCSID 1208.

---

## Common error codes & what they mean

### From the worker

| Code | Cause | RPG response |
|:---|:---|:---|
| `RATE_LIMITED` | Provider 429s exhausted | Mark row failed, retry batch later |
| `PROVIDER_AUTH` | Key invalid | Mark row failed, alert ops |
| `PROVIDER_ERROR` | Provider 5xx exhausted | Mark row failed |
| `TIMEOUT` | Provider didn't respond in time | Mark row failed, may retry |
| `INVALID_REQUEST` | Malformed request | Mark row errored, alert ops |
| `PROFILE_NOT_FOUND` | `profile_ref` invalid | Mark row failed, alert ops |
| `INTERNAL` | Bug in worker | Mark row errored, capture details |

### IBM i SQL state codes worth knowing

| SQLSTATE | Meaning |
|:---|:---|
| `00000` | Success |
| `02000` | No data found (cursor exhausted, queue empty) |
| `42704` | Object not found (typo'd table or library) |
| `42501` | Not authorized |
| `42818` | Type mismatch in expression |
| `54011` | Too many columns in expression |
| `57033` | Deadlock or row lock timeout |

### IBM i error message IDs worth knowing

| MSGID | Meaning |
|:---|:---|
| `CPF1124` | Job started |
| `CPF1240` | Job ended normally |
| `CPF1241` | Job ended abnormally |
| `MCH3601` | Pointer not initialized (often a YAJL/DATA-INTO issue) |
| `MCH0601` | Invalid argument |
| `RNX1107` | DATA-INTO error (parser issue) |

---

## Useful one-liners

### Watch the queue depth in real time (every 2 seconds)

From QSH:

```sh
while true; do
  db2 -tx "SELECT MESSAGES_ON_QUEUE FROM TABLE(QSYS2.DATA_QUEUE_INFO('AIOUTQ', 'K3SAI'))"
  sleep 2
done
```

### Tail the structured log

```sh
tail -f /QIBM/UserData/K3SAI/log/worker-*.log | jq .
```

(`jq` makes JSON one-liners readable.)

### Test that PHP can talk to the AI provider

```sh
cd /www/k3s-ai-worker
php -r 'require "vendor/autoload.php"; 
        $r = (new GuzzleHttp\Client())->get("https://api.anthropic.com"); 
        echo $r->getStatusCode();'
```

(Should print 401 — authentication required, but TLS/network worked.)

### Test a queue from the command line

Send:

```sh
db2 -tx "CALL QSYS2.SEND_DATA_QUEUE(
           MESSAGE_DATA       => 'test message',
           DATA_QUEUE         => 'AIOUTQ',
           DATA_QUEUE_LIBRARY => 'K3SAI'
         )"
```

Receive:

```sh
db2 -tx "SELECT MESSAGE_DATA FROM TABLE(QSYS2.RECEIVE_DATA_QUEUE(
           DATA_QUEUE         => 'AIOUTQ',
           DATA_QUEUE_LIBRARY => 'K3SAI',
           WAIT_TIME          => 1,
           REMOVE_MESSAGE     => 'YES'
         ))"
```

### Find orphaned reply queues

```sql
SELECT OBJECT_LIBRARY, OBJECT_NAME, CHANGE_TIMESTAMP
  FROM TABLE(QSYS2.OBJECT_STATISTICS('*ALLUSR', '*DTAQ'))
 WHERE OBJECT_NAME LIKE 'RPLY_%'
   AND DAYS(CURRENT_DATE) - DAYS(CHANGE_TIMESTAMP) > 1;
```

---

## Conventions used throughout this guide

| Convention | Meaning |
|:---|:---|
| `<placeholder>` | Substitute your value |
| `*` (in IBM i context) | Wildcard or special value |
| `K3SAI/OBJECT` | Qualified IBM i name (library/object) |
| `K3SAI.TABLE` | SQL-style qualified DB2 name |
| `**FREE` | Modern free-format RPG |
| `Status: Draft V1` | Code/content untested in production |

---

## Glossary

| Term | Definition |
|:---|:---|
| **AI profile** | Row in `K3SAI.AI_PROFILE` defining provider, model, key, limits per customer |
| **AIOUTQ** | The shared inbound queue PHP workers consume from |
| **Autostart job (AJE)** | IBM i mechanism for jobs that start when their subsystem starts |
| **Batch initiator** | Short-lived program that populates work queue and submits workers |
| **BYOK** | Bring Your Own Key — customer provides their own provider API key |
| **CCSID** | Character set identifier; 1208 is UTF-8 |
| **Contract** | The V1 message format between RPG and PHP |
| **DEK** | Data encryption key — encrypts an API key |
| **Hosted tier** | K3S provides AI access using K3S's provider account |
| **K3SAI** | The shared K3S admin library |
| **KEK** | Key encryption key — encrypts the DEKs |
| **KEY_VAULT** | Table holding encrypted customer API keys |
| **Long-lived worker** | Process that runs continuously, processing many requests |
| **Multi-tenancy** | Architecture serving many customers from one infrastructure |
| **Profile resolution** | Looking up an AI profile by `profile_ref` at runtime |
| **Reply queue** | Per-worker queue for the AI worker's response back to RPG |
| **SBMJOB** | IBM i command to submit a job to a job queue |
| **Token bucket** | Algorithm for rate limiting with burst capacity |
| **WORK_QUEUE** | Per-customer queue distributing rows to RPG workers |
| **YAJL** | Yet Another JSON Library — Klement's IBM i JSON tool |

---

## External references

### IBM i

- [IBM i SQL Reference](https://www.ibm.com/docs/en/i)
- [QSYS2 services documentation](https://www.ibm.com/docs/en/i/latest?topic=services-system-defined)
- [Data queue APIs](https://www.ibm.com/docs/en/i/latest?topic=apis-data-queue)
- [Subsystem and job management](https://www.ibm.com/docs/en/i/latest?topic=concepts-work-management)

### IBM i community

- [Scott Klement](https://www.scottklement.com) — RPG, YAJL, integration patterns
- [Liam Allan](https://worksofliam.com) — modern RPG tooling, Code for IBM i
- [Seiden Group](https://www.seidengroup.com) — PHP on IBM i, modernization
- [Profound Logic](https://www.profoundlogic.com) — modern UI on IBM i

### AI providers

- [Anthropic API docs](https://docs.anthropic.com)
- [OpenAI API docs](https://platform.openai.com/docs)
- [Ollama](https://ollama.com)

### Companion sites

- [RPG Tutorial](https://rpgtutorial.k3s.com) — modern RPGLE/CL development guide, also from K3S

### PHP libraries used

- [Guzzle](https://docs.guzzlephp.org) — HTTP client with middleware
- [libsodium](https://www.php.net/manual/en/book.sodium.php) — encryption (AES-256-GCM, key handling)

---

## What's deliberately not in this reference

- **A full RPG syntax cheat sheet.** RPG Tutorial covers this.
- **Full IBM i CL command reference.** IBM's docs cover this.
- **A how-to for installing PHP on IBM i.** Seiden Group's docs cover this.
- **Detailed Anthropic/OpenAI API documentation.** The provider docs cover this.

This chapter focuses on what's specific to the K3S architecture — the conventions, contracts, queries, and idioms that are particular to running this system.
