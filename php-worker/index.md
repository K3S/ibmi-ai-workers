---
title: The PHP worker
nav_order: 7
has_children: false
---

# The PHP worker
{: .no_toc }

**Status:** Draft V3 V2

The PHP worker is the smallest part of this architecture, by design. It pulls a request off a data queue, calls an AI provider, sends a reply to another data queue, and goes back to waiting. That's it. Everything that makes the production worker more interesting than the demo is layered onto that core: configuration, profile resolution, retry middleware, logging, lifecycle management. None of those layers change what the worker fundamentally is.

This chapter walks through the production-shape worker, explaining what each layer adds and why. It builds on the demo from [Quickstart 1]({% link quickstart-1/index.md %}) — the demo is a stripped-down version of what's described here, and the production version is what the demo grows up to be.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What the worker is

A long-lived PHP CLI process. One per "slot" of AI throughput you want to provide. Started by an autostart job at IPL or by a script after a planned restart; runs until killed. No HTTP server, no Apache, no FastCGI. Just `php /opt/k3s/ai-worker/bin/worker.php` running in PASE.

What it does, in a loop:

1. Receive a message from `K3SAI/AIOUTQ` (the shared inbound queue).
2. Parse the message as JSON per the [V1 contract]({% link contract/index.md %}).
3. Resolve the AI profile referenced in the message — provider, model, API key, rate limits.
4. Build a provider-specific request.
5. Send the request via Guzzle, with retry middleware, redaction middleware, and timeout enforcement.
6. Translate the response into the V1 reply format.
7. Send the reply to the named reply queue.
8. Log usage (tokens, latency, cost).
9. Loop.

What it doesn't do:

- Read or write operational tables.
- Know what a customer's business is, what a PE check is, or what to do with an AI response.
- Decide which row to process next — that's the RPG side's job.
- Hold long-term state in memory.
- Talk directly to other PHP workers.

The worker is replaceable, restartable, and shareable. Three properties that emerge naturally when the worker is small.

---

## Project structure

The worker lives in `/opt/k3s/ai-worker/` on the IBM i. One installation, shared by all customers.

```
/opt/k3s/ai-worker/
├── bin/
│   └── worker.php              # Entry point
├── src/
│   ├── Config.php              # Environment & secrets loading
│   ├── Logger.php              # Structured logging to stderr/file/DB2
│   ├── QueueClient.php         # SNDDTAQ/RCVDTAQ wrappers
│   ├── ProfileResolver.php     # AI profile lookup
│   ├── ProviderInterface.php   # Provider abstraction
│   ├── Provider/
│   │   ├── AnthropicProvider.php
│   │   ├── OpenAiProvider.php
│   │   └── OllamaProvider.php
│   ├── Middleware/
│   │   ├── RedactionMiddleware.php
│   │   └── RetryMiddleware.php
│   ├── UsageLogger.php         # Writes per-call usage rows
│   └── Worker.php              # The main loop
├── config/
│   └── worker.php              # Defaults & overrides
├── composer.json
├── composer.lock
├── .env                        # NOT in source control
└── vendor/
```

This is more structured than the demo's single `worker.php`. The reason isn't ceremony — it's that several distinct concerns are easier to evolve when they live in separate files.

---

## The entry point

`bin/worker.php` is small. It bootstraps the dependencies and starts the main loop.

```php
<?php
declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php';

use K3S\AiWorker\Config;
use K3S\AiWorker\Logger;
use K3S\AiWorker\QueueClient;
use K3S\AiWorker\ProfileResolver;
use K3S\AiWorker\UsageLogger;
use K3S\AiWorker\Worker;

$config         = Config::loadFromEnv();
$logger         = new Logger($config);
$queueClient    = new QueueClient($config, $logger);
$profileResolver = new ProfileResolver($config, $logger);
$usageLogger    = new UsageLogger($config, $logger);

$worker = new Worker(
    $queueClient,
    $profileResolver,
    $usageLogger,
    $logger,
    $config,
);

$logger->info('Worker starting', ['pid' => getmypid(), 'queue' => $config->inboundQueue]);

// Signal handling for graceful shutdown
pcntl_async_signals(true);
pcntl_signal(SIGTERM, fn() => $worker->stop());
pcntl_signal(SIGINT,  fn() => $worker->stop());

$worker->run();

$logger->info('Worker stopped cleanly');
exit(0);
```

