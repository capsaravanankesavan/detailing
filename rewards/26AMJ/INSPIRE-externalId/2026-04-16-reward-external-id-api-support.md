# Investigation: Reward externalId API Support
**Date:** 2026-04-16  
**Status:** Ready for Tech Detail  
**Confidence:** HIGH

---

## Problem Statement

Brands want to reference rewards using their own identifier (`externalId`) instead of the internal `id` we generate. The API must support both identifiers without a version bump. Brand-related path segments (`/brand/{brandId}`, `/brand/{brandName}`) are deprecated and must not appear in new paths. The gateway must be able to distinguish calls using `externalId` from calls using the internal `rewardId` for metrics and observability. `externalId` is unique within an org (tenant) — not globally unique.

---

## Root Cause / Design Decision

No `externalId` field exists on the `Reward` entity today. The v1 filter (`RequestFilterV1`) validates brand from the URI for most paths, but already has an established bypass mechanism (`APIS_WITHOUT_BRAND_IDENTIFIER` + inline regex conditions at line 93–101) that skips brand validation for specific paths and commits directly after org authentication. Extending this bypass with a single condition (`uri.contains("/external/")`) is sufficient to support brand-free externalId paths within v1 — no new version or filter needed.

### Why Not a New Version (v2)

Adding `externalId` variants is purely additive — no existing contract changes. A new version is only warranted for breaking changes. The v1 filter's existing bypass pattern was designed exactly for this scenario.

### Why Not v1.1

v1.1 only covers user-reward operations (`UserRewardResourceV1x1`). The catalog/admin operations (`RewardResource`) have no v1.1 equivalent. Extending v1.1 would be incomplete and inconsistent.

---

## Evidence Trail

| File | Line | Finding |
|------|------|---------|
| `RequestFilterV1.java` | 93–101 | Brand bypass block — already skips validation for matched URI patterns; extend with `/external/` |
| `RequestFilterV1.java` | 63–78 | `orgId` extracted from header and set on `AuthDetails` before the bypass check — auth is already complete |
| `URLPrefix.java` | 70–71 | `APIS_WITHOUT_BRAND_IDENTIFIER` list — established pattern for brand-free paths in v1 |
| `Reward.java` | 37 | Internal `id` (Long) is the only identifier today — no `externalId` field |
| `UserRewardResource.java` | 146–157 | `POST /reward/{rewardId}/issue` already has no brand in path — already on bypass |
| `URLPrefix.java` | 12–13 | Only `v1` and `v1.1` exist today — no v2 |

---

## System Map (Affected Path)

**Call chain:** Gateway → `RequestFilterV1` → JAX-RS dispatcher → `RewardResource` / `UserRewardResource` → `RewardFacade` / `UserRewardFacade` → `RewardRepository` → `TBL_REWARD`

**Tenant isolation points:** `orgId` from `X-CAP-API-AUTH-ORG-ID` header, set at `RequestFilterV1:78`. All lookups must scope by `orgId`. `externalId` is unique per `(orgId, externalId)` — composite key.

**Upstreams called:** Intouch service (for customer/points operations — not affected by this change)

**Downstreams affected:** None — this is a read/write path on the reward catalog and issuance. No downstream event contracts change.

---

## New URL Surface

### Reward Catalog — `RewardResource` (`/core/v1/reward/`)

| Operation | Existing (keep) | New — internal ID (clean) | New — externalId |
|-----------|----------------|--------------------------|-----------------|
| Get reward | `GET /{id}/brand/{brandId}` | `GET /{rewardId}` | `GET /external/{externalId}` |
| Update reward | `PUT /{id}/brand/{brandId}` | `PUT /{rewardId}` | `PUT /external/{externalId}` |
| Set status | `PUT /{id}/status/{status}/brand/{brandId}` | `PUT /{rewardId}/status/{status}` | `PUT /external/{externalId}/status/{status}` |
| Create reward | `POST /create` | `POST /create` — add `externalId` to request body | — |
| List rewards | `GET /list/brand/{brandId}` | `GET /list` | — |
| Available rewards | `GET /brand/{brandId}/available` | `GET /available` | — |
| Disable constraint | `PUT /{id}/constraint/{cId}/disable` | unchanged — already on bypass | — |

### User Rewards — `UserRewardResource` (`/core/v1/user/`)

