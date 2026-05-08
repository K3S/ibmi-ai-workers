---
title: AI provider concerns
nav_order: 13
has_children: false
---

# AI provider concerns
{: .no_toc }

**Status:** Draft V1

This is the chapter where the AI part stops being abstract. It covers how the worker actually talks to providers, how customers' API keys get stored without leaking, how K3S can offer both BYOK and hosted tiers without writing two different workers, how rate limits get enforced fairly across customers, and what the per-call cost story looks like.

This is also the chapter with the most security-relevant decisions in the entire guide. K3S is taking on real responsibility the moment we accept a customer's API key. The patterns described here are designed to make that responsibility manageable, not to eliminate it — there's no eliminating it.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The provider abstraction

The provider abstraction is the seam that lets the worker be provider-agnostic. The PHP worker holds a `ProviderInterface` and calls `send()`. The concrete implementation behind that interface is chosen at runtime based on the AI profile.

```php
namespace K3S\AiWorker;

interface ProviderInterface
{
    public function send(array $request, Profile $profile): ProviderResponse;
    public function healthCheck(Profile $profile): bool;
    public function estimateCost(array $request, Profile $profile): ?CostEstimate;
}
```

Three implementations ship with V1: `AnthropicProvider`, `OpenAiProvider`, `OllamaProvider`. A fourth (Google, Mistral, custom on-prem) is a new class implementing the same interface.

Each provider does four things:

1. **Translates the V1 contract request into the provider's wire format.** Anthropic's `/v1/messages` body shape ≠ OpenAI's `/v1/chat/completions` body shape ≠ Ollama's `/api/generate` body shape.
2. **Makes the HTTP call** with the provider's auth headers and timeout.
3. **Parses the provider's response into a normalized shape.** Tokens in, tokens out, content text, finish reason — all common across providers in concept, all named differently in protocol.
4. **Translates provider-specific errors into the worker's domain exceptions.** A 429 from any provider becomes `ProviderRateLimitException`; the worker doesn't know which provider it came from.

This is what makes the rest of the system simple: the main worker loop, the retry middleware, the rate limiter, the usage logger — none of them know which provider is in use. They see normalized inputs and outputs.

### Why the abstraction earns its place in V1

The principle of YAGNI ("you aren't gonna need it") would normally argue against building an abstraction before you have multiple implementations. The reason we do it anyway:

- We already need three providers to support BYOK + hosted + on-prem. Multiple implementations exist on day one.
- The abstraction is small. The interface is three methods; the cost of building it is one afternoon.
- Without the abstraction, every retry/log/middleware decision has to be made in each provider's code separately. Drift is inevitable.

So: build it.

---

## BYOK and hosted tiers

K3S offers two ways to use AI:

**Bring Your Own Key (BYOK).** The customer has their own Anthropic, OpenAI, or other provider account. They give K3S their API key. K3S calls the provider on their behalf using their key. The customer is billed by the provider directly. K3S charges them for the K3S service, not for AI.

**Hosted.** The customer pays K3S a monthly fee. K3S calls Anthropic (or whichever provider K3S chooses) using K3S's own API key. K3S absorbs the AI cost into its pricing. The customer never sees an API bill.

Both tiers run through the same worker, the same provider abstraction, the same contract. The difference is in the AI profile: a `MODE = 'B'` profile points at the customer's key; a `MODE = 'H'` profile points at K3S's key, with a quota.

What changes between them:

| Concern | BYOK | Hosted |
|:---|:---|:---|
| Who pays the provider | Customer | K3S |
| Who K3S charges | Service fee only | Service fee + AI usage |
| Rate limit budget | Customer's account limit | K3S's K3S-allocated portion |
| Quota enforcement | None (customer's account, customer's problem) | K3S enforces monthly cap |
| Failure if key revoked | Customer's account is gone, profile suspended | N/A |
| K3S's responsibility | Custody of customer's key | Custody of K3S's key + per-customer accounting |

The core pattern is the same; the operational surface is different. Both deserve real attention.

---

## Key custody

This is the most security-sensitive part of the entire system. When a customer hands K3S their Anthropic API key, K3S has accepted custody of a credential that grants spending power on their account. The patterns in this section exist to make that custody safe.

### What "safe" means concretely

Three properties to enforce:

1. **The plaintext key is never in DB2.** Encrypted at rest, always.
2. **The plaintext key is never in logs.** Stripped before any log statement that touches HTTP.
3. **The plaintext key lives in process memory only as long as the call needs it.** Loaded just before the HTTP request, zeroed after.

