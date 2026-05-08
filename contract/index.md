---
title: The data queue contract
nav_order: 9
has_children: false
---

# The data queue contract
{: .no_toc }

**Status:** Draft V1 — under review

This chapter specifies the API between RPG and PHP. Every other chapter assumes the contract defined here. If you only read three chapters, this is the third.

This is the most important chapter to get right. A wrong decision here propagates into both halves of the system and is painful to walk back. We've tried to be specific about what's settled and what's still open. Sections marked **Open for discussion** are the places we expect feedback, especially from RPG developers reading this.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why this chapter matters

The architecture chapter described two halves of a system separated by a queue. This chapter pins down the exact shape of every message that crosses the boundary. Once it's settled, both sides can be built independently — the RPG side is written to produce and consume the formats specified here; the PHP worker is written against the same specification; neither side needs to read the other's code to get integration right.

The cost of getting it wrong is paid every time the contract is revised. Every change has to ship to both sides, in compatible order, without breaking in-flight messages. Get the first version right and most contract changes are additive. Get the first version wrong and you'll spend a year managing migrations.

---

## The shape of the conversation

The boundary uses a request/reply pattern over data queues:

1. RPG worker assembles a prompt and writes a request message to **AI_OUT_QUEUE**, including the name of its private reply queue.
2. A PHP worker pulls the request off AI_OUT_QUEUE, calls the AI provider, and writes a response message to the reply queue named in the request.
3. The originating RPG worker reads from its reply queue and processes the response.

Three things are worth noting up front:

- **Request and reply are different message formats.** They share a request_id for correlation but otherwise have different fields.
- **Reply queues are per-RPG-worker, not per-batch and not per-customer.** Each long-lived RPG worker creates its own reply queue at startup and tears it down at shutdown. This avoids contention and gives natural isolation.
- **AI_OUT_QUEUE is shared across all customers.** PHP workers don't filter by customer — they take the next request and serve it. The customer context travels in the message itself.

---

## Queue inventory

Three queues exist in the contract. They have different lifetimes, different scopes, and different creation parameters.

### AI_OUT_QUEUE

- **Scope:** Shared across all customers.
- **Library:** Lives in a K3S admin library. Working name: `K3SAI` (open for discussion).
- **Lifetime:** Permanent. Created once at install time, exists for the life of the system.
- **Direction:** RPG worker → PHP worker.
- **Type:** Sequential (FIFO). Not keyed.
- **Producer:** Any RPG worker, in any customer's batch.
- **Consumer:** Any PHP worker.

### Reply queues

- **Scope:** Per RPG worker.
- **Library:** Lives in the customer's library.
- **Lifetime:** Created when an RPG worker starts; deleted when it exits.
- **Naming convention (V1 proposal):** `RPLY_{worker_id}` where `worker_id` is a 6-digit zero-padded number assigned by the batch initiator. So a worker assigned ID 7 in customer ACME has reply queue `ACME_5DTA/RPLY_000007`.
- **Direction:** PHP worker → RPG worker.
- **Type:** Sequential (FIFO).
- **Producer:** PHP worker (whichever one served the request).
- **Consumer:** Exactly one RPG worker — the one that owns the queue.

### WORK_QUEUE

WORK_QUEUE is mentioned in the architecture chapter but is **not part of the RPG/PHP contract.** It's an internal RPG-to-RPG queue used by the batch initiator to feed work to RPG workers. PHP never reads from it. Its format is up to your RPG team and is covered in [The RPG worker pool]({% link rpg-pool/index.md %}).

---

## Queue creation parameters

These are the recommended `CRTDTAQ` parameters for V1. Open for review by your IBM i team.

### AI_OUT_QUEUE

```
CRTDTAQ DTAQ(K3SAI/AIOUTQ)
        TYPE(*STD)
        MAXLEN(2000000)
        SEQ(*FIFO)
        FORCE(*NO)
        AUT(*USE)
        TEXT('K3S AI Worker - inbound request queue')
        CCSID(1208)
```

**Notes:**

