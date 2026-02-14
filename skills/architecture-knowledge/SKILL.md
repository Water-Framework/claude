---
name: architecture-knowledge
description: Central knowledge base for Water Framework architecture, DDD patterns, microservices design, module construction best practices, and enterprise engineering patterns. Use when designing, reviewing, or building Water modules, services, or entire microservice architectures.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework architect with deep knowledge of Domain-Driven Design (DDD), microservices architecture, and enterprise Java patterns. You use this knowledge to guide design decisions, review code, and help build well-structured Water Framework modules.

---

# WATER FRAMEWORK ARCHITECTURE KNOWLEDGE BASE

## Table of Contents

1. [Framework Philosophy & Pillars](#1-framework-philosophy--pillars)
2. [Architectural Overview](#2-architectural-overview)
3. [Core Module Deep Dive](#3-core-module-deep-dive)
4. [Domain-Driven Design in Water](#4-domain-driven-design-in-water)
5. [Microservices Patterns](#5-microservices-patterns)
6. [Module Construction Guide](#6-module-construction-guide)
7. [Entity Modeling Patterns](#7-entity-modeling-patterns)
8. [Service Layer Architecture](#8-service-layer-architecture)
9. [Repository & Persistence Patterns](#9-repository--persistence-patterns)
10. [REST API Design](#10-rest-api-design)
11. [Security & Permission System](#11-security--permission-system)
12. [Interceptor & Cross-Cutting Concerns](#12-interceptor--cross-cutting-concerns)
13. [Component Lifecycle & Registry](#13-component-lifecycle--registry)
14. [Distribution & Deployment](#14-distribution--deployment)
15. [Testing Strategy](#15-testing-strategy)
16. [Anti-Patterns to Avoid](#16-anti-patterns-to-avoid)
17. [Decision Trees](#17-decision-trees)

---

## 1. Framework Philosophy & Pillars

Water Framework is a **cross-framework** solution: it allows writing modular applications that run on different Java runtimes (Spring, OSGi, Quarkus) while providing out-of-the-box enterprise features.

> *"Empty your mind, be formless. Shapeless, like water."* - The framework takes the form of its container.

### Three Pillars

| Pillar | Purpose |
|--------|---------|
| **Water Framework** | Core abstractions, security, permissions, event streaming, entity management |
| **Water Generator** | Yeoman-based scaffolding, dependency analysis, stability metrics, build automation |
| **Runtimes** | OSGi (Karaf), Spring Boot, Quarkus, Standalone - same code, multiple targets |

### Convention Over Coding

Water goes beyond "convention over configuration" by defining:
- **Structural conventions**: Model/API/Service layer separation enforced by module structure
- **Code conventions**: Annotations (`@FrameworkComponent`, `@Inject`, `@OnActivate`) replace XML/YAML configuration
- **Build conventions**: Generator manages dependency ordering, circular dependency detection, and stability metrics
- **Interface remoting**: Automatic REST endpoint generation from service interfaces

---

## 2. Architectural Overview

### Layered Architecture

```
                    +-----------------------+
                    |     REST Layer        |  RestApi, @FrameworkRestApi
                    |  (Rest module)        |  @LoggedIn, JWT, JAX-RS
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |    API Layer          |  BaseEntityApi<T>
                    | (Permission-checked)  |  @AllowPermissions, @AllowRoles
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |  System API Layer     |  BaseEntitySystemApi<T>
                    | (Internal, no perms)  |  Validation, Events
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |  Repository Layer     |  BaseRepository<T>
                    | (Data access)         |  JPA, QueryBuilder, Pagination
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |  Database / Store     |
                    +-----------------------+
```

### Module Dependency Graph

```
Core-api (foundation - all interfaces)
    |
    +---> Core-bundle, Core-interceptors, Core-model, Core-permission,
    |     Core-registry, Core-security, Core-service, Core-validation
    |
    +---> Repository-entity, Repository-persistence, Repository-service
    |
    +---> JpaRepository-api
    |         +---> JpaRepository-spring
    |         +---> JpaRepository-osgi
    |
    +---> Rest-api, Rest-service, Rest-persistence, Rest-security
    |         +---> Rest-spring-api
    |         +---> Rest-jaxrs-api (CXF)
    |
    +---> SharedEntity-model, SharedEntity-api, SharedEntity-service
    |
    +---> Implementation-spring
    +---> Implementation-osgi
    |
    +---> Distribution-spring, Distribution-osgi, Distribution-karaf, Distribution-quarkus
```

### Key Principle: Abstraction Stability

Water enforces the **Stable Abstractions Principle (SAP)**:
- Modules with high abstraction (interfaces/APIs) should be **stable** (many dependents)
- Modules with implementations should be **unstable** (few dependents, easy to change)
- The generator calculates stability metrics to verify this

---

## 3. Core Module Deep Dive

### Core Submodules (11 total)

| Submodule | Purpose |
|-----------|---------|
| **Core-api** | All interfaces and abstract contracts (foundation) |
| **Core-bundle** | Component initialization, runtime configuration |
| **Core-interceptors** | Method interception framework (AOP-like) |
| **Core-model** | Base data models, exceptions, error handling |
| **Core-permission** | Permission management, action-based access control |
| **Core-registry** | Component discovery, lifecycle management |
| **Core-security** | SecurityContext, authentication, encryption |
| **Core-service** | Abstract service implementations |
| **Core-testing-utils** | Mock implementations for testing |
| **Core-validation** | Jakarta Validation integration |

### Service Interface Hierarchy

```
Service (marker - root abstraction)
  +-- BaseApi (user-facing)
  |     +-- BaseEntityApi<T> (CRUD with permission checks)
  +-- BaseSystemApi (system-facing, internal)
  |     +-- BaseEntitySystemApi<T> (CRUD without permission checks)
  +-- RestApi (REST endpoint marker)
  +-- BaseRepository<T> (data access)
  +-- PermissionManager (permission engine)
  +-- SecurityContext (current user state)
  +-- Runtime (runtime environment access)
```

### Key Distinction: Api vs SystemApi vs Repository

| Layer | Interface | Permission Checks | Validation | Events | Use Case |
|-------|-----------|-------------------|------------|--------|----------|
| **Api** | `BaseEntityApi<T>` | YES (@AllowPermissions) | Delegated | Delegated | External/user calls |
| **SystemApi** | `BaseEntitySystemApi<T>` | NO | YES | YES | Internal/framework calls |
| **Repository** | `BaseRepository<T>` | NO | NO | NO | Pure data access |

---

## 4. Domain-Driven Design in Water

### Bounded Context = Water Module

Each Water module represents a **Bounded Context** in DDD terms:

```
my-module/
  my-module-model/     --> Domain Model (Entities, Value Objects, Aggregates)
  my-module-api/       --> Domain Services (interfaces), Repository contracts
  my-module-service/   --> Application Services (implementations), Infrastructure
```

### Entity Classification (DDD Tactical Patterns)

Water provides interfaces that map directly to DDD concepts:

| Water Interface | DDD Concept | Purpose |
|-----------------|-------------|---------|
| `BaseEntity` | Entity | Identity + lifecycle (id, version, timestamps) |
| `ProtectedEntity` | Entity + Access Control | Entity with permission checks |
| `OwnedResource` | Entity + Ownership | Entity belonging to a specific user (Aggregate Root ownership) |
| `OwnedChildResource` | Child Entity | Entity owned through parent relationship |
| `SharedEntity` | Shared Aggregate | Entity shareable across users |
| `ExpandableEntity` | Extensible Entity | Entity with runtime field expansion (Open/Closed Principle) |
| `EntityExtension` | Entity Extension | Additional fields for an expandable entity |

### Aggregate Design Rules in Water

1. **One Aggregate Root per module**: The primary entity in `*-model` is the Aggregate Root
2. **Child entities reference parent via ID**: Use `OwnedChildResource` with `getRootParentFieldPath()`
3. **Cross-aggregate references by ID only**: Never embed entities from other modules directly
4. **Entity Extensions for open aggregates**: Use `ExpandableEntity` + `EntityExtension` instead of modifying the root entity class
5. **Shared access through SharedEntity**: Cross-user access is managed by `WaterSharedEntity` records, not by copying data

### Value Objects

Water entities use JPA `@Embeddable` for value objects:
```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
}
```

### Domain Events

Water provides a built-in event system:
```
PreSaveEvent / PostSaveEvent
PreUpdateEvent / PostUpdateEvent
PreUpdateDetailedEvent / PostUpdateDetailedEvent (carries before/after state)
PreRemoveEvent / PostRemoveEvent
```

Produced via `ApplicationEventProducer` and consumed via `ApplicationEventListener<T>`.

---

## 5. Microservices Patterns

### Pattern 1: Database per Service

Each Water module has its own persistence unit:
```java
// In RepositoryImpl
public MyEntityRepositoryImpl() {
    super(MyEntity.class, "my-module-persistence-unit");
}
```

This ensures data isolation between bounded contexts.

### Pattern 2: API Gateway / REST Composition

Water REST modules expose fine-grained endpoints. Compose at the API gateway level:
- Each module exposes its own `@Path("/resource")` endpoints
- JWT tokens propagate across services
- `SharedEntityIntegrationClient` enables cross-service entity sharing queries

### Pattern 3: Integration Clients

Water uses **Integration Client** pattern for cross-service communication:

```java
// Defined in Core-api
public interface UserIntegrationClient extends EntityIntegrationClient {
    User fetchUserByEmailAddress(String email);
    User fetchUserByUsername(String username);
}

public interface SharedEntityIntegrationClient extends EntityIntegrationClient {
    Collection<Long> fetchSharingUsersIds(String entityResourceName, long entityId);
}
```

Local implementations provided by default; remote implementations can override via higher priority.

### Pattern 4: Strangler Fig (Incremental Migration)

Water's priority-based component registry enables incremental replacement:
```java
// Default implementation (priority 1)
@FrameworkComponent(services = MyService.class, priority = 1)
public class DefaultMyService implements MyService { }

// Override with higher priority
@FrameworkComponent(services = MyService.class, priority = 10)
public class CustomMyService implements MyService { }
```

### Pattern 5: Modulith to Microservices

Water supports the full spectrum:

| Architecture | Water Configuration |
|-------------|---------------------|
| **Monolith** | All modules in single WAR/JAR |
| **Modulith** | Modules deployed together, logical separation |
| **Microservices** | Each module as independent deployable |
| **Hybrid** | Some modules co-deployed, others independent |

Configuration via `water.layer` property: `microservices` or `modulith`.

### Pattern 6: Cross-Framework Portability

Write once, deploy on any runtime:
```java
@FrameworkComponent(services = OrderService.class)
public class OrderServiceImpl implements OrderService {
    @Inject private OrderRepository repository;
    @Inject private ComponentRegistry registry;

    @OnActivate
    public void init() { /* lifecycle hook */ }
}
```

This code runs identically on Spring, OSGi, and Quarkus.

### Pattern 7: Saga Pattern (via Events)

Use Water's event system for distributed transactions:
```java
@FrameworkComponent(services = ApplicationEventListener.class)
public class OrderSagaListener implements ApplicationEventListener<Order> {
    @Override
    public void consumerEvent(Order order, Event event) {
        if (event instanceof PostSaveEvent) {
            // Trigger next step in saga
            inventoryService.reserve(order.getItems());
        }
    }
}
```

---

## 6. Module Construction Guide

### Standard Module Structure

```
my-module/
  build.gradle                    # Parent build
  settings.gradle                 # Module includes
  gradle.properties               # Versions, framework properties
  .yo-rc.json                     # Generator config (DO NOT manually edit)

  my-module-model/                # Domain Model Layer
    src/main/java/
      com/example/mymodule/model/
        MyEntity.java             # JPA entity (@Entity)
        MyEntityPK.java           # Composite PK (if needed)

  my-module-api/                  # API Contract Layer
    src/main/java/
      com/example/mymodule/api/
        MyEntityApi.java          # Public API (extends BaseEntityApi<T>)
        MyEntitySystemApi.java    # System API (extends BaseEntitySystemApi<T>)
        MyEntityRepository.java   # Repository contract (extends BaseRepository<T>)
        rest/
          MyEntityRestApi.java    # REST contract (extends RestApi)

  my-module-service/              # Implementation Layer
    src/main/java/
      com/example/mymodule/
        service/
          MyEntityServiceImpl.java        # Api implementation
          MyEntitySystemServiceImpl.java  # SystemApi implementation
          rest/
            MyEntityRestControllerImpl.java   # REST controller
        repository/
          MyEntityRepositoryImpl.java     # Repository implementation
    src/test/java/
      com/example/mymodule/
        MyEntityApiTest.java              # API tests
        MyEntityRestApiTest.java          # REST tests
    src/test/resources/karate/
        MyEntity-crud.feature             # Karate integration tests
```

### Module Creation via Generator

```bash
# Entity module with full CRUD + REST
yo water:new-project --inlineArgs \
  --projectName=order-management \
  --projectTechnology=water \
  --applicationType=entity \
  --modelName=Order \
  --isProtectedEntity=true \
  --isOwnedEntity=true \
  --hasRestServices=true \
  --restContextRoot=/orders \
  --hasAuthentication=true

# Service module (no persistence)
yo water:new-project --inlineArgs \
  --projectName=notification-service \
  --projectTechnology=water \
  --applicationType=service \
  --hasModel=false \
  --hasRestServices=true \
  --restContextRoot=/notifications

# Add entity to existing project
yo water:add-entity --inlineArgs \
  --projectName=order-management \
  --modelName=OrderItem \
  --isProtectedEntity=true \
  --isOwnedEntity=false
```

### Layer Dependency Rules

```
model  <--  api  <--  service
  |           |          |
  |           |          +-- Depends on: model, api, JpaRepository, Core-*
  |           +-- Depends on: model, Core-api
  +-- Depends on: Core-api (minimal)
```

**RULE**: Dependencies flow inward (model has fewest, service has most). Never reverse.

---

## 7. Entity Modeling Patterns

### Base Entity (AbstractJpaEntity)

All persistent entities extend `AbstractJpaEntity`:

```java
@MappedSuperclass
public abstract class AbstractJpaEntity implements BaseEntity {
    @Id @GeneratedValue
    private long id;

    @Version
    @Column(name = "entity_version", columnDefinition = "INTEGER default 1")
    private Integer entityVersion;            // Optimistic locking

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "entity_create_date")
    private Date entityCreateDate;            // Auto-set on @PrePersist

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "entity_modify_date")
    private Date entityModifyDate;            // Auto-set on @PrePersist and @PreUpdate
}
```

### Protected Entity Pattern

For entities requiring permission checks:

```java
@Entity
@Table(uniqueConstraints = @UniqueConstraint(columnNames = "code"))
@AccessControl(
    availableActions = {CrudActions.SAVE, CrudActions.UPDATE, CrudActions.FIND,
                        CrudActions.FIND_ALL, CrudActions.REMOVE},
    rolesPermissions = {
        @AccessControlRole(roleName = "orderManager",
            actions = {CrudActions.SAVE, CrudActions.UPDATE, CrudActions.FIND,
                       CrudActions.FIND_ALL, CrudActions.REMOVE}),
        @AccessControlRole(roleName = "orderViewer",
            actions = {CrudActions.FIND, CrudActions.FIND_ALL})
    }
)
public class Order extends AbstractJpaEntity implements ProtectedEntity, OwnedResource {

    private String code;
    private BigDecimal total;

    @Column(name = "owner_user_id")
    private Long ownerUserId;

    @Override
    public Long getOwnerUserId() { return ownerUserId; }

    @Override
    public void setOwnerUserId(Long userId) { this.ownerUserId = userId; }

    @Override
    public String getResourceName() { return Order.class.getName(); }
}
```

### Shareable Entity Pattern

For entities that can be shared across users:

```java
@Entity
public class Document extends AbstractJpaEntity
    implements SharedEntity, ProtectedEntity {

    @Column(name = "owner_user_id")
    private Long ownerUserId;

    private String title;
    private String content;

    // SharedEntity extends OwnedResource
    @Override
    public Long getOwnerUserId() { return ownerUserId; }

    @Override
    public void setOwnerUserId(Long userId) { this.ownerUserId = userId; }
}
```

Sharing is managed by `WaterSharedEntity` records (composite PK: entityResourceName + entityId + userId).

### Expandable Entity Pattern

For entities supporting runtime field expansion:

```java
@Entity
public class Product extends AbstractJpaExpandableEntity implements ProtectedEntity {

    private String name;
    private BigDecimal price;

    // Extra fields automatically serialized/deserialized via @JsonAnyGetter/@JsonAnySetter
    // EntityExtension persisted separately via extension repository
}
```

### Owned Child Entity Pattern

For child entities whose ownership derives from parent:

```java
@Entity
public class OrderItem extends AbstractJpaEntity
    implements OwnedChildResource, ProtectedEntity {

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;

    private String productName;
    private int quantity;

    @Override
    public Object getParent() { return order; }

    @Override
    public String getRootParentFieldPath() { return "order.id"; }
}
```

### Composite Primary Key Pattern

For entities with composite keys (e.g., SharedEntity):

```java
@Entity
@IdClass(MyCompositePK.class)
public class MyJoinEntity extends AbstractJpaEntity {
    @Id private String fieldA;
    @Id private Long fieldB;
    @Id private Long fieldC;
}

public class MyCompositePK implements Serializable {
    private String fieldA;
    private Long fieldB;
    private Long fieldC;
}
```

### JSON Serialization Views

Control JSON output per context:

```java
@JsonView({
    WaterJsonView.Extended.class,   // Full detail
    WaterJsonView.Compact.class,    // Summary
    WaterJsonView.Internal.class,   // Internal use
    WaterJsonView.Privacy.class,    // Privacy-filtered
    WaterJsonView.Public.class      // Public-facing
})
private String name;

@JsonView(WaterJsonView.Internal.class)  // Only visible in internal context
private String internalCode;
```

---

## 8. Service Layer Architecture

### Three-Layer Implementation Pattern

#### 1. API Layer (User-facing, permission-checked)

```java
@FrameworkComponent(services = OrderApi.class)
public class OrderServiceImpl extends BaseEntityServiceImpl<Order> implements OrderApi {

    @Inject @Setter
    private OrderSystemApi systemService;

    @Inject @Setter
    private ComponentRegistry componentRegistry;

    public OrderServiceImpl() {
        super(Order.class);
    }

    @Override
    protected BaseEntitySystemApi<Order> getSystemService() {
        return systemService;
    }

    @Override
    protected ComponentRegistry getComponentRegistry() {
        return componentRegistry;
    }

    // Custom business methods with permission checks
    @AllowPermissions(actions = {CrudActions.UPDATE})
    public Order confirmOrder(Order order) {
        order.setStatus("CONFIRMED");
        return systemService.update(order);
    }
}
```

**BaseEntityServiceImpl provides automatically:**
- `save(T entity)` - with `@AllowPermissions(actions = {CrudActions.SAVE})`
- `update(T entity)` - with `@AllowPermissions(actions = {CrudActions.UPDATE})`
- `remove(long id)` - with `@AllowPermissions(actions = CrudActions.REMOVE, checkById = true)`
- `find(long id)` - with `@AllowPermissions(actions = CrudActions.FIND, checkById = true)`
- `find(Query filter)` - with `@AllowGenericPermissions` + `@AllowPermissionsOnReturn`
- `findAll(...)` - with `@AllowGenericPermissions(actions = CrudActions.FIND_ALL)`
- `countAll(Query filter)` - with `@AllowGenericPermissions(actions = CrudActions.FIND)`

**Automatic ownership filtering**: For `OwnedResource`/`SharedEntity`, queries are automatically filtered:
- Non-admin users see only their own resources + resources shared with them
- Admin users see all resources

#### 2. System API Layer (Internal, no permission checks)

```java
@FrameworkComponent(services = OrderSystemApi.class)
public class OrderSystemServiceImpl extends BaseEntitySystemServiceImpl<Order>
    implements OrderSystemApi {

    @Inject @Setter
    private OrderRepository repository;

    public OrderSystemServiceImpl() {
        super(Order.class);
    }

    @Override
    protected BaseRepository<Order> getRepository() {
        return repository;
    }
}
```

**BaseEntitySystemServiceImpl provides automatically:**
- Validation before save/update (via `WaterValidator`)
- Event production (Pre/Post Save/Update/Remove events)
- Entity extension management (for `ExpandableEntity`)
- Query builder access

#### 3. Repository Layer (Data access)

```java
@FrameworkComponent(services = OrderRepository.class)
public class OrderRepositoryImpl extends WaterJpaRepositoryImpl<Order>
    implements OrderRepository {

    public OrderRepositoryImpl() {
        super(Order.class, "order-persistence-unit");
    }

    // Custom query methods
    public List<Order> findByStatus(String status) {
        Query filter = getQueryBuilderInstance()
            .field("status").equalTo(status);
        return findAll(100, 1, filter, null).getResults().stream().toList();
    }
}
```

### Service Communication Patterns

#### Intra-module (same bounded context)
```
Api --> SystemApi --> Repository
```
Direct injection, same transaction context.

#### Inter-module (cross bounded context)
```
ModuleA.Api --> IntegrationClient --> ModuleB.Api
```
Use `IntegrationClient` interfaces. Default: local implementation. Remote: override with higher priority.

---

## 9. Repository & Persistence Patterns

### JPA Repository Architecture

```
BaseRepository<T> (Core-api interface)
    |
    +-- BaseJpaRepositoryImpl<T> (JpaRepository-api, technology-agnostic)
         |
         +-- SpringBaseJpaRepositoryImpl<T> (Spring TransactionTemplate)
         +-- OsgiBaseJpaRepository<T> (JTA/Narayana transactions)
```

### Query Building

#### String-based (simple filters from REST)
```java
Query filter = queryBuilder.createQueryFilter("age > 18 AND city LIKE 'Rome'");
Query complex = queryBuilder.createQueryFilter(
    "(status = 'ACTIVE' OR status = 'PENDING') AND amount > 100");
```

#### Fluent API (type-safe in code)
```java
QueryBuilder qb = repository.getQueryBuilderInstance();
Query filter = qb.field("status").equalTo("ACTIVE")
    .and(qb.field("amount").greaterThan(100))
    .or(qb.field("priority").equalTo("HIGH"));
```

#### Available Operations
| Operation | String Syntax | Fluent API |
|-----------|--------------|------------|
| Equal | `field = 'value'` | `.equalTo(value)` |
| Not Equal | `field != 'value'` | `.notEqualTo(value)` |
| Greater Than | `field > 10` | `.greaterThan(10)` |
| Greater Or Equal | `field >= 10` | `.greaterOrEqualThan(10)` |
| Less Than | `field < 10` | `.lowerThan(10)` |
| Less Or Equal | `field <= 10` | `.lowerOrEqualThan(10)` |
| Like | `field like 'pat%'` | `.like("pat%")` |
| In | - | `.in(List.of(1,2,3))` |
| AND | `expr AND expr` | `.and(query)` |
| OR | `expr OR expr` | `.or(query)` |
| NOT | - | `.not()` |

### Pagination

```java
// Repository level
PaginableResult<Order> result = repository.findAll(
    /*delta*/ 20,        // Items per page
    /*page*/  1,         // Page number (1-indexed)
    /*filter*/ query,
    /*order*/ queryOrder
);

// Query ordering
DefaultQueryOrder order = new DefaultQueryOrder();
order.addOrderField("createdDate", false);  // DESC
order.addOrderField("name", true);          // ASC

// Result access
result.getResults();       // Collection<T>
result.getCurrentPage();   // Current page number
result.getNumPages();      // Total pages
result.getNextPage();      // Next page (-1 if last)
result.getDelta();         // Items per page
```

### Transaction Management

| Runtime | Mechanism | Transaction Types |
|---------|-----------|-------------------|
| **Spring** | `TransactionTemplate` | REQUIRED, REQUIRES_NEW, SUPPORTS, NEVER, MANDATORY, NOT_SUPPORTED |
| **OSGi** | JTA (Narayana/ArjunaTS) | Same propagation types with suspend/resume |
| **Base** | Resource-local | Manual begin/commit/rollback |

```java
// Execute within transaction (repository level)
repository.persist(entity, () -> {
    // This runs in the same transaction
    auditRepository.persist(auditEntry);
});

// Explicit transaction control (JpaRepository)
repository.txExpr(Transactional.TxType.REQUIRES_NEW, em -> {
    em.persist(entity);
    em.flush();
});
```

### Constraint Validation

Automatic duplicate detection via `@UniqueConstraint`:
```java
@Table(uniqueConstraints = {
    @UniqueConstraint(columnNames = "code"),
    @UniqueConstraint(columnNames = {"tenantId", "email"})
})
```

The `DuplicateConstraintValidator` automatically:
1. Reads `@UniqueConstraint` from `@Table`
2. Builds query for each constraint
3. Checks for existing records with same values
4. Throws `DuplicateEntityException` with field names

### Persistence Unit Configuration

Water avoids `persistence.xml` via programmatic configuration (`WaterPersistenceUnitInfo`):
- Entity class list discovery
- Transaction type selection (JTA or RESOURCE_LOCAL)
- Hibernate properties configuration
- DataSource binding

---

## 10. REST API Design

### REST Module Architecture

```
Rest-api           --> Core interfaces, security annotations
Rest-service       --> Base implementations, Jackson config
Rest-persistence   --> BaseEntityRestApi<T> (CRUD REST base)
Rest-security      --> JWT tokens, authentication filters
Rest-jaxrs-api     --> JAX-RS default endpoints (CXF)
Rest-spring-api    --> Spring controllers and config
Rest-api-manager-apache-cxf  --> CXF server management
```

### Two-Tier REST Pattern

**1. API Interface (Contract)**
```java
@Path("/orders")
@FrameworkRestApi
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public interface OrderRestApi extends RestApi {

    @POST
    @LoggedIn
    @JsonView(WaterJsonView.Public.class)
    Response save(Order entity);

    @PUT
    @LoggedIn
    @JsonView(WaterJsonView.Public.class)
    Response update(Order entity);

    @GET @Path("/{id}")
    @LoggedIn
    @JsonView(WaterJsonView.Public.class)
    Response find(@PathParam("id") long id);

    @GET
    @LoggedIn
    @JsonView(WaterJsonView.Public.class)
    Response findAll(
        @QueryParam("delta") Integer delta,
        @QueryParam("page") Integer page);

    @DELETE @Path("/{id}")
    @LoggedIn
    Response remove(@PathParam("id") long id);

    // Custom endpoint
    @POST @Path("/{id}/confirm")
    @LoggedIn
    Response confirmOrder(@PathParam("id") long id);
}
```

**2. Controller Implementation**
```java
@FrameworkRestController(referredRestApi = OrderRestApi.class)
public class OrderRestControllerImpl extends BaseEntityRestApi<Order>
    implements OrderRestApi {

    @Inject @Setter
    private OrderApi orderApi;

    @Override
    protected BaseEntityApi<Order> getEntityService() {
        return orderApi;
    }

    @Override
    public Response confirmOrder(long id) {
        try {
            Order order = orderApi.find(id);
            Order confirmed = orderApi.confirmOrder(order);
            return Response.ok(confirmed).build();
        } catch (Exception e) {
            return handleException(e);
        }
    }
}
```

### JWT Authentication

- **@LoggedIn** annotation on REST methods triggers JWT validation
- Token from `Authorization: Bearer <token>` header or `HIT-AUTH` cookie
- `JwtTokenService` handles generation and validation
- `JwtSecurityContext` stores authenticated principals

### Error Handling

Automatic exception mapping to HTTP status codes:

| Exception | HTTP Status |
|-----------|-------------|
| `DuplicateEntityException` | 409 CONFLICT |
| `ValidationException` | 422 UNPROCESSABLE ENTITY |
| `EntityNotFound` / `NoResultException` | 404 NOT FOUND |
| `UnauthorizedException` | 401 UNAUTHORIZED |
| `RuntimeException` | 500 INTERNAL SERVER ERROR |

### Default Pagination

REST `findAll` uses default page size of 20 items. Parameters:
- `delta` - items per page (default: 20)
- `page` - page number (default: 1)

### Spring vs CXF

| Aspect | Spring | CXF (OSGi) |
|--------|--------|-------------|
| Controllers | `@RestController` | JAX-RS resources |
| Auth Filter | `HandlerInterceptor` | `ContainerRequestFilter` |
| Server | Embedded Tomcat | CXF Server |
| Config | `WaterRestSpringConfiguration` | `CxfRestApiManager` |
| Mapping | Spring MVC | JAX-RS |

---

## 11. Security & Permission System

### Action-Based Permission Model

Actions use **power-of-2 encoding** for bitwise operations:
```
SAVE     = 1  (2^0)
UPDATE   = 2  (2^1)
REMOVE   = 4  (2^2)
FIND     = 8  (2^3)
FIND_ALL = 16 (2^4)
SHARE    = 32 (2^5)
```

User permissions stored as bitmask: `SAVE | FIND | FIND_ALL = 1 + 8 + 16 = 25`

### Permission Annotations

| Annotation | When | What |
|------------|------|------|
| `@AllowPermissions` | Before method | Check permission on entity (optionally by ID) |
| `@AllowGenericPermissions` | Before method | Check permission on resource class (no instance) |
| `@AllowPermissionsOnReturn` | After method | Check permission on returned entity |
| `@AllowRoles` | Before method | Require specific role names |
| `@AllowLoggedUser` | Before method | Require authenticated user |

### @AccessControl on Entities

```java
@AccessControl(
    availableActions = {
        CrudActions.SAVE, CrudActions.UPDATE, CrudActions.FIND,
        CrudActions.FIND_ALL, CrudActions.REMOVE
    },
    rolesPermissions = {
        @AccessControlRole(
            roleName = "orderManager",
            actions = {CrudActions.SAVE, CrudActions.UPDATE, CrudActions.FIND,
                       CrudActions.FIND_ALL, CrudActions.REMOVE}
        ),
        @AccessControlRole(
            roleName = "orderViewer",
            actions = {CrudActions.FIND, CrudActions.FIND_ALL}
        )
    }
)
public class Order extends AbstractJpaEntity implements ProtectedEntity { }
```

### Permission Checking Flow

```
User Request with JWT
    |
    v
[SecurityContext filled with Principals]
    |
    v
[Permission Annotation Interceptor]
    |-- @AllowPermissions: PermissionManager.checkPermission(username, entity, action)
    |-- @AllowRoles: PermissionManager.userHasRoles(username, roleNames)
    |-- @AllowLoggedUser: SecurityContext.isLoggedIn()
    |-- @AllowGenericPermissions: PermissionManager.checkPermission(username, className, action)
    |
    v
[If denied: throw UnauthorizedException -> 401]
[If allowed: proceed to method execution]
```

### Ownership Filtering (Automatic)

`BaseEntityServiceImpl` automatically applies ownership filters:

```
findAll(filter) called by non-admin user
    |
    v
Is entity OwnedResource?
    |-- YES: filter AND (ownerUserId = currentUser.id)
    |          |
    |          Is entity SharedEntity?
    |          |-- YES: OR (id IN sharedEntityIds)
    |          |-- NO: just ownership filter
    |
    |-- NO: return original filter (no ownership filtering)
```

---

## 12. Interceptor & Cross-Cutting Concerns

### Interceptor Architecture

```
Method Call --> Proxy/AOP
    |
    +--> BeforeMethodInterceptor.interceptMethod()    [method annotations]
    +--> BeforeMethodFieldInterceptor.interceptMethod() [field annotations]
    |
    +--> Actual Method Execution
    |
    +--> AfterMethodInterceptor.interceptMethod()     [with result]
    +--> AfterMethodFieldInterceptor.interceptMethod()  [with result]
    |
    v
Return Result
```

### Implementation by Runtime

| Runtime | Mechanism |
|---------|-----------|
| **OSGi** | Java Dynamic Proxies (`InvocationHandler`) + ServiceHooks |
| **Spring** | AspectJ AOP (`@Aspect`, `@Before`, `@AfterReturning`) |

### Custom Interceptor Pattern

**Step 1: Define annotation**
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@InterceptorExecutor(interceptor = AuditInterceptor.class)
public @interface Auditable {
    String action();
}
```

**Step 2: Implement interceptor**
```java
@FrameworkComponent(services = BeforeMethodInterceptor.class)
public class AuditInterceptor implements BeforeMethodInterceptor<Auditable> {

    @Override
    public <S extends Service> void interceptMethod(
            S destination, Method m, Object[] args, Auditable annotation) {
        // Pre-processing logic
        log.info("Audit: {} on {}", annotation.action(), m.getName());
    }

    @Override
    public Class<Auditable> getAnnotation() {
        return Auditable.class;
    }
}
```

**Step 3: Use annotation**
```java
@Auditable(action = "ORDER_CREATION")
public Order createOrder(Order order) { ... }
```

### Built-in Interceptors

- **Permission interceptors**: `@AllowPermissions`, `@AllowRoles`, `@AllowLoggedUser`
- **Logging**: `@LogMethodExecution`, `@LogMethodExecutionTime`
- **Security**: JWT validation via `@LoggedIn`

---

## 13. Component Lifecycle & Registry

### Component Discovery

Water uses **Atteo ClassIndex** for compile-time component indexing:
- No runtime classpath scanning
- `@FrameworkComponent` classes indexed at compile time
- Near-instant discovery at startup

### Registration Flow

```
1. [Discovery] ClassIndex finds @FrameworkComponent classes
2. [Instantiation] Create instance, read properties/priority
3. [Registration] Store in ComponentRegistry with ComponentConfiguration
4. [Injection] Resolve @Inject fields from registry
5. [Activation] Execute @OnActivate methods (params auto-injected)
6. [Active] Component ready for use
7. [Deactivation] Execute @OnDeactivate methods
8. [Removal] Unregister from ComponentRegistry
```

### @FrameworkComponent

```java
@FrameworkComponent(
    services = {OrderApi.class, OrderService.class},  // Exposed interfaces
    properties = {"module=order-management"},          // Registration properties
    priority = 1                                       // Default priority
)
public class OrderServiceImpl implements OrderApi, OrderService { }
```

### Dependency Injection (@Inject)

```java
@Inject(injectOnceAtStartup = true)   // Static injection (global singletons)
private ComponentRegistry registry;

@Inject(injectOnceAtStartup = false)  // Dynamic injection (re-resolved each access)
private SomeService dynamicService;
```

### Lifecycle Hooks

```java
@OnActivate
public void initialize(ComponentRegistry registry, ApplicationProperties props) {
    // Parameters auto-injected from registry by type
    String dbUrl = props.getPropertyOrDefault("db.url", "jdbc:h2:mem:test");
}

@OnDeactivate
public void cleanup() {
    // Called before component removal
}
```

### Priority-Based Override

Higher priority wins when multiple components implement same interface:
```java
// Framework default (priority 1)
@FrameworkComponent(services = EmailService.class, priority = 1)
public class DefaultEmailService implements EmailService { }

// Custom override (priority 10)
@FrameworkComponent(services = EmailService.class, priority = 10)
public class SmtpEmailService implements EmailService { }

// Registry returns SmtpEmailService for EmailService lookups
```

### Component Filtering

```java
ComponentFilterBuilder builder = registry.getComponentFilterBuilder();
ComponentFilter filter = builder.createFilter("module", "order-management")
    .and("version", "2.0");
List<Service> services = registry.findComponents(Service.class, filter);
```

---

## 14. Distribution & Deployment

### Distribution Strategies

| Distribution | Packaging | Deployment |
|-------------|-----------|------------|
| **OSGi/Karaf** | ShadowJar + Karaf features XML | Karaf container with features |
| **Spring Boot** | ShadowJar (fat JAR) | `java -jar app.jar` |
| **Quarkus** | Gradle build / Native image | `java -jar` or native binary |

### OSGi/Karaf Distribution

**Feature Hierarchy:**
```
water-core-features (base)
    +-- water-jpa-repository (extends core)
        +-- water-rest (extends jpa-repository)
            +-- water-core-features-test (extends rest, for testing)
```

**Start Levels (boot order):**
- Level 78: water-distribution-osgi (core framework)
- Level 79: JpaRepository-osgi
- Level 80-81: REST services
- Level 83: Test bundles

**Key Configuration Files (Karaf `etc/`):**
- `it.water.application.cfg` - Framework settings (testMode, layer, nodeId, keystore)
- `org.ops4j.datasource-water.cfg` - Database configuration
- `org.apache.cxf.osgi.cfg` - REST context path
- `org.apache.karaf.management.cfg` - JMX settings

### Spring Boot Distribution

Single fat JAR with all framework modules:
```bash
java -jar Water-Distribution-spring.jar
java -jar Water-Distribution-spring.jar --spring.profiles.active=production
```

Configuration via `application.properties` / `application.yml`.

### ShadowJar Merging

All distributions use ShadowJar to merge:
- Framework core modules
- Implementation modules (Spring or OSGi)
- `META-INF/annotations/it.water.core.interceptors.annotations.FrameworkComponent` files

---

## 15. Testing Strategy

### Test Utilities (Core-testing-utils)

Water provides `Core-testing-utils` with:
- Default mock implementations
- Test component registry
- Test security contexts
- JUnit 5 extension

### Unit Testing Pattern

```java
@ExtendWith(WaterJUnit5Extension.class)
public class OrderApiTest {

    @Inject
    private ComponentRegistry registry;

    @Inject
    private OrderApi orderApi;

    @Test
    void testCreateOrder() {
        Order order = new Order();
        order.setCode("ORD-001");
        order.setTotal(new BigDecimal("99.99"));

        Order saved = orderApi.save(order);

        assertNotNull(saved.getId());
        assertEquals("ORD-001", saved.getCode());
    }

    @Test
    void testPermissionDenied() {
        // Without proper security context
        assertThrows(UnauthorizedException.class, () -> {
            orderApi.find(1L);
        });
    }
}
```

### Integration Testing

**OSGi (PaxExam + Karaf):**
```java
@RunWith(PaxExam.class)
public class OrderOsgiTest {
    @Configuration
    public Option[] config() {
        return karafDistributionConfiguration()
            .features(waterFeatures, "water-rest")
            .unpackDirectory(new File("target/exam"))
            .useDeployFolder(false);
    }
}
```

**Spring Boot:**
```java
@SpringBootTest
@EnableWaterFramework
public class OrderSpringTest {
    @Autowired
    private OrderApi orderApi;

    @Test
    void testCrud() { /* ... */ }
}
```

### REST Integration Testing (Karate)

```gherkin
Feature: Order CRUD

  Background:
    * url baseUrl + '/orders'
    * header Authorization = 'Bearer ' + authToken

  Scenario: Create and retrieve order
    Given request { code: 'ORD-001', total: 99.99 }
    When method POST
    Then status 200
    And match response.code == 'ORD-001'

    Given path response.id
    When method GET
    Then status 200
    And match response.code == 'ORD-001'
```

---

## 16. Anti-Patterns to Avoid

### 1. Bypassing the Service Layer
```java
// WRONG: Calling repository directly from REST controller
@GET @Path("/{id}")
public Response find(@PathParam("id") long id) {
    return Response.ok(repository.find(id)).build();  // No permissions!
}

// CORRECT: Go through Api layer
@GET @Path("/{id}")
public Response find(@PathParam("id") long id) {
    return Response.ok(orderApi.find(id)).build();  // Permission-checked
}
```

### 2. Circular Module Dependencies
```
// WRONG: ModuleA depends on ModuleB AND ModuleB depends on ModuleA
// The generator detects this and fails the build

// CORRECT: Use Integration Clients for cross-module communication
// ModuleA --> IntegrationClient interface (in Core-api)
// ModuleB --> implements IntegrationClient
```

### 3. Embedding Entities from Other Modules
```java
// WRONG: Direct entity reference across bounded contexts
@ManyToOne
private User user;  // User is from another module!

// CORRECT: Reference by ID
@Column(name = "user_id")
private Long userId;
```

### 4. Putting Business Logic in Repository
```java
// WRONG: Business rules in repository
public class OrderRepositoryImpl {
    public Order createOrder(Order order) {
        if (order.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ValidationException("Invalid total");
        }
        return persist(order);
    }
}

// CORRECT: Business rules in SystemApi/Api
public class OrderSystemServiceImpl {
    public Order save(Order order) {
        validate(order);  // Validation in service layer
        return repository.persist(order);
    }
}
```

### 5. Ignoring Priority-Based Override
```java
// WRONG: Creating parallel registration mechanism
ServiceLocator.register(MyService.class, new CustomImpl());

// CORRECT: Use framework priority
@FrameworkComponent(services = MyService.class, priority = 10)
public class CustomImpl implements MyService { }
```

### 6. Manual Transaction Management in Services
```java
// WRONG: Manual transaction handling in service code
entityManager.getTransaction().begin();
// ...
entityManager.getTransaction().commit();

// CORRECT: Use repository transaction helpers
repository.persist(entity, () -> {
    // Additional operations in same transaction
});
```

### 7. Skipping the Generator
```bash
# WRONG: Manually creating project structure
mkdir my-module && mkdir my-module-model && ...

# CORRECT: Use the generator
yo water:new-project --inlineArgs --projectName=my-module ...
```

---

## 17. Decision Trees

### New Module Decision Tree

```
Need a new module?
  |
  +-- Has persistence (entities)?
  |     |-- YES --> applicationType=entity
  |     |     |-- Needs permission control? --> isProtectedEntity=true
  |     |     |-- Entities owned by users? --> isOwnedEntity=true
  |     |     |-- Entities shareable? --> SharedEntity interface + moreModules=shared-entity-integration
  |     |     +-- Entities extensible? --> ExpandableEntity interface
  |     |
  |     +-- NO --> applicationType=service
  |           |-- Needs its own model? --> hasModel=true
  |           +-- Pure integration? --> hasModel=false
  |
  +-- Needs REST API?
  |     |-- YES --> hasRestServices=true
  |     |     |-- Needs authentication? --> hasAuthentication=true
  |     |     +-- Custom context root? --> restContextRoot=/path
  |     +-- NO --> hasRestServices=false
  |
  +-- Technology choice?
        |-- Cross-framework (portable) --> projectTechnology=water
        |-- Spring-only --> projectTechnology=spring
        |-- OSGi-only --> projectTechnology=osgi
        +-- Cloud-native --> projectTechnology=quarkus
```

### Entity Design Decision Tree

```
Designing an entity?
  |
  +-- Needs access control?
  |     |-- YES --> implements ProtectedEntity
  |     |     |-- Per-user ownership? --> implements OwnedResource
  |     |     |-- Child of owned entity? --> implements OwnedChildResource
  |     |     +-- Shareable across users? --> implements SharedEntity
  |     +-- NO --> just extend AbstractJpaEntity
  |
  +-- Needs runtime extensions?
  |     |-- YES --> extends AbstractJpaExpandableEntity
  |     +-- NO --> extends AbstractJpaEntity
  |
  +-- Has composite key?
  |     |-- YES --> @IdClass + separate PK class
  |     +-- NO --> @Id @GeneratedValue (auto)
  |
  +-- Has unique constraints?
        |-- YES --> @Table(uniqueConstraints = ...)
        +-- NO --> default table mapping
```

### Permission Design Decision Tree

```
Securing a method?
  |
  +-- Check permission on specific entity?
  |     |-- By entity ID parameter? --> @AllowPermissions(checkById=true, idParamIndex=0)
  |     |-- By entity parameter? --> @AllowPermissions(actions=...)
  |     +-- On return value? --> @AllowPermissionsOnReturn(actions=...)
  |
  +-- Check permission on resource class (generic)?
  |     --> @AllowGenericPermissions(actions=...)
  |
  +-- Require specific roles?
  |     --> @AllowRoles(rolesNames={"admin","manager"})
  |
  +-- Require any authenticated user?
        --> @AllowLoggedUser
```

---

## Quick Reference Card

### Annotations Cheat Sheet

| Annotation | Target | Purpose |
|------------|--------|---------|
| `@FrameworkComponent` | Class | Register as framework component |
| `@Inject` | Field | Dependency injection |
| `@OnActivate` | Method | Post-construction lifecycle hook |
| `@OnDeactivate` | Method | Pre-destruction lifecycle hook |
| `@AllowPermissions` | Method | Entity-level permission check |
| `@AllowGenericPermissions` | Method | Class-level permission check |
| `@AllowPermissionsOnReturn` | Method | Permission check on return value |
| `@AllowRoles` | Method | Role-based access restriction |
| `@AllowLoggedUser` | Method | Require authentication |
| `@AccessControl` | Class | Define entity permissions and roles |
| `@FrameworkRestApi` | Class | Mark as REST API interface |
| `@FrameworkRestController` | Class | Mark as REST controller impl |
| `@LoggedIn` | Method | Require JWT authentication on REST |
| `@InterceptorExecutor` | Annotation | Link annotation to interceptor |
| `@LogMethodExecution` | Method | Log method entry/exit |
| `@LogMethodExecutionTime` | Method | Log method duration |

### Key Interfaces Cheat Sheet

| Interface | Module | Purpose |
|-----------|--------|---------|
| `Service` | Core-api | Root marker for all services |
| `BaseEntityApi<T>` | Core-api | User-facing CRUD with permissions |
| `BaseEntitySystemApi<T>` | Core-api | Internal CRUD without permissions |
| `BaseRepository<T>` | Core-api | Data access abstraction |
| `RestApi` | Core-api | REST endpoint marker |
| `ComponentRegistry` | Core-api | Component discovery and lifecycle |
| `PermissionManager` | Core-api | Permission checking engine |
| `SecurityContext` | Core-api | Current authentication state |
| `Runtime` | Core-api | Runtime environment access |
| `ApplicationProperties` | Core-api | Configuration access |
| `JpaRepository<T>` | JpaRepository-api | JPA-specific repository |

### Standard CRUD Actions

| Action | Bitmask | Description |
|--------|---------|-------------|
| `SAVE` | 1 | Create new entity |
| `UPDATE` | 2 | Modify existing entity |
| `REMOVE` | 4 | Delete entity |
| `FIND` | 8 | Read single entity |
| `FIND_ALL` | 16 | Read entity collection |
| `SHARE` | 32 | Share entity with other users |