The signal handlers matter. Without them, an `ENDJOB` or `kill` mid-request leaves the in-flight AI call orphaned and the requesting RPG worker waiting on a reply that never comes. With them, the worker finishes its current request, sends the reply, and exits. The blast radius of a restart is one row, not five.

---

## The main loop

`Worker::run()` is the heart of the process. About 60 lines.

```php
public function run(): void
{
    while ($this->running) {
        $message = $this->queueClient->receive(
            library: $this->config->inboundLibrary,
            queue:   $this->config->inboundQueue,
            waitSeconds: 30,
        );

        if ($message === null) {
            continue; // timeout, keep listening
        }

        $this->processMessage($message);
    }
}

private function processMessage(string $rawMessage): void
{
    $request = $this->parseRequest($rawMessage);
    if ($request === null) {
        $this->logger->warning('Discarded unparseable message', ['raw' => substr($rawMessage, 0, 200)]);
        return;
    }

    $startTime = microtime(true);
    $reply = $this->buildReply($request, $startTime);

    try {
        $this->queueClient->send(
            library: $request['reply_queue']['library'],
            queue:   $request['reply_queue']['name'],
            message: json_encode($reply, JSON_UNESCAPED_SLASHES),
        );
    } catch (Throwable $e) {
        $this->logger->error('Failed to send reply', [
            'request_id' => $request['request_id'],
            'error' => $e->getMessage(),
        ]);
    }

    $this->usageLogger->record($request, $reply);
}

private function buildReply(array $request, float $startTime): array
{
    try {
        $profile = $this->profileResolver->resolve($request['profile_ref']);
        $provider = $this->providerFor($profile);
        $response = $provider->send($request, $profile);

        return $this->successReply($request, $response, $startTime);
    } catch (ProfileNotFoundException $e) {
        return $this->errorReply($request, 'PROFILE_NOT_FOUND', $e->getMessage());
    } catch (ProviderRateLimitException $e) {
        return $this->errorReply($request, 'RATE_LIMITED', $e->getMessage(), $e->attempts);
    } catch (ProviderAuthException $e) {
        return $this->errorReply($request, 'PROVIDER_AUTH', $e->getMessage());
    } catch (ProviderTimeoutException $e) {
        return $this->errorReply($request, 'TIMEOUT', $e->getMessage());
    } catch (Throwable $e) {
        return $this->errorReply($request, 'INTERNAL', $e->getMessage());
    }
}
```

A few things worth noticing:

The main loop has three states: receive, process, repeat. There's no other branching at the top level. Everything sophisticated is delegated to the helper classes.

The `processMessage` method is designed to always send *some* reply. Even when things go wrong — bad JSON, unknown profile, provider timeout — the requesting RPG worker gets a structured error reply and can move on. The worker enforces this here.

That said: "designed to" is not "guaranteed." The PHP process can crash between receiving and replying. The IBM i can reboot mid-request. A bug can throw before reaching the send call. Any of these leave RPG waiting on a reply that never arrives. **RPG must always set a `WAIT_TIME` on its receive call and have a recovery path for timeouts** — typically marking the row as `TIMEOUT` and either retrying it later or flagging it for review. The contract is the worker's commitment, not a guarantee against process failure.

The exception types are domain-specific. Each maps cleanly to one of the V1 contract's error codes. This is what lets the worker translate provider-specific failures into a stable API.

---

## The provider abstraction

This is where one of the most important production properties lives: the worker doesn't know which AI provider it's talking to. It holds a `ProviderInterface`, gets a concrete implementation from a factory based on the AI profile, and calls it.

