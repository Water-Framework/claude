---
name: multitenancy-knowledge
description: Knowledge base for Water Framework Company-based multitenancy — tenant markers (TenantResource / MultiTenantResource), companyId JWT claim, login membership gate, user-level impersonation, tenant enforcement in the Api layer, the non-scoped admin model, module isolation, and the enablement/lenient rules. Use when designing, implementing, reviewing, testing, or answering any question about multitenancy, tenants, companies, tenant scoping, or impersonation in Water modules.
allowed-tools: Read, Glob, Grep
---

You are an expert on **Water Framework Company-based multitenancy**. You use this knowledge to answer questions and guide correct design/implementation/testing of tenant-aware modules. This skill is the authoritative reference for the multitenancy design, rules, and implementation status; when you need ground truth, verify against the current code at the file anchors listed in §14.

---

# WATER FRAMEWORK — COMPANY-BASED MULTITENANCY

## 0. One-paragraph model
Tenancy is added **additively and opt-in**: entities opt in via marker interfaces, the active company (tenant) travels as an **encrypted JWT claim** minted once at login, and enforcement happens in the **Api service layer**. The whole mechanism is **lenient / backward compatible**: with no active company the framework behaves exactly like single-tenant. `Company` is a separate microservice; downstream services never call it per-request — they read the tenant from the token. `companyId` is always an opaque `Long`, never a JPA relation (module isolation).

## Table of Contents
1. Enablement & the lenient rule
2. Entity model & tenant markers
3. SecurityContext & JWT claims
4. Login membership gate
5. User-level impersonation
6. Tenant enforcement (the seam)
7. Admin model
8. Module isolation
9. Data model
10. Distributed enforcement (trust boundary)
11. Properties
12. Testing multitenancy
13. Implementation status (done vs deferred)
14. Key files & anchors
15. Gotchas
16. FAQ

---

## 1. Enablement & the lenient rule

- **Enablement is authoritative on the token ISSUER** (Authentication), via `water.authentication.multitenant.enabled` (default `false`). "MT enabled" ⇔ login embeds the `companyId` claim. There is NO per-resource-service "enable" flag.
- **Lenient / backward compatible**: every tenant filter/check applies **only when `SecurityContext.getActiveCompanyId() != null`**. Null (MT off, non-scoped admin, legacy token) → no filtering, no deny → identical to single-tenant. Existing tests/deployments are unaffected.
- **No `isAdmin()` special-casing in the tenant dimension** (unlike the ownership filter, which admins bypass): admin scoping derives purely from whether a company is active.

## 2. Entity model & tenant markers (`it.water.core.api.entity.tenant`)

- **`TenantResource extends BaseEntity`** — single-company entity. Carries a nullable opaque `Long companyId` (constant `COMPANY_ID_FIELD_NAME = "companyId"`). `null` = global / cross-tenant instance (e.g. a global role). Column-based.
- **`MultiTenantResource extends BaseEntity`** — empty marker for M:N tenancy (an instance belongs to several companies, e.g. `WaterUser`). NO column; membership lives in a domain table and is resolved by a `TenantMembershipResolver`.
- **Base `@MappedSuperclass` classes** (`JpaRepository-api`) carry the `companyId` column so column-based entities just extend them. The choice is dictated by expandability and MUST preserve it:
  - `AbstractJpaTenantEntity extends AbstractJpaEntity implements TenantResource` (non-expandable — e.g. `WaterRole`)
  - `AbstractJpaExpandableTenantEntity extends AbstractJpaExpandableEntity implements TenantResource` (expandable — e.g. `Document`)
  - `companyId` is `@JsonIgnore` + PROPERTY access (server-assigned, not client-settable — mirrors `ownerUserId`). Field is duplicated across the two superclasses (single-inheritance; do NOT push it into `AbstractJpaEntity` — that would break opt-in).
