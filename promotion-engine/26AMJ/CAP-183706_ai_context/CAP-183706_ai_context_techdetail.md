# Tech Detail: `.context/` Folder — Developer & AI Agent Guide
**Date:** 2026-04-30
**Author:** Saravanan Kesavan
**Status:** Accepted
**Introduced in:** CAP-183706 — PR #788 (2026-04-17)

---

## Table of Contents

1. [What Is `.context/` and Why It Exists](#1-what-is-context-and-why-it-exists)
2. [How Claude Code Uses This Folder](#2-how-claude-code-uses-this-folder)
   - [2a. The CLAUDE.md Entry Point](#2a-the-claudemd-entry-point)
   - [2b. How the Agent Applies Guardrails in Practice](#2b-how-the-agent-applies-guardrails-in-practice)
   - [2c. Use Cases: AI-Led Development](#2c-use-cases-ai-led-development)
   - [2d. Keeping CLAUDE.md Effective](#2d-keeping-claudemd-effective)
   - [2e. Verifying the Agent Is Following Guardrails](#2e-verifying-the-agent-is-following-guardrails)
3. [File Map](#3-file-map)
4. [Use Cases: During Development](#4-use-cases-during-development)
   - [UC-DEV-1 — Adding a New Cache](#uc-dev-1--adding-a-new-cache)
   - [UC-DEV-2 — Writing a Promotion Mutation](#uc-dev-2--writing-a-promotion-mutation-create--update--activate--deactivate)
   - [UC-DEV-3 — Using `@Cacheable` or `@CacheEvict`](#uc-dev-3--using-cacheable-or-cacheevict)
   - [UC-DEV-4 — Reading PromotionMeta in a Hot Path](#uc-dev-4--reading-promotionmeta-in-a-hot-path-multiple-promotions-per-request)
   - [UC-DEV-5 — Creating or Updating a MongoDB BO Class](#uc-dev-5--creating-or-updating-a-mongodb-bo-class)
   - [UC-DEV-6 — Adding Observability to a New Service or Cache](#uc-dev-6--adding-observability-to-a-new-service-or-cache)
   - [UC-DEV-7 — Touching Deployment Config or Adding Environment Variables](#uc-dev-7--touching-deployment-config-or-adding-environment-variables)
   - [UC-DEV-8 — Writing a New Architectural Decision Record (MADR)](#uc-dev-8--writing-a-new-architectural-decision-record-madr)
5. [Use Cases: During Code Review](#5-use-cases-during-code-review)
   - [UC-CR-1 — PR Adds a New Cache or `@Cacheable` Annotation](#uc-cr-1--pr-adds-a-new-cache-or-cacheable-annotation)
   - [UC-CR-2 — PR Contains a Promotion Mutation](#uc-cr-2--pr-contains-a-promotion-mutation)
   - [UC-CR-3 — PR Adds a New Field or Collection to MongoDB](#uc-cr-3--pr-adds-a-new-field-or-collection-to-mongodb)
   - [UC-CR-4 — PR Adds a New Service Method or Controller Endpoint](#uc-cr-4--pr-adds-a-new-service-method-or-controller-endpoint)
   - [UC-CR-5 — PR Introduces a New Tenant-Scoped Operation](#uc-cr-5--pr-introduces-a-new-tenant-scoped-operation)
   - [UC-CR-6 — PR Touches Deployment or Infrastructure Config](#uc-cr-6--pr-touches-deployment-or-infrastructure-config)
6. [How to Add a New Guardrail or MADR](#6-how-to-add-a-new-guardrail-or-madr)
7. [Security Considerations for `.context/`](#7-security-considerations-for-context)
   - [7a. What Must Never Appear in `.context/` Files](#7a-what-must-never-appear-in-context-files)
   - [7b. Prompt Injection Risk](#7b-prompt-injection-risk)
   - [7c. Change Control and Audit Trail](#7c-change-control-and-audit-trail)
   - [7d. AI Agent Permissions Boundary](#7d-ai-agent-permissions-boundary)
   - [7e. Information Disclosure Classification](#7e-information-disclosure-classification)
   - [7f. Sensitive Patterns in Generated Code](#7f-sensitive-patterns-in-generated-code)
8. [Known Open Issues in `.context/`](#8-known-open-issues-in-context)
9. [Quick Reference — Guardrail Violation Checklist](#9-quick-reference--guardrail-violation-checklist)

---

## 1. What Is `.context/` and Why It Exists

Before this folder was added, the Promotion Engine's architectural guardrails lived in PR descriptions, Slack threads, and the memories of a handful of senior engineers. That meant a new developer had no reliable way to discover that `@Cacheable` on a private method is silently ignored by Spring AOP, or that every promotion mutation *must* route through `CacheEvictionPublisher` — not call `invalidateCache` directly.

The `.context/` folder solves this by putting all non-negotiable guardrails, infrastructure standards, and architectural decisions directly in the repo — version-controlled, searchable, and readable by **both human developers and AI coding agents**. This repo uses **Claude Code** for AI-led development. The `.context/` files are the primary mechanism by which Claude Code knows what is and is not acceptable in this codebase, before it writes or reviews a single line of code.

---

## 2. How Claude Code Uses This Folder

### 2a. The CLAUDE.md Entry Point

Claude Code reads `CLAUDE.md` automatically at the start of every session. It acts as a concise directive sheet that tells the agent *which context files to load and when*. The current directives are:

```
# Promotion Engine — Claude Code guardrails

Treat `.context/code.md` as a non-negotiable guardrail for every code change and review.

If the change touches caching, MongoDB schema, Redis, RabbitMQ, or deployment config —
read `.context/infra.md` before proceeding.

If the change touches PromotionMetaCacheService, CacheEvictionPublisher, or any new cache —
read `.context/code.md` (Caching decision hierarchy) first, then the relevant MADR from
`.context/overview.md`. New caches must pass Step 0 before any cache infrastructure is introduced.
```

These three lines do the following:

| Directive | What Claude Code does |
|-----------|----------------------|
| `Treat .context/code.md as non-negotiable` | Reads and internalises all code guardrails before proposing any implementation |
| `If change touches caching / Redis / MongoDB … read infra.md` | Loads infra constraints before touching stateful resources |
| `New caches must pass Step 0` | Forces the agent to apply the caching decision hierarchy before generating any cache infrastructure code |

### 2b. How the Agent Applies Guardrails in Practice

When a developer asks Claude Code to implement a feature, the agent:

1. Reads `CLAUDE.md` at session start (automatic).
2. Checks which `.context/` files are relevant to the stated task.
3. Reads those files **before generating any code**.
4. Validates its own proposed implementation against the guardrails — catching issues like `@Cacheable` on a private method or a missing `CacheEvictionPublisher` call before the code reaches a PR.
5. Flags violations inline: *"This would violate the caching decision hierarchy in `.context/code.md` Step 0 — consider passing a `Map<>` parameter instead of adding a cache."*

### 2c. Use Cases: AI-Led Development

**UC-AI-1 — Agent proposes a new cache**

The developer says: *"Cache the result of this service method."*

Expected agent behaviour (enforced by CLAUDE.md + `.context/code.md`):
1. Reads `.context/code.md` §Caching Decision Hierarchy.
2. Challenges Step 0 first: *"Can this be solved with a bulk-load `Map` at the entry point instead?"*
3. If Step 0 doesn't apply, proposes Redis `@Cacheable` (Step 1), not Caffeine.
4. Only proposes Caffeine L1 if it can prove bounded cardinality ≤ 1,000 entries.

If the agent skips Step 0 and jumps straight to adding a Caffeine cache, that is a signal that CLAUDE.md or `.context/code.md` needs a stronger directive. Treat it as a guardrail gap.

---

**UC-AI-2 — Agent writes a promotion mutation**

The developer says: *"Add an endpoint to deactivate a promotion."*

Expected agent behaviour (enforced by MADR-0001 §AI/Code-Editor Notes):
1. Reads `CLAUDE.md` → recognises `CacheEvictionPublisher` is in scope.
2. Reads MADR-0001 §AI/Code-Editor Notes.
3. Generates the mutation code with `cacheEvictionPublisher.publishPromotionEviction(orgId, List.of(promotionId))` immediately after the DB write.
4. Does **not** call `invalidateCache` or `evictPromotionMetaCache` directly.

If the generated code calls `invalidateCache` directly, the agent missed the MADR. Ask it: *"Check MADR-0001 §AI/Code-Editor Notes item 3 — is the eviction correct?"*

---

**UC-AI-3 — Agent reviews a PR (code review mode)**

The developer says: *"Review this PR."*

Expected agent behaviour:
1. Reads `CLAUDE.md` and the relevant `.context/` sections for the diff's scope.
2. Checks each changed file against the applicable guardrails.
3. Reports violations citing the specific rule and file (e.g., *"`@Cacheable` on line 42 is on a private method — silently bypassed by Spring AOP. See `.context/code.md` §Cache Rules."*).
4. Flags missing `CacheEvictionPublisher` calls on any mutation path.
5. Checks `lastUpdatedOn` presence on BO writes.

The §4 checklist in this document maps directly to what the agent should flag. If the agent misses an item, it is a gap in either `.context/code.md` clarity or the CLAUDE.md directive.

---

**UC-AI-4 — Agent proposes a MongoDB schema change**

The developer says: *"Add a `lastRedemptionDate` field to `PromotionRedemption`."*

Expected agent behaviour (enforced by `.context/code.md` §MongoDB / BO Classes):
1. Confirms `lastUpdatedOn` is set on every write path in the affected class.
2. Asks whether a compound index needs updating.
3. Confirms the migration is backwards-compatible.
4. Does **not** hardcode a MongoDB connection string.

---

### 2d. Keeping CLAUDE.md Effective

CLAUDE.md directives should be:
- **Conditional** — "If the change touches X, read Y." Unconditional directives to read all files on every task add noise and slow the agent down.
- **Specific** — Name the class, file, or section. "Read `.context/code.md` §Caching Decision Hierarchy" is actionable. "Read the codebase standards" is not.
- **Imperative** — Use "must", "never", "always". The agent treats these as hard constraints, not suggestions.

**When to update CLAUDE.md:**
- A new area of the codebase gets guardrails added to `.context/` → add a conditional directive pointing to the relevant section.
- An existing directive is being ignored by the agent → make it more specific or move the rule into `CLAUDE.md` directly as a one-liner (last resort for critical non-negotiables).
- A new MADR is accepted → add a directive: *"If the change touches `<Class>`, read `<madr-NNNN>` §AI/Code-Editor Notes first."*

### 2e. Verifying the Agent Is Following Guardrails

During an AI coding session, watch for these signals that guardrails are active:

| Signal | Means |
|--------|-------|
| Agent reads `.context/code.md` before writing code | CLAUDE.md directive is working |
| Agent mentions "Step 0" when a cache is proposed | Caching hierarchy is being applied |
| Agent adds `publishPromotionEviction` unprompted on mutation | MADR-0001 is being read |
| Agent flags `@Cacheable` on private method as a problem | Spring AOP guardrail is active |
| Agent blocks new Caffeine cache with cardinality > 1,000 | Step 2 guardrail is working |

If any of these are missing, ask: *"What does `.context/code.md` say about [topic]?"* — this forces the agent to re-read the relevant section.

---

## 3. File Map

| File | When to Read |
|------|-------------|
| [`.context/overview.md`](../../.context/overview.md) | **First stop for any task.** ADR registry, coding standards index, and task-based navigation: "I'm about to do X — which file do I need?" |
| [`.context/code.md`](../../.context/code.md) | **Every code change.** GC health, caching decision hierarchy (Step 0 → 1 → 2), Spring `@Cacheable` gotchas, logging, observability, MongoDB BO rules |
| [`.context/infra.md`](../../.context/infra.md) | **Any change touching Redis, MongoDB, RabbitMQ, or deployment config.** Submodule setup, shard manager, Redis Sentinel, ETL guardrails, Facets deployment |
| [`.context/adr/README.md`](../../.context/adr/README.md) | **Before writing a new MADR.** Naming conventions, status values, template, and rules on when an ADR is warranted |
| [`.context/adr/madr-0001-promotion-meta-caffeine-redis-l1-l2-cache.md`](../../.context/adr/madr-0001-promotion-meta-caffeine-redis-l1-l2-cache.md) | **Any change to PromotionMeta caching** — L1/L2 design, eviction path, key format, or cache size. Also read for understanding the current known L2 bug (inactive Redis layer). |

**Quick navigation rule** (from `overview.md`):

| What you are about to do | Read this first |
|--------------------------|----------------|
| Any code change | `.context/code.md` |
| Caching, Redis, MongoDB, RabbitMQ, or deployment | `.context/infra.md` |
| Promotion mutation path | MADR-0001 §AI/Code-Editor Notes |
| New cache anywhere in the codebase | `.context/code.md` §Caching Decision Hierarchy (Step 0 first) |
| Repeated entity lookups in a single request | `overview.md` §Task-based Navigation — use request-boundary pre-load, not shared cache |
| Writing a new architectural decision | `.context/adr/README.md` |

---

## 4. Use Cases: During Development

### UC-DEV-1 — Adding a New Cache

**Trigger:** You notice a method repeatedly fetching the same data from the DB or Redis.

**Read:** [`.context/code.md`](../../.context/code.md) §Caching Decision Hierarchy

**Rules:**
- **Step 0 — Design first (mandatory).** Ask: *why is this fetched more than once?* Solutions that eliminate the cache entirely:
  - Bulk-load all needed entities at the service entry point; pass as `Map<K, V>` through the call chain
  - Use a `@RequestScope` Spring bean to hold within-request state
  - Make the processor stateless; restructure the loop to avoid repeated lookups
- **Step 1 — Redis `@Cacheable` (default).** If the data must survive across requests or pods, use Redis. Multi-tenant cardinality is unbounded; a local Caffeine cache in this scenario will churn under load and add GC pressure with zero latency benefit.
- **Step 2 — Caffeine L1 (last resort).** Only valid when ALL three conditions hold:
  1. Cardinality is provably bounded — not driven by org count or entities-per-org
  2. Sized as a hot-spot absorber: ≤ 1,000 entries, 5–10 min TTL
  3. Observability in place: `recordStats()` called, `CaffeineCacheMetrics.monitor` registered

> If you need > 1,000 entries for an acceptable hit rate → use Redis, not Caffeine.

---

### UC-DEV-2 — Writing a Promotion Mutation (Create / Update / Activate / Deactivate)

**Trigger:** Any write path that modifies a promotion entity.

**Read:** [MADR-0001](../../.context/adr/madr-0001-promotion-meta-caffeine-redis-l1-l2-cache.md) §AI/Code-Editor Notes (items 3–4)

**Rules:**
- **Always** call `CacheEvictionPublisher.publishPromotionEviction(orgId, promotionIds)` after the DB write.
- **Never** call `PromotionMetaCacheService.invalidateCache` or `RedisCacheUtil.evictPromotionMetaCache` directly — they bypass the pub/sub broadcast that invalidates L1 on all other pods.
- `promotionIds` must be a non-empty list of specific IDs. Passing `null` or an empty list logs an error and does nothing — L1 caches on all pods will remain stale for up to 1 hour.

```java
// CORRECT
cacheEvictionPublisher.publishPromotionEviction(orgId, List.of(promotionId));

// WRONG — bypasses cross-pod invalidation
promotionMetaCacheService.invalidateCache(orgId, promotionId);
```

---

### UC-DEV-3 — Using `@Cacheable` or `@CacheEvict`

**Trigger:** Adding a Spring cache annotation to any method.

**Read:** [`.context/code.md`](../../.context/code.md) §Cache Rules — *Spring AOP gotcha*

**Rules:**
- The annotated method **must be `public`** on a `@Service` bean.
- It **must be called via an injected bean reference**, not via `this.method()` (self-call).
- `@Cacheable` on a `private` method or a self-call is **silently bypassed by Spring AOP** — the cache is never populated, no error is thrown, and the bug is invisible until you check Redis and find it empty.

```java
// WRONG — private method, AOP proxy never intercepts it
@Cacheable("ONE_DAY_CACHE")
private PromotionMeta loadFromDb(String orgId, String id) { ... }

// CORRECT — public method on a separate @Service bean, called via injection
@Service
public class PromotionMetaDbLoader {
    @Cacheable("ONE_DAY_CACHE")
    public PromotionMeta load(String orgId, String id) { ... }
}
```

> This is the root cause of the current L2 Redis cache being inactive (MADR-0001 §Known bug, item 5). The fix is tracked separately.

---

### UC-DEV-4 — Reading PromotionMeta in a Hot Path (Multiple Promotions per Request)

**Trigger:** A service method needs `PromotionMeta` for several promotions within a single request.

**Read:** [MADR-0001](../../.context/adr/madr-0001-promotion-meta-caffeine-redis-l1-l2-cache.md) §AI/Code-Editor Notes item 1; [`.context/overview.md`](../../.context/overview.md) §Task-based Navigation

**Rules:**
- Bulk-load all needed `PromotionMeta` objects **once** at the service entry point using `preWarmCacheWithPromotionIds` or a direct bulk fetch.
- Pass the result as `Map<String, PromotionMeta>` through the call chain as a method parameter.
- **Do not** call `PromotionMetaCacheService.getPromotionMeta(...)` inside a loop — each call is a separate cache lookup (and potentially a separate MongoDB read on miss).

```java
// CORRECT — bulk load once, thread the map through
Map<String, PromotionMeta> metaMap = promotionMetaCacheService
    .preWarmCacheWithPromotionIds(orgId, promotionIds);
processPromotions(orgId, promotionIds, metaMap);

// WRONG — repeated cache lookups inside a loop
for (String id : promotionIds) {
    PromotionMeta meta = promotionMetaCacheService.getPromotionMeta(orgId, id); // N round-trips
    process(meta);
}
```

- **Never store** a `PromotionMeta` reference in another object's field. Always look it up at the point of use — a held reference silently serves stale data after eviction.
- **Never mutate** a `PromotionMeta` object or any of its internal attribute objects — the cached instance is shared across all concurrent requests on the pod.

---

### UC-DEV-5 — Creating or Updating a MongoDB BO Class

**Trigger:** New collection, new field on an existing BO, or schema migration.

**Read:** [`.context/code.md`](../../.context/code.md) §MongoDB / BO Classes; [`.context/infra.md`](../../.context/infra.md) §MongoDB

**Rules:**
- Set `lastUpdatedOn` on **every write** to any BO that has it — ETL delta loads are driven by this field; missing it silently drops records from downstream analytics.
- Declare compound indexes via `@CompoundIndexes` on the BO class; apply to all environments before or with the deploy.
- New access patterns requiring a new index must have that index added and backfilled before the query is deployed (full-table-scans at production scale under `orgId` compound indexes are expensive).
- **Never hardcode** a MongoDB connection string — always obtain the connection via Shard Manager dynamic resolution. The connection string is org-specific and resolved at startup/on-demand.
- Schema changes must be **backwards-compatible** (rolling deployments; old and new code run simultaneously during rollout).
- `IAttributionMigration` methods marked `@Deprecated(forRemoval = true)` are migration scaffolding — do not add new callers.

---

### UC-DEV-6 — Adding Observability to a New Service or Cache

**Trigger:** New service method, new Caffeine cache, or new controller endpoint.

**Read:** [`.context/code.md`](../../.context/code.md) §Observability

**Rules:**
- Add `@Trace(metricName = "Custom/<ServiceName>/<Operation>")` to cache read and eviction handling methods.
- Write logs to **stdout only**; always include `orgId` in the log line (or in MDC) for Loki filterability.
- Register every new Caffeine cache: `CaffeineCacheMetrics.monitor(meterRegistry, cache, "<cacheName>")`.
- **Do not** log at `INFO` level inside per-request hot paths — use `DEBUG` or `TRACE` for cache hit/miss events.

---

### UC-DEV-7 — Touching Deployment Config or Adding Environment Variables

**Trigger:** Changes to the service baseline JSON, Redis Helm config, or new Spring properties.

**Read:** [`.context/infra.md`](../../.context/infra.md) §Critical Guardrails; §Deployment Config

**Rules:**
- `service/instances/promotion-engine-a.json` (in `cc-stack-crm` submodule) propagates to **all clusters**. For cluster-specific tuning, use Deployer UI overrides — do not modify the baseline file for a single cluster's need.
- New env vars with production defaults are fine. Do **not** modify existing `spring.*`, `server.*`, or third-party library properties without explicit team review.
- The `cc-stack-crm` submodule is not auto-populated in Git worktrees — run `git submodule update --init src/test/resources/cc-stack-crm` in each new worktree before running tests that depend on infra config.

---

### UC-DEV-8 — Writing a New Architectural Decision Record (MADR)

**Trigger:** A non-obvious architectural trade-off is being made; the decision constrains how future code must be written; the reasoning would not be recoverable from the code alone.

**Read:** [`.context/adr/README.md`](../../.context/adr/README.md)

**Rules:**
- Write the MADR **after** the decision is validated and the team agrees — not during exploration.
- File name: `madr-NNNN-short-kebab-title.md` (increment from highest existing number).
- Register the new MADR in the table in [`.context/overview.md`](../../.context/overview.md).
- Never delete a MADR — mark it `Superseded by MADR-XXXX` and keep the file.
- Guardrails declared in the MADR apply immediately once status is `Accepted`.

**Do NOT write a MADR for:** routine refactors, bug fixes, implementation details that don't constrain future code.

---

## 5. Use Cases: During Code Review

For each scenario below, the listed `.context/` file is the authoritative source for the review question. Open it, find the relevant section, and use it to frame your comment.

---

### UC-CR-1 — PR Adds a New Cache or `@Cacheable` Annotation

**Check:** [`.context/code.md`](../../.context/code.md) §Caching Decision Hierarchy

**Review questions:**
1. Did the author document why Step 0 (design pattern) couldn't eliminate the cache? If not, request justification.
2. If Caffeine L1: is cardinality provably bounded (not driven by org count)? Is `maximumSize` ≤ 1,000? Is TTL 5–10 min?
3. Is `maximumSize` **and** TTL set on every Caffeine cache? (Unbounded cache = OOM risk under multi-tenant load.)
4. Is `recordStats()` called? Is `CaffeineCacheMetrics.monitor(meterRegistry, cache, "<name>")` registered?
5. Is `@Cacheable` on a `public` method of a separate `@Service` bean, not a private method or self-call?

---

### UC-CR-2 — PR Contains a Promotion Mutation

**Check:** [MADR-0001](../../.context/adr/madr-0001-promotion-meta-caffeine-redis-l1-l2-cache.md) §Guardrails & Follow-ups item 1

**Review questions:**
1. Does every mutation code path call `CacheEvictionPublisher.publishPromotionEviction(orgId, promotionIds)`?
2. Is `promotionIds` always a non-empty list? (Empty/null = silent no-op; all pods serve stale data for up to 1 hour.)
3. Is `invalidateCache` or `evictPromotionMetaCache` called anywhere directly? If yes, **block the PR** — this bypasses cross-pod pub/sub invalidation.

---

### UC-CR-3 — PR Adds a New Field or Collection to MongoDB

**Check:** [`.context/code.md`](../../.context/code.md) §MongoDB / BO Classes; [`.context/infra.md`](../../.context/infra.md) §MongoDB

**Review questions:**
1. Is `lastUpdatedOn` set on every write path for BOs that have it? (Missed writes silently corrupt ETL delta loads.)
2. Is a compound index declared via `@CompoundIndexes`? Is `orgId` part of the compound key?
3. Is the migration backwards-compatible for rolling deployments?
4. If the new field will be included in the ETL dimension table: is it a scalar? One-to-many arrays or `attribution` fields cannot be exported without a separate ETL pipeline change.
5. Is the new access pattern's index applied to all environments?

---

### UC-CR-4 — PR Adds a New Service Method or Controller Endpoint

**Check:** [`.context/code.md`](../../.context/code.md) §Observability; §GC Health

**Review questions:**
1. Is `orgId` present in MDC or in every log line emitted from this path? (Required for Loki filterability.)
2. Is `@Trace(metricName = "Custom/...")` added to the new method?
3. Are collections pre-sized with expected capacity? (Prevents resize allocations in hot paths.)
4. Is any request-scoped data being stored in a `static` field or application-scoped bean? (ThreadLocal must be cleaned up after the request.)
5. Are there any `INFO`-level log statements inside per-request loops? (Use `DEBUG`/`TRACE` instead.)

---

### UC-CR-5 — PR Introduces a New Tenant-Scoped Operation

**Check:** [`.context/infra.md`](../../.context/infra.md) §MongoDB; [`.context/code.md`](../../.context/code.md) §GC Health

**Review questions:**
1. Are all DB queries filtered by `orgId`? Is `orgId` part of every compound index that backs those queries?
2. Are cache keys org-scoped? (A cache key missing `orgId` will bleed data across tenants.)
3. If a scheduled job: is it scoped to a single tenant per execution? Can two runs overlap for the same tenant?
4. Is there any shared static or application-scoped state that could accumulate data across org boundaries?

---

### UC-CR-6 — PR Touches Deployment or Infrastructure Config

**Check:** [`.context/infra.md`](../../.context/infra.md) §Critical Guardrails; §Deployment Config

**Review questions:**
1. Does the change modify `promotion-engine-a.json`? If so: is this intentionally for *all clusters*, or should it be a Deployer UI override for a specific cluster?
2. Are any existing `spring.*`, `server.*`, or third-party library properties being modified without team review?
3. Are new Redis connection pool settings or `SimpleAsyncTaskExecutor` limits being changed? Verify against the infra constraints in `infra.md §Redis Sentinel`.

---

## 6. How to Add a New Guardrail or MADR

### When to add a rule to `.context/code.md` or `.context/infra.md`

Add a guardrail line when:
- A pattern recurs across multiple PRs (you've commented the same thing twice).
- A mistake is hard to detect at review time (e.g., silent Spring AOP bypass).
- The rule is actionable in one sentence.

Simply open the relevant file and add under the appropriate section. No MADR needed.

### When to write a full MADR

Write a MADR when:
- The decision involves real trade-offs between alternatives (not obvious right answer).
- The decision *constrains* how future code must be written — i.e., teams need context to not undo it.
- The reasoning would not be recoverable from the code alone.

See [`.context/adr/README.md`](../../.context/adr/README.md) for the template and full authoring rules.

### What does NOT need a MADR
Routine refactors, bug fixes, library upgrades, one-off config changes.

---

## 7. Security Considerations for `.context/`

The `.context/` folder occupies an unusual security position: it is read by both human developers and AI coding agents as **trusted input**. A careless or malicious change to any file in this folder can silently alter how the AI agent behaves for everyone on the team. The considerations below must be assessed whenever a new file is added, an existing file is updated, or the folder structure is extended.

---

### 7a. What Must Never Appear in `.context/` Files

`.context/` files are committed to the repository and read verbatim by AI agents. They must never contain:

| Category | Examples | Why |
|----------|----------|-----|
| Credentials / secrets | Passwords, API keys, bearer tokens, service account keys | Committed to git history; read by agents and potentially surfaced in agent output |
| Production hostnames / IPs | Specific Redis Sentinel master hostnames, MongoDB shard IP addresses, internal DNS names | Reveals production topology; creates attack surface if repo is ever shared externally |
| Hardcoded connection strings | `mongodb://user:pass@host:port/db` | Violates the infra guardrail in `infra.md` and risks credential exposure |
| Real `orgId` / customer data | Production org IDs used as examples, customer promotion IDs | PII / data privacy risk; agents may echo these in generated code or logs |
| Vulnerability details beyond what's needed | Specific exploit paths, unpatched CVE details | MADR-0001 already documents known bugs at a design level — do not add exploit-level detail |

**Current state of this repo:** `infra.md` references the Redis pub/sub channel name (`promotion_eviction`), MongoDB shard naming convention (`emf`, `emf-2`…), and the `cc-stack-crm` submodule path. These are acceptable for an **internal private repo** but would require audit before any external sharing or open-sourcing.

---

### 7b. Prompt Injection Risk

AI agents treat `CLAUDE.md` and `.context/` files as **trusted system instructions** — they carry the same authority as a human prompt. This creates a prompt injection surface:

**Attack vector:** A malicious or careless commit to `CLAUDE.md` or any `.context/` file could instruct the agent to:
- Introduce a deliberate vulnerability (e.g., remove an `orgId` filter from a DB query)
- Bypass a guardrail ("ignore the caching decision hierarchy for this class")
- Grant itself wider permissions (e.g., push to main without review)
- Silently remove a critical check from generated code

**Mitigations:**
- Treat every PR that touches `CLAUDE.md` or any `.context/` file as a **security-sensitive change** — require review from a senior engineer who understands the AI agent's trust model.
- Do not merge `.context/` changes via auto-merge or bot accounts.
- Review the *intent and effect* of each directive change, not just the text diff. Ask: *"What would the agent now do differently?"*

---

### 7c. Change Control and Audit Trail

Because `.context/` and `CLAUDE.md` changes affect AI agent behaviour for the entire team:

- **Every change must go through a PR** — no direct commits to `main`.
- **PR description must state** what guardrail changed, why it changed, and what the agent will now do differently.
- **At least one reviewer** must be someone who has used Claude Code on this repo and understands the agent trust model.
- Changes that *remove or weaken* an existing guardrail require a stronger justification than changes that add one.

**Versioning note:** Because git history is the audit trail for `.context/` changes, PR descriptions carry the "why" that the files themselves should not — keep files directive and concise; put rationale in the PR.

---

### 7d. AI Agent Permissions Boundary

`CLAUDE.md` must not — intentionally or accidentally — instruct the agent to take autonomous destructive or externally-visible actions without explicit human approval. Review any new `CLAUDE.md` directive against this checklist:

| Action category | Policy |
|-----------------|--------|
| Read files, grep, explore codebase | Safe — no approval needed |
| Edit source files, write tests | Safe — user reviews before commit |
| Run tests, build | Safe — output is local |
| Commit to git | Requires explicit user instruction per commit |
| Push to remote / create PRs | Must never be automatic; always require explicit user approval |
| Delete branches, reset hard | Must never be in a CLAUDE.md directive |
| Modify CI/CD pipelines | Must never be in a CLAUDE.md directive |
| Call external APIs / send messages | Must never be in a CLAUDE.md directive |

If a proposed `CLAUDE.md` directive would cause the agent to take any action in the lower half of this table autonomously, **reject it**.

---

### 7e. Information Disclosure Classification

Assess every new `.context/` file or MADR against the following before merging:

| Content type | Internal repo | External / public repo |
|-------------|--------------|----------------------|
| Coding guardrails and design patterns | OK | OK (no sensitive data) |
| Cache key patterns (e.g., `promo_meta_{orgId}_{promotionId}`) | OK | Review — reveals data model |
| Internal topology (shard names, Redis channel names) | OK | **Audit required before sharing** |
| Known bugs / inactive code paths (e.g., L2 not active) | OK — internal risk register | **Must not disclose publicly** — reveals exploitable state |
| Deployment cluster names, Helm chart versions | OK | **Audit required before sharing** |
| Connection strings, credentials | **Never** | **Never** |

**Action if repo visibility changes:** Before making this repo public or sharing with a third party, audit `infra.md` and all MADRs against the "External" column above and redact or generalise as needed.

---

### 7f. Sensitive Patterns in Generated Code

AI agents informed by `.context/` files may inadvertently surface sensitive patterns in generated code or log output. Watch for:

- Agent generating a cache key that embeds a real `orgId` value (picked up from a `.context/` example)
- Agent echoing a production hostname from `infra.md` in a generated config snippet
- Agent including a known-bug description from a MADR in a code comment visible in production logs

If any of these are observed, the relevant `.context/` file should be edited to remove or generalise the sensitive value and the generated code must be reviewed before commit.

---

## 8. Known Open Issues in `.context/`

| Issue | File | Detail |
|-------|------|--------|
| L2 Redis cache inactive | MADR-0001 §item 5 | `@Cacheable` on private `loadPromotionMetaFromDatabase` is bypassed by Spring AOP — Redis L2 never populated. Fix: extract to a new public `@Service` bean. Pre-requisite: `PromotionMeta` must implement `Serializable`. |
| L1 cardinality risk | MADR-0001 §item 8 | L1 max size is 30,000 entries (set before multi-tenant cardinality was fully understood). If steady-state evictions are observed in production → right-size to ≤ 1,000 entries or remove L1 entirely. Do not raise the cap. |

---

## 9. Quick Reference — Guardrail Violation Checklist

Use this as a final gate before marking a PR approved:

- [ ] No `@Cacheable` / `@CacheEvict` on private methods or self-calls
- [ ] Every promotion mutation calls `CacheEvictionPublisher.publishPromotionEviction` with non-empty `promotionIds`
- [ ] New Caffeine caches: `maximumSize` set, TTL set, `recordStats()` + `CaffeineCacheMetrics.monitor` registered
- [ ] New caches justified through Step 0 (design pattern considered)
- [ ] `lastUpdatedOn` set on every write to BO classes that carry it
- [ ] All MongoDB queries scoped to `orgId`; compound indexes include `orgId`
- [ ] No hardcoded MongoDB connection strings
- [ ] `@Trace` added to new service / cache methods; New Relic metric name follows `Custom/<Service>/<Op>`
- [ ] No INFO-level logs in per-request hot paths
- [ ] Request-scoped state not stored in `static` fields; ThreadLocals cleaned up after request
- [ ] Deployment config changes intended for all clusters (not sneaking a cluster-specific change into the baseline)

**`.context/` / CLAUDE.md changes (additional gate):**
- [ ] No credentials, hostnames, connection strings, or real `orgId` values in `.context/` files
- [ ] Change went through a PR with rationale in the PR description — no direct commits to main
- [ ] PR reviewed by a senior engineer familiar with the AI agent trust model
- [ ] New `CLAUDE.md` directive does not grant the agent autonomous push, delete, or CI/CD actions
- [ ] If repo visibility could change: new file audited against information disclosure classification (§7e)
