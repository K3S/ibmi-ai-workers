---
title: The data queue contract
nav_order: 9
has_children: false
---

# The data queue contract
{: .no_toc }

**Status:** Draft V2

The architecture chapter described two halves of a system separated by a queue. This chapter pins down the exact shape of every message that crosses the boundary. Once it's settled, both sides can be built independently — the RPG side is written to produce and consume the formats specified here; the PHP worker is written against the same specification; neither side needs to read the other's code to get integration right.

The contract isn't fancy. It's just JSON, sent over IBM i data queues, with carefully chosen fields. But it's the most important chapter in this guide to get right, because changes to it later are expensive.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why a contract

Without a contract, RPG and PHP each end up encoding assumptions about each other in their code. The PHP worker assumes "RPG sends a JSON blob with a `prompt` field." RPG assumes "PHP sends back a JSON blob with `status` and `response` fields." When those assumptions diverge — and they will — debugging becomes archaeology.

A contract pulls all of those assumptions out into one place. Both sides reference it. Changes are deliberate, versioned, communicated. The contract is the API; everything else is implementation.

The contract's job is to:

- Specify the structure of every message
- Specify required vs optional fields
- Specify how errors are communicated
- Specify how the contract itself can evolve over time

This chapter does all four.

---

## The transport: data queues, UTF-8

The contract uses two IBM i data queues per round trip:

- `K3SAI/AIOUTQ` — the shared inbound queue. RPG writes requests; PHP reads them. One queue, all customers, FIFO.
- A per-worker reply queue named in each request. PHP writes replies to the queue address the request specified.

Both directions use **UTF-8**. This is a hard requirement, not a suggestion.

### Why UTF-8 is non-negotiable

JSON requires a Unicode encoding. The standard JSON spec assumes UTF-8. PHP strings are bytes; when PHP runs `json_encode()`, the output is UTF-8 bytes. Every AI provider's API expects UTF-8 in its request body. Every modern HTTP API operates in UTF-8.

But IBM i jobs typically run in CCSID 37 (EBCDIC). If you put text on a data queue from an EBCDIC job and read it from a UTF-8 process, the bytes get mangled in conversion. Garbage characters. JSON parsing fails on any non-ASCII content.

The fix is to use the **UTF-8-specific variants** of the SQL data queue procedures:

- **Sending:** `QSYS2.SEND_DATA_QUEUE_UTF8` (not `SEND_DATA_QUEUE`)
- **Receiving:** select `MESSAGE_DATA_UTF8` column from `QSYS2.RECEIVE_DATA_QUEUE` (not `MESSAGE_DATA`)

These variants tell IBM i: "the bytes I'm giving you are UTF-8, store them as such" and "give me back UTF-8 bytes regardless of my job's CCSID." Combined, they make the whole round trip CCSID-agnostic.

In RPG, declare any variable that holds queue payload data with `ccsid(*utf8)`:

```rpg
dcl-s requestJson varchar(4000) ccsid(*utf8);
dcl-s responseJson varchar(4000) ccsid(*utf8);
```

This makes RPG handle the encoding conversion when binding to SQL.

In PHP, no special handling is needed — PHP strings are already UTF-8 bytes by convention.

### The rule

**Whenever data crosses the RPG↔PHP boundary via a queue, use the UTF-8 variants. No exceptions.**

Every code sample in this chapter and every quickstart follows this rule.

---

## Creating a data queue

A data queue is created with `CRTDTAQ`. The relevant parameters for our contract:

```
CRTDTAQ DTAQ(K3SAI/AIOUTQ) +
        TYPE(*STD) +
        MAXLEN(64512) +
        SEQ(*FIFO) +
        FORCE(*NO) +
        AUT(*USE) +
        TEXT('K3S AI Worker - inbound request queue')
```

Notes:

**`MAXLEN(64512)` is the maximum allowed for a standard data queue.** Plenty for V1 contract messages (typically 500-1500 bytes for a demo, 5-30 KB for production purchasing prompts). If your real prompts ever approach this limit, switch to a "prompt-by-reference" pattern (described below) rather than expanding to a different queue type.