- An entity can be **both** `OwnedResource`/`SharedEntity` AND `TenantResource` (e.g. `Document`) — it must satisfy BOTH the ownership and the tenant conditions.

## 3. SecurityContext & JWT claims

- `SecurityContext.getActiveCompanyId()` (nullable `Long`) — the SINGLE runtime carrier read by all enforcement. `getImpersonatedBy()` (nullable `String`) / `isImpersonated()` — impersonation marker. All additive `default` methods (return null/false) so no implementor breaks.
- Populated by the JWT filter: `NimbusJwtTokenService.getPrincipals` reads claims → `UserPrincipal` (nullable `companyId`, `impersonatedBy`) → `WaterAbstractSecurityContext` (fields populated in `setLoggedPrincipals`).
- Claims (`JWTConstants`): `JWT_CLAIM_COMPANY_ID = "companyId"`, `JWT_CLAIM_IMPERSONATED_BY = "impersonatedBy"`. `generateClaims` emits each ONLY when non-null → tokens without a company/impersonation are byte-identical to legacy. Tokens are JWE-encrypted → claims are not exposed to the client.

## 4. Login membership gate

- The company is chosen **once at login** and baked into the token; validation is the ONLY security gate (downstream trusts the claim).
- `AuthenticationProvider` gets an additive overload `login(username, password, Long companyId)` (default delegates to the 2-arg). `AuthenticationSystemServiceImpl.login` calls the 3-arg only when MT is enabled; otherwise the legacy 2-arg.
- The gate lives IN the provider (`UserAuthenticationProvider`, User domain — module isolation), not in Authentication core: after credential validation, `resolveActiveCompany(user, requested)`:
  - **admin → `null`** (non-scoped, cross-tenant; admins enter a tenant only via impersonation);
  - non-admin + requested `companyId` → must be in the user's `UserCompany` membership (`existsByUserAndCompany`), else `UnauthorizedException`;
  - non-admin + none → the `primary` company (may be null).
- REST login accepts an optional `@FormParam("companyId")` (JAX-RS) / `@RequestParam companyId` (Spring). Switching company = re-login/impersonate; there is NO per-request company header.

## 5. User-level impersonation

- Impersonation is **user-level** (not tenant-level): a caller mints a token acting AS a specific target user (the tenant comes transitively from that user).
- `AuthenticationProvider.impersonate(targetUsername, callerUsername, companyId)` (default throws `UnsupportedOperationException`; impl in `UserAuthenticationProvider`). `AuthenticationApi.impersonate(targetUsername, companyId)` (caller from `SecurityContext`). Endpoint `POST /water/authentication/impersonate` (authenticated).
- **Permission-gated** via the `UserActions.IMPERSONATE` action on `WaterUser` (checked with `PermissionManager.checkPermission(caller, WaterUser.class, IMPERSONATE)`): an **admin passes by construction**; a normal user only if that permission is granted (NOT granted to any default role). NOT a hardcoded `isAdmin()` check.
- The token carries the target's identity/company/roles + claim `impersonatedBy=<caller>` → surfaces as `SecurityContext.getImpersonatedBy()` for audit ("caller X acted as user Y"). Not MT-flag-gated; same TTL; no operational restrictions (audit-only).

## 6. Tenant enforcement (the seam)

Everything is in the **Api layer** (`BaseEntityServiceImpl`, `Repository-service`) + the permission manager — NEVER SystemApi (SystemApi bypasses all of it; used for trusted internal/enforcement machinery). Gated on `getActiveCompanyId() != null`:
- **save()**: auto-assigns `companyId` from the active company for `TenantResource`.
- **find/findAll/countAll**: `createConditionForTenantResource(...)` ANDs a tenant condition (independent of the owner/shared condition):
  - `TenantResource` → `companyId = active OR companyId IS NULL` (null = global visible cross-tenant).
  - `MultiTenantResource` → `id IN (ids)` where ids come from the module's `TenantMembershipResolver`; empty set → `id = -1` (zero rows).