```php
namespace K3S\AiWorker;

interface ProviderInterface
{
    /**
     * Send a request to the AI provider.
     *
     * @param array $request The V1 contract request
     * @param Profile $profile The resolved AI profile (provider config + key)
     * @return ProviderResponse
     * @throws ProviderRateLimitException
     * @throws ProviderAuthException
     * @throws ProviderTimeoutException
     * @throws ProviderException
     */
    public function send(array $request, Profile $profile): ProviderResponse;

    /**
     * Light health check. Used by ops tooling, not by the worker loop.
     */
    public function healthCheck(Profile $profile): bool;

    /**
     * Estimate cost for a request without sending it. Optional; may return null.
     */
    public function estimateCost(array $request, Profile $profile): ?CostEstimate;
}
```

The worker holds an `ProviderInterface`, never a concrete class. Three implementations ship with V1:

- `AnthropicProvider` — speaks Anthropic's `/v1/messages` API.
- `OpenAiProvider` — speaks OpenAI's `/v1/chat/completions` API.
- `OllamaProvider` — speaks Ollama's `/api/generate` for on-premises deployments.

Each one knows its own provider's quirks: request body shape, header format, response parsing, error code translation, retry semantics. The interface is small enough that adding a fourth provider (Google, Mistral, a custom on-prem service) is a matter of writing one class.

Here's a sketch of the Anthropic implementation, abbreviated:

```php
class AnthropicProvider implements ProviderInterface
{
    public function __construct(
        private Client $http,
        private LoggerInterface $logger,
    ) {}

    public function send(array $request, Profile $profile): ProviderResponse
    {
        $body = [
            'model'       => $request['model_override'] ?? $profile->model,
            'max_tokens'  => $request['max_tokens'] ?? 1024,
            'temperature' => $request['temperature'] ?? 0.0,
            'messages'    => [['role' => 'user', 'content' => $request['prompt']]],
        ];
        if (isset($request['system_prompt'])) {
            $body['system'] = $request['system_prompt'];
        }

        try {
            $response = $this->http->post($profile->endpoint . '/v1/messages', [
                'headers' => [
                    'x-api-key'         => $profile->apiKey->reveal(),
                    'anthropic-version' => '2023-06-01',
                    'content-type'      => 'application/json',
                ],
                'json'    => $body,
                'timeout' => ($request['timeout_ms'] ?? 60000) / 1000,
            ]);

            return $this->parseResponse($response);
        } catch (RequestException $e) {
            throw $this->translateException($e);
        }
    }

    private function translateException(RequestException $e): ProviderException
    {
        $status = $e->getResponse()?->getStatusCode();

        return match (true) {
            $status === 401 => new ProviderAuthException('Anthropic auth failed'),
            $status === 429 => new ProviderRateLimitException('Anthropic rate-limited'),
            $status >= 500  => new ProviderException('Anthropic server error: ' . $status),
            default         => new ProviderException('Anthropic request failed: ' . $e->getMessage()),
        };
    }
}
```

The provider does the work of speaking the provider's protocol and translating its errors into the worker's domain language. The worker doesn't see HTTP at all by the time the response gets back to it.

---

## Middleware