**No `CCSID` parameter on `CRTDTAQ`.** The queue itself doesn't have a CCSID — it stores raw bytes. CCSID handling happens at the send/receive boundary via the `_UTF8` procedure variants.

**`TYPE(*STD)`.** Standard data queue. Other types (keyed, distributed) are interesting in advanced architectures but not in V1.

**`SEQ(*FIFO)`.** First in, first out. Important for fair ordering across customers.

**`AUT(*USE)`.** Default authority is "use" (read and write). Production may want tighter, with explicit authority grants per user profile.

The same parameters apply to per-worker reply queues. They get created and destroyed dynamically by RPG workers as they start and stop.

---

## Verifying the SQL services exist

Before going further, confirm your IBM i has the SQL data queue services. From any SQL session:

```sql
SELECT ROUTINE_NAME 
  FROM QSYS2.SYSROUTINES
 WHERE ROUTINE_SCHEMA = 'QSYS2'
   AND ROUTINE_NAME LIKE '%DATA_QUEUE%'
 ORDER BY ROUTINE_NAME;
```

You should see at least:

- `RECEIVE_DATA_QUEUE`
- `SEND_DATA_QUEUE`
- `SEND_DATA_QUEUE_UTF8`
- `SEND_DATA_QUEUE_BINARY`
- `CLEAR_DATA_QUEUE`
- `DATA_QUEUE_INFO`

If `SEND_DATA_QUEUE_UTF8` is missing, your DB2 PTF Group is too old for the contract as written. You'll need to either upgrade DB2 services, or use a different approach (calling the data queue APIs directly via SRVPGM rather than SQL services).

---

## The request format

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
| `request_id` | string | yes | UUID, unique per request. Used for log correlation |
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

### Field-by-field rationale

**`version`.** First field, fixed format. Future contract revisions can read this to know how to parse the rest. Cheap insurance against the inevitable need to change the format.

**`request_id`.** Required. Used for log correlation across RPG, PHP, and any external services. Must be unique per request. UUIDs are convenient; sequential numbers from a database also work.

**`customer`.** Identifies which customer this request is for. PHP uses this to scope profile lookups, rate limit buckets, and usage logging.

**`profile_ref`.** Points at a row in `K3SAI.AI_PROFILE`. PHP resolves the actual provider, model, and key from this. RPG doesn't need to know what the resolution is.

**`reply_queue`.** The address to send the response to. This is what makes the "request-reply" pattern work over a fire-and-forget queue.

**`prompt`.** The text that goes into the AI's user-message slot. Built by the customer's `AIPRE` logic.

**`system_prompt`.** Optional. Some use cases need a system-level instruction; others don't. Profile defaults usually cover this.

**`max_tokens`, `temperature`, `model_override`, `timeout_ms`.** Optional overrides for the profile's defaults. Most calls don't override anything.

**`metadata`.** Free-form. RPG puts whatever it needs (row ID, batch ID, etc.) and PHP echoes it back unchanged. The architectural rule: PHP never inspects `metadata`. It's RPG's space.

---

## The reply format — success

JSON sent from PHP to the named reply queue.

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

| Field | Type | Notes |
|:---|:---|:---|
| `version` | string | `"1.0"` |
| `request_id` | string | Echo of request's `request_id` |
| `status` | string | `"success"` |
| `response` | string | The AI's text response, content of `content[0].text` from Anthropic |
| `model_used` | string | Actual model that processed the request |
| `tokens_in` | integer | Input tokens consumed |
| `tokens_out` | integer | Output tokens produced |
| `latency_ms` | integer | Wall-clock time spent in the AI call |
| `finish_reason` | string | Why the AI stopped — `end_turn`, `max_tokens`, etc. |
| `metadata` | object | Echo of request's `metadata` |

### Why "response" is a single string, not the full provider envelope

We deliberately don't pass through Anthropic's full response envelope. The `content` array, the `role` field, the various message-shape complexities — none of that crosses to RPG. What RPG gets is the AI's text, normalized.