- **update()** (+ `BaseJpaRepositoryImpl.doUpdate` defense-in-depth): restores `companyId` from the DB (no tenant-hijack).
- **by-id access** (`PermissionManagerDefault.checkUserOwnsResource` → `checkEntityBelongsToActiveTenant`): a `TenantResource`'s `companyId` must be null or == active; a `MultiTenantResource`'s id must be in the resolver set. Runs AFTER the `isAdmin()` short-circuit.
- **`TenantMembershipResolver`** (`it.water.core.api.service.integration`): `boolean supports(String entityResourceName)` + `Set<Long> getEntityIdsInCompany(String entityResourceName, long companyId)`. Contract in Core-api; impl in the entity's own module (e.g. `UserTenantMembershipResolver` queries `UserCompany`). Resolved from the `ComponentRegistry` by type (like `AuthenticationProvider` by issuer). Uses SystemApi internally to avoid recursion.

## 7. Admin model

- Admin is **non-scoped by default** (login → `null` company → cross-tenant, for provisioning: create companies, users, memberships). Enforcement: `null` + admin → no filter → cross-tenant; `null` + non-admin → (lenient) no filter today, strict mode would deny.
- Admin enters a specific tenant ONLY via **user-level impersonation** (§5). There is intentionally no god-view via a "super-admin sees all" flag — cross-tenant reach is provisioning (non-scoped) or impersonation.

## 8. Module isolation (hard constraint)
- Tenant management lives in the **module of the entity**, never in Company. Membership for `WaterUser` is `UserCompany` in `User-model` + its resolver in `User-service`. No central membership table.
- `companyId` is an opaque `Long` everywhere — never a JPA relationship to the `Company` entity. Company module is not a compile dependency of tenantized modules.

## 9. Data model
- **`UserCompany`** (`User-model`) — M:N membership `(userId, companyId, primary)`, unique `(userId, companyId)`, `companyId` opaque. `primary` = default company at login.
- **`WaterRole`** — `TenantResource` (companyId null = global role, else company-scoped).
- **`Document`** — `TenantResource` (auto-assigned company on save).
- **`WaterUser`** — `MultiTenantResource` (M:N via `UserCompany`).

## 10. Distributed enforcement (trust boundary)
- **Presence of the `companyId` claim is authoritative and obliges enforcement**, everywhere, on every tenantized entity. A service must NOT re-decide "am I multi-tenant" from local config; ignoring a present claim = tenant leak.
- The claim is trustworthy because the token is server-signed+encrypted; the only validation point is issuance (login/impersonate). At enablement, use short token TTL + forced re-login to drain legacy (claim-less) tokens.

## 11. Properties
- `water.authentication.multitenant.enabled` (Authentication, default `false`) — issuer-side enablement. Declared in `AuthenticationOption.isMultiTenantEnabled()`, in `it.water.application.properties`, and in the **Water Pins descriptor** (`waterDescriptor { properties { ... } }` in `Authentication-service/build.gradle` → `water-descriptor.json`; the Spring module inherits via `inheritsFrom`).

