---
name: persistence-knowledge
description: Comprehensive knowledge base for Water Framework persistence layer, including Repository pattern, JPA implementation, QueryBuilder fluent API, transaction management, and SystemService integration. Use when designing, implementing, or debugging data access, queries, pagination, or transaction logic in Water modules.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework persistence architect with deep knowledge of the Repository abstraction layer, JPA implementation, QueryBuilder API, transaction management, and SystemService patterns. You use this knowledge to guide correct data access implementations, query building, and transactional design in Water modules.

---

# WATER FRAMEWORK PERSISTENCE KNOWLEDGE BASE

## Table of Contents

1. [Persistence Architecture Overview](#1-persistence-architecture-overview)
2. [Module Dependency Map](#2-module-dependency-map)
3. [BaseRepository Interface](#3-baserepository-interface)
4. [JpaRepository Extension](#4-jparepository-extension)
5. [AbstractJpaEntity Base Class](#5-abstractjpaentity-base-class)
6. [QueryBuilder Fluent API](#6-querybuilder-fluent-api)
7. [Query Operations Hierarchy](#7-query-operations-hierarchy)
8. [String-based Query Parser](#8-string-based-query-parser)
9. [PredicateBuilder - JPA Translation](#9-predicatebuilder---jpa-translation)
10. [Pagination & Ordering](#10-pagination--ordering)
11. [Transaction Management](#11-transaction-management)
12. [BaseEntitySystemServiceImpl](#12-baseentitysystemserviceimpl)
13. [BaseJpaRepositoryImpl Deep Dive](#13-basejparepositoryimpl-deep-dive)
14. [Spring JPA Integration](#14-spring-jpa-integration)
15. [Entity Extensions (Expandable Entity)](#15-entity-extensions-expandable-entity)
16. [Database Constraints Validation](#16-database-constraints-validation)
17. [Entity Lifecycle Events](#17-entity-lifecycle-events)
18. [Persistence Unit Configuration](#18-persistence-unit-configuration)
19. [Best Practices & Anti-Patterns](#19-best-practices--anti-patterns)
20. [Decision Trees](#20-decision-trees)
21. [Quick Reference Tables](#21-quick-reference-tables)

---

## 1. Persistence Architecture Overview

Water Framework implements a **technology-agnostic repository pattern** with a clear separation between query abstraction and JPA implementation.

### Layer Architecture

```
Api Layer (BaseEntityApi)
  |  annotated with security (@AllowPermissions, etc.)
  v
SystemService Layer (BaseEntitySystemServiceImpl)
  |  validation, events, asset management
  v
Repository Layer (BaseRepository / JpaRepository)
  |  CRUD operations, query execution, transactions
  v
JPA / EntityManager
  |  entity mapping, SQL generation
  v
Database
```

### Key Principle

- **Core-api** defines `BaseRepository`, `Query`, `QueryBuilder` interfaces (technology-agnostic)
- **Repository** module provides `DefaultQueryBuilder`, `QueryParser`, `BaseEntitySystemServiceImpl`
- **JpaRepository** module provides `BaseJpaRepositoryImpl`, `PredicateBuilder`, `AbstractJpaEntity`
- Application code NEVER directly touches `EntityManager` — always goes through Repository

---

## 2. Module Dependency Map

```
Core-api (interfaces: BaseRepository, Query, QueryBuilder, QueryOrder, PaginableResult)
  |
  +--- Repository-entity (exception classes: EntityNotFound, DuplicateEntityException, NoResultException)
  |
  +--- Repository-persistence (DefaultQueryBuilder, QueryParser)
  |         depends on: Core-api
  |
  +--- Repository-service (BaseEntitySystemServiceImpl)
  |         depends on: Core-api, Repository-entity
  |
  +--- JpaRepository-api (BaseJpaRepositoryImpl, AbstractJpaEntity, PredicateBuilder, JpaRepository)
  |         depends on: Core-api, Repository-entity, Repository-persistence
  |
  +--- JpaRepository-spring (SpringBaseJpaRepositoryImpl)
            depends on: JpaRepository-api, Spring Framework
```

---

## 3. BaseRepository Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/repository/BaseRepository.java`

```java
public interface BaseRepository<T extends BaseEntity> extends Service {
    // Entity type
    Class<T> getEntityType();

    // CREATE
    T persist(T entity);
    T persist(T entity, Runnable executeInTransaction);

    // UPDATE
    T update(T entity);
    T update(T entity, Runnable executeInTransaction);

    // DELETE
    void remove(long id);
    void remove(long id, Runnable executeInTransaction);
    void remove(T entity);
    void removeAllByIds(Iterable<Long> ids);
    void removeAll(Iterable<T> entities);
    void removeAll();

    // READ
    T find(long id);
    T find(Query filter);
    T find(String filterStr);
    PaginableResult<T> findAll(int delta, int page, Query filter, QueryOrder queryOrder);
    long countAll(Query filter);

    // Utility
    QueryBuilder getQueryBuilderInstance();
}
```

### Key Features

- **Transactional callbacks**: `persist(entity, runnable)` and `update(entity, runnable)` execute the `Runnable` within the SAME transaction as the main operation.
- **Three find modes**: by ID, by `Query` object, or by string filter expression.
- **QueryBuilder access**: Each repository provides a `QueryBuilder` for constructing type-safe queries.

---

## 4. JpaRepository Extension

**File:** `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/api/JpaRepository.java`

```java
public interface JpaRepository<T extends BaseEntity> extends BaseRepository<T> {
    EntityManager getEntityManager();

    // Execute code in transaction without result
    void txExpr(Transactional.TxType txType, Consumer<EntityManager> function);

    // Execute code in transaction with result
    <R> R tx(Transactional.TxType txType, Function<EntityManager, R> function);
}
```

### Transaction Types

| TxType | Behavior |
|--------|----------|
| `REQUIRED` | Uses existing transaction or creates new one (default) |
| `REQUIRES_NEW` | Always creates a new transaction |
| `SUPPORTS` | Uses existing transaction if available, no TX otherwise |
| `NOT_SUPPORTED` | Suspends current transaction |
| `MANDATORY` | Requires existing transaction, throws if none |
| `NEVER` | Throws if transaction exists |

### Custom Transaction Usage

```java
// Execute arbitrary code in a transaction
repository.tx(Transactional.TxType.REQUIRED, em -> {
    // em is a managed EntityManager
    MyEntity entity = em.find(MyEntity.class, id);
    entity.setStatus("processed");
    em.merge(entity);
    return entity;
});

// Execute without returning a result
repository.txExpr(Transactional.TxType.REQUIRED, em -> {
    em.createQuery("DELETE FROM MyEntity e WHERE e.status = 'expired'")
      .executeUpdate();
});
```

---

## 5. AbstractJpaEntity Base Class

**File:** `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/model/AbstractJpaEntity.java`

```java
@MappedSuperclass
@Embeddable
public abstract class AbstractJpaEntity extends AbstractEntity implements BaseEntity {
    @Id @GeneratedValue
    public long getId();

    @Version
    @Column(name = "entity_version")
    public Integer getEntityVersion();      // Optimistic locking

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "entity_create_date")
    public Date getEntityCreateDate();      // Auto-set on persist

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "entity_modify_date")
    public Date getEntityModifyDate();      // Auto-set on persist & update

    @PrePersist
    protected void prePersist() { ... }     // Sets dates automatically

    @PreUpdate
    protected void preUpdate() { ... }      // Updates modify date
}
```

### Built-in Features

- **Optimistic locking** via `@Version` — prevents concurrent modification
- **Automatic timestamps** — `entityCreateDate` and `entityModifyDate` managed by JPA lifecycle
- **Auto-generated IDs** — `@Id @GeneratedValue`
- **Extensible lifecycle hooks** — Override `prePersist()` / `preUpdate()` for custom logic

### Defining a New Entity

```java
@Entity
@Table(name = "my_entity")
@Access(AccessType.FIELD)
public class MyEntity extends AbstractJpaEntity {
    @Column @NotBlank
    private String name;

    @Column
    private int quantity;

    // getters/setters via Lombok or manual
}
```

---

## 6. QueryBuilder Fluent API

**Interface:** `Core/Core-api/src/main/java/it/water/core/api/repository/query/QueryBuilder.java`
**Implementation:** `Repository/Repository-persistence/src/main/java/it/water/repository/query/DefaultQueryBuilder.java`

```java
public interface QueryBuilder {
    Query createQueryFilter(String filter);     // Parse string filter
    FieldNameOperand field(String name);         // Start fluent query
}
```

### Getting a QueryBuilder

```java
// From Repository
QueryBuilder qb = repository.getQueryBuilderInstance();

// From SystemApi
QueryBuilder qb = systemApi.getQueryBuilderInstance();

// Standalone (for unit tests)
QueryBuilder qb = new DefaultQueryBuilder();
```

### Fluent API Usage

**FieldNameOperand** (`Core/Core-api/src/main/java/it/water/core/api/repository/query/operands/FieldNameOperand.java`) provides:

| Method | SQL Equivalent | Accepts |
|--------|---------------|---------|
| `equalTo(value)` | `field = value` | Object |
| `notEqualTo(value)` | `field <> value` | Object |
| `greaterThan(value)` | `field > value` | Number, Date |
| `greaterOrEqualThan(value)` | `field >= value` | Number, Date |
| `lowerThan(value)` | `field < value` | Number, Date |
| `lowerOrEqualThan(value)` | `field <= value` | Number, Date |
| `like(value)` | `field LIKE value` | String |

### Composing Queries

```java
QueryBuilder qb = repository.getQueryBuilderInstance();

// Simple equality
Query q = qb.field("name").equalTo("John");

// AND composition
Query q = qb.field("name").equalTo("John")
    .and(qb.field("age").greaterThan(25));

// OR composition
Query q = qb.field("status").equalTo("active")
    .or(qb.field("status").equalTo("pending"));

// Complex nested query
Query nameFilter = qb.field("name").like("%mario%");
Query ageFilter = qb.field("age").greaterThan(20)
    .and(qb.field("age").lowerThan(50));
Query combined = nameFilter.or(ageFilter);

// IN clause
Query q = qb.field("id").equalTo(1).in(List.of(1L, 2L, 3L));

// NOT
Query q = qb.field("deleted").equalTo(true).not();

// Nested field paths (JPA relationships)
Query q = qb.field("user.address.city").equalTo("Rome");
```

### Query Definition Output

Every `Query` has a `getDefinition()` method that returns a human-readable string:
```java
qb.field("name").equalTo("John").getDefinition()
// Returns: "name = John"

qb.field("age").greaterThan(25).and(qb.field("status").like("%active%")).getDefinition()
// Returns: "age > 25 AND status LIKE %active%"
```

---

## 7. Query Operations Hierarchy

**Base path:** `Core/Core-api/src/main/java/it/water/core/api/repository/query/operations/`

```
AbstractOperation (implements Query, QueryFilterOperation)
|
+-- UnaryOperation (1 operand)
|   +-- NotOperation          -> NOT
|
+-- BinaryOperation (2 operands)
    |
    +-- BinaryLogicOperation
    |   +-- AndOperation      -> AND
    |   +-- OrOperation       -> OR
    |
    +-- BinaryValueOperation (field + value)
    |   +-- EqualTo           -> =
    |   +-- NotEqualTo        -> <>
    |   +-- GreaterThan       -> >
    |   +-- GreaterOrEqualThan -> >=
    |   +-- LowerThan         -> <
    |   +-- LowerOrEqualThan  -> <=
    |   +-- Like              -> LIKE
    |
    +-- BinaryValueListOperation
        +-- In                -> IN
```

### Operand Types

| Class | Purpose |
|-------|---------|
| `FieldNameOperand` | Represents a field name (e.g., `"name"`, `"user.address"`) |
| `FieldValueOperand` | Represents a single value (e.g., `"John"`, `42`, `true`) |
| `FieldValueListOperand` | Represents a list of values for IN clause |
| `ParenthesisNode` | Groups nested expressions |

---

## 8. String-based Query Parser

**File:** `Repository/Repository-persistence/src/main/java/it/water/repository/query/parser/QueryParser.java`

Parses string filter expressions into `Query` objects using `StreamTokenizer`.

### Supported Syntax

```
<field> <operator> <value> [AND|OR <field> <operator> <value>]*

Operators: =, <>, >, >=, <, <=, LIKE, IN, NOT
Logic: AND, OR
Grouping: ( )
Values: 'quoted string', "double quoted", numeric, true/false
```

### Examples

```java
QueryBuilder qb = new DefaultQueryBuilder();

// Simple
Query q = qb.createQueryFilter("name = 'Mario'");

// Multiple conditions
Query q = qb.createQueryFilter("age > 20 AND name LIKE 'mario'");

// IN clause
Query q = qb.createQueryFilter("id IN (1,2,3)");

// Parentheses
Query q = qb.createQueryFilter("(age > 20 AND name LIKE 'mario') OR status = 'active'");

// Nested fields
Query q = qb.createQueryFilter("user.address.city = 'Rome'");

// Mixed
Query q = qb.createQueryFilter("ownerUserId = 52 OR id IN (452,454)");
```

### When to Use String vs Fluent API

| Use String Parser | Use Fluent API |
|-------------------|----------------|
| Filters from REST query parameters | Programmatic query building in code |
| User-defined dynamic filters | Type-safe queries in services |
| Simple expressions | Complex nested conditions |
| Test scenarios | Production service code |

---

## 9. PredicateBuilder - JPA Translation

**File:** `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/query/PredicateBuilder.java`

Converts Water `Query` objects into JPA `Predicate` objects for `CriteriaBuilder`.

### Translation Map

| Water Operation | JPA CriteriaBuilder Method |
|----------------|---------------------------|
| `AndOperation` | `cb.and(left, right)` |
| `OrOperation` | `cb.or(left, right)` |
| `NotOperation` | `cb.not(operand)` |
| `EqualTo` | `cb.equal(path, value)` / `cb.isNull(path)` for null |
| `NotEqualTo` | `cb.notEqual(path, value)` |
| `GreaterThan` | `cb.greaterThan(path, value)` |
| `GreaterOrEqualThan` | `cb.greaterThanOrEqualTo(path, value)` |
| `LowerThan` | `cb.lessThan(path, value)` |
| `LowerOrEqualThan` | `cb.lessThanOrEqualTo(path, value)` |
| `Like` | `cb.like(path, value)` |
| `In` | `path.in(values)` |

### Type Conversion

PredicateBuilder automatically converts value types:
- **String** → direct use
- **Number** (Integer, Long, Float, Double) → parsed from string
- **Boolean** → `Boolean.parseBoolean()`
- **Date types** → `java.util.Date`, `java.sql.Date`, `java.sql.Timestamp`, `Instant`, `LocalDateTime`, `LocalDate`
- **Enum** → tries `Enum.valueOf()` first, falls back to ordinal

### Nested Field Paths

Dot-separated field names are resolved through JPA `Path.get()`:
```
"user.address.city" → root.get("user").get("address").get("city")
```

---

## 10. Pagination & Ordering

### PaginableResult

**File:** `Core/Core-api/src/main/java/it/water/core/api/model/PaginableResult.java`

```java
public interface PaginableResult<T extends Resource> {
    int getNumPages();          // Total pages
    int getCurrentPage();       // Current page (1-based)
    int getNextPage();          // Next page (or 1 if last)
    int getDelta();             // Items per page
    Collection<T> getResults(); // Page results
}
```

### QueryOrder

**File:** `Core/Core-api/src/main/java/it/water/core/api/repository/query/QueryOrder.java`

```java
public interface QueryOrder {
    QueryOrder addOrderField(String name, boolean asc);
    List<QueryOrderParameter> getParametersList();
}
```

### Usage Examples

```java
QueryBuilder qb = repository.getQueryBuilderInstance();
Query filter = qb.field("status").equalTo("active");

// Page 1, 10 items per page, no ordering
PaginableResult<MyEntity> page1 = repository.findAll(10, 1, filter, null);

// With ordering (ascending by name)
QueryOrder order = new DefaultQueryOrder();
order.addOrderField("name", true);
PaginableResult<MyEntity> sorted = repository.findAll(10, 1, filter, order);

// Multiple order fields (descending by date, ascending by name)
QueryOrder order = new DefaultQueryOrder();
order.addOrderField("entityCreateDate", false);  // DESC
order.addOrderField("name", true);               // ASC
PaginableResult<MyEntity> results = repository.findAll(20, 2, filter, order);

// Count matching records
long total = repository.countAll(filter);

// Find single entity by filter
MyEntity entity = repository.find(qb.field("email").equalTo("john@example.com"));
```

### Pagination Notes

- `delta` = -1 means no pagination (return all results)
- `page` is 1-based
- `filter` can be null (no filtering)
- `queryOrder` can be null (no ordering)

---

## 11. Transaction Management

### Transaction Flow in BaseJpaRepositoryImpl

```
persist(entity) / update(entity)
  |
  v
tx(TxType.REQUIRED, em -> doPersist(entity, null, em))
  |
  v
startTransactionIfNeeded(em)
  |  - If isTransactionalSupported() == false (non-Spring):
  |      em.getTransaction().begin()
  |  - If Spring: no-op (Spring manages TX)
  v
dbConstraintsValidatorManager.runCheck(entity)    // Unique constraint check
  |
  v
em.persist(entity) / em.merge(entity)
  |
  v
doPersistOnExpandableEntity(entity)               // Handle extensions
  |
  v
commitTransactionIfNeeded(em)
  |  - If isTransactionalSupported() == false:
  |      em.getTransaction().commit()
  |  - If Spring: no-op (Spring commits at TX boundary)
  v
Return entity
  |
  (On RuntimeException: rollback if non-Spring)
```

### Transactional Callback Pattern

Execute additional operations within the SAME transaction as the entity save:

```java
// Save entity and create related records in the same TX
repository.persist(entity, () -> {
    // This runs INSIDE the same transaction
    relatedRepository.persist(relatedEntity);
    auditRepository.persist(auditRecord);
});
```

### Spring vs Non-Spring Transaction Handling

| Aspect | Spring | Non-Spring (OSGi/Standalone) |
|--------|--------|------------------------------|
| TX start | `TransactionTemplate` | `em.getTransaction().begin()` |
| TX commit | Automatic at TX boundary | `em.getTransaction().commit()` |
| TX rollback | Automatic on exception | Manual `em.getTransaction().rollback()` |
| `isTransactionalSupported()` | `true` | `false` |
| TX propagation | Spring `TransactionDefinition` | Manual management |

### Custom Transaction in Repository

```java
// Get result from transactional code
MyEntity result = repository.tx(Transactional.TxType.REQUIRED, em -> {
    MyEntity entity = em.find(MyEntity.class, id);
    entity.setProcessed(true);
    return em.merge(entity);
});

// Execute without result
repository.txExpr(Transactional.TxType.REQUIRES_NEW, em -> {
    em.createQuery("UPDATE MyEntity e SET e.status = 'archived' WHERE e.age > 365")
      .executeUpdate();
});
```

---

## 12. BaseEntitySystemServiceImpl

**File:** `Repository/Repository-service/src/main/java/it/water/repository/service/BaseEntitySystemServiceImpl.java`

The bridge between Api/SystemApi and Repository. All CRUD operations go through this class.

### Architecture

```java
public abstract class BaseEntitySystemServiceImpl<T extends BaseEntity>
    extends BaseAbstractSystemService implements BaseEntitySystemApi<T> {

    @Inject protected ComponentRegistry componentRegistry;
    @Inject protected WaterValidator waterValidator;

    // Subclasses MUST provide the repository
    protected abstract BaseRepository<T> getRepository();
}
```

### CRUD Operation Flow

**Save:**
```
save(entity)
  -> validate(entity)
  -> validateEntityExtension(entity)
  -> produceEvent(entity, PreSaveEvent.class)
  -> repository.persist(entity)
  -> manageAssets(entity, ADD)
  -> produceEvent(entity, PostSaveEvent.class)
  -> return entity
```

**Update:**
```
update(entity)
  -> validate(entity)
  -> validateEntityExtension(entity)
  -> entityBeforeUpdate = find(entity.getId())
  -> produceEvent(entity, PreUpdateEvent.class)
  -> produceDetailedEvent(before, entity, PreUpdateDetailedEvent.class)
  -> repository.update(entity)
  -> manageAssets(entity, UPDATE)
  -> produceDetailedEvent(before, updated, PostUpdateDetailedEvent.class)
  -> return updatedEntity
```

**Remove:**
```
remove(id)
  -> entity = find(id)
  -> if entity == null: throw EntityNotFound
  -> produceEvent(entity, PreRemoveEvent.class)
  -> repository.remove(id)
  -> manageAssets(entity, DELETE)
  -> produceEvent(entity, PostRemoveEvent.class)
```

### Implementing a SystemService

```java
@FrameworkComponent
public class MyEntitySystemServiceImpl
    extends BaseEntitySystemServiceImpl<MyEntity>
    implements MyEntitySystemApi {

    @Inject @Setter
    private MyEntityRepository repository;

    public MyEntitySystemServiceImpl() {
        super(MyEntity.class);
    }

    @Override
    protected BaseRepository<MyEntity> getRepository() {
        return repository;
    }

    // Custom business methods...
    public MyEntity findByEmail(String email) {
        QueryBuilder qb = getQueryBuilderInstance();
        return getRepository().find(
            qb.field("email").equalTo(email)
        );
    }
}
```

---

## 13. BaseJpaRepositoryImpl Deep Dive

**File:** `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/BaseJpaRepositoryImpl.java`

Core JPA repository (~735 lines) handling all persistence operations.

### Key Configuration

```java
public abstract class BaseJpaRepositoryImpl<T extends BaseEntity>
    implements JpaRepository<T> {

    // Persistence unit name (default: "water-default-persistence-unit")
    protected String persistenceUnitName;

    // Global EntityManager (OSGi)
    private static EntityManager globalEntityManager;

    // Instance EntityManager
    private EntityManager entityManager;
}
```

### EntityManager Resolution

The repository resolves `EntityManager` in this priority:
1. Instance-level `entityManager` (if set, e.g., by Spring)
2. Global static `entityManager` (if set, e.g., by OSGi)
3. Create new from `EntityManagerFactory` via persistence unit name

### Find with Filter

```java
@Override
public T find(Query filter) {
    return tx(Transactional.TxType.REQUIRED, em -> {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<T> cq = cb.createQuery(type);
        Root<T> root = cq.from(type);

        PredicateBuilder pb = new PredicateBuilder(root, cb, cq);
        Predicate predicate = pb.buildPredicate(filter);
        cq.select(root).where(predicate);

        TypedQuery<T> query = em.createQuery(cq);
        return query.getSingleResult();
    });
}
```

### FindAll with Pagination

```java
@Override
public PaginableResult<T> findAll(int delta, int page, Query filter, QueryOrder order) {
    return tx(Transactional.TxType.REQUIRED, em -> {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<T> cq = cb.createQuery(type);
        Root<T> root = cq.from(type);

        // Apply filter
        if (filter != null) {
            Predicate predicate = new PredicateBuilder(root, cb, cq).buildPredicate(filter);
            cq.select(root).where(predicate);
        }

        // Apply ordering
        if (order != null) {
            List<Order> orders = order.getParametersList().stream()
                .map(p -> p.isAsc() ? cb.asc(root.get(p.getName())) : cb.desc(root.get(p.getName())))
                .collect(Collectors.toList());
            cq.orderBy(orders);
        }

        TypedQuery<T> query = em.createQuery(cq);

        // Pagination
        if (delta > 0) {
            query.setFirstResult((page - 1) * delta);
            query.setMaxResults(delta);
        }

        // Build result
        List<T> results = query.getResultList();
        long count = countAll(filter);
        return new WaterPaginableResult<>(...);
    });
}
```

---

## 14. Spring JPA Integration

**File:** `JpaRepository/JpaRepository-spring/src/main/java/it/water/repository/jpa/spring/SpringBaseJpaRepositoryImpl.java`

### Spring-specific Features

```java
public abstract class SpringBaseJpaRepositoryImpl<T extends BaseEntity>
    extends BaseJpaRepositoryImpl<T> {

    @Autowired private PlatformTransactionManager transactionManager;
    @Autowired private EntityManagerFactory entityManagerFactory;
    private TransactionTemplate transactionTemplate;

    @PostConstruct
    void init() {
        transactionTemplate = new TransactionTemplate(transactionManager);
    }

    @Override
    public <R> R tx(Transactional.TxType txType, Function<EntityManager, R> function) {
        transactionTemplate.setPropagationBehavior(mapTxType(txType));
        return transactionTemplate.execute(status -> {
            EntityManager em = EntityManagerFactoryUtils
                .getTransactionalEntityManager(entityManagerFactory);
            return function.apply(em);
        });
    }
}
```

### TxType to Spring Propagation Mapping

| JPA TxType | Spring Propagation |
|------------|-------------------|
| `REQUIRED` | `PROPAGATION_REQUIRED` |
| `REQUIRES_NEW` | `PROPAGATION_REQUIRES_NEW` |
| `SUPPORTS` | `PROPAGATION_SUPPORTS` |
| `NEVER` | `PROPAGATION_NEVER` |
| `MANDATORY` | `PROPAGATION_MANDATORY` |
| `NOT_SUPPORTED` | `PROPAGATION_NOT_SUPPORTED` |

---

## 15. Entity Extensions (Expandable Entity)

**File:** `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/model/AbstractJpaExpandableEntity.java`

Entities can be extended at runtime using the Expandable Entity pattern:

```java
public interface ExpandableEntity {
    EntityExtension getExtension();
    void setExtension(EntityExtension extension);
    boolean isExpandableEntity();
}
```

- Extension data is stored in a separate table (one-to-one relationship)
- Extension is validated alongside the main entity
- Extension is persisted/updated within the same transaction

---

## 16. Database Constraints Validation

**Pre-persist validation** checks unique constraints BEFORE sending to the database:

```java
dbConstraintsValidatorManager.runCheck(entity, type, this);
```

This prevents duplicate entity exceptions by checking constraints proactively, providing clearer error messages via `DuplicateEntityException`.

---

## 17. Entity Lifecycle Events

`BaseEntitySystemServiceImpl` generates events at each CRUD stage:

| Event Class | When | Contains |
|------------|------|----------|
| `PreSaveEvent` | Before persist | New entity |
| `PostSaveEvent` | After persist | Saved entity (with ID) |
| `PreUpdateEvent` | Before update | Updated entity |
| `PreUpdateDetailedEvent` | Before update | Before + After entity |
| `PostUpdateDetailedEvent` | After update | Before + After entity |
| `PreRemoveEvent` | Before remove | Entity to remove |
| `PostRemoveEvent` | After remove | Removed entity |

Events are produced via `ApplicationEventProducer` (if registered). If no producer is registered, events are silently skipped.

---

## 18. Persistence Unit Configuration

### Default Persistence Unit

Name: `water-default-persistence-unit`

### Example persistence.xml

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence" version="3.0">
    <persistence-unit name="water-default-persistence-unit" transaction-type="RESOURCE_LOCAL">
        <class>it.water.mymodule.model.MyEntity</class>
        <properties>
            <property name="javax.persistence.jdbc.driver" value="org.hsqldb.jdbcDriver"/>
            <property name="javax.persistence.jdbc.url" value="jdbc:hsqldb:mem:testdb"/>
            <property name="javax.persistence.jdbc.user" value="sa"/>
            <property name="javax.persistence.jdbc.password" value=""/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.HSQLDialect"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.hbm2ddl.auto" value="create-drop"/>
        </properties>
    </persistence-unit>
</persistence>
```

### Custom Persistence Unit

Modules can define their own persistence unit by overriding `persistenceUnitName` in the repository:

```java
public class MyEntityRepositoryImpl extends BaseJpaRepositoryImpl<MyEntity> {
    public MyEntityRepositoryImpl() {
        super("my-custom-persistence-unit", MyEntity.class);
    }
}
```

---

## 19. Best Practices & Anti-Patterns

### Best Practices

1. **Always use QueryBuilder fluent API for programmatic queries.** It's type-safe and technology-agnostic.

2. **Use string parser for REST filters only.** Let the `QueryParser` handle user input from REST query parameters.

3. **Never access EntityManager directly in services.** Always go through `BaseRepository` or `JpaRepository` methods.

4. **Use transactional callbacks for multi-entity saves.**
   ```java
   repository.persist(parent, () -> {
       childRepository.persist(child);
   });
   ```

5. **Always extend AbstractJpaEntity for JPA entities.** It provides ID generation, optimistic locking, and timestamps.

6. **Implement custom finders in SystemServiceImpl, not in Api.** Keep business query logic in the system service layer.

7. **Use `TxType.REQUIRED` as default.** Only use `REQUIRES_NEW` when you need an independent transaction (e.g., audit logging that must persist even if the main TX fails).

8. **Handle `NoResultException` and `EntityNotFound`.** Repository throws these when `find()` returns no results.

9. **Use `countAll(filter)` before `findAll()` for accurate pagination.** The framework does this automatically in `BaseJpaRepositoryImpl`.

10. **Leverage `QueryOrder` for consistent sorting.** Don't rely on database default ordering.

### Anti-Patterns

1. **Calling `repository.removeAll()` in production.** This deletes ALL records without any filter.

2. **Building JPA CriteriaQuery manually in service code.** Always use QueryBuilder.

3. **Ignoring optimistic locking.** If you bypass `entityVersion`, concurrent updates may overwrite each other silently.

4. **Using `find(String filterStr)` with user input without sanitization.** The QueryParser has basic safety but isn't designed for arbitrary untrusted input.

5. **Creating multiple transactions when one would suffice.** Use the callback pattern instead of separate persist calls.

6. **Putting query logic in the Api layer.** The Api layer is for security-annotated methods; query building belongs in SystemService.

---

## 20. Decision Trees

### How to Query Data

```
Need to query entities?
  |
  +-- Single entity by ID?
  |     --> repository.find(id)
  |
  +-- Single entity by criteria?
  |     |
  |     +-- Programmatic (in service code)?
  |     |     --> qb.field("email").equalTo(email)
  |     |     --> repository.find(query)
  |     |
  |     +-- From REST parameter string?
  |           --> qb.createQueryFilter(filterString)
  |           --> repository.find(query)
  |
  +-- Multiple entities with pagination?
  |     --> repository.findAll(delta, page, filter, order)
  |
  +-- Just need the count?
        --> repository.countAll(filter)
```

### How to Handle Transactions

```
Need transactional behavior?
  |
  +-- Simple save/update/remove?
  |     --> Use repository.persist/update/remove (auto-managed TX)
  |
  +-- Save + related operations in same TX?
  |     --> repository.persist(entity, () -> { /* related ops */ })
  |
  +-- Custom EntityManager operations?
  |     --> repository.tx(TxType.REQUIRED, em -> { /* custom logic */ })
  |
  +-- Need independent TX (audit, logging)?
        --> repository.tx(TxType.REQUIRES_NEW, em -> { ... })
```

---

## 21. Quick Reference Tables

### BaseRepository Methods

| Method | Description | Returns |
|--------|-------------|---------|
| `persist(entity)` | Save new entity | Saved entity with ID |
| `persist(entity, runnable)` | Save + execute in same TX | Saved entity |
| `update(entity)` | Update existing entity | Updated entity |
| `update(entity, runnable)` | Update + execute in same TX | Updated entity |
| `remove(id)` | Delete by ID | void |
| `remove(entity)` | Delete by entity | void |
| `removeAll()` | Delete ALL records | void |
| `find(id)` | Find by ID | Entity or exception |
| `find(query)` | Find by Query | Entity or exception |
| `find(filterStr)` | Find by string filter | Entity or exception |
| `findAll(delta, page, filter, order)` | Paginated search | PaginableResult |
| `countAll(filter)` | Count matching | long |

### QueryBuilder Operations

| Fluent Method | Operator | String Syntax |
|--------------|----------|---------------|
| `field("x").equalTo(v)` | `=` | `x = v` |
| `field("x").notEqualTo(v)` | `<>` | `x <> v` |
| `field("x").greaterThan(v)` | `>` | `x > v` |
| `field("x").greaterOrEqualThan(v)` | `>=` | `x >= v` |
| `field("x").lowerThan(v)` | `<` | `x < v` |
| `field("x").lowerOrEqualThan(v)` | `<=` | `x <= v` |
| `field("x").like(v)` | `LIKE` | `x LIKE v` |
| `query.and(query2)` | `AND` | `... AND ...` |
| `query.or(query2)` | `OR` | `... OR ...` |
| `query.not()` | `NOT` | `NOT ...` |
| `query.in(list)` | `IN` | `x IN (1,2,3)` |

### Key File Paths

| Component | Path |
|-----------|------|
| BaseRepository interface | `Core/Core-api/src/main/java/it/water/core/api/repository/BaseRepository.java` |
| Query interface | `Core/Core-api/src/main/java/it/water/core/api/repository/query/Query.java` |
| QueryBuilder interface | `Core/Core-api/src/main/java/it/water/core/api/repository/query/QueryBuilder.java` |
| FieldNameOperand | `Core/Core-api/src/main/java/it/water/core/api/repository/query/operands/FieldNameOperand.java` |
| Query operations | `Core/Core-api/src/main/java/it/water/core/api/repository/query/operations/` |
| QueryOrder interface | `Core/Core-api/src/main/java/it/water/core/api/repository/query/QueryOrder.java` |
| PaginableResult | `Core/Core-api/src/main/java/it/water/core/api/model/PaginableResult.java` |
| BaseEntitySystemApi | `Core/Core-api/src/main/java/it/water/core/api/service/BaseEntitySystemApi.java` |
| DefaultQueryBuilder | `Repository/Repository-persistence/src/main/java/it/water/repository/query/DefaultQueryBuilder.java` |
| QueryParser | `Repository/Repository-persistence/src/main/java/it/water/repository/query/parser/QueryParser.java` |
| BaseEntitySystemServiceImpl | `Repository/Repository-service/src/main/java/it/water/repository/service/BaseEntitySystemServiceImpl.java` |
| BaseJpaRepositoryImpl | `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/BaseJpaRepositoryImpl.java` |
| AbstractJpaEntity | `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/model/AbstractJpaEntity.java` |
| JpaRepository interface | `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/api/JpaRepository.java` |
| PredicateBuilder | `JpaRepository/JpaRepository-api/src/main/java/it/water/repository/jpa/query/PredicateBuilder.java` |
| SpringBaseJpaRepositoryImpl | `JpaRepository/JpaRepository-spring/src/main/java/it/water/repository/jpa/spring/SpringBaseJpaRepositoryImpl.java` |
| Entity exceptions | `Repository/Repository-entity/src/main/java/it/water/repository/entity/model/exceptions/` |