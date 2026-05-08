---
title: Foundations
nav_order: 2
has_children: false
---

# Foundations
{: .no_toc }

**Status:** Draft

This chapter tells you what you need to know, what you need to have running, and what permissions you need before this guide becomes useful. It's deliberately specific. If something on these lists is unfamiliar or unavailable, the honest move is to address it before going further.

The chapter applies to both paths in this guide: pure RPG and RPG + PHP. Some of the requirements only apply to the PHP path; we mark those clearly.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Expertise you need

You don't need to be expert in all of these. You do need to be at least *comfortable* in each one, in the sense that you can read existing code in that area and make small changes without getting stuck.

### RPG and CL — comfortable

You can read modern free-format RPGLE. You can write a CL program that calls another program with parameters. You understand how `SBMJOB` differs from `CALL`, and you've used at least one of them in production. You know what a service program is, even if you haven't written many.

If you're earlier in your RPG journey, our companion site [RPG Tutorial](https://rpgtutorial.k3s.com) covers the modern RPGLE basics this guide assumes.

### DB2 for i — comfortable

You can write SQL against DB2 for i, including joins and subqueries. You understand the difference between SQL access and native record access (RLA), and you know which one you're using when. You've used `RUNSQLSTM` or STRSQL or an SQL editor against your system.

### IBM i job management — comfortable

You know what a subsystem is, what a job queue is, and how `WRKACTJOB` differs from `WRKSBMJOB`. You've watched a job run, looked at its joblog, and used `WRKSPLF` to see what it produced.

### Library lists — comfortable

You know what a library list is, why it matters, and how it gets set when a job starts. You understand the difference between qualified and unqualified table references in SQL and in RPG. You've debugged at least one "wrong table got read" problem caused by library list confusion.

This guide leans heavily on library lists as a multi-tenant boundary. If library lists feel like magic to you, the multi-tenancy chapter will not be useful.

### PHP — comfortable, *if* you're going to take the RPG + PHP path

You can read and write PHP code at the level of a real application. You understand Composer and `vendor/` directories. You've used at least one third-party PHP library by reading its docs and adding it to a project.

You don't need to be an expert in async PHP, Guzzle internals, or Mezzio middleware. We'll cover the parts of those that matter as we go.

If you've never run PHP on IBM i specifically — only on Linux or in a web framework — that's fine. The Seiden Group's PHP for IBM i installation is straightforward.

**You only need PHP expertise if you're following the RPG + PHP path.** The pure-RPG path doesn't use PHP at all.

### HTTP and JSON — comfortable

You know what a REST API is. You can read JSON and write it by hand. You understand HTTP status codes well enough to know what a 200, a 401, a 429, and a 500 each mean. You've used `curl` or Postman to test an API at least once.

### LLMs — at least one round trip

You've made at least one LLM call yourself, somehow, somewhere. You've sent a prompt to an API and gotten a response back. You understand at a high level that LLMs are billed by tokens, that they have rate limits, and that they sometimes return unexpected output.

You do not need to understand transformers, embeddings, fine-tuning, or RAG. We treat the LLM as a black box.

If you've literally never made an LLM call from anywhere, do that first. Sign up for an Anthropic or OpenAI account, send one request from `curl`, see the response. Half an hour. This guide will make much more sense afterward.

---

## What you need running

### An IBM i system you can deploy to

Any supported IBM i version will work. We've built and run on IBM i 7.4 and 7.5; older versions will work with adjustments. You need:

- The ability to create libraries, source files, and objects.
- The ability to compile RPG and CL programs.
- The ability to create data queues (`CRTDTAQ`).
- The ability to submit jobs to a subsystem of your choice.

For the pure-RPG path, you also need:

- IBM i 7.4 or newer, or older versions with `SYSTOOLS.HTTPPOSTCLOB` available. The pure-RPG demos use SQL HTTP services, which require a recent enough IBM i.

For the RPG + PHP path, you also need:

- A PHP installation that can run from PASE/QSH as a CLI process, with `composer` available, with `ext-curl` enabled, and with `ext-ibm_db2` enabled.