- **MAXLEN(2000000)** — 2 MB. Prompts with embedded product context, vendor history, and policy text can get large. 2 MB gives margin without being absurd. If you find yourself needing more, consider the "prompt-by-reference" pattern below.
- **SEQ(*FIFO)** — Strict first-in-first-out. We're not using priority queuing in V1. If you add it later, the field is in the message format already.
- **CCSID(1208)** — UTF-8. JSON is UTF-8 by convention, and AI provider APIs expect UTF-8. RPG jobs running in EBCDIC will need to convert on the way in.
- **FORCE(*NO)** — Don't force writes to disk synchronously. Faster, and we don't need durability guarantees stronger than "queue survives normal operation."

### Reply queues

```
CRTDTAQ DTAQ({customer_lib}/RPLY_{worker_id})
        TYPE(*STD)
        MAXLEN(2000000)
        SEQ(*FIFO)
        FORCE(*NO)
        AUT(*EXCLUDE)
        TEXT('K3S AI Worker - reply queue for worker {worker_id}')
        CCSID(1208)
```

**Notes:**

- **AUT(*EXCLUDE)** with explicit grants to the worker's user profile and the PHP worker's user profile. Reply queues should not be readable by anyone else.
- Reply queues should be **deleted on worker shutdown.** The RPG worker's exit routine should `DLTDTAQ` its own queue. Stale reply queues from crashed workers are cleaned up by a daily job (covered in operations).

---

## Message format philosophy

V1 uses **JSON over UTF-8 data queue messages.** Here's why, and what we considered.

### Why JSON

Three reasons:

1. **Self-describing.** Adding a new field doesn't require recompiling either side. Old consumers ignore unknown fields; new consumers default the missing ones. This is the property that makes contract evolution survivable.
2. **Native to PHP.** Zero friction.
3. **Workable in modern RPG.** YAJL, the SQL-based `JSON_OBJECT`/`JSON_VALUE` functions, and the `DATA-INTO`/`DATA-GEN` built-ins all handle JSON well. Not as cheap as fixed-format binary, but cheap enough.

### What we considered and rejected

- **Fixed-format binary structures (RPG-style data structures laid out in memory).** Faster on the RPG side, but every change is a recompile on both sides, and PHP would need to mirror the RPG layout. The coupling is too tight for a contract that will evolve.
- **XML.** Verbose, slow to parse on both sides, and nobody reaches for it in 2026 unless they're talking to a SOAP service.
- **Protocol Buffers.** Tempting for performance, but adds a code-generation step on both sides and isn't idiomatic on IBM i. Maybe in V3 if we have throughput problems we don't have today.

### CCSID handling

Data queues store bytes. JSON is UTF-8. RPG jobs typically run in EBCDIC (CCSID 37 or similar). Two implications:

- **RPG must convert outgoing strings to UTF-8 before SNDDTAQ.** YAJL's encoder handles this when configured for CCSID 1208 output. The SQL `JSON_OBJECT` function with `RETURNING CLOB CCSID 1208` also works.
- **RPG must convert incoming UTF-8 to job CCSID after RCVDTAQ.** Same tools in reverse.
- **PHP side is UTF-8 native** in PASE. No conversion needed there.

If you're new to CCSID conversion in RPG, this is the most likely place to lose a day. Test with a non-ASCII character (an em-dash, a fancy quote, a vendor name with a Spanish ñ) early. It will reveal CCSID mistakes that ASCII-only tests miss.

---

## AI_OUT_QUEUE message format (RPG → PHP)

Every message on AI_OUT_QUEUE is a single JSON object. The fields are:

### Required fields

| Field | Type | Description |
|:---|:---|:---|
| `version` | string | Contract version. V1 is `"1.0"`. PHP rejects unknown major versions. |
| `request_id` | string (UUID) | Generated by RPG. Used for correlation and logging. Must be unique. |
| `customer` | string (3-10 chars) | K3S customer code (e.g., `"ACME"`, `"DMO"`). |
| `profile_ref` | string | Lookup key into the AI profile table. Determines provider, model, key. |
| `reply_queue` | object | The queue PHP should send the response to. See below. |
| `prompt` | string | The full text to send to the AI. UTF-8. |