This is a design choice. Pro: RPG doesn't have to track provider-specific response shapes. Con: if the provider ever returns multi-part content (text + images + function calls), this format can't represent that. For the V1 contract, the simplification is worth it. We'll evolve when we need to.

---

## The reply format — error

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

`attempts` tells RPG how many tries the worker made before giving up. Useful for batch-level observability.

---

## How PHP reads requests

The reference pattern, in PHP:

```php
$sql = "SELECT MESSAGE_DATA_UTF8 
          FROM TABLE(QSYS2.RECEIVE_DATA_QUEUE(
            DATA_QUEUE         => ?,
            DATA_QUEUE_LIBRARY => ?,
            REMOVE             => 'YES',
            WAIT_TIME          => 30
          ))";

$stmt = db2_prepare($conn, $sql);
db2_execute($stmt, [$queueName, $queueLibrary]);
$row = db2_fetch_assoc($stmt);

if ($row && !empty($row['MESSAGE_DATA_UTF8'])) {
    $request = json_decode($row['MESSAGE_DATA_UTF8'], true, 512, JSON_THROW_ON_ERROR);
}
```

Three things to notice:

**`SELECT MESSAGE_DATA_UTF8`** — not `MESSAGE_DATA`. The UTF-8 column.

**Parameter `REMOVE`** — not `REMOVE_MESSAGE`. The actual parameter name expected by the function.

**`WAIT_TIME => 30`** — PHP blocks for up to 30 seconds. If a message arrives, the call returns immediately with the message. If 30 seconds pass with no message, the call returns empty and PHP loops to call again.

During the wait, PHP uses zero CPU. The OS handles the suspend and wakeup at the kernel level.

---

## How PHP sends replies

```php
$sql = "CALL QSYS2.SEND_DATA_QUEUE_UTF8(
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
```

**`SEND_DATA_QUEUE_UTF8`** — the UTF-8 variant. Stores the bytes as UTF-8 so RPG can read them back cleanly.

