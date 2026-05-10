---
title: Multi-tenancy
nav_order: 9
has_children: false
---

# Multi-tenancy
{: .no_toc }

**Status:** Draft V3 V2

This chapter is about how one shared PHP worker serves dozens of customers without knowing which one is which, and how each customer's choice of AI provider, model, and key gets resolved at runtime without polluting the worker code with customer-specific branches. The IBM i platform makes this easier than it would be on most stacks — library lists do most of the work — but there are real decisions to make about where shared admin data lives and how customer profiles are looked up.

If you've read [The PHP worker]({% link php-worker/index.md %}) and [The RPG worker pool]({% link rpg-pool/index.md %}), you've already seen the multi-tenancy pattern in pieces. This chapter pulls it together explicitly.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The shape

Two libraries per customer (the standard K3S split: `<CODE>_5DTA` for data, `<CODE>_5OBJ` for objects). One shared admin library for K3S itself: `K3SAI`. The PHP worker lives in IFS at `/opt/k3s/ai-worker/`, one installation, shared.

```
                  ┌──────────────────────┐
                  │  K3SAI               │  ← shared K3S admin library
                  │                      │
                  │   AIOUTQ        (DQ) │
                  │   AI_PROFILE   (TBL) │
                  │   USAGE_LOG    (TBL) │
                  │   K3SCMN       (PGMs)│
                  └──────────┬───────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
┌────────────┐        ┌────────────┐        ┌────────────┐
│ ACME_5DTA  │        │ DMO_5DTA   │  ...   │ XYZ_5DTA   │
│            │        │            │        │            │
│ WORK_QUEUE │        │ WORK_QUEUE │        │ WORK_QUEUE │
│ RPLY_*     │        │ RPLY_*     │        │ RPLY_*     │
│ AI_BATCH   │        │ AI_BATCH   │        │ AI_BATCH   │
│ PO_LINES   │        │ PO_LINES   │        │ PO_LINES   │
│ ...        │        │ ...        │        │ ...        │
└────────────┘        └────────────┘        └────────────┘

┌────────────┐        ┌────────────┐        ┌────────────┐
│ ACME_5OBJ  │        │ DMO_5OBJ   │  ...   │ XYZ_5OBJ   │
│            │        │            │        │            │
│ AIBATSTRT  │        │ AIBATSTRT  │        │ AIBATSTRT  │
│ AIWORKER   │        │ AIWORKER   │        │ AIWORKER   │
│ AIPRE      │        │ AIPRE      │        │ AIPRE      │
│ AIPOST     │        │ AIPOST     │        │ AIPOST     │
└────────────┘        └────────────┘        └────────────┘
```

What's in `K3SAI`:

- `AIOUTQ` — the shared inbound data queue. PHP workers consume from this. Any customer's RPG worker can write to it.
- `AI_PROFILE` — the table of AI profiles. One row per (customer, profile) combination.
- `USAGE_LOG` — every AI call writes a row. Used for billing, capacity planning, and debugging.
- Common admin programs (`K3SCMN`-prefixed) that work across customers — health checks, batch monitors, profile management UI backend.

What's in each customer's library:

- `WORK_QUEUE` — per-customer work distribution.
- Reply queues (`RPLY_*`) — created and destroyed by workers as they run.
- `AI_BATCH` — per-customer batch metadata.
- The customer's own operational tables (PO_LINES, vendors, demand, etc.).
- `AIBATSTRT`, `AIWORKER`, `AIPRE`, `AIPOST` — the worker programs, in the customer's `*_5OBJ`. Compiled per-customer because pre/post logic is customer-specific.

The PHP worker doesn't sit in any of these libraries. It lives in IFS and isn't an IBM i object at all in the traditional sense.

---

## How tenant isolation works

The IBM i library list is the tenant boundary. When `AIBATSTRT` runs in a job whose library list starts with `ACME_5OBJ ACME_5DTA K3SAI ...`, every unqualified table reference in its SQL resolves to ACME's tables. The same code, run with `BARCO_5OBJ BARCO_5DTA K3SAI ...`, reads BARCO's tables. The code doesn't change; the environment changes around it.

