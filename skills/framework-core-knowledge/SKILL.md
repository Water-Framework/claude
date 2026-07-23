---
name: framework-core-knowledge
description: >
  Core knowledge base for Water Framework: philosophy, all base interfaces with full Java signatures,
  entity model hierarchy, service layer (Api/SystemApi), repository layer (BaseRepository, QueryBuilder),
  REST layer (BaseEntityRestApi), component lifecycle, permission system, and all framework annotations.
  Use as the primary reference for any Water Framework implementation task — covers Core-api, Repository, and Rest modules.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework developer with deep knowledge of all core APIs, interfaces, annotations, and architectural contracts. This skill provides the authoritative reference for all framework-level types — use it to guide correct implementation without reading source code.

---

# WATER FRAMEWORK CORE KNOWLEDGE BASE

## Table of Contents

0. [Package & Import Reference (on-demand)](#0-package--import-reference)
1. [Framework Philosophy](#1-framework-philosophy)
2. [Entity Model Hierarchy](#2-entity-model-hierarchy)
3. [Service Layer — Api & SystemApi](#3-service-layer--api--systemapi)
4. [Repository Layer](#4-repository-layer)
5. [QueryBuilder Fluent API](#5-querybuilder-fluent-api)
6. [REST Layer](#6-rest-layer)
7. [Component Lifecycle & Registry](#7-component-lifecycle--registry)
8. [Permission & Security System](#8-permission--security-system)
9. [Framework Annotations Reference](#9-framework-annotations-reference)
10. [Validation Annotations](#10-validation-annotations)
11. [Interceptor System](#11-interceptor-system)
12. [Implementation Patterns](#12-implementation-patterns)
13. [Anti-Patterns & Critical Rules](#13-anti-patterns--critical-rules)
14. [Quick Reference Tables](#14-quick-reference-tables)

---

## 0. Package & Import Reference

> **Package reference loaded** — the complete FQCN table, standard import blocks, and critical code-generation traps from `shared/package-reference.md` are already available in this skill's context.

The shared file contains:
- **Section 1**: Master FQCN table (65+ classes with package and module)
- **Section 2**: Source file paths for every key interface (read directly for exact signatures)
- **Section 3**: Ready-to-paste import blocks by use-case (ServiceImpl, SystemServiceImpl, Repository, REST, Security, Entity)
- **Section 4**: Critical code-generation traps (findAll parameter order, @Path prefix, etc.)

### Key packages — quick lookup (most frequently needed)

| Layer | Key class | Package |
|---|---|---|
| Entity | `BaseEntity`, `PaginableResult` | `it.water.core.api.model` |
| Service | `BaseEntityApi`, `BaseEntitySystemApi` | `it.water.core.api.service` |
| Repository | `BaseRepository`, `QueryBuilder`, `Query` | `it.water.core.api.repository[.query]` |
| Registry | `ComponentRegistry` | `it.water.core.api.registry` |
| Runtime | `Runtime`, `ApplicationProperties` | `it.water.core.api.bundle` |
| Security | `PermissionManager`, `SecurityContext` | `it.water.core.api.permission` |
| Annotations | `@FrameworkComponent`, `@Inject` | `it.water.core.interceptors.annotations` |
| Lifecycle | `@OnActivate`, `@OnDeactivate` | `it.water.core.api.interceptors` |
| Permission annotations | `@AllowPermissions`, `@AccessControl` | `it.water.core.permission.annotations` |
| Impl bases | `BaseEntitySystemServiceImpl`, `BaseEntityServiceImpl` | `it.water.repository.service` |
| JPA entity | `AbstractJpaEntity` | `it.water.repository.jpa.model` |

### Critical traps — always apply these rules

**findAll parameter order differs by layer:**
```
BaseRepository.findAll(int delta, int page, Query filter, QueryOrder order)      // Query = 3rd
BaseEntitySystemApi.findAll(Query filter, int delta, int page, QueryOrder order) // Query = 1st <- DIFFERENT
BaseEntityApi.findAll(Query filter, int delta, int page, QueryOrder order)       // Query = 1st <- DIFFERENT
```

**PaginableResult.getResults() returns Collection\<T\>, not List\<T\>:**
```java
List<T> list = new ArrayList<>(result.getResults()); // always wrap
```

---

## 1. Framework Philosophy

Water Framework is a **cross-runtime** solution — the same application code runs on OSGi (Karaf), Spring Boot, and Quarkus without modification.

### Three Pillars

| Pillar | Role |
|--------|------|
| **Water Framework** | Core abstractions: security, permissions, event streaming, entity management |
| **Water Generator** | Yeoman scaffolding: dependency analysis, stability metrics, build automation |
| **Runtimes** | OSGi (Karaf), Spring Boot, Quarkus — same code, multiple deploy targets |

### Convention Over Coding

Water goes beyond "convention over configuration":
- **Structural conventions**: Model / Api / Service layer separation enforced by module structure
- **Code conventions**: Annotations replace XML/YAML (`@FrameworkComponent`, `@Inject`, `@OnActivate`)
- **Build conventions**: Generator manages dependency ordering and stability metrics
- **Interface remoting**: REST endpoints auto-generated from service interfaces via proxy pattern

### Module Layer Structure (Generated by `yo water`)

```
<ProjectName>-model/         JPA entities — extend AbstractEntity
<ProjectName>-api/
  api/<Entity>Api.java        Public CRUD with permission checks
  api/<Entity>SystemApi.java  Internal CRUD without permission checks
  api/<Entity>Repository.java Repository contract
  api/rest/<Entity>RestApi.java  REST interface
<ProjectName>-service/
  service/<Entity>ServiceImpl.java           Implements Api
  service/<Entity>SystemServiceImpl.java     Implements SystemApi
  repository/<Entity>RepositoryImpl.java     Implements Repository
  service/rest/<Entity>RestControllerImpl.java  Implements RestApi
<ProjectName>-service-spring/  (or -service-osgi / -service-quarkus)
  Application.java             Spring Boot / OSGi / Quarkus bootstrap
  RestControllerImpl.java      Runtime-specific REST wiring
  application.properties       Runtime-specific config
```

**Sub-module roles and dependencies:**

| Sub-module | Depends on | Purpose |
|------------|-----------|---------|
| `<Name>-model` | Core-api, Jakarta Persistence/Validation | JPA entities, enums, DTOs. No Water service layer. |
| `<Name>-api` | model + Core-api, Repository-api, Core-rest-api, JAX-RS | `*Api`, `*SystemApi`, `*RestApi`, `*Repository` interfaces |
| `<Name>-service` | api + Core-service, Repository-jpa, Water-rest, Hibernate | `*ServiceImpl`, `*RepositoryImpl` — runtime-agnostic |
| `<Name>-service-spring` / `-osgi` / `-quarkus` | service + runtime libs | Standalone fat JAR / OSGi bundle: REST controllers, bootstrap, runtime glue |

**Rules:**
- The fourth module is only needed when the module deploys **standalone as a microservice**. Embeddable library modules stop at `-service`.
- For **connector/integration modules** (wraps external systems, no owned entities): use `applicationType=service` in the generator — produces model/api/service without entity CRUD boilerplate.
- `yo water:new-project` scaffolds the correct layout automatically based on `applicationType` and `projectTechnology`.

---

## 2. Entity Model Hierarchy

### 2.1 Resource — Root Interface

```java
// it.water.core.api.model.Resource
public interface Resource {
    default String getResourceName() {
        return this.getClass().getName();
    }
}
```
Marker interface for all objects managed by the framework.

---

### 2.2 BaseEntity — Persisted Entity Contract

```java
// it.water.core.api.model.BaseEntity
public interface BaseEntity extends Resource {
    long getId();
    Date getEntityCreateDate();
    Date getEntityModifyDate();
    Integer getEntityVersion();
    void setEntityVersion(Integer entityVersion);
    default boolean isExpandableEntity() {
        return ExpandableEntity.class.isAssignableFrom(this.getClass());
    }
    default long[] getCategoryIds()  { return new long[0]; }
    default void setCategoryIds(long[] categoryIds) { }
    default long[] getTagIds()       { return new long[0]; }
    default void setTagIds(long[] tagIds) { }
}
```

**Base fields present in every persisted entity:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | `long` | Auto-generated primary key |
| `entityCreateDate` | `Date` | Timestamp of creation (auto-set) |
| `entityModifyDate` | `Date` | Timestamp of last update (auto-set) |
| `entityVersion` | `Integer` | Optimistic locking counter — incremented on every update |

---

### 2.3 AbstractEntity — JPA Base Class

```java
// it.water.repository.entity.model.AbstractEntity
public abstract class AbstractEntity extends AbstractResource implements BaseEntity {
    protected long id;
    protected Integer entityVersion;
    protected Date entityCreateDate;
    protected Date entityModifyDate;
    protected long[] categoryIds;
    protected long[] tagIds;

    protected AbstractEntity();
    protected AbstractEntity(long id);
    protected AbstractEntity(long id, Date entityCreateDate);
}
```
**Always extend this class** for JPA entities. The generator does this automatically.

---

### 2.4 ProtectedEntity — Permission-Controlled Entity

```java
// it.water.core.api.permission.ProtectedEntity
public interface ProtectedEntity extends BaseEntity { }
```
Marker: entity participates in the permission system. Combine with `@AccessControl` annotation.

---

### 2.5 OwnedResource — User-Owned Entity

```java
// it.water.core.api.entity.owned.OwnedResource
public interface OwnedResource extends BaseEntity {
    static final String OWNER_USER_ID_FIELD_NAME = "ownerUserId";

    Long getOwnerUserId();
    void setOwnerUserId(Long userId);

    static String getOwnerUserIdFieldName() {
        return OWNER_USER_ID_FIELD_NAME;
    }
}
```
When an entity implements `OwnedResource`, the framework auto-assigns `ownerUserId` to the currently logged-in user on creation. Operations are restricted to the owner unless explicitly overridden.

---

### 2.6 OwnedChildResource — Child of Owned Entity

```java
// it.water.core.api.entity.owned.OwnedChildResource
public interface OwnedChildResource extends OwnedResource {
    BaseEntity getParent();
}
```
Inherits ownership semantics from parent. Framework propagates ownership checks up the parent chain.

---

### 2.7 SharedEntity — Shareable Owned Entity

```java
// it.water.core.api.entity.shared.SharedEntity
public interface SharedEntity extends OwnedResource { }
```
Extends `OwnedResource` with sharing capabilities. Integrates with the `SharedEntity` module for cross-user access.

---

### 2.7-bis Tenant markers — Company-based multitenancy (`it.water.core.api.entity.tenant`)

Two opt-in markers make an entity tenant-scoped (see the `multitenancy-knowledge` skill). Enforcement lives in `BaseEntityServiceImpl` (Api layer) and applies **only when `SecurityContext.getActiveCompanyId() != null`** (lenient: MT off / non-scoped admin / legacy token → no filtering → fully backward compatible; NO `isAdmin()` special-casing).

```java
// single-company entity: carries a nullable opaque companyId column (null = global/cross-tenant)
public interface TenantResource extends BaseEntity {
    String COMPANY_ID_FIELD_NAME = "companyId";
    Long getCompanyId();  void setCompanyId(Long companyId);
}
// M:N entity: NO column — membership lives in a domain table, resolved by a TenantMembershipResolver
public interface MultiTenantResource extends BaseEntity { }
```

- **Column-based** entities extend a base `@MappedSuperclass` (JpaRepository-api) that already carries `companyId`: `AbstractJpaTenantEntity` (non-expandable) / `AbstractJpaExpandableTenantEntity` (expandable — an expandable entity MUST stay expandable when tenantized). `companyId` is `@JsonIgnore` (server-assigned, like `ownerUserId`). Examples: `Document`, `WaterRole` (null=global role).
- **M:N** entities (e.g. `WaterUser` via `UserCompany`) implement `MultiTenantResource`; the module provides a `TenantMembershipResolver` (`it.water.core.api.service.integration`) — `boolean supports(String type)` + `Set<Long> getEntityIdsInCompany(String type, long companyId)` — resolved from the ComponentRegistry, used to filter `id IN (...)`.
- `companyId` is an **opaque `Long`**, never a JPA relation to `Company` (module isolation). Filter: `TenantResource` → `companyId = active OR companyId IS NULL`; `MultiTenantResource` → `id IN (resolver ids)`.

---

### 2.8 ProtectedResource — Fine-Grained Access Resource

```java
// it.water.core.api.model.ProtectedResource
public interface ProtectedResource extends Resource {
    String getResourceId();
}
```
Marker for resources requiring granular access control (not entity-level, but resource-level).

---

### 2.9 ExpandableEntity — Dynamic Fields Entity

```java
// it.water.core.api.model.ExpandableEntity
public interface ExpandableEntity extends BaseEntity {
    Map<String, Object> getExtraFields();
    void setExtraFields(Map<String, Object> extraFields);
    EntityExtension getExtension();
    void setExtension(EntityExtension extension);
}
```
Supports dynamic extra fields stored as JSON and typed extensions via the entity extension system.

---

### 2.10 User — System User

```java
// it.water.core.api.model.User
public interface User extends Authenticable {
    long getId();
    String getName();
    String getLastname();
    String getEmail();
    String getUsername();
    boolean isAdmin();

    @Override default String getScreenName()          { return this.getUsername(); }
    @Override default String getScreenNameFieldName() { return "username"; }
    @Override default boolean isActive()              { return false; }
}
```

---

### 2.11 Role — RBAC Role

```java
// it.water.core.api.model.Role
public interface Role {
    default long getId() { return 0; }
    String getName();
}
```

---

### 2.12 Entity Interface Combinations

| Combination | Use when |
|-------------|----------|
| `AbstractEntity` only | Simple entity, no permission control |
| `AbstractEntity` + `ProtectedEntity` + `@AccessControl` | Entity with role-based permission |
| `AbstractEntity` + `OwnedResource` | Entity owned by a specific user |
| `AbstractEntity` + `OwnedResource` + `ProtectedEntity` | Owned + permission-controlled |
| `AbstractEntity` + `SharedEntity` | Owned but shareable with other users |
| `AbstractEntity` + `ExpandableEntity` | Entity with dynamic extra fields |

---

## 3. Service Layer — Api & SystemApi

### 3.1 Service — Base Marker

```java
// it.water.core.api.service.Service
public interface Service { }
```
All framework-managed services implement this marker. Required for `@Inject` to work.

---

### 3.2 BaseApi — Permission-Checked Service Base

```java
// it.water.core.api.service.BaseApi
public interface BaseApi extends Service { }
```
Custom service interfaces extend this. Methods decorated with `@AllowPermissions`, `@AllowRoles`, or `@AllowGenericPermissions` are intercepted for permission checks.

---

### 3.3 BaseSystemApi — Internal Service Base (No Permission Checks)

```java
// it.water.core.api.service.BaseSystemApi
public interface BaseSystemApi extends Service { }
```
For internal operations that bypass the permission layer. **Only use for legitimately trusted internal calls.**

---

### 3.4 BaseEntityApi<T> — CRUD Api with Permissions

```java
// it.water.core.api.service.BaseEntityApi
public interface BaseEntityApi<T extends BaseEntity> extends BaseApi {
    T save(T entity);
    T update(T entity);
    void remove(long id);
    T find(long id);
    T find(Query filter);
    PaginableResult<T> findAll(Query filter, int delta, int page, QueryOrder queryOrder);
    long countAll(Query filter);
    Class<T> getEntityType();
}
```
**Every method must have a permission annotation** on the interface definition. No exception.

---

### 3.5 BaseEntitySystemApi<T> — CRUD SystemApi Without Permissions

```java
// it.water.core.api.service.BaseEntitySystemApi
public interface BaseEntitySystemApi<T extends BaseEntity> extends BaseSystemApi {
    T save(T entity);
    T update(T entity);
    void remove(long id);
    T find(long id);
    T find(Query filter);
    PaginableResult<T> findAll(Query filter, int delta, int page, QueryOrder queryOrder);
    long countAll(Query filter);
    Class<T> getEntityType();
    QueryBuilder getQueryBuilderInstance();
}
```
Used internally by services that need to operate on entities without triggering permission checks (e.g., cascade operations, background jobs).

---

### 3.6 Api vs SystemApi Decision Rule

Place an operation in `*SystemApi` ONLY when **ALL** four conditions hold:

1. It is called from another service **in the same module** (or an internal scheduler / lifecycle hook).
2. The caller already has authority to act, but does not naturally hold permissions on the target aggregate.
3. The operation is **NEVER** exposed via REST.
4. The operation supports a **cross-aggregate transition** or a data-integrity reconciliation.

If any condition fails → put it in `*Api` and protect with `@AllowPermissions`.

```
External call (from REST, from other domain's public API)
  └─> ALWAYS Api — permission checks are mandatory

Internal call (from same module, trusted orchestration)
  └─> SystemApi ONLY if all 4 conditions above hold
  └─> Api otherwise — "convenience" or "skip check" is never a valid justification

WARNING: Never expose SystemApi via REST.
WARNING: Never inject SystemApi into a class reachable from the REST layer.
WARNING: Over-using SystemApi creates silent privilege escalation paths.
```

**Applying the rule:** for every SystemApi method, write a one-line security justification. If the justification reads "convenience" or "skip the check", move the method to Api.

---

### 3.7 Cross-Aggregate State Transitions via Paired SystemApi

When two aggregates have mutually-dependent state (one's status mirrors the other's lifecycle), expose a SystemApi method on **each side** and have each `*ServiceImpl` depend on the sibling's SystemApi. Never let the sibling carry the permission burden of the caller.

**Pattern:**
```java
// BookSystemApi — called by LoanServiceImpl when a loan opens/closes
void transitionStatusInternal(long bookId, BookStatus newStatus);

// LoanSystemApi — called by BookServiceImpl.markAsLost()
void forceCloseOpenLoanForBook(long bookId, ReturnReason reason);
```

**Why:** Without this, the caller would need both `Book.update` AND `Loan.update` permissions even though it is invoking a narrow legitimate transition. Forcing the broad permission breaks role separation (e.g., a `librarian` who can lend should not need to mutate book metadata).

**Rules:**
- Pair them: caller-side calls callee-side SystemApi, never the reverse in the same transaction.
- Co-locate both writes in **one `@Transactional` boundary** at the originating service method.
- Both `*SystemApi` interfaces live in `-api` — implementations in `-service` — breaking the circular dependency because both impls depend only on interfaces.
- Always cross-link to the Api vs SystemApi criteria (§3.6) when justifying the SystemApi methods.

---

## 4. Repository Layer

### 4.1 BaseRepository<T> — Data Access Contract

```java
// it.water.core.api.repository.BaseRepository
public interface BaseRepository<T extends BaseEntity> extends Service {
    Class<T> getEntityType();

    // Persist (INSERT)
    T persist(T entity);
    T persist(T entity, Runnable executeInTransaction);

    // Update (UPDATE)
    T update(T entity);
    T update(T entity, Runnable executeInTransaction);

    // Remove (DELETE)
    void remove(long id);
    void remove(long id, Runnable executeInTransaction);
    void remove(T entity);
    void removeAllByIds(Iterable<Long> ids);
    void removeAll(Iterable<T> entities);
    void removeAll();

    // Find (SELECT)
    T find(long id);
    T find(Query filter);
    T find(String filterStr);

    // Find All (SELECT with pagination)
    PaginableResult<T> findAll(int delta, int page, Query filter, QueryOrder queryOrder);

    // Count
    long countAll(Query filter);

    // Query builder factory
    QueryBuilder getQueryBuilderInstance();
}
```

**Repository is the ONLY layer that talks to the database.**
- `ServiceImpl` delegates to `SystemApi`
- `SystemApi` delegates to `Repository`
- `Repository` uses JPA / QueryBuilder

---

### 4.2 PaginableResult<T> — Paginated Response

```java
// it.water.core.api.model.PaginableResult
public interface PaginableResult<T extends Resource> {
    int getNumPages();      // total pages
    int getCurrentPage();   // current page (1-based)
    int getNextPage();      // next page number (0 if last)
    int getDelta();         // page size
    Collection<T> getResults(); // entities on this page
}
```

**Standard usage in REST layer:**
```java
// delta = page size (default 20), page = page number (1-based)
PaginableResult<MyEntity> result = entityApi.findAll(filter, 20, 1, null);
```

---

## 5. QueryBuilder Fluent API

### 5.1 QueryBuilder — Entry Point

```java
// it.water.core.api.repository.query.QueryBuilder
public interface QueryBuilder {
    Query createQueryFilter(String filter);  // parse string filter expression
    FieldNameOperand field(String name);     // start fluent query on field
}
```

Obtain an instance from the repository or SystemApi:
```java
QueryBuilder qb = myEntitySystemApi.getQueryBuilderInstance();
// or
QueryBuilder qb = myRepository.getQueryBuilderInstance();
```

---

### 5.2 FieldNameOperand — Comparison Methods

```java
// it.water.core.api.repository.query.operands.FieldNameOperand
public class FieldNameOperand extends AbstractOperand<String> {
    public Query equalTo(Object value);
    public Query notEqualTo(Object value);
    public Query greaterThan(Number value);
    public Query greaterThan(Date value);
    public Query greaterOrEqualThan(Number value);
    public Query greaterOrEqualThan(Date value);
    public Query lowerThan(Number value);
    public Query lowerThan(Date value);
    public Query lowerOrEqualThan(Number value);
    public Query lowerOrEqualThan(Date value);
    public Query like(String value);
}
```

---

### 5.3 Query — Logical Composition

```java
// it.water.core.api.repository.query.Query
public interface Query {
    void defineOperands(Query... operands);
    String getDefinition();
    Query and(Query rightQuery);
    Query or(Query rightQuery);
    Query in(List<?> values);
    Query not();
}
```

---

### 5.4 QueryOrder — Sorting

```java
// it.water.core.api.repository.query.QueryOrder
public interface QueryOrder {
    QueryOrder addOrderField(String name, boolean asc);
    List<QueryOrderParameter> getParametersList();
}
```

---

### 5.5 Complete QueryBuilder Examples

```java
QueryBuilder qb = systemApi.getQueryBuilderInstance();

// Simple equality
Query byName = qb.field("name").equalTo("Admin");

// Like (pattern matching — % is wildcard)
Query byNameLike = qb.field("name").like("%mario%");

// Numeric comparison
Query byAge = qb.field("age").greaterThan(18);
Query byPrice = qb.field("price").lowerOrEqualThan(100.0);

// Date comparison
Query recent = qb.field("entityCreateDate").greaterThan(new Date(since));

// AND composition
Query activeAdults = qb.field("status").equalTo("ACTIVE")
    .and(qb.field("age").greaterOrEqualThan(18));

// OR composition
Query adminOrManager = qb.field("role").equalTo("ADMIN")
    .or(qb.field("role").equalTo("MANAGER"));

// NOT
Query notDeleted = qb.field("deleted").equalTo(true).not();

// IN (list of values)
List<String> statuses = Arrays.asList("ACTIVE", "PENDING");
Query byStatuses = qb.field("status").equalTo("ACTIVE").in(statuses);

// Complex composition
Query complex = qb.field("status").equalTo("ACTIVE")
    .and(qb.field("age").greaterThan(18))
    .and(qb.field("name").like("%mario%").or(qb.field("name").like("%luigi%")));

// Paginated find with query and ordering
QueryOrder order = ... ; // obtained from framework
order.addOrderField("name", true).addOrderField("entityCreateDate", false);
PaginableResult<MyEntity> page = repository.findAll(20, 1, complex, order);

// String filter (alternative — RSQL-like syntax)
Query fromString = qb.createQueryFilter("status==ACTIVE;age>18");
```

---

## 6. REST Layer

### 6.1 RestApi — Marker Interface

```java
// it.water.core.api.service.rest.RestApi
public interface RestApi extends Service { }
```

---

### 6.2 @FrameworkRestApi — JAX-RS REST Resource Annotation

```java
// it.water.core.api.service.rest.FrameworkRestApi
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface FrameworkRestApi { }
```
**Target**: CLASS (on the REST interface)
Apply to JAX-RS interface (`@Path`, `@GET`, `@POST`, etc.). Used by CXF and OSGi runtime.

---

### 6.3 @FrameworkRestController — Spring MVC Controller Annotation

```java
// it.water.core.api.service.rest.FrameworkRestController
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface FrameworkRestController {
    Class<? extends RestApi> referredRestApi();
}
```
**Target**: CLASS (on the REST controller implementation)
**Attribute**: `referredRestApi` — the JAX-RS interface this class implements.
Required on every `XxxRestControllerImpl`.

---

### 6.4 @LoggedIn — Authentication Guard

```java
// it.water.service.rest.api.security.LoggedIn
@NameBinding
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface LoggedIn {
    String[] issuers() default {"it.water.core.api.model.User"};
}
```
**Target**: CLASS or METHOD
**Attribute**: `issuers` — valid JWT issuer classes
Place on REST interface methods (or class) to require authentication. Works as a JAX-RS `@NameBinding` filter.

---

### 6.5 BaseEntityRestApi<T> — CRUD REST Base

```java
// it.water.service.rest.persistence.BaseEntityRestApi
public abstract class BaseEntityRestApi<T extends BaseEntity> implements Service {
    public static final int HYPERIOT_DEFAULT_PAGINATION_DELTA = 20;

    public T save(T entity);
    public T update(T entity);
    public void remove(long id);
    public T find(long id);
    public PaginableResult<T> findAll(Integer delta, Integer page, Query filter, QueryOrder order);
    public PaginableResult<T> findAll();

    protected abstract BaseEntityApi<T> getEntityService();
}
```
The generated `XxxRestControllerImpl` extends this. Override `getEntityService()` to return the injected `XxxApi`.

---

### 6.6 WaterJsonView — JSON Serialization Views

```java
// it.water.core.api.service.rest.WaterJsonView
public class WaterJsonView {
    public interface Compact  { }                        // id + primary key only
    public interface Extended extends Compact { }        // all public non-sensitive fields
    public interface Public   extends Extended { }       // safe for external APIs
    public interface Internal extends Extended { }       // service-to-service communication
    public interface Secured  extends Extended { }       // authenticated users only
    public interface Privacy  extends Extended { }       // PII / sensitive fields
}
```

Usage with Jackson `@JsonView`:
```java
@JsonView(WaterJsonView.Public.class)
public String getName() { return name; }

@JsonView(WaterJsonView.Privacy.class)
public String getSocialSecurityNumber() { return ssn; }
```

---

### 6.7 REST Path Convention (CRITICAL)

```
@Path annotation in JAX-RS interfaces MUST NOT include the /water prefix.

CXF adds /water automatically as the base context root.

WRONG:  @Path("/water/api/products")    → URL becomes /water/water/api/products (404)
CORRECT: @Path("/api/products")         → URL becomes /water/api/products (200)

Full URL = http://host:port/water + @Path value
```

**Why this matters (confirmed regression):** The ApiGateway module had paths like `@Path("/water/api/gateway/routes")`. CXF doubled the prefix to `/water/water/api/gateway/routes`, causing HTTP 404 across all endpoints. The fix was stripping the `/water` prefix from every `@Path` annotation. Any `@Path` starting with `/water` is a bug — flag it immediately during design review and code review.

**In documentation and analysis:** when listing REST paths, write them **without** `/water` in the "declared `@Path`" column and show the full external URL separately:

| Declared `@Path` | Full external URL |
|-----------------|------------------|
| `/api/products` | `/water/api/products` |
| `/api/products/{id}` | `/water/api/products/{id}` |

---

## 7. Component Lifecycle & Registry

### 7.1 ComponentRegistry — Service Locator

```java
// it.water.core.api.registry.ComponentRegistry
public interface ComponentRegistry {
    // Lookup
    <T> List<T> findComponents(Class<T> componentClass, ComponentFilter filter);
    <T> T findComponent(Class<T> componentClass, ComponentFilter filter);

    // Registration (used by framework, rarely by application code)
    <T, K> ComponentRegistration<T, K> registerComponent(
        Class<? extends T> componentClass,
        T component,
        ComponentConfiguration configuration);

    // Unregistration
    <T> boolean unregisterComponent(ComponentRegistration<T, ?> registration);
    <T> boolean unregisterComponent(Class<T> componentClass, T component);

    // Filter builder factory
    ComponentFilterBuilder getComponentFilterBuilder();

    // Specialized lookups
    <T extends BaseEntitySystemApi> T findEntitySystemApi(String entityClassName);
    <T extends BaseRepository>      T findEntityRepository(String entityClassName);
    <T extends BaseEntity> BaseRepository<T> findEntityExtensionRepository(Class<T> type);

    // Lifecycle invocation
    default <T> void invokeLifecycleMethod(
        Class<? extends Annotation> annotation,
        Class<?> componentServiceClass,
        T component);
}
```

**Standard usage** in services:
```java
@Inject
@Setter
private ComponentRegistry componentRegistry;

// Lookup a service by interface
MyService svc = componentRegistry.findComponent(MyService.class, null);

// Lookup with filter
ComponentFilter filter = componentRegistry.getComponentFilterBuilder()
    .createFilter("type", "redis");
MyConnector connector = componentRegistry.findComponent(MyConnector.class, filter);
```

---

### 7.2 ComponentRegistration<T, K>

```java
// it.water.core.api.registry.ComponentRegistration
public interface ComponentRegistration<T, K> {
    T getComponent();
    ComponentConfiguration getConfiguration();
    Class<? extends T> getRegistrationClass();
    K getRegistration();
}
```

---

### 7.3 ComponentFilter & ComponentFilterBuilder

```java
// it.water.core.api.registry.filter.ComponentFilter
public interface ComponentFilter {
    String getFilter();
    ComponentFilter and(ComponentFilter filter);
    ComponentFilter and(String propertyName, String propertyValue);
    ComponentFilter or(ComponentFilter filter);
    ComponentFilter or(String propertyName, String propertyValue);
    ComponentFilter not();
    boolean isNot();
    boolean matches(Properties props);
}

// it.water.core.api.registry.filter.ComponentFilterBuilder
public interface ComponentFilterBuilder {
    ComponentFilter createFilter(String name, String value);
}
```

---

### 7.4 @FrameworkComponent — Component Registration

```java
// it.water.core.interceptors.annotations.FrameworkComponent
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface FrameworkComponent {
    String[] properties() default {};   // registration properties
    Class<?>[] services() default {};   // interfaces to expose
    int priority() default 1;           // override priority (>1 replaces lower priority)
}
```
Marks a class as a framework-managed component. The runtime (OSGi/Spring/Quarkus) automatically registers it in `ComponentRegistry`.

**Priority**: if two components implement the same interface, the one with `priority > 1` wins. This is the extension/override mechanism.

---

### 7.5 @Inject — Dependency Injection

```java
// it.water.core.interceptors.annotations.Inject
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@InterceptorExecutor(interceptor = WaterComponentsInjector.class)
@IndexAnnotated
public @interface Inject {
    boolean injectOnceAtStartup() default false;
}
```
**Target**: FIELD
**Attribute**: `injectOnceAtStartup` — if `true`, injects at startup and never refreshes; if `false` (default), re-resolves on each access (dynamic binding).

**IMPORTANT**: Combine with Lombok `@Setter` — the framework uses the setter to inject:
```java
@Inject
@Setter
private MyService myService;
```

---

### 7.6 @OnActivate — Initialization Hook

```java
// it.water.core.api.interceptors.OnActivate
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface OnActivate { }
```
**Target**: METHOD
Called when the component is activated/started. Use for initialization logic (loading properties, establishing connections).

```java
@OnActivate
public void initialize() {
    this.timeout = ApplicationProperties.getInstance()
        .getPropertyOrDefault("my.module.timeout", 30, Integer.class);
}
```

---

### 7.7 @OnDeactivate — Shutdown Hook

```java
// it.water.core.api.interceptors.OnDeactivate
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface OnDeactivate { }
```
**Target**: METHOD
Called when the component is deactivated/stopped. Use for resource cleanup (close connections, flush buffers).

---

## 8. Permission & Security System

### 8.1 SecurityContext — Current User Access

```java
// it.water.core.api.permission.SecurityContext
public interface SecurityContext extends Service {
    String getLoggedUsername();
    boolean isLoggedIn();
    boolean isAdmin();
    long getLoggedEntityId();
    // multitenancy (additive, default null): the active company (tenant) for the session,
    // and — for impersonation tokens — the caller who impersonated this user.
    default Long getActiveCompanyId() { return null; }
    default String getImpersonatedBy() { return null; }
    default boolean isImpersonated() { return getImpersonatedBy() != null; }
}
```
Inject this to read the currently authenticated user. Always request it from `ComponentRegistry` (it is request-scoped in web contexts). `getActiveCompanyId()` is the SINGLE carrier read by tenant enforcement (`!= null` → scope to that company; `null` → no tenant scope). Populated by the JWT filter from the `companyId`/`impersonatedBy` claims via `UserPrincipal` → `WaterAbstractSecurityContext`.

---

### 8.2 PermissionManager — Authorization Checks

```java
// it.water.core.api.permission.PermissionManager
public interface PermissionManager extends Service {
    boolean userHasRoles(String username, String[] rolesNames);

    void addPermissionIfNotExists(Role r, Class<? extends Resource> resourceClass, Action action);

    boolean checkPermission(String username, Resource entity, Action action);
    boolean checkPermission(String username, Class<? extends Resource> resource, Action action);
    boolean checkPermission(String username, String resourceName, Action action);

    boolean checkPermissionAndOwnership(
        String username, String resourceName, Action action, Resource... entities);
    boolean checkPermissionAndOwnership(
        String username, Resource resource, Action action, Resource... entities);

    boolean checkUserOwnsResource(User user, Object resource);

    static boolean isProtectedEntity(Object entity);
    static boolean isProtectedEntity(String resourceName);

    Map<String, Map<String, Map<String, Boolean>>> entityPermissionMap(
        String username, Map<String, List<Long>> entityPks);
}
```

---

### 8.3 Action — Atomic Permission Unit

```java
// it.water.core.api.action.Action
public interface Action {
    String getActionName();   // e.g., "save", "update", "find", "remove"
    String getActionType();   // resource class name
    long getActionId();       // power of 2: 1, 2, 4, 8, 16, ...
}
```
Actions are combined with bitmask arithmetic. Standard actions:

| Action Name | Typical ID |
|-------------|-----------|
| `save`      | 1 |
| `update`    | 2 |
| `find`      | 4 |
| `findAll`   | 8 |
| `remove`    | 16 |

**Naming convention for action identifiers:** `<ResourceClassName>.<actionVerb>`

- Standard CRUD verbs (always lowercase): `find`, `save`, `update`, `remove`
- Custom action verbs: lowercase verb that names the state transition — `close`, `archive`, `restore`, `lost`, `cancel`, `approve`, `publish`, `suspend`, etc.

Examples:
```
Book.find      Book.save      Book.update    Book.remove
Book.lost      Book.archive
Loan.find      Loan.save      Loan.update    Loan.remove    Loan.close
```

**Custom actions must be explicitly registered** at module activation via the Water action manager inside `@OnActivate`. Standard CRUD actions are auto-registered by the framework.

```java
@OnActivate
public void initialize() {
    // Register custom actions not covered by standard CRUD
    actionManager.registerAction("lost",    Book.class);
    actionManager.registerAction("archive", Book.class);
    actionManager.registerAction("close",   Loan.class);
}
```

**In design documents:** always produce a permission matrix table (resource.action × role) in §3.5 Security Design. Flag every custom action so the backend-architect knows to register it during `@OnActivate`.

---

### 8.4 @AccessControl — Entity Permission Declaration

```java
// it.water.core.permission.annotations.AccessControl
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface AccessControl {
    String[] availableActions() default {};
    DefaultRoleAccess[] rolesPermissions() default {};
}
```
**Target**: CLASS (on entity)

```java
// it.water.core.permission.annotations.DefaultRoleAccess
@Target({ElementType.LOCAL_VARIABLE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DefaultRoleAccess {
    String roleName() default "";
    String[] actions() default {};
}
```

**Example on a ProtectedEntity:**
```java
@AccessControl(
    availableActions = {"save", "update", "find", "findAll", "remove"},
    rolesPermissions = {
        @DefaultRoleAccess(roleName = "MANAGER", actions = {"save","update","find","findAll","remove"}),
        @DefaultRoleAccess(roleName = "VIEWER",  actions = {"find","findAll"})
    }
)
public class MyEntity extends AbstractEntity implements ProtectedEntity { }
```

---

## 9. Framework Annotations Reference

### 9.1 @AllowPermissions — Entity-Level Permission Check

```java
// it.water.core.permission.annotations.AllowPermissions
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowPermissions {
    String[] actions() default {};
    boolean checkById() default false;  // check permission for specific entity by ID
    int idParamIndex() default 0;       // method param index that contains the entity ID
    String systemApiRef() default "";   // SystemApi bean name for entity lookup
}
```
**Target**: METHOD (on Api interface methods)

```java
// In XxxApi.java interface:
@AllowPermissions(actions = {"save"})
MyEntity save(MyEntity entity);

@AllowPermissions(actions = {"find"}, checkById = true, idParamIndex = 0)
MyEntity find(long id);

@AllowPermissions(actions = {"remove"}, checkById = true, idParamIndex = 0)
void remove(long id);
```

---

### 9.2 @AllowRoles — Role-Level Check

```java
// it.water.core.permission.annotations.AllowRoles
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowRoles {
    String[] rolesNames() default {};
}
```
**Target**: METHOD

```java
@AllowRoles(rolesNames = {"ADMIN", "MANAGER"})
void adminOperation();
```

---

### 9.3 @AllowGenericPermissions — Resource-Scoped Check

```java
// it.water.core.permission.annotations.AllowGenericPermissions
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowGenericPermissions {
    String[] actions() default {};
    String resourceName() default "";       // explicit resource class name
    String resourceParamName() default "";  // method param name that holds the resource
}
```
**Target**: METHOD
Use when the entity is not identified by an ID parameter but by a resource reference.

---

### 9.4 @AllowLoggedUser — Authentication-Only Check

```java
// it.water.core.permission.annotations.AllowLoggedUser
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowLoggedUser { }
```
Allows any authenticated user, regardless of roles or permissions. Equivalent to "logged in" check only.

---

### 9.5 @AllowPermissionsOnReturn — Post-Execution Filter

```java
// it.water.core.permission.annotations.AllowPermissionsOnReturn
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowPermissionsOnReturn {
    String[] actions() default {};
}
```
Filters the return value of a method based on permissions on the returned object.

---

### 9.6 @LogMethodExecution — Execution Logging

```java
// it.water.core.interceptors.annotations.LogMethodExecution
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface LogMethodExecution {
    boolean logDebug() default true;  // true = DEBUG, false = INFO
}
```
Apply on class or method to enable automatic logging of method entry/exit with parameters.

---

## 10. Validation Annotations

### 10.1 @NotNullOnPersist — Required on Persist

```java
// it.water.core.validation.annotations.NotNullOnPersist
@NotNull
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = {})
public @interface NotNullOnPersist {
    String message() default "{water.validation.constraints.NotNullOnPersist.message}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```
Field cannot be null when persisting. Use on entity fields that are required.

**NOTE**: The generator adds an `exampleField` with `@NotNullOnPersist` as a placeholder. **Always remove it** if it has no domain meaning and replace with real fields.

---

### 10.2 @NoMalitiusCode — XSS/Injection Prevention

```java
// it.water.core.validation.annotations.NoMalitiusCode
@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NoMalitiusCodeValidator.class)
public @interface NoMalitiusCode {
    String message() default ValidationMessages.NO_MALITIUS_CODE_VALIDATION;
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```
Apply on `String` fields received from user input to prevent XSS and injection attacks. The validator rejects HTML tags and script patterns.

---

## 11. Interceptor System

### 11.1 MethodInterceptor<A> — Custom Interceptor Base

```java
// it.water.core.api.interceptors.MethodInterceptor
public interface MethodInterceptor<A extends Annotation> {
    Class<A> getAnnotation();
}
```
Implement this to create a custom interceptor triggered by a custom annotation.

---

### 11.2 @InterceptorExecutor — Link Annotation to Interceptor

```java
// it.water.core.api.interceptors.InterceptorExecutor
@Target({ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface InterceptorExecutor {
    Class<? extends MethodInterceptor> interceptor();
}
```
Meta-annotation placed on a custom annotation to associate it with its `MethodInterceptor` implementation.

**Pattern for custom interceptor:**
```java
// 1. Define custom annotation
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@InterceptorExecutor(interceptor = MyInterceptor.class)
public @interface MyAnnotation { }

// 2. Implement interceptor
@FrameworkComponent
public class MyInterceptor implements MethodInterceptor<MyAnnotation> {
    @Override
    public Class<MyAnnotation> getAnnotation() { return MyAnnotation.class; }
    // pre/post processing logic...
}

// 3. Use on service methods
@MyAnnotation
public void myMethod() { ... }
```

---

## 12. Implementation Patterns

### 12.1 Standard Entity Implementation

```java
@AccessControl(
    availableActions = {"save","update","find","findAll","remove"},
    rolesPermissions = {
        @DefaultRoleAccess(roleName = "MANAGER", actions = {"save","update","find","findAll","remove"}),
        @DefaultRoleAccess(roleName = "VIEWER",  actions = {"find","findAll"})
    }
)
public class Product extends AbstractEntity implements ProtectedEntity {

    @NotNullOnPersist
    @NoMalitiusCode
    private String name;

    @NotNullOnPersist
    private Double price;

    @Column(nullable = true)
    private String description;

    // Lombok @Data or manual getters/setters
}
```

---

### 12.2 Standard Api Interface

```java
public interface ProductApi extends BaseEntityApi<Product> {

    @AllowPermissions(actions = {"save"})
    @Override
    Product save(Product entity);

    @AllowPermissions(actions = {"update"})
    @Override
    Product update(Product entity);

    @AllowPermissions(actions = {"find"}, checkById = true, idParamIndex = 0)
    @Override
    Product find(long id);

    @AllowPermissions(actions = {"findAll"})
    @Override
    PaginableResult<Product> findAll(Query filter, int delta, int page, QueryOrder order);

    @AllowPermissions(actions = {"remove"}, checkById = true, idParamIndex = 0)
    @Override
    void remove(long id);

    // Custom method example
    @AllowRoles(rolesNames = {"MANAGER"})
    List<Product> findByCategory(String category);
}
```

---

### 12.3 Standard SystemApi Interface

```java
public interface ProductSystemApi extends BaseEntitySystemApi<Product> {
    // No permission annotations needed — SystemApi bypasses security
    List<Product> findActiveProducts();
    void archiveExpiredProducts();
}
```

---

### 12.4 Standard ServiceImpl

```java
@FrameworkComponent(services = {ProductApi.class, ProductSystemApi.class})
public class ProductServiceImpl extends BaseEntityServiceImpl<Product>
    implements ProductApi, ProductSystemApi {

    @Inject
    @Setter
    private ProductRepository productRepository;

    @OnActivate
    public void initialize() {
        // Load configuration, setup caches, etc.
    }

    @Override
    protected BaseRepository<Product> getRepository() {
        return productRepository;
    }

    @Override
    public List<Product> findByCategory(String category) {
        QueryBuilder qb = productRepository.getQueryBuilderInstance();
        Query q = qb.field("category").equalTo(category);
        return productRepository.findAll(20, 1, q, null).getResults()
            .stream().collect(Collectors.toList());
    }
}
```

---

### 12.5 Standard REST Controller

```java
@FrameworkRestController(referredRestApi = ProductRestApi.class)
public class ProductRestControllerImpl extends BaseEntityRestApi<Product>
    implements ProductRestApi {

    @Inject
    @Setter
    private ProductApi productApi;

    @Override
    protected BaseEntityApi<Product> getEntityService() {
        return productApi;
    }
}
```

```java
// REST Api interface
@FrameworkRestApi
@Path("/api/products")   // NO /water prefix — CXF adds it automatically
public interface ProductRestApi extends RestApi {

    @POST
    @Produces(MediaType.APPLICATION_JSON)
    @Consumes(MediaType.APPLICATION_JSON)
    @LoggedIn
    Response save(Product product);

    @GET
    @Path("/{id}")
    @Produces(MediaType.APPLICATION_JSON)
    @LoggedIn
    Response find(@PathParam("id") long id);
}
```

---

### 12.6 Component Lookup Pattern

```java
// Direct lookup (when @Inject is not available)
ProductApi productApi = componentRegistry.findComponent(ProductApi.class, null);

// Lookup with filter (when multiple implementations exist)
ComponentFilter filter = componentRegistry.getComponentFilterBuilder()
    .createFilter("runtime", "redis");
CacheService cacheService = componentRegistry.findComponent(CacheService.class, filter);

// Find SystemApi by entity class name
ProductSystemApi systemApi = componentRegistry.findEntitySystemApi(Product.class.getName());
```

---

## 13. Anti-Patterns & Critical Rules

### ❌ Never use SystemApi from REST layer
```java
// WRONG — exposes internal API externally
@FrameworkRestController(referredRestApi = ProductRestApi.class)
public class ProductRestControllerImpl {
    @Inject
    private ProductSystemApi systemApi; // WRONG
}

// CORRECT — use Api with permission checks
    @Inject
    private ProductApi productApi;      // CORRECT
```

---

### ❌ Never annotate SystemApi methods with @AllowPermissions
```java
// WRONG — SystemApi bypasses security by design
public interface ProductSystemApi extends BaseEntitySystemApi<Product> {
    @AllowPermissions(actions = {"save"}) // WRONG
    Product save(Product entity);
}
```

---

### ❌ Never include /water in @Path
```java
// WRONG
@Path("/water/api/products")  // → doubles to /water/water/api/products → 404

// CORRECT
@Path("/api/products")        // → /water/api/products ✓
```

---

### ❌ Never leave exampleField in generated entities
```java
// Remove ALL traces of exampleField:
// 1. Field declaration
// 2. @UniqueConstraint column reference
// 3. @EqualsAndHashCode entry
// 4. Constructor parameters
```

---

### ❌ Never edit .yo-rc.json manually
The generator uses `.yo-rc.json` as its state store. Editing it breaks future generator runs.

---

### ❌ Never run ./gradlew directly for builds
Always use `yo water:build --projects ModuleName` — it respects dependency ordering and framework conventions.

---

### ✅ Always add @Setter (Lombok) to @Inject fields
```java
@Inject
@Setter                   // Required for Water injection mechanism
private ProductApi api;
```

---

### ✅ Public Api methods MUST have permission annotations
Every method on `XxxApi` that extends `BaseEntityApi` must have at least one of:
- `@AllowPermissions`
- `@AllowRoles`
- `@AllowGenericPermissions`
- `@AllowLoggedUser`

---

## 14. Quick Reference Tables

### Entity Interfaces

| Interface | Package | Purpose |
|-----------|---------|---------|
| `Resource` | `it.water.core.api.model` | Root marker |
| `BaseEntity` | `it.water.core.api.model` | Persisted entity with id/version/dates |
| `AbstractEntity` | `it.water.repository.entity.model` | JPA base class to extend |
| `ProtectedEntity` | `it.water.core.api.permission` | Permission-controlled entity |
| `OwnedResource` | `it.water.core.api.entity.owned` | User-owned entity |
| `OwnedChildResource` | `it.water.core.api.entity.owned` | Child of owned entity |
| `SharedEntity` | `it.water.core.api.entity.shared` | Shareable owned entity |
| `ExpandableEntity` | `it.water.core.api.model` | Entity with dynamic extra fields |

### Service Interfaces

| Interface | Package | Has Permissions |
|-----------|---------|----------------|
| `Service` | `it.water.core.api.service` | — (marker only) |
| `BaseApi` | `it.water.core.api.service` | Yes (via interceptors) |
| `BaseSystemApi` | `it.water.core.api.service` | No |
| `BaseEntityApi<T>` | `it.water.core.api.service` | Yes — all methods intercepted |
| `BaseEntitySystemApi<T>` | `it.water.core.api.service` | No — trusted internal |

### Permission Annotations

| Annotation | Target | When to use |
|------------|--------|-------------|
| `@AllowPermissions` | METHOD | Entity-level permission check by action |
| `@AllowRoles` | METHOD | Role-based access only |
| `@AllowGenericPermissions` | METHOD | Permission check on resource name |
| `@AllowLoggedUser` | METHOD | Any authenticated user |
| `@AllowPermissionsOnReturn` | METHOD | Filter result set by permissions |
| `@AccessControl` | CLASS | Declare entity actions and default role mappings |

### Lifecycle Annotations

| Annotation | Target | When called |
|------------|--------|-------------|
| `@FrameworkComponent` | CLASS | Register as framework component |
| `@Inject` | FIELD | Inject dependency from ComponentRegistry |
| `@OnActivate` | METHOD | Component startup/activation |
| `@OnDeactivate` | METHOD | Component shutdown/deactivation |
| `@LogMethodExecution` | CLASS/METHOD | Log method calls automatically |

### REST Annotations

| Annotation | Target | Purpose |
|------------|--------|---------|
| `@FrameworkRestApi` | CLASS | JAX-RS REST interface marker (CXF/OSGi) |
| `@FrameworkRestController` | CLASS | Spring MVC REST controller marker |
| `@LoggedIn` | CLASS/METHOD | Require JWT authentication |

### Validation Annotations

| Annotation | Target | Purpose |
|------------|--------|---------|
| `@NotNullOnPersist` | FIELD/METHOD | Required field at persist time |
| `@NoMalitiusCode` | FIELD/METHOD | Prevent XSS/script injection on strings |