| Operation | Existing (keep) | New — internal ID (clean) | New — externalId |
|-----------|----------------|--------------------------|-----------------|
| Get reward details | `GET /reward/{rewardId}/brand/{brandName}` | `GET /reward/{rewardId}` | `GET /reward/external/{externalId}` |
| Get reward catalog | `GET /reward/brand/{brandName}` | `GET /rewards` | — |
| Get vouchers (mobile) | `GET /{mobile}/reward/{rewardId}/vouchers/brand/{brandName}` | `GET /{mobile}/reward/{rewardId}/vouchers` | `GET /{mobile}/reward/external/{externalId}/vouchers` |
| Get vouchers (identifier) | `GET /reward/{rewardId}/vouchers/brand/{brandName}` | `GET /reward/{rewardId}/vouchers` | `GET /reward/external/{externalId}/vouchers` |
| Issue reward | `POST /reward/{rewardId}/issue` | unchanged — no brand today | `POST /reward/external/{externalId}/issue` |
| Bulk issue | `POST /rewards/issue` | unchanged | Add `externalId` as optional field in `RewardIssueRequest` body alongside `rewardId` |
| User reward list | `GET /userReward/brand/{brandName}` | `GET /userReward` | — |

> **JAX-RS routing note:** JAX-RS resolves literal path segments (`external`) with higher specificity than path params (`{rewardId}`). No collision between `GET /reward/{rewardId}` and `GET /reward/external/{externalId}`.

---

## Solution: What Changes

### 1. `RequestFilterV1.java` — 2 lines
Add `/external/` bypass to the existing brand-skip block (line 93–101):
```java
|| httpServletRequest.getRequestURI().contains("/external/")
```

Add MDC + New Relic tagging immediately after line 75 (after orgId MDC is set):
```java
boolean isExternalIdCall = httpServletRequest.getRequestURI().contains("/external/");
MDC.put("reward_id_type", isExternalIdCall ? "external" : "internal");
metricsService.addCustomParameter("reward_id_type", isExternalIdCall ? "external" : "internal");
```

### 2. `URLPrefix.java` — 1 line
```java
public static final String EXTERNAL = "external";
```

### 3. DB Migration — `TBL_REWARD`
```sql
ALTER TABLE TBL_REWARD ADD COLUMN external_id VARCHAR(255);

-- Partial unique index: only enforces uniqueness for rows where external_id is set.
-- Existing rewards without externalId are unaffected.
CREATE UNIQUE INDEX idx_reward_org_external_id
    ON TBL_REWARD (org_id, external_id)
    WHERE external_id IS NOT NULL;
```

### 4. `Reward.java` — entity field
```java
@Column(name = "external_id")
private String externalId;
```

### 5. `RewardRepository` — 1 method
```java
Optional<Reward> findByOrgIdAndExternalId(Long orgId, String externalId);
```

### 6. `RewardFacade` — resolver helper
Add a private resolver that all externalId operations route through. This resolves the `externalId` to the internal `Reward` and then delegates to the existing business logic unchanged:
```java
private Reward resolveByExternalId(String externalId, Long orgId) {
    return rewardRepository.findByOrgIdAndExternalId(orgId, externalId)
        .orElseThrow(() -> new MarvelException(StatusCode.REWARD_NOT_FOUND));
}
```

### 7. `RewardResource.java` — new endpoint methods
Add new `@Path` methods for:
- `GET /{rewardId}` (clean internal, no brand)
- `GET /external/{externalId}`
- `PUT /{rewardId}` (clean internal)
- `PUT /external/{externalId}`
- `PUT /{rewardId}/status/{status}`
- `PUT /external/{externalId}/status/{status}`
- `GET /list` (no brand)
- `GET /available` (no brand)

### 8. `UserRewardResource.java` — new endpoint methods
Add new `@Path` methods for:
- `GET /reward/{rewardId}` (clean, no brand)
- `GET /reward/external/{externalId}`
- `GET /reward/{rewardId}/vouchers` (clean)
- `GET /reward/external/{externalId}/vouchers`
- `GET /{mobile}/reward/{rewardId}/vouchers` (clean)
- `GET /{mobile}/reward/external/{externalId}/vouchers`
- `POST /reward/external/{externalId}/issue`
- `GET /rewards` (clean catalog, no brand)
- `GET /userReward` (clean, no brand)

### 9. `RewardIssueRequest` (bulk issue body)
Add optional `externalId` field. Service resolves whichever is present — `rewardId` takes priority if both are supplied, `externalId` used as fallback. Log a warning if both are present.

---

## What Does NOT Change (Explicit Out of Scope)

- All existing v1 brand-path URLs — keep working, no deprecation in this change
- `RequestFilterV1_1` and all v1.1 endpoints — untouched
- `BrandResource` — no externalId concept applies to brands
- `TransactionResource` / `TransactionResourceV1x1` — transactions reference `rewardTransactionId`, not `rewardId`; out of scope for this change
- `VendorResource` — unrelated
- Intouch/downstream service contracts — unchanged