The Seiden Group's distribution covers all the PHP requirements out of the box. If you have an existing Zend Server installation, that works too — just confirm CLI invocation works.

### YAJL for RPG JSON

Both paths use YAJL for JSON building (`DATA-GEN`) and parsing (`DATA-INTO`). Most modern IBM i shops have YAJL installed already. If yours doesn't, it's a free download from Scott Klement.

### Outbound HTTPS to your AI provider

The IBM i needs to be able to make outbound HTTPS connections to whichever AI provider you're using — `api.anthropic.com`, `api.openai.com`, your own on-premises endpoint, whatever. This means:

- DNS resolution working from the IBM i for that hostname.
- Outbound TCP 443 open through whatever firewall sits between the IBM i and the internet.
- TLS trust established — your IBM i needs to trust the certificate the AI endpoint presents.

Verify by running, from QSH:

```
curl -v https://api.anthropic.com/v1/messages
```

You should get a 401 (because you didn't send a key) — that's a successful test.

### An AI provider account with API access

Sign up for whichever provider you'll use first. Get an API key. Confirm you can make a call with it from your laptop using `curl` before involving the IBM i.

### A test data set

You'll want a representative DB2 table with some realistic-looking rows you can use to drive batch processing during development. A subset of production data with PII scrubbed, or synthetic data of the same shape.

---

## Permissions and authority

### Object authority on IBM i

The user profile that runs your batch jobs needs:

- `*USE` authority on the libraries containing the source code.
- `*CHANGE` authority on the libraries containing the data tables it reads and writes.
- `*USE` authority on the data queues it produces and consumes from.
- `*USE` authority on any service programs it calls.

Most shops handle this with group profiles. If yours doesn't, plan the authority model deliberately rather than running everything as a powerful profile.

### Authority for the PHP worker (RPG + PHP path only)

The PHP worker runs under a user profile too. That profile needs:

- `*USE` on the directory containing the PHP code.
- `*CHANGE` on any DB2 tables the worker writes to.
- `*USE` on the data queues it consumes from and produces to.

The worker should *not* have authority on customer operational tables. That's a deliberate constraint.

### API keys and secrets

API keys are sensitive. For both paths:

- Never commit keys to source control.
- Never log keys, even in development.
- For the demos, the API key lives in a tightly-restricted data area (pure RPG) or environment variable (PHP). Production handles BYOK keys with proper encryption — see [providers chapter]({% link providers/index.md %}).

### Network access

Confirm with whoever owns your network:

- Outbound HTTPS to your AI provider is allowed.
- The IBM i is not behind a proxy that requires authentication, or if it is, that PHP/curl is configured to use it.
- TLS inspection (where a corporate firewall terminates and re-signs TLS) is either disabled for AI provider hostnames or your IBM i trusts the firewall's CA.

The TLS inspection one is a quiet killer.

---

## What you don't need

- **You don't need a GPU server.** This guide works equally well calling cloud AI providers and calling on-premises models.
- **You don't need Kubernetes, Docker, or any container infrastructure.**
- **You don't need a separate Linux server.** PHP runs on the IBM i in PASE.
- **You don't need to migrate anything.** This guide is additive.
- **You don't need PHP** if you're staying with the pure-RPG path.

---

## Ready check

Before going further, you should be able to truthfully answer "yes" to:

1. I can read modern RPGLE and write CL.
2. I can write SQL against DB2 for i, including joins.
3. I understand library lists well enough to debug a wrong-table problem.
4. I have an IBM i system I can deploy to, with authority to create libraries and compile programs.
5. The IBM i can make outbound HTTPS calls to my chosen AI provider.
6. I have an AI provider API key that I've successfully used from my laptop.
7. I have YAJL installed (or can install it) for RPG JSON parsing.
8. I have a representative DB2 table I can develop against.

If you're following the RPG + PHP path, also:

9. PHP runs from QSH on my system, with `ext-curl`, `ext-ibm_db2`, and Composer available.
10. I (or someone on my team) can read and modify PHP code.

If any of these are "no," resolving the gap is the next thing to do.

---

[Next: Quickstart 1 — One worker (RPG only)]({% link quickstart-1-rpg/index.md %}) →