If any of these three fails, the architecture is broken. The custody design exists to make all three the easy path.

### Envelope encryption

Two-layer encryption: a per-customer **data encryption key (DEK)** encrypts the API key; a **key encryption key (KEK)** encrypts the DEK.

```
Customer API key (plaintext)
        │
        │ AES-256-GCM with DEK
        ▼
Encrypted API key   ──┐
                       │ stored in DB2 K3SAI.KEY_VAULT
DEK                   │
        │              │
        │ AES-256-GCM with KEK
        ▼              ▼
Encrypted DEK   ──────┘

KEK lives outside DB2: in *VLDL or IFS file with tight authority
```

Why two layers? Because rotating the KEK doesn't require re-encrypting every API key. You decrypt each DEK with the old KEK, re-encrypt with the new KEK, and the API keys themselves never get touched. This is the standard envelope encryption pattern; it's worth using even though it adds complexity.

### The KEY_VAULT table

```sql
CREATE TABLE K3SAI.KEY_VAULT (
    KEY_REF             VARCHAR(60)   NOT NULL,
    CUSTOMER            VARCHAR(10)   NOT NULL,
    PROVIDER            VARCHAR(20)   NOT NULL,
    KEY_VERSION         INTEGER       NOT NULL,
    ENCRYPTED_DEK       VARBINARY(512) NOT NULL,    -- DEK encrypted with KEK
    ENCRYPTED_KEY       VARBINARY(2048) NOT NULL,   -- API key encrypted with DEK
    NONCE_DEK           VARBINARY(32) NOT NULL,
    NONCE_KEY           VARBINARY(32) NOT NULL,
    KEK_VERSION         INTEGER       NOT NULL,
    STATUS              VARCHAR(20)   NOT NULL,    -- ACTIVE, ROTATED, REVOKED
    CREATED_AT          TIMESTAMP     NOT NULL,
    LAST_USED_AT        TIMESTAMP,
    REVOKED_AT          TIMESTAMP,
    PRIMARY KEY (KEY_REF)
);

CREATE INDEX K3SAI.KEY_VAULT_CUST_STAT 
       ON K3SAI.KEY_VAULT (CUSTOMER, STATUS);
```

A few notes:

The `KEY_REF` is what `AI_PROFILE.KEY_REF` points at. Stable for the life of the key version. New version of the key for the same customer = new `KEY_REF`.

`KEY_VERSION` lets multiple versions coexist during a rotation. Old version stays `ACTIVE` for a grace period; new version is the one new requests use; old becomes `ROTATED` after the grace period; gets `REVOKED` eventually.

`KEK_VERSION` records which KEK encrypted this row's DEK. KEK rotation re-encrypts each DEK with the new KEK and updates this column.

Authority on this table is tight. Read access is granted only to the user profile under which the PHP worker runs. Write access is granted only to the user profile under which the admin tooling runs. No interactive user has either.

### KEK storage

The KEK has to live somewhere the worker can read it but humans can't easily extract it. Options on IBM i:

1. **Validation list (`*VLDL`).** Built-in IBM i object designed for storing credentials. Authority granular. Survives IPL. Probably the best fit.
2. **IFS file with tight authority.** A binary file at, say, `/QIBM/UserData/K3SAI/kek/master.bin`, owned by the worker user profile, mode 600. Simple. Easy to back up (carefully).
3. **System value.** Tempting but too persistent and too visible.
4. **HSM.** Real cryptographic hardware. Right answer for high-stakes deployments; overkill for V1.

V1 picks option 2 (IFS file). Reason: simpler to set up than `*VLDL`, well-understood permission model, easy to rotate (write a new file, atomically rename). Switching to `*VLDL` later is straightforward.

The KEK file is loaded once at worker startup, held in worker memory (zeroed on shutdown), and never written to log.

### The KeyVault class

In PHP, the encryption/decryption logic is encapsulated in one class:

```php
namespace K3S\AiWorker;

class KeyVault
{
    private string $kek;

    public function __construct(string $kekFilePath)
    {
        if (!is_readable($kekFilePath)) {
            throw new RuntimeException("KEK file not readable: {$kekFilePath}");
        }
        $this->kek = file_get_contents($kekFilePath);
        if (strlen($this->kek) !== 32) {
            throw new RuntimeException('KEK must be exactly 32 bytes');
        }
    }

    public function resolve(string $keyRef): SecretValue
    {
        $row = $this->fetchKeyRow($keyRef);
        if ($row === null) {
            throw new KeyNotFoundException($keyRef);
        }
        if ($row['STATUS'] !== 'ACTIVE') {
            throw new KeyInactiveException($keyRef, $row['STATUS']);
        }

        $dek = $this->decryptDek($row['ENCRYPTED_DEK'], $row['NONCE_DEK']);

        try {
            $apiKey = $this->decryptApiKey(
                $row['ENCRYPTED_KEY'],
                $row['NONCE_KEY'],
                $dek,
            );
            return new SecretValue($apiKey);
        } finally {
            sodium_memzero($dek);
        }
    }

    private function decryptDek(string $ciphertext, string $nonce): string
    {
        $plaintext = sodium_crypto_aead_aes256gcm_decrypt(
            $ciphertext,
            '',
            $nonce,
            $this->kek,
        );
        if ($plaintext === false) {
            throw new DecryptionException('DEK decryption failed');
        }
        return $plaintext;
    }

    // ... decryptApiKey similar
}

class SecretValue
{
    private string $value;

    public function __construct(string $value)
    {
        $this->value = $value;
    }

    public function reveal(): string
    {
        return $this->value;
    }

    public function __destruct()
    {
        if (function_exists('sodium_memzero')) {
            sodium_memzero($this->value);
        }
    }

    public function __toString(): string
    {
        return '[REDACTED SecretValue]';
    }

    public function __debugInfo(): array
    {
        return ['value' => '[REDACTED]'];
    }
}
```

The `SecretValue` wrapper is tiny but does important work: it ensures the key gets zeroed when the wrapper is destroyed (between requests), and it makes accidentally logging the wrapper produce `[REDACTED SecretValue]` instead of the actual key. Defense in depth — the redaction middleware handles HTTP headers; this wrapper handles application code.

### Storing a new key

When a customer onboards with BYOK:

1. K3S admin tool prompts the customer to paste their API key (over TLS).
2. The admin tool generates a new DEK (32 random bytes from `random_bytes(32)`).
3. Encrypts the API key with the DEK (AES-256-GCM, fresh nonce).
4. Encrypts the DEK with the KEK (AES-256-GCM, fresh nonce).
5. Inserts a row into `KEY_VAULT` with `STATUS = 'ACTIVE'`, `KEY_VERSION = 1`.
6. Updates the customer's `AI_PROFILE` to point at the new `KEY_REF`.

The plaintext key is in memory in the admin tool for the duration of step 1-3, then zeroed. It's never in DB2. It's never in logs.

### Rotating a key

Customer rotates their Anthropic key:

1. Customer creates a new key in their Anthropic console.
2. Customer pastes new key into K3S admin tool.
3. New `KEY_VAULT` row is inserted with `STATUS = 'ACTIVE'`, `KEY_VERSION = 2`. Old `STATUS` flipped to `'ROTATED'`.
4. `AI_PROFILE.KEY_REF` updated to point at the new row.
5. PHP worker's profile cache is cleared (signal handler).
6. New requests use the new key. Anthropic still accepts the old key for the grace period (typically 30 days).
7. After grace period, customer revokes the old key in Anthropic. K3S marks the old `KEY_VAULT` row `'REVOKED'`.

Zero downtime, zero leaked keys, full audit trail.

### Rotating the KEK

Less common but should still be possible:

1. Generate new KEK file. Save to disk under a different name first.
2. For each `ACTIVE` and `ROTATED` row in `KEY_VAULT`: decrypt DEK with old KEK, re-encrypt with new KEK, update row, increment `KEK_VERSION`.
3. Atomically rename the new KEK file to the canonical path.
4. Restart PHP workers (so they re-load the new KEK).
5. Move the old KEK file to a backup location (in case of recovery need) or destroy.

This needs to happen offline or at least during a maintenance window for safety.

---

## Rate limit fairness

When customers share an account (the hosted tier), they share that account's rate limit. Without explicit fairness logic, one customer's batch can starve another's.

### The fairness problem, made concrete

Suppose K3S's Anthropic account has a 4000 RPM limit. ACME starts a 50,000-row batch at noon. Without fairness, ACME's workers fire 4000 requests/minute, hitting the rate limit. BARCO starts a small batch at 12:01. Their requests get 429s because ACME is consuming the entire budget.

Even worse: ACME doesn't notice they're being rate-limited because retry middleware backs them off; they just see slightly slower throughput. BARCO sees nothing but failures.

This is unacceptable. Fairness is not a "nice to have."

### The token bucket