### `reply_queue` sub-object

| Field | Type | Description |
|:---|:---|:---|
| `library` | string | Library name. Normally the customer library. |
| `name` | string | Queue name. E.g., `"RPLY_000007"`. |

### Optional fields

| Field | Type | Default | Description |
|:---|:---|:---|:---|
| `model_override` | string | (profile default) | Override the profile's default model for this request. |
| `temperature` | number | (profile default) | LLM sampling temperature, 0.0 to 2.0. |
| `max_tokens` | integer | (profile default) | Maximum response tokens. |
| `system_prompt` | string | (profile default or none) | System prompt. Many providers use this; some don't. |
| `timeout_ms` | integer | 60000 | How long PHP waits for the AI before returning a timeout error. |
| `priority` | integer | 5 | 1 (highest) to 9 (lowest). Reserved for future use; PHP ignores in V1. |
| `metadata` | object | `{}` | Opaque to PHP. Echoed back unchanged in the response. Use this for `batch_id`, `line_id`, anything you need on the way back. |

### Example

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "customer": "ACME",
  "profile_ref": "ACME_DEFAULT",
  "reply_queue": {
    "library": "ACME_5DTA",
    "name": "RPLY_000007"
  },
  "prompt": "You are a purchasing analyst. Review the following PO line...\n\nItem: 12oz canned soup\nVendor minimum: 24\nOn hand: 8\nDemand last 30 days: 50\n\nRespond with one word: OK or FLAG.",
  "max_tokens": 50,
  "timeout_ms": 30000,
  "metadata": {
    "batch_id": "BATCH_20260507_0001",
    "line_id": 4271,
    "po_number": "PO123456"
  }
}
```

### Notes on field choices

- **`request_id` is a UUID.** RPG can generate UUIDs via `SYSTOOLS.GENERATE_UUID` or via the system API `_GENUUID`. UUIDs are required because PHP uses them for log correlation and deduplication.
- **`metadata` is opaque on purpose.** PHP doesn't parse it, doesn't validate it, just echoes it back. This is the field RPG uses to remember what request this was — line ID, batch ID, anything. Putting these in `metadata` instead of as top-level fields means PHP doesn't need to know about K3S concepts.
- **`profile_ref` is a string lookup, not the profile contents.** The profile (provider, model, key) lives in a DB2 table, not in the message. Sending it in the message would put API keys on the queue, which is a leak risk. The reference is a small, safe identifier; PHP looks up the actual profile when it needs it.

---

## Reply queue message format (PHP → RPG)

Every message on a reply queue is a single JSON object. The fields are:

### Always present

| Field | Type | Description |
|:---|:---|:---|
| `version` | string | Matches the request version. `"1.0"` in V1. |
| `request_id` | string | Echoed from the request. |
| `status` | string | One of: `"success"`, `"error"`, `"timeout"`. |
| `metadata` | object | Echoed unchanged from the request's `metadata`. |

### Present when `status == "success"`

| Field | Type | Description |
|:---|:---|:---|
| `response` | string | The AI's response text. |
| `model_used` | string | The actual model that served the request (may differ from requested if profile overrode). |
| `tokens_in` | integer | Input tokens billed. |
| `tokens_out` | integer | Output tokens billed. |
| `latency_ms` | integer | Wall-clock time from PHP receiving the request to PHP getting the response. |
| `finish_reason` | string | Provider-normalized: `"stop"`, `"length"`, `"content_filter"`, `"other"`. |

### Present when `status == "error"` or `"timeout"`

| Field | Type | Description |
|:---|:---|:---|
| `error_code` | string | Stable code for programmatic handling. See below. |
| `error_message` | string | Human-readable description. Safe to log. |
| `provider_status` | integer | If applicable — the HTTP status from the provider (e.g., 429, 500). |
| `attempts` | integer | How many times PHP retried before giving up. |

### Error codes

V1 defines these `error_code` values:

| Code | Meaning |
|:---|:---|
| `INVALID_REQUEST` | The request itself was malformed (missing required field, bad JSON, unknown profile_ref). |
| `PROFILE_NOT_FOUND` | `profile_ref` doesn't resolve to a known profile. |
| `PROFILE_DISABLED` | Profile exists but is marked inactive. |
| `RATE_LIMITED` | Provider returned 429 and retries were exhausted. |
| `PROVIDER_ERROR` | Provider returned 5xx and retries were exhausted. |
| `PROVIDER_AUTH` | Provider returned 401 — bad or revoked API key. |
| `TIMEOUT` | `timeout_ms` exceeded before the provider responded. |
| `INTERNAL` | Something went wrong inside the PHP worker. The most general code; should be rare. |

RPG decides what to do per code. Most shops will handle `RATE_LIMITED` and `PROVIDER_ERROR` as retryable (re-queue the work unit later) and the rest as permanent (mark the row as errored and move on).

### Example: success

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "success",
  "response": "OK",
  "model_used": "claude-sonnet-4-5",
  "tokens_in": 487,
  "tokens_out": 2,
  "latency_ms": 842,
  "finish_reason": "stop",
  "metadata": {
    "batch_id": "BATCH_20260507_0001",
    "line_id": 4271,
    "po_number": "PO123456"
  }
}
```