Parameter order: `MESSAGE_DATA` first, then `DATA_QUEUE`, then `DATA_QUEUE_LIBRARY`. (Different from the receive procedure's parameter order — using named arguments avoids ambiguity.)

---

## How RPG sends requests

```rpg
dcl-s requestJson varchar(4000) ccsid(*utf8);

// ... build request data structure ...

data-gen request 
         %data(requestJson : 'noprefix=request_') 
         %gen('YAJL/YAJLDTAGEN');

exec sql 
  call qsys2.send_data_queue_utf8(
    MESSAGE_DATA       => :requestJson,
    DATA_QUEUE         => 'AIOUTQ',
    DATA_QUEUE_LIBRARY => 'K3SAI'
  );
```

The `ccsid(*utf8)` declaration on the JSON variable is what makes RPG handle the conversion correctly when binding `:requestJson` to the SQL parameter.

---

## How RPG receives replies

```rpg
dcl-s responseJson varchar(4000) ccsid(*utf8);

exec sql 
  select MESSAGE_DATA_UTF8 
    into :responseJson
    from table(qsys2.receive_data_queue(
      DATA_QUEUE         => :replyQueue,
      DATA_QUEUE_LIBRARY => :replyLib,
      REMOVE             => 'YES',
      WAIT_TIME          => 60
    ));
```

Same pattern as PHP receive: select `MESSAGE_DATA_UTF8`, parameter `REMOVE`, named arguments.

---

## Reply queue naming

Conventions matter, because reply queue names cross the contract.

The pattern: `RPLY_{worker_id_zero_padded}` in the customer's library.

Examples: `RPLY_000001`, `RPLY_000023`, `RPLY_000150`.

Why zero-padding to six digits:

- Allows up to 999,999 worker IDs without restructuring
- Sortable lexically when listing queues
- Predictable length for parsing

The library is the customer's `*_5DTA`. Each worker creates its queue at startup, and deletes it at shutdown. If a worker crashes, an orphaned queue may remain — there's a daily cleanup job that finds and removes them.

Worth deciding for a real shop: which option fits K3S conventions best — RPLY_{id}, RPLY_{job_name}, or some other pattern. The contract works with any of them as long as both sides agree.

---

## Maximum message size

V1 limit: 64 KB (the `MAXLEN(64512)` on the queue).

For the demo, every message is 500-1500 bytes. For production purchasing prompts, expect 5-30 KB depending on how much vendor history and policy text is included. Still well under the limit.

If a prompt approaches the limit, switch to **prompt-by-reference**:

1. Store the actual prompt in a DB2 CLOB table keyed by request_id.
2. Send `{"request_id": "...", "prompt_ref": "DEMOLIB.PROMPTS.123"}` on the queue.
3. PHP fetches the prompt from DB2 using the reference, then makes the AI call.
4. PHP sends the response back through the queue normally.

Adds a DB lookup per call but removes the size ceiling. Worth implementing if and when prompts approach 64 KB. Not in V1.

---

## Error handling principles

The contract guarantees a reply for every accepted request. Even if everything goes wrong, RPG gets *some* reply with a status code it can act on. This is what lets RPG rely on its `RECEIVE_DATA_QUEUE` call without timeouts and retry logic.

Three cases produce error replies:

1. **Provider failure that retries didn't fix.** Status `error`, error_code per the table above. Worker logs the failure but doesn't try further.
2. **Configuration failure.** Bad profile, bad key, missing data. Worker recognizes these don't benefit from retry, returns immediately with `PROFILE_NOT_FOUND` or `PROVIDER_AUTH`.
3. **Internal worker bug.** Something the worker didn't anticipate. Status `error`, error_code `INTERNAL`. RPG should log these and alert ops.

What the worker never does: silent failure. Every request gets a reply. RPG never has to wonder whether its request got lost.

---

## Versioning the contract

The `version` field is `"1.0"` today. Future versions of this contract will increment. The rules:

- **Major version bump (`2.0`)** for breaking changes — fields removed, semantics changed, error codes redefined.
- **Minor version bump (`1.1`)** for additive changes — new optional fields, new error codes.
- **Both sides must coordinate on major version changes.** Old PHP can't speak `2.0`; old RPG can't read `2.0` replies.
- **Minor changes are backward-compatible.** Old code ignores fields it doesn't recognize.

The `version` field at the top of every message lets each side check compatibility. PHP can refuse to process a `2.0` request if it's still on `1.0`. RPG can detect a `2.0` reply and warn ops.

This is light governance. Production deserves more (a contract version registry, automated compatibility checks). V1 doesn't need it.

---

## What's deliberately not in this V2 contract

- **Streaming responses.** Each reply is a single message, not a stream. Future versions may support streaming via multiple queue messages with sequence numbers.
- **Multi-part content (text + images + function calls).** Reply's `response` is a single string. Multi-modal isn't supported.
- **Compression.** No compressing the JSON payload. Could be added if message size becomes a concern.
- **Encryption inside the message.** The queue is on the same IBM i; we trust the local environment. If cross-system queues become a thing, encryption joins the contract.
- **Authentication of sender.** Any program with `*USE` on the queue can send. Production multi-tenancy depends on library-list isolation, not message authentication.

Each is a real direction for the contract to grow. None is needed for V1.

---

## Open for discussion

V1 calls in this contract that are worth revisiting once we have real-shop experience:

- **Reply queue naming.** Pattern is `RPLY_{worker_id}` zero-padded; zero-padding length and exact format are open.
- **Maximum message size.** 64 KB is generous for demos but tight for production prompts. May want to standardize prompt-by-reference earlier.
- **The `metadata` field's "free-form" promise.** Free-form is convenient but means PHP can never understand what's in there. May want to formalize standard sub-fields (request source, debug tag, etc.).
- **Whether to include a `priority` field.** For systems that want some requests to jump the queue. Adds complexity to PHP's pull logic.
- **`error_code` enum extensibility.** New providers may have new error categories. Need a way to extend without breaking deserialization on the RPG side.

---

[Next: The PHP worker]({% link php-worker/index.md %}) →