Two pieces of cross-cutting behavior live in Guzzle middleware: redaction (so we never log API keys) and retry (so transient failures don't fail batches).

### Redaction

```php
class RedactionMiddleware
{
    private const SENSITIVE_HEADERS = [
        'x-api-key',
        'authorization',
        'cookie',
    ];

    public function __invoke(callable $handler): callable
    {
        return function (RequestInterface $request, array $options) use ($handler) {
            // Stash a redacted copy for any logger that wants it
            $options['_redacted_request'] = $this->redactRequest($request);
            return $handler($request, $options);
        };
    }

    private function redactRequest(RequestInterface $request): array
    {
        $headers = [];
        foreach ($request->getHeaders() as $name => $values) {
            $headers[$name] = in_array(strtolower($name), self::SENSITIVE_HEADERS, true)
                ? '[REDACTED]'
                : $values;
        }
        return [
            'method'  => $request->getMethod(),
            'uri'     => (string) $request->getUri(),
            'headers' => $headers,
        ];
    }
}
```

Every logger in the worker uses the redacted version of the request, never the raw one. There's no way to accidentally log an API key — it's stripped before logging is even attempted.

### Retry

```php
class RetryMiddleware
{
    private const MAX_ATTEMPTS = 5;
    private const RETRYABLE_STATUSES = [429, 500, 502, 503, 504, 529];

    public function __invoke(callable $handler): callable
    {
        return Middleware::retry(
            $this->shouldRetry(...),
            $this->backoff(...),
        )($handler);
    }

    private function shouldRetry(
        int $retries,
        RequestInterface $request,
        ?ResponseInterface $response = null,
        ?Throwable $exception = null,
    ): bool {
        if ($retries >= self::MAX_ATTEMPTS) {
            return false;
        }
        if ($response && in_array($response->getStatusCode(), self::RETRYABLE_STATUSES, true)) {
            return true;
        }
        if ($exception instanceof ConnectException) {
            return true; // network errors are retryable
        }
        return false;
    }

    private function backoff(int $retries): int
    {
        // Exponential backoff with jitter, in milliseconds
        $base = 1000 * (2 ** $retries);
        $jitter = random_int(0, $base);
        return $base + $jitter;
    }
}
```

Five attempts maximum, exponential backoff with jitter, retryable on network errors and the documented retryable HTTP statuses. If a request is *still* failing after five attempts, the worker gives up and returns a `RATE_LIMITED` or `PROVIDER_ERROR` to the RPG side. RPG can decide to re-queue the work unit later.

The jitter is important. Without it, when the AI provider hiccups, every in-flight request hits its retries at exactly the same time, doubling load right when the provider is recovering. Jitter spreads the retries across time so the provider can heal.

---

## Profile resolution

The AI profile encodes which provider to use, which model, which API key, which rate limits, and any per-customer overrides. It lives in a DB2 table in the K3S admin library and is looked up by `profile_ref` from each request.

```php
class ProfileResolver
{
    private array $cache = [];

    public function __construct(
        private DB2Connection $db,
        private KeyVault $keyVault,
    ) {}

    public function resolve(string $profileRef): Profile
    {
        if (isset($this->cache[$profileRef])) {
            return $this->cache[$profileRef];
        }

        $row = $this->db->fetchOne(
            'SELECT PROVIDER, MODEL, ENDPOINT, KEY_REF, STATUS
               FROM K3SAI.AI_PROFILE
              WHERE PROFILE_REF = ?',
            [$profileRef],
        );

        if (!$row) {
            throw new ProfileNotFoundException("Unknown profile_ref: {$profileRef}");
        }
        if ($row['STATUS'] !== 'ACTIVE') {
            throw new ProfileNotFoundException("Profile {$profileRef} is {$row['STATUS']}");
        }

        $profile = new Profile(
            ref:      $profileRef,
            provider: $row['PROVIDER'],
            model:    $row['MODEL'],
            endpoint: $row['ENDPOINT'],
            apiKey:   $this->keyVault->resolve($row['KEY_REF']),
        );

        $this->cache[$profileRef] = $profile;
        return $profile;
    }

    public function clearCache(): void
    {
        $this->cache = [];
    }
}
```

Three things:

The cache prevents a DB lookup on every request. For a long-lived worker handling thousands of requests, this matters.

The cache invalidation is intentionally crude: a `clearCache()` method that ops can trigger via a signal handler when profiles change. For V1 this is enough; if profile changes need to be picked up faster, a TTL or pubsub mechanism is the next step.

The API key never lives in the DB row directly. The DB stores a `KEY_REF`, a small handle. The actual key material lives in the key vault, encrypted, and is only loaded into memory when needed. The `Profile::$apiKey` property is a `SecretValue` object that wraps the key and zeroes it from memory when the profile is garbage-collected. Details on the key vault are in the [providers chapter]({% link providers/index.md %}).

---

## Configuration

The worker reads its configuration from environment variables and a config file. Nothing customer-specific lives in either — that's all in DB2 profiles. The config covers worker-level defaults: timeouts, queue names, log destinations, default provider for fallback.

```php
class Config
{
    public function __construct(
        public readonly string $inboundLibrary,
        public readonly string $inboundQueue,
        public readonly string $logDestination,
        public readonly int    $defaultTimeoutMs,
        public readonly int    $maxRetries,
        public readonly string $usageLogTable,
    ) {}

    public static function loadFromEnv(): self
    {
        return new self(
            inboundLibrary:   getenv('K3SAI_INBOUND_LIB')   ?: 'K3SAI',
            inboundQueue:     getenv('K3SAI_INBOUND_QUEUE') ?: 'AIOUTQ',
            logDestination:   getenv('K3SAI_LOG_DEST')      ?: 'stderr',
            defaultTimeoutMs: (int)(getenv('K3SAI_TIMEOUT_MS') ?: 60000),
            maxRetries:       (int)(getenv('K3SAI_MAX_RETRIES') ?: 5),
            usageLogTable:    getenv('K3SAI_USAGE_TABLE')   ?: 'K3SAI.USAGE_LOG',
        );
    }
}
```

This pattern — environment for deployment-specific values, code for defaults — keeps the worker easy to run locally for testing (just set a few env vars) and easy to deploy on Calvin (the autostart job sets the variables once).

---

## Lifecycle

Worker processes are managed as IBM i autostart jobs. One subsystem (`K3SAIWRK` is one possible name) hosts them. Each worker is one job. Job count is the parallelism you want.

The autostart job description:

```
CRTJOBD JOBD(K3SAI/AIWRKJOBD)
        TEXT('K3S AI Worker autostart')
        JOBQ(QSYS/QSYSNOMAX)
        OUTQ(QPRINT)
        USER(K3SAIWRK)
        INLLIBL(K3SAI QGPL QTEMP)
        RQSDTA('CALL PGM(K3SAI/AIWSTART)')
```

`AIWSTART.CLLE` is a tiny CL program that launches `php /opt/k3s/ai-worker/bin/worker.php` from PASE. It's the same as starting it manually from QSH, just packaged so the subsystem can do it.

The subsystem description tells IBM i how many workers to start and in which pool:

```
CRTSBSD SBSD(K3SAI/K3SAIWRK)
        POOLS((1 *BASE))
        TEXT('K3S AI Worker subsystem')
        AUT(*USE)
        SGNDSPF(*NONE)

ADDAJE  SBSD(K3SAI/K3SAIWRK)
        JOB(WORKER1)
        JOBD(K3SAI/AIWRKJOBD)
ADDAJE  SBSD(K3SAI/K3SAIWRK)
        JOB(WORKER2)
        JOBD(K3SAI/AIWRKJOBD)
ADDAJE  SBSD(K3SAI/K3SAIWRK)
        JOB(WORKER3)
        JOBD(K3SAI/AIWRKJOBD)
ADDAJE  SBSD(K3SAI/K3SAIWRK)
        JOB(WORKER4)
        JOBD(K3SAI/AIWRKJOBD)
```

`STRSBS K3SAI/K3SAIWRK` starts all four workers. `ENDSBS` stops them — and because of the SIGTERM handlers, they drain in-flight requests and exit cleanly.

If a worker crashes, the autostart job entry restarts it automatically. This is your built-in supervision: workers come back up on their own, you don't have to write a separate watcher.

---

## When to add or remove workers

You add a worker by adding another autostart job entry and restarting the subsystem (or just running another `STRSBSJOB` once). You remove one by ending its job; the subsystem cleans up.

How many workers should you have? The answer is governed by three constraints:

1. **AI rate limit.** Whatever your provider's RPM/TPM limit is, divided by per-worker throughput. If Anthropic gives you 4000 RPM and a worker can handle 10-12 concurrent requests via Guzzle's pool with ~1s latency each, that's roughly 600-720 RPM per worker — so 6-7 workers saturate the budget. Anything past that is wasted. (If you've measured and increased pool_size to 30+, recompute accordingly.)
2. **IBM i resource budget.** Each worker is ~50 MB resident plus its DB2 connection. Multiple workers compete for memory pool and CPU. Run more workers than your subsystem can host and you'll see queueing.
3. **Variance smoothing.** One worker is a single point of failure. Two is the minimum for HA. Beyond that, more workers smooth out latency variance — when one is mid-call, others are picking up new work.