### Example: rate limited

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "error",
  "error_code": "RATE_LIMITED",
  "error_message": "Provider returned 429 after 5 retries",
  "provider_status": 429,
  "attempts": 5,
  "metadata": {
    "batch_id": "BATCH_20260507_0001",
    "line_id": 4271,
    "po_number": "PO123456"
  }
}
```

### Example: timeout

```json
{
  "version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "timeout",
  "error_code": "TIMEOUT",
  "error_message": "Provider did not respond within 30000ms",
  "attempts": 1,
  "metadata": {
    "batch_id": "BATCH_20260507_0001",
    "line_id": 4271,
    "po_number": "PO123456"
  }
}
```

---

## Correlation and ordering

A few subtle properties that matter:

**One request, one response.** PHP guarantees that for every well-formed request it accepts off AI_OUT_QUEUE, exactly one response is written to the named reply queue — regardless of whether the call succeeded, errored, or timed out. The RPG worker can `RCVDTAQ` from its reply queue with confidence that something will arrive.

**Request_id correlation is the safety net, not the primary mechanism.** Because each RPG worker has its own reply queue and processes one row at a time, the response on the queue is, by construction, the response to the request the worker is waiting for. The request_id is there for log correlation and as a sanity check — RPG should still verify that the response's request_id matches the one it sent, and treat a mismatch as a serious error.

**FIFO is preserved within a reply queue.** Because RPG workers process one row at a time, this is mostly academic — there's only one in-flight request per worker. But if you ever build a worker that pipelines (sends 2-3 requests before waiting for the first response), FIFO ordering is your friend.

**Reply queues are not shared.** Two RPG workers must never share a reply queue. The contract assumes one consumer per reply queue, period.

---

## What's deliberately not in the contract

The contract is small on purpose. These things live elsewhere:

- **Authentication, encryption, key rotation.** Handled by the AI profile lookup, not by message fields. Covered in [AI provider concerns]({% link providers/index.md %}).
- **Rate limit fairness across customers.** Lives in the PHP worker, invisible to the contract. Covered in [AI provider concerns]({% link providers/index.md %}).
- **Logging, monitoring, observability.** Out-of-band of the contract. Covered in [Operating in production]({% link operations/index.md %}).
- **Prompt construction and response parsing.** Owned by RPG. The contract carries the prompt as opaque string and the response as opaque string. What goes inside is not the contract's business.
- **Batch coordination, work distribution, completion detection.** Lives in WORK_QUEUE and the RPG worker pool, not in the AI contract.

---

## Versioning the contract

The `version` field is `"1.0"` for the contract described in this chapter. It will change.

### Compatible (minor) changes — version becomes `"1.1"`, `"1.2"`, etc.

These don't break old senders or old receivers:

- Adding new optional fields with defaults.
- Adding new error codes (old code on the receive side should treat unknown codes as `INTERNAL` until updated).
- Adding new `finish_reason` values (same fallback rule).
- Loosening validation (e.g., accepting longer strings).

PHP and RPG can be upgraded independently for minor versions. Old PHP can serve new RPG (ignoring fields it doesn't know); new PHP can serve old RPG (defaulting fields RPG doesn't send).

### Incompatible (major) changes — version becomes `"2.0"`

These do break compatibility:

- Removing fields.
- Renaming fields.
- Changing field types (string to integer, etc.).
- Changing the meaning of an existing value.

For major changes, the migration plan is:

1. Deploy new PHP that accepts both `"1.x"` and `"2.0"` requests, writes responses matching the request version.
2. Migrate RPG to send `"2.0"`.
3. After all RPG is migrated, deploy newer PHP that drops `"1.x"` support.

Don't undertake major version changes lightly. Most evolution should be minor.

---

## Open for discussion

These are the V1 decisions we're least confident about. Worth working through with your team.

### Library naming

We've used `K3SAI` as the working name for the K3S admin library. Other candidates: `K3SCMN` (common), `K3SOPS` (operations), `K3SSYS` (system). What naming convention does your shop already use for cross-customer admin libraries?

### Reply queue naming

`RPLY_{worker_id}` is fine, but worker_id needs to be unique across all batches running concurrently. Options:

- **Globally unique worker IDs** assigned by a counter in a shared K3S admin table. Simple but introduces a hot row.
- **Per-batch worker IDs** plus the batch_id in the queue name (e.g., `RPLY_BATCH001_007`). More cumbersome.
- **Job-name-based** (e.g., `RPLY_{job_name}_{job_number}`). Naturally unique but uglier.

Worth deciding: which option fits K3S conventions best.

### Maximum message size

2 MB feels generous, but we haven't measured what real K3S prompts look like with full product context embedded. If actual prompts are in the 50KB range, 2 MB is overkill. If they're approaching 1 MB, we should bump it.

If a prompt would exceed the queue's max length, the alternative is "prompt-by-reference": store the prompt in a DB2 CLOB table keyed by request_id, send only the reference on the queue. Adds a DB2 lookup but removes the size ceiling. Worth implementing if we ever need it; not needed in V1.

### Timeout default

`timeout_ms: 60000` (60 seconds) is the V1 default. Cloud AI providers usually respond in under 10 seconds; on-premises models can be slower. This is the timeout *PHP enforces locally* — it doesn't change provider-side behavior. If the provider takes 90 seconds, PHP times out at 60 and returns; the provider's response (if it eventually arrives) is discarded.

Is 60 seconds the right default? Should it be 30? 120? Worth discussing.

### Error code granularity

The eight error codes above are a starting point. We may want to split `PROVIDER_ERROR` into transient vs. permanent (a 503 vs. a 400 are very different). Or add `CONTENT_FILTERED` for cases where the provider refuses to respond. Adding codes is a minor version change, so we don't need to settle this in V1.

### Should `system_prompt` be in the request or the profile?

Argument for request: same prompt template can vary system behavior per-call.

Argument for profile: keeps RPG simpler; the profile knows the role.

V1 puts it in the request as optional (with profile fallback). Open to flipping that.

---

## What V1 gives you

If both halves implement what's specified above:

- An RPG developer can write a worker against the V1 contract without seeing the PHP.
- A PHP developer can write a worker against the V1 contract without seeing the RPG.
- Either side can be tested in isolation by stubbing the other (a tiny CL program that puts canned messages on the queue; a tiny PHP script that reads from the queue and writes a fake response).
- Either side can be replaced or rewritten without touching the other, as long as the contract is preserved.

That's the test of whether the contract is right. If it doesn't enable independent development, it's not a contract — it's a coupling pretending to be one.

---

[Next: The PHP worker]({% link php-worker/index.md %}) →