---

## Assumptions Going Into Tech Detail

1. **`externalId` is set at reward creation time** — breaks if: brands expect to back-fill externalId on existing rewards (need a separate PATCH endpoint if so)
2. **`externalId` is immutable after creation** — breaks if: brands need to rename their IDs (requires update logic and cache invalidation)
3. **orgId is always present in `X-CAP-API-AUTH-ORG-ID` header for new paths** — breaks if: any caller omits it (existing filter already rejects this at line 69–72)
4. **Partial unique index supported by DB engine in use** — verify MySQL/Aurora version supports `WHERE` clause on index

---

## Risks for Implementer

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| JAX-RS path collision between `/{rewardId}` and `/external/{externalId}` | Low | JAX-RS spec §3.7.2: literals beat path params in matching priority |
| Existing callers sending both `rewardId` and `externalId` in bulk issue body | Medium | Define explicit precedence rule (`rewardId` wins); log warning |
| `externalId` back-fill needed for existing rewards | Medium | Confirm with brands — if yes, add a `PUT /reward/{rewardId}/external-id` endpoint |
| MySQL partial index syntax varies by version | Low | Test migration on target DB version before deploy |
| `APIS_WITHOUT_BRAND_IDENTIFIER` bypass covers too broadly if `/external/` appears in non-reward URIs | Low | Audit other resource paths — none currently use the word "external" |

---

## Open Questions (Must Resolve Before Build)

- [ ] Should `externalId` be updatable after creation, or immutable? — owner: Product
- [ ] For bulk issue (`POST /rewards/issue`): are `rewardId` and `externalId` mutually exclusive per line item, or can both be present? — owner: Product
- [ ] Do brands need to back-fill `externalId` on existing rewards? If yes, scope a separate update endpoint — owner: Product
- [ ] Confirm DB engine version supports partial unique index (`WHERE external_id IS NOT NULL`) — owner: Infra

---

## Gateway Tracking

Two complementary signals — both available independently:

| Signal | Where configured | What it captures |
|--------|-----------------|-----------------|
| Route match on `/external/` in path | API Gateway routing rules | All externalId calls, zero service dependency |
| `reward_id_type` MDC key in logs | `RequestFilterV1` (new 2-line addition) | Per-request log correlation |
| `reward_id_type` New Relic custom attribute | `RequestFilterV1` (same 2-line addition) | APM dashboard dimension for split metrics |

Gateway route patterns:
```
/core/v1/**/external/**   →  reward_id_type = external
/core/v1/**               →  reward_id_type = internal  (default)
```

---

## What to Watch Post-Deploy

- `reward_id_type=external` call volume ramps up as brands migrate
- `reward_id_type=internal` (brand-path) call volume trends toward zero as brands cut over
- `REWARD_NOT_FOUND` error rate on `/external/` paths — indicates mismatched externalId values between brand system and ours
- No regression on existing brand-path (`/brand/{brandId}`) error rates

---

## Handoff Note for tech-detailer

Root cause / design decision is confirmed. The fix is bounded to:
- `RequestFilterV1.java` — 2-line addition to bypass block + MDC tagging
- `URLPrefix.java` — 1 constant
- `Reward.java` — 1 field
- `RewardRepository` — 1 method
- `RewardFacade` — 1 resolver helper + wire-up in new facade methods
- `RewardResource.java` — new `@Path` methods (no changes to existing methods)
- `UserRewardResource.java` — new `@Path` methods (no changes to existing methods)
- `RewardIssueRequest` — 1 optional field
- DB migration — 1 column + 1 partial index

Key risks to explore during tech detail:
- JAX-RS path resolution order for `/{rewardId}` vs `/external/{externalId}` — verify with a focused integration test
- Bulk issue body contract — nail down `rewardId` vs `externalId` precedence rule before writing service logic

Contracts to validate:
- `RewardFacade.getReward(rewardId, brandId)` — the externalId path resolves to internal `Reward` then calls this; confirm `brandId` param can be sourced from `authDetails.getOrgId()` → `brandService.getByOrgId()` instead of path

Suggested test coverage emphasis:
- **Unit:** `resolveByExternalId()` — not-found case, wrong org case
- **Integration:** JAX-RS routing — literal `external` segment vs numeric `{rewardId}` — both resolve to correct handlers
- **Tenant isolation:** `GET /core/v1/reward/external/PROMO_1` with orgId=A must not return reward owned by orgId=B even if both have `externalId=PROMO_1`