## 12. Testing multitenancy
- **Establish an active company** with the 4-arg overload: `runtime.fillSecurityContext(TestSecurityContext.createContext(id, username, isAdmin, companyId))` (Core-testing-utils). `impersonateAdmin`/`impersonate` produce a NULL company. Restore a clean context afterwards.
- Test the **filter** in isolation with an admin context that ALSO carries a `companyId` (filter has no `isAdmin()` special-casing). Test the **by-id gate** with a NON-admin context (admins short-circuit) + the needed permission.
- Seed cross-company data via **SystemApi** (bypasses auto-assign + filter, so you control each row's `companyId`); for M:N seed the membership rows. Assert on your own seeded ids (one shared H2 DB across all test classes).
- **Clean up seeded rows in `@AfterAll`** (e.g. by a unique path/uid prefix) — pollution can push a pagination-sensitive Karate `CONTAINS` assertion in a sibling class off its page.
- MT-ON behavioral tests catch bugs MT-OFF regression cannot. Always add: scoped visibility, admin/null cross-tenant, by-id cross-tenant deny, auto-assign, and one backward-compat (null company → unfiltered) case.

## 13. Implementation status
- ✅ **Done (4 tasselli)**: (1) foundations — markers, base classes, SecurityContext/UserPrincipal, JWT claim; (2) login MT — `UserCompany`, provider gate, issuer flag; (3) impersonation — `IMPERSONATE`, `impersonatedBy` claim, `/impersonate`; (4) enforcement — filter/auto-assign/restore, by-id gate, `Document`/`WaterRole`/`WaterUser` tenantized, resolver. All built + tested (regression MT-off + behavioral MT-on).
- ⏳ **Deferred**: company-aware role ASSIGNMENT (`WaterUserRole` companyId dimension) + roles-in-token resolution; granular per-entity opt-out (`water.multitenancy.<fqcn>.shared`, fail-closed); company-scoped branch of impersonation; dedicated Permission-manager by-id unit test; strict fail-closed mode.

## 14. Key files & anchors
- Markers: `Core-api/.../entity/tenant/{TenantResource,MultiTenantResource}.java`; base classes `JpaRepository-api/.../model/AbstractJpa[Expandable]TenantEntity.java`.
- Context: `Core-security/.../principal/UserPrincipal.java`, `.../context/WaterAbstractSecurityContext.java`; `Core-api/.../permission/SecurityContext.java`.
- Token: `Rest-security/.../jwt/{JWTConstants,NimbusJwtTokenService}.java`.
- Login/impersonation: `User-service/.../service/UserAuthenticationProvider.java`; `Authentication-service/.../service/AuthenticationSystemServiceImpl.java` + REST controllers.
- Enforcement: `Repository-service/.../service/BaseEntityServiceImpl.java` (`createConditionForTenantResource`, save/update); `JpaRepository-api/.../BaseJpaRepositoryImpl.java` (`doUpdate`); `Permission-manager/.../PermissionManagerDefault.java` (`checkEntityBelongsToActiveTenant`).
- Membership/resolver: `User-model/.../UserCompany.java`, `User-service/.../UserTenantMembershipResolver.java`.

## 15. Gotchas
- `field("id").in(list)` is **capped at 2 operands** (`IllegalArgumentException: Too much operands for operation!`). For `id IN (many)` build the STRING filter with server-controlled numeric ids: `qb.createQueryFilter("id IN (" + csv + ")")`.
- Spring submodules referencing a newly-tenantized entity may need an explicit `Core-api` dependency in `build.gradle` (the inherited marker type must be on the compile classpath).
- `yo water:add-entity` generates a per-entity persistence unit — in a module with an existing PU this deadlocks the shared HSQLDB test schema migration; align the `*RepositoryImpl` PU to the module's PU.
- The generated entity may use `javax.persistence`; switch to `jakarta.persistence` in jakarta modules.

## 16. FAQ
- **Does enabling MT break single-tenant?** No — lenient rule: no active company → no filtering. Default is off.
- **How does the frontend switch tenant?** Re-login (or impersonate) to get a new token; no client-side/per-request switch. The JWT is opaque to the frontend.
- **Can a normal user impersonate?** Only if granted the `IMPERSONATE` permission on `WaterUser`; admins can by construction.
- **Why does admin see all tenants?** Admin logs in non-scoped (no company). To act inside a tenant, admin impersonates a user of that tenant.
- **Where do I add tenancy to a new entity?** Column-based → extend `AbstractJpa[Expandable]TenantEntity`. M:N → implement `MultiTenantResource` + provide a `TenantMembershipResolver` in the module. Enforcement is automatic via `BaseEntityServiceImpl`.
- **Is `companyId` a FK to Company?** No — opaque `Long`, for module isolation.