The standard fix: each hosted customer gets a per-minute token bucket sized to their proportion of the global budget. ACME's bucket holds 800 RPM, BARCO's holds 600 RPM, etc., summing to ≤ K3S's account limit.

Token bucket pseudocode:

```
For each hosted customer:
  tokens_remaining: integer, refilled at customer's rate per minute
  
On request arrival for customer X:
  if X.tokens_remaining > 0:
    X.tokens_remaining -= 1
    proceed with request
  else:
    return RATE_LIMITED to RPG; do not call provider
```

The bucket refills continuously: the rate is per-minute, but tokens get added incrementally so customers can do bursts up to their bucket size and then sustain at their rate.

### Where the bucket lives

The bucket has to be:

- Shared across all PHP worker processes (otherwise each worker has its own bucket and the sum exceeds the budget)
- Fast to read and update (every request hits it)
- Resilient to worker restarts (a restart shouldn't reset the bucket and double-bill the customer)

Three options:

1. **DB2 row with locking.** Simple, works, slow. Each request reads a row, decrements an integer, writes. Row locking serializes across workers. Probably 10-50ms overhead per request. Probably acceptable.
2. **Data area on IBM i.** Faster than DB2. Atomic operations less convenient.
3. **External store like Redis.** Fastest. New dependency.

V1 picks option 1. DB2 is already there, lock contention isn't bad at the scale we're talking about (single-digit milliseconds), and the alternative dependencies aren't justified yet.

```sql
CREATE TABLE K3SAI.RATE_LIMIT_BUCKET (
    CUSTOMER          VARCHAR(10)   NOT NULL,
    BUCKET_TYPE       VARCHAR(10)   NOT NULL,    -- 'RPM' or 'TPM'
    TOKENS_REMAINING  INTEGER       NOT NULL,
    LAST_REFILL_AT    TIMESTAMP     NOT NULL,
    REFILL_RATE       INTEGER       NOT NULL,    -- tokens per minute
    BUCKET_SIZE       INTEGER       NOT NULL,    -- max tokens
    PRIMARY KEY (CUSTOMER, BUCKET_TYPE)
);
```

The check-and-decrement is one transaction:

```sql
UPDATE K3SAI.RATE_LIMIT_BUCKET
   SET TOKENS_REMAINING = LEAST(BUCKET_SIZE,
           TOKENS_REMAINING + 
           CAST(TIMESTAMPDIFF(2, CHAR(CURRENT_TIMESTAMP - LAST_REFILL_AT)) 
                * REFILL_RATE / 60.0 AS INTEGER)
       ) - 1,
       LAST_REFILL_AT = CURRENT_TIMESTAMP
 WHERE CUSTOMER = ?
   AND BUCKET_TYPE = 'RPM'
   AND TOKENS_REMAINING + 
       CAST(TIMESTAMPDIFF(2, CHAR(CURRENT_TIMESTAMP - LAST_REFILL_AT))
            * REFILL_RATE / 60.0 AS INTEGER) >= 1;
```

Atomic refill-and-decrement. Returns 0 rows if the bucket is empty (request rejected), 1 row if successful.

In the PHP worker, this gets wrapped in `RateLimiter::tryAcquire()` and called before every request. If it returns false, the worker returns `RATE_LIMITED` to RPG without calling the provider.

### BYOK customers

BYOK customers don't share the K3S account, so they don't share the K3S budget. But their own provider account has a limit, and exceeding it means 429s and retry storms.

For BYOK, the bucket protects the customer's *own* account from accidental over-use. K3S configures the bucket to slightly under the customer's actual provider limit (say, 80%). This gives headroom for traffic K3S doesn't know about (other apps the customer runs against the same account).

---

## Retry and backoff

Already covered in [The PHP worker]({% link php-worker/index.md %}#middleware), but worth restating in this context.

Retry policy:

- Retry on: 429, 500, 502, 503, 504, 529 (Anthropic's overloaded), and `ConnectException`.
- Don't retry on: 400 (request was bad, retrying won't fix it), 401 (auth, key is wrong), 404 (endpoint missing).
- Maximum 5 attempts.
- Exponential backoff with jitter: `base = 1000ms * 2^retry, jitter = random(0, base), wait = base + jitter`.
- The total max wait across all retries: roughly 30-60 seconds.

When a request times out client-side (the provider didn't respond within the configured timeout), there's a question of whether to retry or fail. V1 fails (returns `TIMEOUT`). The provider may have processed the request, so retrying could double-bill. RPG decides whether to re-queue.

---

## Per-call cost telemetry

Every successful call writes a row to `K3SAI.USAGE_LOG`. The cost basis is computed at write time, not later, because rate cards rarely change retroactively and computing on read is much slower.

```php
class CostCalculator
{
    private array $rates;

    public function __construct()
    {
        // Rates per million tokens, in USD
        // Update when providers change pricing
        $this->rates = [
            'claude-sonnet-4-5' => ['input' => 3.00, 'output' => 15.00],
            'claude-opus-4-7'   => ['input' => 15.00, 'output' => 75.00],
            'gpt-4o'            => ['input' => 5.00, 'output' => 15.00],
            'gpt-4o-mini'       => ['input' => 0.15, 'output' => 0.60],
        ];
    }

    public function compute(string $model, int $tokensIn, int $tokensOut): float
    {
        $rate = $this->rates[$model] ?? null;
        if ($rate === null) {
            return 0.0; // unknown model, log raw tokens, compute later
        }
        return ($tokensIn * $rate['input'] + $tokensOut * $rate['output']) / 1_000_000;
    }
}
```

Two notes:

For unknown models, we record `0.0` and rely on tokens for later cost reconstruction. The alternative — failing the call because we don't recognize the model — is too brittle. New models will appear faster than this code updates.

For on-prem models (Ollama on K3S hardware), cost is conventionally 0 in the table since the marginal cost is hardware amortization, not per-call. K3S can compute total spend separately if needed.

### Margin for hosted customers

For hosted customers, the cost row is K3S's cost. K3S charges the customer some margin on top of that. The margin computation is *not* in the worker; it's done at billing time by aggregating usage rows and multiplying. This keeps the worker simple — it records facts; billing applies policies.

---

## Provider-specific notes

A few things to know about each provider that the abstraction has to handle.

### Anthropic

- API: `/v1/messages`, header-auth via `x-api-key`.
- Max tokens enforced strictly; request fails if exceeded.
- 529 status code for "overloaded" — similar to 503 but provider-specific. Worth retrying.
- System prompt goes in a top-level `system` field (not in messages array).
- Returns `stop_reason` of `"end_turn"`, `"max_tokens"`, etc.
- Token counts in `usage.input_tokens` and `usage.output_tokens`.

### OpenAI

- API: `/v1/chat/completions`, header-auth via `Authorization: Bearer`.
- System prompt goes as a `messages` array entry with `role: system`.
- Returns `finish_reason` of `"stop"`, `"length"`, etc.
- Token counts in `usage.prompt_tokens` and `usage.completion_tokens`.
- Has an alternative `/v1/responses` API that's similar; we use `/v1/chat/completions` for V1.

### Ollama (on-prem)

- API: `/api/generate` (single-turn) or `/api/chat` (multi-turn).
- No auth header in the default install; trust is by network topology.
- No usage data in some versions; have to count manually or accept null.
- Significantly less polish in error responses; expect HTML error pages for some failures.

The provider abstraction normalizes all of these. Worker code never sees these details.

---

## What's deliberately not here in V1

- **Streaming responses.** All replies are collected fully before returning. Fine for short responses; not for long ones. Future enhancement.
- **Provider failover.** If Anthropic is down, fall back to OpenAI? Not in V1. Keeps things simpler; customer's profile picks one provider.
- **Cost-aware routing.** Pick the cheapest provider that can do the task. Way out of V1 scope.
- **Sub-customer accounting.** Some K3S customers may have their own internal users; tracking AI usage per their internal user isn't supported.
- **Per-request audit logging beyond usage_log.** If compliance requires keeping the actual prompt and response text, that's a separate concern.

---

## Open for discussion

V1 calls in this chapter that need real validation:

- **KEK in IFS file vs. `*VLDL`.** I picked IFS for V1 simplicity; security review may prefer `*VLDL`. Worth a deliberate decision.
- **Token bucket in DB2 vs. faster store.** DB2 is fine for the scale we're at, but the latency hit is real. If we hit a wall, switching is non-trivial.
- **Where the rate card lives.** I have it in PHP code. Putting it in a DB2 table makes updates an admin operation rather than a deploy.
- **Whether usage_log captures the prompt.** Currently it doesn't. Helpful for debugging but big and potentially sensitive.
- **What "RATE_LIMITED" looks like to the user.** RPG returns the error to the row; it gets recorded as failed. But should we surface "your account is rate limited, your batch will be slower" to the user upfront, or just let it slow down silently?

---

[Next: Operating in production]({% link operations/index.md %}) →
