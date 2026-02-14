---
name: authorization-knowledge
description: Comprehensive knowledge base for Water Framework security, authorization, permission system, security annotations, interceptors, and access control patterns. Use when designing, implementing, reviewing, or debugging any security or permission-related feature in Water modules.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework security architect with deep knowledge of the authorization system, permission model, security annotations, interceptors, and access control patterns. You use this knowledge to guide secure design decisions, review security implementations, and help build properly secured Water Framework modules.

---

# WATER FRAMEWORK AUTHORIZATION & SECURITY KNOWLEDGE BASE

## Table of Contents

1. [Security Architecture Overview](#1-security-architecture-overview)
2. [Module Dependency Map](#2-module-dependency-map)
3. [Security Annotations Reference](#3-security-annotations-reference)
4. [Interceptor System Deep Dive](#4-interceptor-system-deep-dive)
5. [Core Security Interfaces](#5-core-security-interfaces)
6. [SecurityContext & Principal Model](#6-securitycontext--principal-model)
7. [PermissionUtil Implementation](#7-permissionutil-implementation)
8. [Action System & Bitwise Permission Model](#8-action-system--bitwise-permission-model)
9. [DefaultActionsManager & Action Registration](#9-defaultactionsmanager--action-registration)
10. [Permission Entity Model](#10-permission-entity-model)
11. [Permission APIs](#11-permission-apis)
12. [PermissionManager Implementation Deep Dive](#12-permissionmanager-implementation-deep-dive)
13. [Ownership & Sharing Authorization](#13-ownership--sharing-authorization)
14. [Authentication System](#14-authentication-system)
15. [Encryption Utilities](#15-encryption-utilities)
16. [Security Exceptions & Error Handling](#16-security-exceptions--error-handling)
17. [Best Practices & Anti-Patterns](#17-best-practices--anti-patterns)
18. [Decision Trees](#18-decision-trees)
19. [Testing Security](#19-testing-security)
20. [Quick Reference Tables](#20-quick-reference-tables)

---

## 1. Security Architecture Overview

Water Framework implements a **layered, annotation-driven security model** built on four core modules that separate interfaces from implementations, annotations from interceptors, and permission logic from persistence.

### Security Request Flow

```
HTTP Request
  |
  v
REST Controller (@LoggedIn on JAX-RS endpoint)
  |
  v
JWT Validation -> SecurityContext creation (UserPrincipal + RolePrincipals)
  |
  v
Runtime.fillSecurityContext(ctx) -> Thread-bound context
  |
  v
Api Method invocation (annotated with @AllowPermissions / @AllowRoles / etc.)
  |
  v
Interceptor Framework detects annotation -> invokes matching BeforeMethodInterceptor
  |
  v
Interceptor calls PermissionUtil / PermissionManager
  |
  +-- checkPermission() -> bitwise AND on actionIds
  +-- checkUserOwnsResource() -> OwnedResource / SharedEntity
  +-- userHasRoles() -> role name matching
  |
  v
ALLOW (method executes) --or-- DENY (UnauthorizedException -> HTTP 403)
  |
  v
AfterMethodInterceptor (e.g., @AllowPermissionsOnReturn) validates return value
  |
  v
Response
```

### Four Security Modules

| Module | Role | Key Packages |
|--------|------|-------------|
| **Core-api** | Interfaces & contracts | `it.water.core.api.permission`, `it.water.core.api.security` |
| **Core-permission** | Annotations + Action classes + Exceptions | `it.water.core.permission.annotations`, `it.water.core.permission.action`, `it.water.core.permission.exceptions` |
| **Core-security** | Interceptor implementations + SecurityContext models + PermissionUtil impl | `it.water.core.security.annotations.implementation`, `it.water.core.security.model`, `it.water.core.security.util` |
| **Permission** (module) | WaterPermission entity, PermissionManagerDefault, REST API, persistence | `it.water.permission.model`, `it.water.permission.manager`, `it.water.permission.service`, `it.water.permission.api` |

---

## 2. Module Dependency Map

```
Core-api (foundation - interfaces only)
  |
  +--- Core-permission (annotations, action classes, exceptions)
  |         depends on: Core-api
  |
  +--- Core-security (interceptors, context models, PermissionUtil impl)
  |         depends on: Core-api, Core-permission
  |
  +--- Permission-model (WaterPermission JPA entity)
  |         depends on: Core-api, Core-permission, JpaRepository
  |
  +--- Permission-api (PermissionApi, PermissionSystemApi, PermissionRepository, PermissionRestApi)
  |         depends on: Core-api, Permission-model
  |
  +--- Permission-service (service implementations)
  |         depends on: Permission-api, Core-security
  |
  +--- Permission-manager (PermissionManagerDefault)
  |         depends on: Core-api, Core-permission, Permission-api
  |
  +--- Permission-service-spring (Spring REST controller)
            depends on: Permission-api
```

### Key Principle

- **Core-api** defines *what* security does (interfaces)
- **Core-permission** defines *how to declare* security (annotations)
- **Core-security** defines *how to enforce* security (interceptors)
- **Permission module** defines *where* security data lives (entity + persistence)

---

## 3. Security Annotations Reference

All annotations are in package `it.water.core.permission.annotations`.
**Base path:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/`

### 3.1 @AccessControl (TYPE-level, configuration)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/AccessControl.java`

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface AccessControl {
    String[] availableActions() default {};
    DefaultRoleAccess[] rolesPermissions() default {};
}
```

**Purpose:** Applied to entity classes to define which actions are available and what default role permissions to create at startup.

**Attributes:**
| Attribute | Type | Description |
|-----------|------|-------------|
| `availableActions` | `String[]` | Action names in order. **Position determines actionId**: `actionId = 2^position` |
| `rolesPermissions` | `DefaultRoleAccess[]` | Default role-to-action mappings created at application startup |

**Critical Rule:** The ORDER of `availableActions` matters because each action's numeric ID is calculated as `Math.pow(2, position)`. Changing the order after deployment breaks existing permission records.

**Example (from WaterPermission entity):**
```java
@AccessControl(
    availableActions = {
        CrudActions.SAVE,               // position 0 -> actionId = 1
        CrudActions.UPDATE,             // position 1 -> actionId = 2
        CrudActions.FIND,               // position 2 -> actionId = 4
        CrudActions.FIND_ALL,           // position 3 -> actionId = 8
        CrudActions.REMOVE,             // position 4 -> actionId = 16
        PermissionsActions.GIVE_PERMISSIONS,  // position 5 -> actionId = 32
        PermissionsActions.LIST_ACTIONS       // position 6 -> actionId = 64
    },
    rolesPermissions = {
        @DefaultRoleAccess(roleName = "permissionManager",
            actions = {CrudActions.SAVE, CrudActions.UPDATE, CrudActions.FIND,
                       CrudActions.FIND_ALL, CrudActions.REMOVE,
                       PermissionsActions.GIVE_PERMISSIONS, PermissionsActions.LIST_ACTIONS}),
        @DefaultRoleAccess(roleName = "permissionViewer",
            actions = {CrudActions.FIND, CrudActions.FIND_ALL, PermissionsActions.LIST_ACTIONS}),
        @DefaultRoleAccess(roleName = "permissionEditor",
            actions = {CrudActions.SAVE, CrudActions.UPDATE, CrudActions.FIND,
                       CrudActions.FIND_ALL, PermissionsActions.LIST_ACTIONS})
    }
)
public class WaterPermission extends AbstractJpaEntity implements Permission, ProtectedEntity { ... }
```

**Note:** `@IndexAnnotated` triggers compile-time discovery by Atteo ClassIndex, so the framework automatically finds all `@AccessControl`-annotated classes at startup.

### 3.2 @DefaultRoleAccess (used inside @AccessControl)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/DefaultRoleAccess.java`

```java
@Target({ElementType.LOCAL_VARIABLE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DefaultRoleAccess {
    String roleName() default "";
    String[] actions() default {};
}
```

**Purpose:** Associates a role name with a set of action names. At startup, `DefaultActionsManager` creates the role (if not existing) and adds the specified permissions.

### 3.3 @AllowPermissions (METHOD-level, BeforeMethodInterceptor)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/AllowPermissions.java`

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowPermissions {
    String[] actions() default {};
    boolean checkById() default false;
    int idParamIndex() default 0;
    String systemApiRef() default "";
}
```

**Purpose:** Entity-specific permission check. Verifies the user has permission to perform the specified action(s) on a specific entity instance.

**Attributes:**
| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `actions` | `String[]` | `{}` | Action names to check (e.g., `{CrudActions.SAVE}`) |
| `checkById` | `boolean` | `false` | If true, finds entity by ID from method parameter |
| `idParamIndex` | `int` | `0` | Index of the method parameter containing the entity ID (used when `checkById=true`) |
| `systemApiRef` | `String` | `""` | Fully qualified class name of a `BaseEntitySystemApi` to use for entity lookup (for non-entity APIs) |

**Usage patterns:**
```java
// Pattern 1: Entity passed as parameter (default)
@AllowPermissions(actions = {CrudActions.SAVE})
public MyEntity save(MyEntity entity) { ... }

// Pattern 2: Entity found by ID
@AllowPermissions(actions = {CrudActions.FIND}, checkById = true, idParamIndex = 0)
public MyEntity find(long id) { ... }

// Pattern 3: Entity found by ID with custom SystemApi
@AllowPermissions(actions = {CrudActions.REMOVE}, checkById = true, idParamIndex = 0,
    systemApiRef = "it.water.mymodule.api.MyEntitySystemApi")
public void removeFromExternalApi(long entityId) { ... }
```

**Rules:**
- MUST be placed on `BaseApi` (Api) class methods only. Throws `WaterRuntimeException` if placed on SystemApi or other services.
- When `checkById=true`, the parameter at `idParamIndex` MUST be of type `long`.

### 3.4 @AllowGenericPermissions (METHOD-level, BeforeMethodInterceptor)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/AllowGenericPermissions.java`

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowGenericPermissions {
    String[] actions() default {};
    String resourceName() default "";
    String resourceParamName() default "";
}
```

**Purpose:** Resource-level (generic) permission check WITHOUT entity ID. Checks if the user has permission on the resource class, not a specific entity instance.

**Attributes:**
| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `actions` | `String[]` | `{}` | Action names to check |
| `resourceName` | `String` | `""` | Explicit resource name (fully qualified class name) |
| `resourceParamName` | `String` | `""` | Name of method parameter containing resource name |

**Resource name resolution order:**
1. If `resourceName` is set → use it directly
2. If `resourceParamName` is set → extract from method parameter
3. If service is `BaseEntityApi` → auto-infer from `entityApi.getEntityType().getName()`
4. If none match and service is NOT `BaseEntityApi` → throws `WaterRuntimeException`

**Usage patterns:**
```java
// Pattern 1: On entity API (auto-infers resource name)
@AllowGenericPermissions(actions = {CrudActions.FIND_ALL})
public PaginableResult<MyEntity> findAll() { ... }

// Pattern 2: Explicit resource name (non-entity service)
@AllowGenericPermissions(actions = {"export"}, resourceName = "it.water.mymodule.model.MyEntity")
public byte[] exportData() { ... }
```

**Rules:**
- MUST be placed on `BaseApi` class methods only.
- For non-`BaseEntityApi` services, MUST provide either `resourceName` or `resourceParamName`.

### 3.5 @AllowRoles (METHOD-level, BeforeMethodInterceptor)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/AllowRoles.java`

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowRoles {
    String[] rolesNames() default {};
}
```

**Purpose:** Role-based access control. Verifies the current user has at least ONE of the specified roles.

**Usage:**
```java
@AllowRoles(rolesNames = {"permissionManager", "systemAdmin"})
public void dangerousOperation() { ... }
```

**Rules:**
- MUST be placed on `BaseApi` class methods only.
- MUST provide at least one role name. Throws `WaterRuntimeException` if empty.
- Use sparingly. Prefer fine-grained `@AllowPermissions` or `@AllowGenericPermissions`.

### 3.6 @AllowLoggedUser (METHOD-level, BeforeMethodInterceptor)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/AllowLoggedUser.java`

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowLoggedUser {
}
```

**Purpose:** Marker annotation. Simply checks that the user is authenticated (has a valid SecurityContext with a non-zero `loggedEntityId`).

**Usage:**
```java
@AllowLoggedUser
public Map<String, Map<String, Map<String, Boolean>>> entityPermissionMap(
    Map<String, List<Long>> entityPks) { ... }
```

**Rules:**
- Does NOT check on which class it is applied (no `checkAnnotationIsOnWaterApiClass` call).
- Throws `UnauthorizedException("No security context found, please login")` if:
  - `currentRuntime` is null, OR
  - `currentRuntime.getSecurityContext()` is null, OR
  - `currentRuntime.getSecurityContext().getLoggedEntityId() == 0`

### 3.7 @AllowPermissionsOnReturn (METHOD-level, AfterMethodInterceptor)

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/annotations/AllowPermissionsOnReturn.java`

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowPermissionsOnReturn {
    String[] actions() default {};
}
```

**Purpose:** Post-execution interceptor. Validates that the user has permission on the entity RETURNED by the method.

**Usage:**
```java
@AllowPermissionsOnReturn(actions = {CrudActions.FIND})
public MyEntity findByCustomQuery(Query filter) { ... }
```

**Rules:**
- MUST be placed on `BaseApi` class methods only.
- If return value is `null` → passes through (no check).
- If return value is not `BaseEntity` → throws `WaterRuntimeException`.
- Only works on `BaseEntityApi` subclasses.
- Currently does NOT support collection return types.

---

## 4. Interceptor System Deep Dive

All interceptors are in package `it.water.core.security.annotations.implementation`.
**Base path:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/`

### 4.1 AbstractPermissionInterceptor (base class)

**File:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/AbstractPermissionInterceptor.java`

Provides common infrastructure for all security interceptors.

**Injected dependencies:**
- `Runtime waterRuntime` — access to SecurityContext
- `PermissionUtil waterPermissionUtil` — simplified permission checking
- `ComponentRegistry componentRegistry` — component lookup
- `ActionsManager actionsManager` — registered action lookup

**Key utility methods:**

| Method | Description |
|--------|-------------|
| `getAction(String className, String actionName)` | Resolves a registered `Action` by resource class name and action name. Throws `WaterRuntimeException` if multiple actions match. |
| `findObjectTypeInParams(Class<T> type, Object[] args)` | Finds first method parameter matching the given type. Used to extract `BaseEntity` from method args. |
| `findMethodParamIndexByName(String paramName, Object[] params)` | Finds method parameter index by name. |
| `checkEntityPermission(SecurityContext ctx, BaseEntity entity, String[] actions)` | Iterates action names, resolves each to an `Action`, checks permission via `PermissionUtil`. Returns true if ANY action is permitted. |
| `checkAnnotationIsOnWaterApiClass(Service s, Annotation a)` | Validates that the annotation is on a `BaseApi` class. Throws `WaterRuntimeException` with clear message if not. |

### 4.2 AllowPermissionInterceptor

**File:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/AllowPermissionInterceptor.java`
**Type:** `BeforeMethodInterceptor<AllowPermissions>`
**Registration:** `@FrameworkComponent(services = {BeforeMethodInterceptor.class})`

**Logic flow:**
1. Verify annotation is on Api class (`checkAnnotationIsOnWaterApiClass`)
2. Check if service is entity-related (`BaseEntityApi` or has `systemApiRef`)
3. If `checkById=true` → find entity by ID from args using `BaseEntityApi.find()` or SystemApi
4. If `checkById=false` → find `BaseEntity` in method params
5. Call `checkEntityPermission(ctx, entity, actions)`
6. If not found → `throw new UnauthorizedException()`

### 4.3 AllowRolesInterceptor

**File:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/AllowRolesInterceptor.java`
**Type:** `BeforeMethodInterceptor<AllowRoles>`

**Logic flow:**
1. Verify annotation is on Api class
2. Validate `rolesNames` array is not null/empty (throws `WaterRuntimeException`)
3. Get `SecurityContext` from Runtime
4. Call `permissionUtil.userHasRoles(ctx.getLoggedUsername(), roles)`
5. If false → `throw new UnauthorizedException()`

### 4.4 AllowGenericPermissionInterceptor

**File:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/AllowGenericPermissionInterceptor.java`
**Type:** `BeforeMethodInterceptor<AllowGenericPermissions>`

**Logic flow:**
1. Verify annotation is on Api class
2. Resolve resource name:
   - If service is `BaseEntityApi` → try `annotation.resourceName()`, then `annotation.resourceParamName()`, then `entityApi.getEntityType().getName()`
   - If service is NOT `BaseEntityApi` → MUST have `resourceName` or `resourceParamName`
3. Get `SecurityContext` from Runtime
4. For each action: call `permissionUtil.checkPermission(resourceName, action)`
5. If no action passes → `throw new UnauthorizedException()`

### 4.5 AllowLoggedUserInterceptor

**File:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/AllowLoggedUserInterceptor.java`
**Type:** `BeforeMethodInterceptor<AllowLoggedUser>`

**Logic flow:**
1. Check `currentRuntime == null || getSecurityContext() == null || getLoggedEntityId() == 0`
2. If any is true → `throw new UnauthorizedException("No security context found, please login")`

### 4.6 AllowPermissionOnReturnInterceptor

**File:** `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/AllowPermissionOnReturnInterceptor.java`
**Type:** `AfterMethodInterceptor<AllowPermissionsOnReturn>`

**Logic flow:**
1. Verify annotation is on Api class
2. If `returnResult == null` → return (pass through)
3. If service is `BaseEntityApi`:
   - If return is `BaseEntity` → `checkEntityPermission(ctx, entity, actions)`
   - If return is NOT `BaseEntity` → throw `WaterRuntimeException` (incompatible return type)
4. If check fails → `throw new UnauthorizedException()`

### Interceptor Execution Order

```
Api Method Call
  |
  +-> [BEFORE] AllowLoggedUserInterceptor     (checks authentication)
  +-> [BEFORE] AllowPermissionInterceptor      (checks entity permission)
  +-> [BEFORE] AllowRolesInterceptor           (checks role membership)
  +-> [BEFORE] AllowGenericPermissionInterceptor (checks class-level permission)
  |
  +-> Actual Method Execution
  |
  +-> [AFTER]  AllowPermissionOnReturnInterceptor (checks permission on return value)
  |
  v
  Return Result (or UnauthorizedException)
```

**Note:** A method can have MULTIPLE security annotations. All Before interceptors must pass for the method to execute.

---

## 5. Core Security Interfaces

All interfaces in `Core/Core-api/src/main/java/it/water/core/api/`.

### 5.1 PermissionManager

**File:** `Core/Core-api/src/main/java/it/water/core/api/permission/PermissionManager.java`

The central service for all permission checks. Operates by username (not SecurityContext).

```java
public interface PermissionManager extends Service {
    boolean userHasRoles(String username, String[] rolesNames);

    void addPermissionIfNotExists(Role r, Class<? extends Resource> resourceClass, Action action);

    // Entity-specific permission check
    boolean checkPermission(String username, Resource entity, Action action);
    // Class-level permission check
    boolean checkPermission(String username, Class<? extends Resource> resource, Action action);
    // Resource-name permission check
    boolean checkPermission(String username, String resourceName, Action action);

    // Permission + ownership checks
    boolean checkPermissionAndOwnership(String username, String resourceName, Action action, Resource... entities);
    boolean checkPermissionAndOwnership(String username, Resource resource, Action action, Resource... entities);

    // Ownership check
    boolean checkUserOwnsResource(User user, Object resource);

    // Bulk permission map
    Map<String, Map<String, Map<String, Boolean>>> entityPermissionMap(
        String username, Map<String, List<Long>> entityPks);

    // Static helpers
    static boolean isProtectedEntity(Object entity) { ... }
    static boolean isProtectedEntity(String resourceName) { ... }
}
```

**Static helpers:** `isProtectedEntity()` checks whether a class implements `ProtectedEntity` by scanning the Atteo ClassIndex for `@AccessControl`-annotated classes.

### 5.2 PermissionUtil

**File:** `Core/Core-api/src/main/java/it/water/core/api/permission/PermissionUtil.java`

Simplified facade that automatically extracts the logged username from `Runtime.getSecurityContext()`.

```java
public interface PermissionUtil extends Service {
    boolean userHasRoles(String username, String[] rolesNames);
    boolean checkPermission(Object o, Action action);
    boolean checkPermission(String resourceName, Action action);
    boolean checkPermissionAndOwnership(String resourceName, Action action, Resource... entities);
    boolean checkPermissionAndOwnership(Object o, Action action, Resource... entities);
}
```

### 5.3 SecurityContext

**File:** `Core/Core-api/src/main/java/it/water/core/api/permission/SecurityContext.java`

```java
public interface SecurityContext extends Service {
    String getLoggedUsername();
    boolean isLoggedIn();
    boolean isAdmin();
    long getLoggedEntityId();
}
```

### 5.4 Permission

**File:** `Core/Core-api/src/main/java/it/water/core/api/permission/Permission.java`

```java
public interface Permission {
    long getId();
    String getName();
    String getResourceName();
    Long getResourceId();
    long getActionIds();
    long getUserId();
    long getRoleId();
}
```

### 5.5 ProtectedEntity

**File:** `Core/Core-api/src/main/java/it/water/core/api/permission/ProtectedEntity.java`

```java
public interface ProtectedEntity extends BaseEntity {
    // Marker interface - entities implementing this require permission checks
}
```

An entity that implements `ProtectedEntity` **must** also have `@AccessControl` annotation to define its available actions.

### 5.6 ProtectedResource

**File:** `Core/Core-api/src/main/java/it/water/core/api/model/ProtectedResource.java`

```java
public interface ProtectedResource extends Resource {
    String getResourceId();
}
```

### 5.7 ContextFactory

**File:** `Core/Core-api/src/main/java/it/water/core/api/permission/ContextFactory.java`

```java
public interface ContextFactory {
    SecurityContext createContext(Map<String, Object> params);
}
```

Used by different runtime implementations (Spring, OSGi) to create `SecurityContext` from technology-specific auth data (e.g., JWT claims).

### 5.8 AuthenticationProvider

**File:** `Core/Core-api/src/main/java/it/water/core/api/security/AuthenticationProvider.java`

```java
public interface AuthenticationProvider extends Service {
    Principal login(String username, String password);
    List<String> issuersNames();
}
```

### 5.9 Authenticable

**File:** `Core/Core-api/src/main/java/it/water/core/api/security/Authenticable.java`

Interface for entities that can authenticate (e.g., Users, Devices).

Key methods: `getScreenName()`, `getPassword()`, `getSalt()`, `isAdmin()`, `isActive()`, `getRoles()`, `getLoggedEntityId()`, `getIssuer()`, `getScreenNameFieldName()`

### 5.10 EncryptionUtil

**File:** `Core/Core-api/src/main/java/it/water/core/api/security/EncryptionUtil.java`

See [Section 15: Encryption Utilities](#15-encryption-utilities).

---

## 6. SecurityContext & Principal Model

All classes in `Core/Core-security/src/main/java/it/water/core/security/model/`.

### 6.1 WaterAbstractSecurityContext

**File:** `Core/Core-security/src/main/java/it/water/core/security/model/context/WaterAbstractSecurityContext.java`

```java
public abstract class WaterAbstractSecurityContext implements SecurityContext {
    private Set<java.security.Principal> loggedPrincipals;
    private java.security.Principal loggedUser;
    private List<RolePrincipal> roles;
    private String permissionImplementation;
    private String issuerClassName;
    private long loggedEntityId;
}
```

**Constructor logic:**
1. Receives `Set<Principal>` of logged principals
2. Iterates and separates:
   - `UserPrincipal` → stored as `loggedUser`, extracts `loggedEntityId` and `issuerClassName`
   - `RolePrincipal` → added to `roles` list
3. Sets `permissionImplementation` to `SecurityConstants.PERMISSION_MANAGER_DEFAULT_IMPLEMENTATION` ("default")

**Key method: `getPermissionManager()`**
Uses `ComponentRegistry` with a filter on `SecurityConstants.PERMISSION_IMPLEMENTATION_COMPONENT_PROPERTY` to find the appropriate `PermissionManager` for the current context. This allows pluggable permission implementations.

**Abstract methods:**
- `isSecure()` — whether connection is secure (HTTPS)
- `getAuthenticationScheme()` — e.g., "Basic", "Bearer"

### 6.2 BasicSecurityContext

**File:** `Core/Core-security/src/main/java/it/water/core/security/model/context/BasicSecurityContext.java`

```java
public class BasicSecurityContext extends WaterAbstractSecurityContext {
    public BasicSecurityContext(Set<Principal> principals) { super(principals); }
    @Override public boolean isSecure() { return false; }
    @Override public String getAuthenticationScheme() { return "Basic"; }
}
```

### 6.3 UserPrincipal

**File:** `Core/Core-security/src/main/java/it/water/core/security/model/principal/UserPrincipal.java`

```java
@AllArgsConstructor @Getter
public class UserPrincipal implements java.security.Principal, Serializable {
    private String name;          // username
    private boolean isAdmin;      // admin flag
    private long loggedEntityId;  // user entity ID in DB
    private String issuer;        // issuer class (e.g., "it.water.user.model.WaterUser")
}
```

### 6.4 RolePrincipal

**File:** `Core/Core-security/src/main/java/it/water/core/security/model/principal/RolePrincipal.java`

```java
@AllArgsConstructor @Getter
public class RolePrincipal implements java.security.Principal, Serializable {
    private String name;  // role name
}
```

### 6.5 SecurityConstants

**File:** `Core/Core-security/src/main/java/it/water/core/security/model/SecurityConstants.java`

```java
public class SecurityConstants {
    public static final String PERMISSION_MANAGER_DEFAULT_IMPLEMENTATION = "default";
    public static final String PERMISSION_IMPLEMENTATION_COMPONENT_PROPERTY = "implementation";
}
```

### 6.6 Building a SecurityContext (example)

```java
Set<Principal> principals = new HashSet<>();
principals.add(new UserPrincipal("john.doe", false, 42L, "it.water.user.model.WaterUser"));
principals.add(new RolePrincipal("permissionManager"));
principals.add(new RolePrincipal("projectEditor"));
SecurityContext ctx = new BasicSecurityContext(principals);
// ctx.getLoggedUsername() -> "john.doe"
// ctx.isAdmin() -> false
// ctx.getLoggedEntityId() -> 42
// ctx.isUserInRole("permissionManager") -> true
```

---

## 7. PermissionUtil Implementation

**File:** `Core/Core-security/src/main/java/it/water/core/security/util/WaterPermissionUtilImpl.java`

```java
@FrameworkComponent(services = PermissionUtil.class)
public class WaterPermissionUtilImpl implements PermissionUtil {
    @Inject private Runtime waterRuntime;
    @Inject private PermissionManager pm;
    ...
}
```

**Pattern:** Every method follows the same structure:
1. Check `PermissionManager.isProtectedEntity()` → if NOT protected, return `true` (no check needed)
2. Check `isAdmin()` → if admin, return `true` (admin bypasses everything)
3. If `pm == null` → return `false` (no permission manager registered)
4. Delegate to `PermissionManager` with `waterRuntime.getSecurityContext().getLoggedUsername()`

**Methods:**
| Method | Delegates to |
|--------|-------------|
| `userHasRoles(username, rolesNames)` | `pm.userHasRoles()` |
| `checkPermission(Object o, Action)` | `pm.checkPermission(username, entity, action)` |
| `checkPermission(String resourceName, Action)` | `pm.checkPermission(username, resourceName, action)` |
| `checkPermissionAndOwnership(String, Action, Resource...)` | `pm.checkPermissionAndOwnership(username, resourceName, action, entities)` |
| `checkPermissionAndOwnership(Object, Action, Resource...)` | `pm.checkPermissionAndOwnership(username, entity, action, entities)` |

**Key insight:** `PermissionUtil` is the bridge between the SecurityContext (thread-bound) and the `PermissionManager` (username-based). Interceptors use `PermissionUtil`; internal code can use `PermissionManager` directly.

---

## 8. Action System & Bitwise Permission Model

### 8.1 How Actions Work

Actions are string identifiers mapped to numeric IDs that are **powers of 2**. This enables bitwise operations for efficient permission storage and checking in a single `long` field (`actionIds`).

**Formula:** `actionId = Math.pow(2, position)` where `position` is the action's index in `@AccessControl.availableActions`.

### 8.2 Predefined Action Constants

**CrudActions** (`Core/Core-permission/src/main/java/it/water/core/permission/action/CrudActions.java`):

| Constant | Value |
|----------|-------|
| `CrudActions.SAVE` | `"save"` |
| `CrudActions.UPDATE` | `"update"` |
| `CrudActions.FIND` | `"find"` |
| `CrudActions.FIND_ALL` | `"find-all"` |
| `CrudActions.REMOVE` | `"remove"` |

**UserActions** (`Core/Core-permission/src/main/java/it/water/core/permission/action/UserActions.java`):

| Constant | Value |
|----------|-------|
| `UserActions.IMPERSONATE` | `"impersonate"` |
| `UserActions.ACTIVATE` | `"activate"` |
| `UserActions.DEACTIVATE` | `"deactivate"` |

**ShareAction** (`Core/Core-permission/src/main/java/it/water/core/permission/action/ShareAction.java`):

| Constant | Value |
|----------|-------|
| `ShareAction.SHARE` | `"share"` |

**PermissionsActions** (`Permission/Permission-model/src/main/java/it/water/permission/actions/PermissionsActions.java`):

| Constant | Value |
|----------|-------|
| `PermissionsActions.GIVE_PERMISSIONS` | `"permissions"` |
| `PermissionsActions.LIST_ACTIONS` | `"actions"` |

### 8.3 Bitwise Permission Table

For a standard CRUD entity with `@AccessControl(availableActions = {SAVE, UPDATE, FIND, FIND_ALL, REMOVE})`:

| Position | Action Name | actionId (2^pos) | Binary |
|----------|------------|------------------|--------|
| 0 | save | 1 | 00001 |
| 1 | update | 2 | 00010 |
| 2 | find | 4 | 00100 |
| 3 | find-all | 8 | 01000 |
| 4 | remove | 16 | 10000 |

### 8.4 Bitwise AND Logic

Permission checking formula:
```java
boolean hasPermission = (permissionActionIds & actionId) == actionId;
```

**Examples:**
```
Permission stored: actionIds = 11 (binary: 01011) = save(1) + update(2) + find-all(8)

Check SAVE (1):      01011 & 00001 = 00001 == 00001 -> TRUE
Check UPDATE (2):    01011 & 00010 = 00010 == 00010 -> TRUE
Check FIND (4):      01011 & 00100 = 00000 != 00100 -> FALSE
Check FIND_ALL (8):  01011 & 01000 = 01000 == 01000 -> TRUE
Check REMOVE (16):   01011 & 10000 = 00000 != 10000 -> FALSE
```

### 8.5 Accumulating Actions

`WaterPermission.withAccumulateActions(long actionsToAdd)` uses bitwise OR to merge new actions without losing existing ones:
```java
long newActionIds = this.actionIds | actionsToAdd;
// Example: existing = 3 (save+update), adding find(4) -> 3 | 4 = 7 (save+update+find)
```

### 8.6 Key Action Classes

**ActionFactory** (`Core/Core-permission/src/main/java/it/water/core/permission/action/ActionFactory.java`):
- `createResourceAction(Class<T>, Action)` — wraps action with resource class
- `createEmptyActionList(Class<T>)` — creates empty list for a resource
- `createBaseCrudActionList(Class<T>)` — creates standard CRUD action list
- `createGenericAction(Class<T>, String actionName, long actionId)` — creates custom action

---

## 9. DefaultActionsManager & Action Registration

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/action/DefaultActionsManager.java`

```java
@FrameworkComponent(priority = 1)
public class DefaultActionsManager implements ActionsManager {
    private Map<String, ActionList<? extends Resource>> actionsRegistry;
    @Inject private RoleManager roleManager;
    @Inject private PermissionManager permissionManager;
}
```

### Registration Flow (at startup)

```
1. Framework discovers all @AccessControl-annotated classes (via ClassIndex)
2. For each class, calls registerActions(Class<T> resourceClass):
   a. Read @AccessControl annotation
   b. For each action in availableActions[]:
      - Create Action with actionId = Math.pow(2, position)
      - Add to ActionList
   c. Store ActionList in actionsRegistry map (key = resourceClass.getName())
   d. For each @DefaultRoleAccess in rolesPermissions[]:
      - roleManager.createIfNotExists(roleName) -> get/create Role
      - For each action name in actions[]:
        - Lookup Action from actionsRegistry
        - permissionManager.addPermissionIfNotExists(role, resourceClass, action)
```

**Key code (action ID assignment):**
```java
for (int i = 0; i < availableActions.length; i++) {
    actionList.addAction(
        ActionFactory.createGenericAction(resourceClass, availableActions[i], (long) Math.pow(2, i))
    );
}
```

---

## 10. Permission Entity Model

**File:** `Permission/Permission-model/src/main/java/it/water/permission/model/WaterPermission.java`

### Entity Fields

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | `String` | `@NotBlank`, `@NoMalitiusCode`, max 255 | Permission display name |
| `actionIds` | `long` | `@Positive` | Bitwise-combined action IDs |
| `entityResourceName` | `String` | `@NotEmpty`, `@NoMalitiusCode`, max 255 | Fully qualified resource class name |
| `resourceId` | `Long` | nullable | Specific entity ID (null = class-level permission) |
| `roleId` | `long` | — | Associated role ID (0 = user-level permission) |
| `userId` | `long` | — | Associated user ID (0 = role-level permission) |

### Unique Constraint

```java
@Table(uniqueConstraints = @UniqueConstraint(
    columnNames = {"roleId", "userId", "entityResourceName", "resourceId"}
))
```

This ensures one permission record per role/user + resource + specific entity combination.

### Permission Types

| roleId | userId | resourceId | Type | Meaning |
|--------|--------|-----------|------|---------|
| > 0 | 0 | null | Role-general | Role has permission on ALL instances of resource |
| > 0 | 0 | > 0 | Role-specific | Role has permission on SPECIFIC entity |
| 0 | > 0 | null | User-general | User has permission on ALL instances of resource |
| 0 | > 0 | > 0 | User-specific | User has permission on SPECIFIC entity |

### Default Roles

| Role Name | Permissions |
|-----------|-------------|
| `permissionManager` | SAVE, UPDATE, FIND, FIND_ALL, REMOVE, GIVE_PERMISSIONS, LIST_ACTIONS |
| `permissionViewer` | FIND, FIND_ALL, LIST_ACTIONS |
| `permissionEditor` | SAVE, UPDATE, FIND, FIND_ALL, LIST_ACTIONS |

### Constructor

```java
WaterPermission perm = new WaterPermission(
    "permName",           // name
    actionIds,            // bitwise combined action IDs
    "full.class.Name",   // entityResourceName
    resourceId,           // specific entity ID (or 0L)
    roleId,               // role ID (or 0L)
    userId                // user ID (or 0L)
);
```

---

## 11. Permission APIs

### 11.1 PermissionApi (user-facing)

**File:** `Permission/Permission-api/src/main/java/it/water/permission/api/PermissionApi.java`

```java
public interface PermissionApi extends BaseEntityApi<WaterPermission> {
    Map<String, Map<String, Map<String, Boolean>>> entityPermissionMap(
        Map<String, List<Long>> entityPks);
}
```

Inherits CRUD from `BaseEntityApi`. The `entityPermissionMap` method calculates a permission matrix for the logged user.

**Permission map format:**
```json
{
  "it.water.mymodule.model.MyEntity": {
    "38": { "save": true, "update": true, "find": true, "remove": false },
    "54": { "save": false, "update": true, "find": true, "remove": false }
  }
}
```

### 11.2 PermissionSystemApi (internal, bypasses security)

**File:** `Permission/Permission-api/src/main/java/it/water/permission/api/PermissionSystemApi.java`

```java
public interface PermissionSystemApi extends BaseEntitySystemApi<WaterPermission> {
    WaterPermission findByUserAndResource(long userId, Resource resource);
    WaterPermission findByUserAndResourceName(long userId, String resourceName);
    WaterPermission findByUserAndResourceNameAndResourceId(long userId, String resourceName, long id);
    WaterPermission findByRoleAndResource(long roleId, Resource resource);
    WaterPermission findByRoleAndResourceName(long roleId, String resourceName);
    Collection<WaterPermission> findByRole(long roleId);
    WaterPermission findByRoleAndResourceNameAndResourceId(long roleId, String resourceName, long resourceId);
    void checkOrCreatePermissions(long roleId, List<ResourceAction<?>> actions);
    void checkOrCreatePermissionsSpecificToEntity(long roleId, long entityId, List<ResourceAction<?>> actions);
    boolean permissionSpecificToEntityExists(String resourceName, long resourceId);
}
```

### 11.3 PermissionRepository

**File:** `Permission/Permission-api/src/main/java/it/water/permission/api/PermissionRepository.java`

Extends `BaseRepository<WaterPermission>`. Mirrors SystemApi methods at the JPA persistence level.

**Implementation:** `Permission/Permission-service/src/main/java/it/water/permission/repository/PermissionRepositoryImpl.java`
Uses persistence unit `permission-persistence-unit`.

### 11.4 PermissionRestApi

**File:** `Permission/Permission-api/src/main/java/it/water/permission/api/rest/PermissionRestApi.java`

```java
@Path("/permissions")
@Api(produces = MediaType.APPLICATION_JSON, tags = "Permission API")
@FrameworkRestApi
public interface PermissionRestApi extends RestApi {
    @LoggedIn @POST
    WaterPermission save(WaterPermission permission);

    @LoggedIn @PUT
    WaterPermission update(WaterPermission permission);

    @LoggedIn @GET @Path("/{id}")
    WaterPermission find(@PathParam("id") long id);

    @LoggedIn @GET
    PaginableResult<WaterPermission> findAll();

    @LoggedIn @DELETE @Path("/{id}")
    void remove(@PathParam("id") long id);

    @LoggedIn @POST @Path("/map")
    Map<String, Map<String, Map<String, Boolean>>> elaboratePermissionMap(
        Map<String, List<Long>> entityPks);
}
```

All endpoints require `@LoggedIn` (authenticated user).

---

## 12. PermissionManager Implementation Deep Dive

**File:** `Permission/Permission-manager/src/main/java/it/water/permission/manager/PermissionManagerDefault.java`

```java
@FrameworkComponent(properties = {
    PermissionManagerComponentProperties.PERMISSION_MANAGER_IMPLEMENTATION_PROP + "=" +
    PermissionManagerComponentProperties.PERMISSION_MANAGER_DEFAILT_IMPLEMENTATION
})
public class PermissionManagerDefault implements PermissionManager { ... }
```

### 12.1 Generic Permission Check (resource name, no entity ID)

`hasPermission(String username, String resourceName, Action action)`:

```
1. Fetch User by username
2. If user == null -> return false
3. If user.isAdmin() -> return true
4. Fetch user roles (roleIntegrationClient.fetchUserRoles)
5. If no roles -> return false
6. For each role:
   a. Find permission by role + resourceName
   b. If permission exists AND hasPermission(permission.actionIds, action.actionId):
      -> return true
7. return false
```

### 12.2 Entity-Specific Permission Check

`hasPermission(User user, ProtectedEntity entity, Action action)`:

```
1. If user.isAdmin() -> return true
2. Fetch user roles
3. If no roles -> return false
4. For each role:
   a. permissionSpecific = findByRoleAndResourceNameAndResourceId(role, entity)
   b. userPermissionSpecific = findByUserAndResourceNameAndResourceId(user, entity)
   c. permissionImpersonation = findByRoleAndResourceName(role, User.class.getName())
   d. hasGeneralPermission = check role + user general permission on resourceName
   e. hasEntityPermission = check specific permission on entity ID
   f. existPermissionSpecificToEntity = any permission exists for this entity
   g. userOwnsResource = checkUserOwnsResource(user, entity)
   h. userSharesResource = checkUserSharesResource(user, entity)
   i. hasImpersonationPermission = check impersonate action on User resource
   j. result |= calculatePermission(...) || hasImpersonationPermission
5. return result
```

### 12.3 The calculatePermission Formula

```java
private boolean calculatePermission(
    Permission permissionSpecific,
    Permission userPermissionSpecific,
    boolean hasEntityPermission,
    boolean hasGeneralPermission,
    boolean userOwnsResource,
    boolean userSharesResource,
    boolean existPermissionSpecificToEntity
) {
    return (
        ((permissionSpecific != null || userPermissionSpecific != null) && hasEntityPermission)
        || (permissionSpecific == null && userPermissionSpecific == null && hasGeneralPermission)
    ) && (
        userOwnsResource
        || ((userSharesResource && !existPermissionSpecificToEntity && hasGeneralPermission)
            || (userSharesResource && (permissionSpecific != null || userPermissionSpecific != null) && hasEntityPermission))
    );
}
```

**In natural language:**

The user has permission if:

**Permission Clause** (at least one must be true):
- There IS a specific permission for this entity AND the action matches, OR
- There is NO specific permission for this entity AND the general (class-level) permission allows the action

**AND Ownership Clause** (at least one must be true):
- The user OWNS the resource, OR
- The resource is SHARED to the user AND:
  - No entity-specific permission exists AND the general permission allows it, OR
  - An entity-specific permission exists AND the action matches

### 12.4 Bitwise Check

```java
private boolean hasPermission(long permissionActionIds, long actionId) {
    return (permissionActionIds & actionId) == actionId;
}
```

### 12.5 Permission + Ownership Check

`checkPermissionAndOwnership(username, resourceName, action, entities...)`:
1. Check generic permission on `resourceName`
2. If has permission AND entities provided:
   - For each entity, verify `checkUserOwnsResource(user, entity)`
   - ALL entities must be owned (AND logic)

---

## 13. Ownership & Sharing Authorization

### 13.1 OwnedResource Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/entity/owned/OwnedResource.java`

```java
public interface OwnedResource {
    String OWNER_USER_ID_FIELD_NAME = "ownerUserId";
    Long getOwnerUserId();
    void setOwnerUserId(Long ownerUserId);
}
```

Entities implementing `OwnedResource` are automatically filtered: non-admin users can only access entities they own.

### 13.2 OwnedChildResource Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/entity/owned/OwnedChildResource.java`

```java
public interface OwnedChildResource extends OwnedResource {
    BaseEntity getParent();
}
```

For child entities that inherit ownership from a parent. The framework traverses the parent chain to verify ownership.

### 13.3 SharedEntity Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/entity/shared/SharedEntity.java`

```java
public interface SharedEntity extends OwnedResource {
    // A shared entity is always owned first, but can be shared to other users
}
```

### 13.4 checkUserOwnsResource Algorithm

```
1. If user.isAdmin() -> return true
2. Check if entity implements OwnedResource
   a. If yes, get ownerUserId
   b. If owner doesn't match user AND resource is not shared -> return false
3. If entity has persisted ID (id != 0):
   a. Load persisted entity from DB
   b. Re-check ownerUserId from persisted version
   c. If not OwnedResource -> resource is not owned, pass
4. Return true if user.id == ownerUserId OR user shares resource
```

### 13.5 checkUserSharesResource Algorithm

```
1. If no SharedEntityIntegrationClient available -> return false
2. Walk up the entity hierarchy:
   a. If entity is SharedEntity:
      - Fetch sharing user IDs from SharedEntityIntegrationClient
      - If user's entity ID not in shared list -> return false
      - Otherwise stop walking
   b. If entity is OwnedChildResource:
      - Move to parent entity and repeat
   c. Otherwise -> stop walking
3. Load persisted entity and re-verify sharing
4. Return true if entity ID is in shared list
```

---

## 14. Authentication System

### 14.1 Authentication Flow

```
1. Client sends credentials (username + password)
2. AuthenticationProvider.login(username, password) verifies credentials
3. On success: creates Set<Principal> with UserPrincipal + RolePrincipals
4. SecurityContext is created from principals
5. JWT token is generated containing principal information
6. Subsequent requests include JWT in Authorization header
7. REST layer validates JWT and recreates SecurityContext
8. Runtime.fillSecurityContext(ctx) binds context to current thread
```

### 14.2 @LoggedIn Annotation (REST-level)

Applied to REST API interface methods. Triggers JWT validation BEFORE the API method is called. This is separate from the method-level security annotations (`@AllowPermissions`, etc.).

### 14.3 Authenticable Interface

Entities that can authenticate must implement `Authenticable`:
- `getScreenName()` — login identifier
- `getPassword()` — hashed password
- `getSalt()` — password salt
- `isAdmin()` — admin flag
- `isActive()` — account active flag
- `getRoles()` — user's roles
- `getScreenNameFieldName()` — field name for login lookup (e.g., "username" or "email")

---

## 15. Encryption Utilities

**Interface:** `Core/Core-api/src/main/java/it/water/core/api/security/EncryptionUtil.java`
**Implementation:** `Core/Core-security/src/main/java/it/water/core/security/util/WaterEncryprionUtilImpl.java`

### Method Categories (44+ methods)

| Category | Key Methods |
|----------|-------------|
| **SSL/Certificate** | `generateSSLKeyPairValue()`, `generateCertificationRequest()`, `createServerClientX509Cert()` |
| **Key Management** | `getServerKeyPair()`, `getPublicKeyString()`, `getPrivateKeyString()`, `getPublicKeyFromString()`, `getPrivateKeyFromString()` |
| **RSA Encryption** | `getCipherRSA()`, `encodeMessageWithPublicKey()`, `decodeMessageWithPublicKey()`, `decodeMessageWithServerPrivateKey()` |
| **AES Encryption** | `getCipherAES()`, `encryptWithAES()`, `decryptWithAES()`, `generateRandomAESPassword()`, `generateRandomAESInitVector()` |
| **Signing** | `signDataWithServerCert()`, `verifyDataSignedWithServerCert()` |
| **Password Hashing** | `hashPassword(String password, byte[] salt)`, `generate16BytesSalt()` |

---

## 16. Security Exceptions & Error Handling

### UnauthorizedException

**File:** `Core/Core-permission/src/main/java/it/water/core/permission/exceptions/UnauthorizedException.java`

```java
@AllArgsConstructor
public class UnauthorizedException extends WaterRuntimeException {
    public UnauthorizedException() { super(); }
    public UnauthorizedException(String message) { super(message); }
}
```

Thrown by ALL security interceptors when authorization fails. Maps to HTTP 403 in the REST layer.

### WaterRuntimeException

Thrown for security configuration errors:
- Annotation placed on wrong class type (not BaseApi)
- Missing required annotation attributes
- Entity ID parameter not found or wrong type
- SystemApi reference not found

---

## 17. Best Practices & Anti-Patterns

### Best Practices

1. **Annotate Api methods, not implementations.** Security annotations MUST be on `BaseApi` interface methods. The interceptor framework validates this and throws if violated.

2. **Use `@AllowPermissions` for entity CRUD.** When the method operates on a specific entity instance (save, update, find by ID, remove), use entity-specific permission checking.

3. **Use `@AllowGenericPermissions` for class-level operations.** For `findAll()`, `count()`, bulk operations, and non-entity services, use generic (resource-name-level) permission checking.

4. **Use `@AllowRoles` sparingly.** Role checks are coarse-grained. Prefer fine-grained `@AllowPermissions` or `@AllowGenericPermissions` that use the bitwise permission model.

5. **Always define `@AccessControl` on `ProtectedEntity` classes.** Without `@AccessControl`, the action registry won't have entries for the entity and permission checks will fail silently.

6. **Never change the order of `availableActions` after deployment.** Action IDs are derived from array position. Changing order changes IDs, breaking existing permission records.

7. **Add new actions at the END of `availableActions`.** Append new actions to preserve existing action IDs.

8. **Use `PermissionUtil` for programmatic checks, annotations for declarative.** In code that needs runtime permission decisions, inject `PermissionUtil`. For method-level enforcement, use annotations.

9. **`SystemApi` intentionally bypasses security.** Use `SystemApi` for internal operations (startup, migration, inter-service). Never expose `SystemApi` to REST or user-facing code.

10. **Always include `@LoggedIn` on REST endpoints.** This ensures JWT validation occurs before any business logic.

### Anti-Patterns

1. **Putting security annotations on SystemApi or Repository classes.** The interceptor explicitly rejects this with `WaterRuntimeException`. SystemApi bypasses security by design.

2. **Using `@AllowPermissions` on non-entity APIs without `systemApiRef`.** The interceptor can't find the entity to check and throws `UnsupportedOperationException`.

3. **Calling Repository directly from REST layer.** This bypasses ALL security (interceptors only work on Api/SystemApi). Always go through the Api layer.

4. **Forgetting to implement `ProtectedEntity` on secured entities.** Without `ProtectedEntity`, `PermissionManager.isProtectedEntity()` returns false and no permission checks occur.

5. **Using non-power-of-2 action IDs.** Manually setting action IDs that aren't powers of 2 breaks the bitwise AND logic. Always let `@AccessControl` and `DefaultActionsManager` handle ID assignment.

6. **Hardcoding permission checks instead of using annotations.** This duplicates logic, is error-prone, and doesn't benefit from the interceptor framework's consistent enforcement.

---

## 18. Decision Trees

### How to Secure a Method

```
Need to secure a method?
  |
  +-- Does the method operate on a SPECIFIC entity instance?
  |     |
  |     +-- Is the entity passed as a METHOD PARAMETER?
  |     |     --> @AllowPermissions(actions = {CrudActions.XXX})
  |     |
  |     +-- Is only the entity ID passed as parameter?
  |     |     --> @AllowPermissions(actions = {CrudActions.XXX},
  |     |             checkById = true, idParamIndex = <param-index>)
  |     |
  |     +-- Is the entity only available in the RETURN VALUE?
  |     |     --> @AllowPermissionsOnReturn(actions = {CrudActions.XXX})
  |     |
  |     +-- Does the method belong to a NON-ENTITY API?
  |           --> @AllowPermissions(actions = {CrudActions.XXX},
  |                   checkById = true, idParamIndex = <param-index>,
  |                   systemApiRef = "fully.qualified.SystemApiClass")
  |
  +-- Does the method operate at the RESOURCE CLASS level (no specific entity)?
  |     |
  |     +-- Is it on a BaseEntityApi subclass?
  |     |     --> @AllowGenericPermissions(actions = {CrudActions.XXX})
  |     |         (resource name auto-inferred from entity type)
  |     |
  |     +-- Is it on a non-entity service?
  |           --> @AllowGenericPermissions(actions = {...},
  |                   resourceName = "fully.qualified.ClassName")
  |
  +-- Does the method just need ROLE-BASED access?
  |     --> @AllowRoles(rolesNames = {"roleName1", "roleName2"})
  |
  +-- Does the method just need the user to be LOGGED IN?
        --> @AllowLoggedUser
```

### How to Define Entity Security

```
Defining a new entity?
  |
  +-- Does it need access control?
  |     |
  |     +-- YES:
  |     |     1. Implement ProtectedEntity
  |     |     2. Add @AccessControl with availableActions and rolesPermissions
  |     |     |
  |     |     +-- Is it owned by a user?
  |     |     |     --> Also implement OwnedResource (adds ownerUserId field)
  |     |     |
  |     |     +-- Can it be shared between users?
  |     |     |     --> Also implement SharedEntity (extends OwnedResource)
  |     |     |
  |     |     +-- Is it a child of an owned entity?
  |     |           --> Also implement OwnedChildResource (adds getParent())
  |     |
  |     +-- NO:
  |           Just extend AbstractJpaEntity (no security annotations needed)
```

### Which API Layer to Use

```
Need to call a secured operation?
  |
  +-- From REST/external client?
  |     --> Use PermissionRestApi endpoints (HTTP + JWT)
  |
  +-- From Api layer (user context)?
  |     --> Use PermissionApi (security enforced via annotations)
  |
  +-- From internal service / startup / migration?
  |     --> Use PermissionSystemApi (security bypassed)
  |
  +-- From data layer?
        --> Use PermissionRepository (direct DB, no security)
        --> WARNING: Only from SystemApi implementations!
```

---

## 19. Testing Security

### 19.1 Test Framework Setup

```java
@ExtendWith(WaterTestExtension.class)
public class MySecurityTest {
    @Inject private ComponentRegistry componentRegistry;
    @Inject private Runtime runtime;
    @Inject private PermissionUtil permissionUtil;
    @Inject private RoleManager roleManager;
    @Inject private ActionsManager actionsManager;
    ...
}
```

### 19.2 Impersonation Pattern

Use `TestRuntimeInitializer` to impersonate users in tests:

```java
// Impersonate a specific user
TestRuntimeInitializer initializer = TestRuntimeInitializer.getInstance();
initializer.impersonate(userEntity, runtime);

// Now all PermissionUtil calls use this user's context
boolean canSave = permissionUtil.checkPermission(entity, saveAction);

// Reset to no user
initializer.resetSecurityContext(runtime);
```

### 19.3 Creating Test Users and Roles

```java
// Create a user via SystemApi
User testUser = createTestUser("testuser", "Test", "User", "test@example.com", "Password1!");
// testUser is now in the DB with an ID

// Create a role
Role testRole = roleManager.createIfNotExists("testRoleName");

// Assign role to user
roleManager.addRole(testUser.getId(), testRole);

// Create permission for role
Action saveAction = actionsManager.getActions()
    .get(MyEntity.class.getName())
    .getAction(CrudActions.SAVE);
permissionManager.addPermissionIfNotExists(testRole, MyEntity.class, saveAction);
```

### 19.4 Testing Permission Denied

```java
// Impersonate user WITHOUT the required role/permission
initializer.impersonate(unprivilegedUser, runtime);

// Verify UnauthorizedException is thrown
assertThrows(UnauthorizedException.class, () -> {
    myEntityApi.save(newEntity);
});
```

### 19.5 Testing Permission Granted

```java
// Impersonate user WITH the required role/permission
initializer.impersonate(privilegedUser, runtime);

// Verify operation succeeds
MyEntity saved = myEntityApi.save(newEntity);
assertNotNull(saved);
assertTrue(saved.getId() > 0);
```

### 19.6 Testing Permission Map

```java
initializer.impersonate(testUser, runtime);

Map<String, List<Long>> entityPks = Map.of(
    MyEntity.class.getName(), List.of(entity1.getId(), entity2.getId())
);
Map<String, Map<String, Map<String, Boolean>>> permMap =
    permissionApi.entityPermissionMap(entityPks);

assertTrue(permMap.get(MyEntity.class.getName())
    .get(String.valueOf(entity1.getId()))
    .get("save"));
```

---

## 20. Quick Reference Tables

### Annotation Quick Reference

| Annotation | Target | Interceptor Type | When | Key Attributes |
|-----------|--------|-----------------|------|----------------|
| `@AccessControl` | TYPE | (none - config) | Startup | `availableActions`, `rolesPermissions` |
| `@DefaultRoleAccess` | (inner) | (none - config) | Startup | `roleName`, `actions` |
| `@AllowPermissions` | METHOD | Before | Pre-exec | `actions`, `checkById`, `idParamIndex`, `systemApiRef` |
| `@AllowGenericPermissions` | METHOD | Before | Pre-exec | `actions`, `resourceName`, `resourceParamName` |
| `@AllowRoles` | METHOD | Before | Pre-exec | `rolesNames` |
| `@AllowLoggedUser` | METHOD | Before | Pre-exec | (none - marker) |
| `@AllowPermissionsOnReturn` | METHOD | After | Post-exec | `actions` |

### Interface Quick Reference

| Interface | Module | Package | Purpose |
|-----------|--------|---------|---------|
| `PermissionManager` | Core-api | `it.water.core.api.permission` | Central permission checking (by username) |
| `PermissionUtil` | Core-api | `it.water.core.api.permission` | Simplified permission checking (logged user context) |
| `SecurityContext` | Core-api | `it.water.core.api.permission` | Current user session info |
| `Permission` | Core-api | `it.water.core.api.permission` | Permission data model |
| `ProtectedEntity` | Core-api | `it.water.core.api.permission` | Marker for entities requiring permission checks |
| `ProtectedResource` | Core-api | `it.water.core.api.model` | Resource with ID |
| `ContextFactory` | Core-api | `it.water.core.api.permission` | Creates SecurityContext from params |
| `AuthenticationProvider` | Core-api | `it.water.core.api.security` | Login authentication |
| `Authenticable` | Core-api | `it.water.core.api.security` | Entity that can authenticate |
| `EncryptionUtil` | Core-api | `it.water.core.api.security` | Encryption/hashing operations |

### Action Constants Quick Reference

| Class | Constants | Module |
|-------|----------|--------|
| `CrudActions` | SAVE, UPDATE, FIND, FIND_ALL, REMOVE | Core-permission |
| `UserActions` | IMPERSONATE, ACTIVATE, DEACTIVATE | Core-permission |
| `ShareAction` | SHARE | Core-permission |
| `PermissionsActions` | GIVE_PERMISSIONS, LIST_ACTIONS | Permission-model |

### Security Model Classes Quick Reference

| Class | Module | Purpose |
|-------|--------|---------|
| `WaterAbstractSecurityContext` | Core-security | Base SecurityContext with principal separation |
| `BasicSecurityContext` | Core-security | Simple implementation (scheme=Basic) |
| `UserPrincipal` | Core-security | User identity (name, isAdmin, entityId, issuer) |
| `RolePrincipal` | Core-security | Role identity (name) |
| `SecurityConstants` | Core-security | Permission manager implementation constants |
| `UnauthorizedException` | Core-permission | Security denial exception |
| `WaterPermissionUtilImpl` | Core-security | PermissionUtil implementation |
| `WaterPermission` | Permission-model | JPA entity storing permission data |
| `PermissionManagerDefault` | Permission-manager | Default PermissionManager implementation |
| `DefaultActionsManager` | Core-permission | Action registration from @AccessControl |

### File Path Quick Reference

| Component | Path |
|-----------|------|
| Security annotations | `Core/Core-permission/src/main/java/it/water/core/permission/annotations/` |
| Action classes | `Core/Core-permission/src/main/java/it/water/core/permission/action/` |
| Security interfaces | `Core/Core-api/src/main/java/it/water/core/api/permission/` |
| Auth interfaces | `Core/Core-api/src/main/java/it/water/core/api/security/` |
| Interceptors | `Core/Core-security/src/main/java/it/water/core/security/annotations/implementation/` |
| Context models | `Core/Core-security/src/main/java/it/water/core/security/model/` |
| PermissionUtil impl | `Core/Core-security/src/main/java/it/water/core/security/util/` |
| Permission entity | `Permission/Permission-model/src/main/java/it/water/permission/model/` |
| Permission APIs | `Permission/Permission-api/src/main/java/it/water/permission/api/` |
| Permission manager | `Permission/Permission-manager/src/main/java/it/water/permission/manager/` |
| Permission service | `Permission/Permission-service/src/main/java/it/water/permission/service/` |
| Permission REST (Spring) | `Permission/Permission-service-spring/src/main/java/it/water/permission/service/rest/spring/` |