Starting point: 4 workers. Adjust based on what you see. The subsystem makes adjustment a 30-second operation.

---

## Logging and observability

The worker logs four kinds of events, each at a different volume:

| Event | Volume | Where it goes |
|:---|:---|:---|
| Per-request usage | High (every call) | `K3SAI.USAGE_LOG` table |
| Errors | Low (when things go wrong) | stderr + log file |
| Lifecycle events | Very low (start/stop/profile change) | log file |
| Debug | Off in production | stderr if enabled |

Usage logging is the most important of these — it's what you'll use for billing customers, capacity planning, and debugging "why was that batch slow?" Every successful and unsuccessful call writes one row.

```sql
CREATE TABLE K3SAI.USAGE_LOG (
    LOG_ID           BIGINT GENERATED ALWAYS AS IDENTITY,
    WORKER_PID       INTEGER       NOT NULL,
    REQUEST_ID       VARCHAR(36)   NOT NULL,
    CUSTOMER         VARCHAR(10),
    PROFILE_REF      VARCHAR(20),
    PROVIDER         VARCHAR(20),
    MODEL            VARCHAR(50),
    STATUS           VARCHAR(20)   NOT NULL,
    TOKENS_IN        INTEGER,
    TOKENS_OUT       INTEGER,
    LATENCY_MS       INTEGER,
    COST_BASIS_USD   DECIMAL(10,6),
    ERROR_CODE       VARCHAR(30),
    LOGGED_AT        TIMESTAMP     NOT NULL DEFAULT CURRENT TIMESTAMP,
    PRIMARY KEY (LOG_ID)
);

CREATE INDEX K3SAI.USAGE_LOG_CUSTOMER_TIME 
       ON K3SAI.USAGE_LOG (CUSTOMER, LOGGED_AT);
CREATE INDEX K3SAI.USAGE_LOG_REQUEST 
       ON K3SAI.USAGE_LOG (REQUEST_ID);
```