This pattern has three properties that matter for multi-tenancy:

**1. Customer code knows nothing about other customers.** `AIPRE` reads `PO_LINES` (unqualified). It doesn't know what library that resolves to, can't accidentally read someone else's data, and would fail to compile if it tried to qualify a name from another customer's library (because that library isn't in the build path).

**2. Authority is enforced by IBM i, not by the application.** The user profile that runs ACME's batch jobs has no authority on BARCO's tables. Even if a bug caused unqualified `PO_LINES` to somehow resolve to BARCO's library, the read would fail with a `*OBJOPR` authority error. The platform refuses to leak data; the application doesn't have to.

**3. The worker (PHP) is genuinely tenant-agnostic.** It reads from `K3SAI/AIOUTQ`, runs in its own user profile context, never touches operational tables. The customer context arrives in each message as the `customer` field, used only to scope the AI profile lookup. PHP could not look up an ACME row even if it tried — its profile doesn't have authority on ACME's tables.

The whole multi-tenancy story is summarized as: **business logic runs in the tenant's library list; transport runs in the shared library list; the platform enforces the boundary.**

---

## The AI profile table

This is the central admin object that maps "customer X wants to use Y" to "here's how to do that."

```sql
CREATE TABLE K3SAI.AI_PROFILE (
    PROFILE_REF      VARCHAR(40)   NOT NULL,
    CUSTOMER         VARCHAR(10)   NOT NULL,
    PROFILE_NAME     VARCHAR(50)   NOT NULL,
    MODE             CHAR(1)       NOT NULL,    -- 'B' = BYOK, 'H' = hosted
    PROVIDER         VARCHAR(20)   NOT NULL,    -- 'anthropic', 'openai', 'k3s_onprem'
    MODEL            VARCHAR(80)   NOT NULL,
    ENDPOINT         VARCHAR(200)  NOT NULL,
    KEY_REF          VARCHAR(60),               -- handle into KEY_VAULT; null for on-prem
    DEFAULT_MAX_TOKENS  INTEGER    DEFAULT 1024,
    DEFAULT_TEMP        DECIMAL(3,1) DEFAULT 0.0,
    SYSTEM_PROMPT       VARCHAR(4000),
    RATE_LIMIT_RPM      INTEGER,                -- requests per minute
    RATE_LIMIT_TPM      INTEGER,                -- tokens per minute
    MONTHLY_QUOTA_TOKENS BIGINT,                -- hosted only; null = unlimited
    STATUS              VARCHAR(20)   NOT NULL, -- 'ACTIVE', 'SUSPENDED', 'OVER_QUOTA'
    NOTES               VARCHAR(2000),
    CREATED_AT          TIMESTAMP     NOT NULL DEFAULT CURRENT TIMESTAMP,
    UPDATED_AT          TIMESTAMP     NOT NULL DEFAULT CURRENT TIMESTAMP,
    PRIMARY KEY (PROFILE_REF)
);

CREATE UNIQUE INDEX K3SAI.AI_PROFILE_CUST_NAME 
       ON K3SAI.AI_PROFILE (CUSTOMER, PROFILE_NAME);
```

A profile reference looks something like `ACME_DEFAULT`, `ACME_FAST`, `ACME_BUDGET` — short, readable, sortable when listed. It's what RPG sends in the `profile_ref` field of each AI request. PHP looks it up to figure out what to do.

A few notes:

**`PROFILE_REF` is the primary key, customer + name is a unique index.** This lets RPG send a stable reference that doesn't change when a profile gets renamed in admin UI, while also letting humans browse profiles by customer and name.

**`PROFILE_REF` is globally unique by convention.** The convention is `<CUSTOMER>_<PROFILE_NAME>` — for example, `ACME_DEFAULT`, `BARCO_DEFAULT`. This is why ACME's "DEFAULT" and BARCO's "DEFAULT" don't collide on the primary key — they're stored as `ACME_DEFAULT` and `BARCO_DEFAULT`. The schema doesn't *enforce* this convention; an admin tool inserting bare names without the customer prefix would compile but produce confusing collisions and look-up failures. If your insertion paths are anything other than carefully-controlled admin UI, consider adding a CHECK constraint that `PROFILE_REF LIKE CUSTOMER || '\_%'`. For V1, the convention is documented and trusted.

An alternative — making the primary key `(CUSTOMER, PROFILE_REF)` — is also defensible. We chose single-column PK so RPG can send just the ref without also sending customer (the request envelope already includes customer for other reasons; it's redundant on the lookup). Either design works; pick whichever fits your shop's habits.

**`MODE` is one character because it's hot.** Read on every request. `'B'` for bring-your-own-key, `'H'` for K3S-hosted. The full word is unnecessary at the cost of a tiny readability hit in raw SELECT output.

**`KEY_REF` is a handle, not the key itself.** Resolving the handle to the actual API key is the [providers chapter]({% link providers/index.md %})'s job. The profile table is unencrypted; the keys are not in it.

**`RATE_LIMIT_RPM` and `RATE_LIMIT_TPM` are per-customer.** For BYOK customers, this should match what their provider account allows, with some safety margin. For hosted customers, this is the budget K3S has carved out for them within K3S's own provider account. Both are enforced by the PHP worker; details in [providers]({% link providers/index.md %}).

**`MONTHLY_QUOTA_TOKENS` is for hosted billing.** When a hosted customer hits their cap, the worker returns a `RATE_LIMITED` error and the customer sees an alert. BYOK customers don't have this column populated.

---

## Profile resolution at runtime

In RPG, resolving the profile is done on the K3S side, not by RPG. RPG just sends the `profile_ref` it was configured with. PHP does the lookup.

But how does RPG *know* which profile_ref to send? Three possible patterns, in order of how I'd order them:

**Pattern A: Per-batch parameter.** The batch initiator is invoked with a profile name. The user picks "ACME_DEFAULT" or "ACME_FAST" depending on what they want. Pro: explicit, flexible. Con: requires user to know what the profiles are.

**Pattern B: Default profile per customer, configurable per batch.** Each customer has a `DEFAULT_PROFILE` setting in their config (a row in a per-customer `K3S_CONFIG` table or a data area). Batches use that unless overridden. Pro: transparent to users; advanced users can override. Con: more moving parts.

**Pattern C: Hardcoded per program.** Each batch type (PE check, vendor classification, forecast summary) hardcodes its profile name. Pro: simple. Con: changing providers means recompiling RPG.

V1 should probably do A or B. C is too brittle for production but fine for the demo.

The contract chapter's `profile_ref` field carries whatever RPG decides, opaquely. PHP looks it up.

---

## What "shared" really means

The phrase "shared PHP worker" can mislead — it suggests one big shared compute resource that customers compete for. That's not quite the right mental model. The actual sharing is:

**Shared:**

- The PHP worker pool (M processes, all tenant-agnostic)
- `K3SAI/AIOUTQ` (the inbound queue)
- `K3SAI/USAGE_LOG` (the usage tracking)
- The provider abstraction code, retry middleware, etc.
- The K3S Anthropic/OpenAI account, for hosted-tier customers
- Compute time on the PHP worker processes

**Not shared:**

- Customer libraries and their contents
- API keys (each profile's key is per-customer, even for hosted-tier where the *underlying* account is K3S's, the customer has a logical scope)
- Rate limit budgets (each customer's RPM/TPM is separately accounted)
- Reply queues (per-worker, hence per-batch, hence per-customer)
- AI profiles (per-customer rows in the shared table)
- Operational data (each customer's tables, in their library)

The model that fits is **shared compute, partitioned configuration, isolated data**. Compute (PHP processes) doesn't care about tenancy. Configuration is keyed by tenant. Data is fully partitioned.

---

## Concurrency across customers

Multiple customers can run batches simultaneously without configuration. The flow:

1. ACME starts a batch. Their `AIBATSTRT` runs in their library list, populates `ACME_5DTA/WORK_QUEUE`, submits 20 ACME workers.
2. While those run, BARCO starts a batch. Their `AIBATSTRT` populates `BARCO_5DTA/WORK_QUEUE`, submits 20 BARCO workers.
3. All 40 workers concurrently put requests on `K3SAI/AIOUTQ`.
4. PHP workers pull from `AIOUTQ` and serve them, looking up profiles by `profile_ref` (which carries the customer code).
5. PHP sends replies to whatever reply queue the request named.
6. Each customer's workers process their own work, against their own data, oblivious to the other.

The only place customer concurrency interacts is at the AI provider rate limit. If ACME and BARCO both use the same K3S Anthropic account (hosted tier for both), they share a global RPM budget. Without fairness logic, ACME's bigger batch could starve BARCO's. The token-bucket-per-customer fairness logic in the [providers chapter]({% link providers/index.md %}) prevents this.

If they're both BYOK, they have separate accounts and separate budgets, and the only shared resource is K3S's PHP worker pool. The pool is rarely a bottleneck — workers handle 30+ concurrent calls each via Guzzle pool — so this is usually fine without special handling.

---

## Onboarding a new customer

What do you do when ACME signs up?

1. Create the libraries: `CRTLIB ACME_5DTA`, `CRTLIB ACME_5OBJ` (probably already done as part of K3S onboarding regardless of AI).
2. Create the customer's user profile and group (also part of standard K3S onboarding).
3. Create `WORK_QUEUE` and `AI_BATCH` in `ACME_5DTA`. (One-time per customer.)
4. Compile `AIBATSTRT`, `AIWORKER`, `AIPRE`, `AIPOST` from K3S source into `ACME_5OBJ`. The PRE and POST programs may have customer-specific overrides; if not, they're identical to the standard versions.
5. Insert one or more rows into `K3SAI.AI_PROFILE` for ACME. At minimum: `ACME_DEFAULT` with whatever provider/model they're using.
6. If BYOK: get their API key, store it via the key vault (covered in [providers]({% link providers/index.md %})), and link the profile to the resulting `KEY_REF`.
7. If hosted: pick the right rate limit and quota for their tier; profile points at K3S's central key reference.
8. Smoke test by running a small batch and confirming the round trip works.

This whole sequence is scriptable. A `K3SCMN/ADDAICUST` admin program that takes a customer code, profile name, mode (B or H), provider, and model and does steps 3-7 makes onboarding a five-minute task.

---

## Removing a customer

When a customer leaves:

1. Set their AI profile rows to `STATUS = 'SUSPENDED'`. Existing batches finish; new ones can't start.
2. After the grace period: set to `STATUS = 'TERMINATED'` or delete the rows entirely (your retention policy).
3. If BYOK: revoke the key from the vault.
4. The customer's libraries get cleaned up via standard K3S offboarding.

The key thing: the profile change takes effect immediately for new requests, but doesn't interrupt in-flight ones. Workers that were already processing a request when the profile got suspended finish their current row and exit naturally when `WORK_QUEUE` empties.

---

## Cross-customer admin queries

Once everything's flowing through `K3SAI/USAGE_LOG`, you can answer cross-customer questions easily:

**Top spenders today:**

```sql
SELECT CUSTOMER, COUNT(*) AS REQUESTS,
       SUM(TOKENS_IN + TOKENS_OUT) AS TOTAL_TOKENS,
       SUM(COST_BASIS_USD) AS TOTAL_COST_USD
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT DATE
 GROUP BY CUSTOMER
 ORDER BY TOTAL_COST_USD DESC;
```

**Provider mix:**

```sql
SELECT PROVIDER, COUNT(DISTINCT CUSTOMER) AS CUSTOMERS,
       COUNT(*) AS REQUESTS_30D
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT DATE - 30 DAYS
 GROUP BY PROVIDER;
```

**Error rates per customer:**

```sql
SELECT CUSTOMER,
       COUNT(*) AS TOTAL,
       SUM(CASE WHEN STATUS = 'success' THEN 1 ELSE 0 END) AS SUCCESS,
       SUM(CASE WHEN STATUS = 'success' THEN 0 ELSE 1 END) AS ERRORS,
       100.0 * SUM(CASE WHEN STATUS = 'success' THEN 0 ELSE 1 END) / COUNT(*) AS ERROR_PCT
  FROM K3SAI.USAGE_LOG
 WHERE LOGGED_AT >= CURRENT DATE - 7 DAYS
 GROUP BY CUSTOMER
HAVING COUNT(*) > 100
 ORDER BY ERROR_PCT DESC;
```

This is the operational benefit of having shared usage logging. Per-customer billing and forecasting would be much harder if usage data lived in each customer's library.

---

## Library list management

Two specific patterns to be deliberate about.

**For batch jobs:** the `SBMJOB` command's `INLLIBL` parameter sets the library list. The batch initiator, when submitting workers, should pass the customer library list explicitly. Don't rely on the user's interactive library list at submit time — it's brittle and surprising.

**For PHP workers:** the autostart job's job description sets the initial library list. PHP workers running in their own subsystem should have `K3SAI` and the system libraries on their list, but no customer libraries. Customer-specific lookups happen via qualified names from PHP (e.g., `K3SAI.AI_PROFILE`), never via library list resolution.

The asymmetry is intentional: RPG workers run *as* customers and rely on library list; PHP workers run *for* K3S and don't.

---

## What's deliberately not here in V1

- **Per-customer subsystems.** All customers' batches run in the same `QBATCH` (or whatever shared subsystem). Some shops prefer per-customer subsystems for resource isolation. Achievable later if needed.
- **Cross-region or cross-LPAR multi-tenancy.** The chapter assumes one IBM i system. Distributing across multiple LPARs adds significant complexity around the shared queue and profile table.
- **Customer-level audit logging.** `USAGE_LOG` records what happened, but doesn't track who initiated what. If you need compliance-grade audit trails, that's a separate concern.
- **Quota enforcement at the boundary.** PHP enforces quotas when calls happen, but doesn't reject batch initiation up front. A batch could start, run for 10 minutes against the quota, and only then fail when over. Pre-flight quota checks would be a nice addition.
- **Profile inheritance / templates.** All profiles are fully specified. If many customers want "the same as ACME_DEFAULT but with a longer system prompt," they have to copy the row. A template/inheritance pattern is doable, not in V1.

---

## Open for discussion

V1 calls in this chapter that may want different answers:

- **Library naming for K3S admin.** I've used `K3SAI`. Could be `K3SCMN`, `K3SOPS`, `K3SADM`. K3S already has conventions; this should match them.
- **Profile name format.** I've used `<CUSTOMER>_<NAME>` like `ACME_DEFAULT`. Could be `<CUSTOMER>.<NAME>` or just `<NAME>` per-customer. The customer prefix makes the global table easier to scan but is redundant with the `CUSTOMER` column.
- **Whether `AIPRE` and `AIPOST` are in customer-specific libraries or in `K3SAI` shared.** Customer-specific keeps logic isolated; shared simplifies maintenance for customers using identical logic. Probably some hybrid.
- **The `MODE` enum (`'B'`/`'H'`).** Single character is hot, but readability suffers. `'BYOK'`/`'HOSTED'` is clearer. Net of measurement, probably worth more readability.
- **Rate limit enforcement granularity.** Per-profile or per-customer? Some customers may have multiple profiles (cheap+slow, expensive+fast); should rate limits be per-profile (each gets its budget) or per-customer (shared across profiles)? Worth deciding.

---

[Next: AI provider concerns]({% link providers/index.md %}) →