The `COST_BASIS_USD` column is computed at log time using the profile's published rates and the actual tokens. Computing it later from the rates table is possible but slower; computing it at write time means cost reports are a single SELECT.

The two indexes are sized for different queries: `(CUSTOMER, LOGGED_AT)` for "show me ACME's usage today," `(REQUEST_ID)` for "what happened on that one row that errored?"

---

## What's deliberately not here in V1

The worker is intentionally smaller than what a fully production-grade async worker could be. Things we're not doing yet:

- **No worker-side queue depth awareness.** The worker doesn't slow down or push back when AI_OUT_QUEUE grows; it just consumes as fast as it can. If the queue grows, that's a signal for ops, not an action for the worker.
- **No prefetching.** The worker handles one request at a time. We could improve throughput-per-worker by having one worker hold N requests in flight simultaneously via Guzzle's async pool. Worth doing if measurement shows it's needed.
- **No rate-limit fairness across customers.** The worker pulls FIFO from `AIOUTQ`. If one customer fires a 10,000-row batch, they monopolize the worker pool for the duration. Fairness is a real concern but lives in the [providers chapter]({% link providers/index.md %}) — it's where the token bucket logic belongs.
- **No streaming responses.** Every reply is buffered fully before being put on the queue. This is fine for short responses; for long ones it adds latency that streaming could remove. Not a V1 concern.
- **No request batching.** Some providers (Anthropic, OpenAI) support batch APIs that are cheaper but slower. Worth using for non-time-sensitive checks. Not in V1.

Each of these is a real direction for the worker to grow. None is needed for V1.

---

## Open for discussion

V1 calls in this chapter that are worth revisiting:

- **Subsystem name.** `K3SAIWRK` is a placeholder; pick something matching K3S conventions.
- **Default worker count.** Four is reasonable but unmeasured. May want fewer at start, more once we have data.
- **Provider abstraction granularity.** The `ProviderInterface` is small. We may find we want to split it (e.g., a separate `StreamingProviderInterface` later) or push more into it.
- **Cost basis computation timing.** Computing at write time is fast for reads but assumes the rate doesn't change retroactively. If it does, you have to re-run the rate calc to fix history. Trade-off worth thinking about.
- **The KeyVault interface.** Sketched here as a black box. Detail covered in [providers chapter]({% link providers/index.md %}); revisit as a design once that chapter is settled.

---

[Next: The RPG worker pool]({% link rpg-pool/index.md %}) →
