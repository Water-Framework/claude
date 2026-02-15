---
name: authentication-knowledge
description: >
  Comprehensive knowledge base for Water Framework authentication system, including JWT token management,
  login flow, user registration/activation, AuthenticationProvider pattern, REST security filters,
  and the User module. Use when designing, implementing, reviewing, or debugging any authentication,
  login, JWT, or user management feature in Water modules.
allowed-tools: Read, Glob, Grep
---

# Water Framework Authentication Knowledge Base

This skill provides comprehensive documentation of the Water Framework authentication system,
covering the full login-to-token lifecycle, JWT management, user registration, password management,
and REST security filter integration. It serves as the single source of truth for all authentication
patterns and best practices.

---

## Table of Contents

1. [Authentication Architecture Overview](#1-authentication-architecture-overview)
2. [Module Dependency Map](#2-module-dependency-map)
3. [Core Security Interfaces](#3-core-security-interfaces)
4. [Authentication Module Deep Dive](#4-authentication-module-deep-dive)
5. [AuthenticationProvider Pattern](#5-authenticationprovider-pattern)
6. [User Module Overview](#6-user-module-overview)
7. [WaterUser Entity Model](#7-wateruser-entity-model)
8. [UserAuthenticationProvider Implementation](#8-userauthenticationprovider-implementation)
9. [Login Flow End-to-End](#9-login-flow-end-to-end)
10. [JWT Token System](#10-jwt-token-system)
11. [JWT Security Filter Chain](#11-jwt-security-filter-chain)
12. [Framework-Specific Filters (Spring, CXF)](#12-framework-specific-filters-spring-cxf)
13. [@LoggedIn Annotation & Endpoint Protection](#13-loggedin-annotation--endpoint-protection)
14. [SecurityContext & Principal Model](#14-securitycontext--principal-model)
15. [JWT Configuration Properties](#15-jwt-configuration-properties)
16. [User Registration & Activation Flow](#16-user-registration--activation-flow)
17. [Password Management](#17-password-management)
18. [User Module Configuration Properties](#18-user-module-configuration-properties)
19. [Authentication Module Configuration](#19-authentication-module-configuration)
20. [JAAS Integration](#20-jaas-integration)
21. [Testing Authentication](#21-testing-authentication)
22. [Best Practices & Anti-Patterns](#22-best-practices--anti-patterns)
23. [Decision Trees](#23-decision-trees)
24. [Quick Reference Tables](#24-quick-reference-tables)

---

## 1. Authentication Architecture Overview

The Water Framework authentication system is organized across multiple modules with clear
separation of concerns:

```
+-------------------------------------------------------------------+
|                     REST Layer (Incoming Request)                   |
|                                                                     |
|  POST /authentication/login  ──>  AuthenticationRestApi             |
|  GET /users (with @LoggedIn) ──>  JWT Filter validates token        |
+-------------------------------------------------------------------+
          │                                    │
          v                                    v
+---------------------------+    +-----------------------------+
| Authentication Module     |    | Rest-security Module        |
|                           |    |                             |
| AuthenticationApi         |    | GenericJWTAuthFilter        |
| AuthenticationSystemApi   |    | NimbusJwtTokenService       |
| AuthenticationOption      |    | JwtSecurityContext           |
+---------------------------+    +-----------------------------+
          │                                    │
          v                                    v
+---------------------------+    +-----------------------------+
| Core-api Interfaces       |    | Core-security Model         |
|                           |    |                             |
| AuthenticationProvider    |    | WaterAbstractSecurityContext |
| Authenticable             |    | UserPrincipal               |
| User                      |    | RolePrincipal               |
| EncryptionUtil            |    | BasicSecurityContext         |
+---------------------------+    +-----------------------------+
          │
          v
+---------------------------+
| User Module               |
|                           |
| WaterUser (entity)        |
| UserAuthenticationProvider|
| UserSystemApi             |
| UserOptions               |
+---------------------------+
```

### Key Principles

1. **Separation of Authentication from User Management**: The Authentication module handles
   login orchestration and token generation. The User module handles user CRUD, registration,
   and provides an `AuthenticationProvider` implementation.

2. **Issuer-Based Provider Selection**: Multiple `AuthenticationProvider` implementations can
   coexist. The system selects the correct one based on the configured **issuer name**.

3. **JWT as the Token Format**: After successful authentication, a JWT token is generated
   using RSA256 signing. All subsequent requests use this token for identity verification.

4. **Framework-Agnostic Core**: The core authentication logic is framework-agnostic. Spring
   and CXF provide thin adapter filters.

---

## 2. Module Dependency Map

```
Authentication-api                    Rest-api
├── AuthenticationApi                 ├── LoggedIn (annotation)
├── AuthenticationSystemApi           ├── JwtTokenService (interface)
├── AuthenticationOption              ├── RsSecurityContext
└── AuthenticationRestApi             └── JwtSecurityOptions
         │                                      │
         v                                      v
Authentication-service                Rest-security
├── AuthenticationServiceImpl         ├── GenericJWTAuthFilter
├── AuthenticationSystemServiceImpl   ├── NimbusJwtTokenService
├── AuthenticationOptionImpl          ├── JwtSecurityContext
├── AuthenticationConstants           ├── JWTConstants
├── AuthenticationModule (JAAS)       └── JwtSecurityOptionsImpl
└── AuthenticationRestControllerImpl
         │                                      │
         v                                      v
Core-api                              Core-security
├── AuthenticationProvider            ├── UserPrincipal
├── Authenticable                     ├── RolePrincipal
├── User                              └── WaterAbstractSecurityContext
├── EncryptionUtil
└── Role
         │
         v
User-api / User-model / User-service
├── WaterUser (entity, implements User + Authenticable)
├── UserAuthenticationProvider (implements AuthenticationProvider)
├── UserApi / UserSystemApi
├── UserRestApi
├── UserOptions / UserOptionsImpl
└── UserConstants
```

### Key Source File Locations

| File | Path |
|------|------|
| AuthenticationApi | `Authentication/Authentication-api/src/main/java/it/water/authentication/api/AuthenticationApi.java` |
| AuthenticationSystemApi | `Authentication/Authentication-api/src/main/java/it/water/authentication/api/AuthenticationSystemApi.java` |
| AuthenticationOption | `Authentication/Authentication-api/src/main/java/it/water/authentication/api/options/AuthenticationOption.java` |
| AuthenticationRestApi | `Authentication/Authentication-api/src/main/java/it/water/authentication/api/rest/AuthenticationRestApi.java` |
| AuthenticationServiceImpl | `Authentication/Authentication-service/src/main/java/it/water/authentication/service/AuthenticationServiceImpl.java` |
| AuthenticationSystemServiceImpl | `Authentication/Authentication-service/src/main/java/it/water/authentication/service/AuthenticationSystemServiceImpl.java` |
| AuthenticationOptionImpl | `Authentication/Authentication-service/src/main/java/it/water/authentication/service/AuthenticationOptionImpl.java` |
| AuthenticationRestControllerImpl | `Authentication/Authentication-service/src/main/java/it/water/authentication/service/rest/AuthenticationRestControllerImpl.java` |
| AuthenticationProvider | `Core/Core-api/src/main/java/it/water/core/api/security/AuthenticationProvider.java` |
| Authenticable | `Core/Core-api/src/main/java/it/water/core/api/security/Authenticable.java` |
| User | `Core/Core-api/src/main/java/it/water/core/api/model/User.java` |
| EncryptionUtil | `Core/Core-api/src/main/java/it/water/core/api/security/EncryptionUtil.java` |
| LoggedIn | `Rest/Rest-api/src/main/java/it/water/service/rest/api/security/LoggedIn.java` |
| JwtTokenService | `Rest/Rest-api/src/main/java/it/water/service/rest/api/security/jwt/JwtTokenService.java` |
| JwtSecurityOptions | `Rest/Rest-api/src/main/java/it/water/service/rest/api/options/JwtSecurityOptions.java` |
| GenericJWTAuthFilter | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/GenericJWTAuthFilter.java` |
| NimbusJwtTokenService | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/NimbusJwtTokenService.java` |
| JwtSecurityContext | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JwtSecurityContext.java` |
| JWTConstants | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JWTConstants.java` |
| JwtSecurityOptionsImpl | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JwtSecurityOptionsImpl.java` |
| WaterUser | `User/User-model/src/main/java/it/water/user/model/WaterUser.java` |
| UserAuthenticationProvider | `User/User-service/src/main/java/it/water/user/service/UserAuthenticationProvider.java` |
| UserApi | `User/User-api/src/main/java/it/water/user/api/UserApi.java` |
| UserSystemApi | `User/User-api/src/main/java/it/water/user/api/UserSystemApi.java` |
| UserRestApi | `User/User-api/src/main/java/it/water/user/api/rest/UserRestApi.java` |
| UserOptions | `User/User-api/src/main/java/it/water/user/api/options/UserOptions.java` |
| UserOptionsImpl | `User/User-service/src/main/java/it/water/user/service/UserOptionsImpl.java` |
| UserConstants | `User/User-model/src/main/java/it/water/user/model/UserConstants.java` |
| SpringJwtAuthFilter | `Rest/Rest-spring-api/src/main/java/it/water/service/rest/spring/security/SpringJwtAuthenticationFilter.java` |
| CxfJwtAuthFilter | `Rest/Rest-api-manager-apache-cxf/src/main/java/it/water/service/rest/manager/cxf/security/filters/jwt/CxfJwtAuthenticationFilter.java` |

---

## 3. Core Security Interfaces

### Authenticable Interface

The fundamental contract for any entity that can authenticate (users, devices, sensors).

**File:** `Core/Core-api/src/main/java/it/water/core/api/security/Authenticable.java`

```java
public interface Authenticable {
    Long getLoggedEntityId();       // Unique ID of the logged entity
    String getIssuer();             // Issuer name (e.g., "it.water.core.api.model.User")
    String getScreenNameFieldName();// Field name used as screen name (e.g., "username")
    String getScreenName();         // The display name (e.g., the username)
    String getPassword();           // Hashed password
    String getSalt();               // Base64-encoded salt
    boolean isAdmin();              // True if admin
    boolean isActive();             // True if activated for authentication
    Collection<Role> getRoles();    // Associated roles
}
```

**Key Points:**
- Any entity that implements `Authenticable` can participate in the authentication flow
- The `issuer` field is critical: it determines which `AuthenticationProvider` handles this entity
- `isActive()` must return `true` for authentication to succeed (checked in JAAS module)
- `getRoles()` is used to populate JWT claims and SecurityContext principals

### User Interface

Extends `Authenticable` with user-specific fields.

**File:** `Core/Core-api/src/main/java/it/water/core/api/model/User.java`

```java
public interface User extends Authenticable {
    long getId();
    String getName();
    String getLastname();
    String getEmail();
    String getUsername();
    boolean isAdmin();

    // Default implementations
    default String getScreenName() { return this.getUsername(); }
    default String getScreenNameFieldName() { return "username"; }
    default boolean isActive() { return false; }  // Override in entity!
}
```

**Important:** The default `isActive()` returns `false`. The concrete `WaterUser` entity
overrides this with a persisted `active` field.

### AuthenticationProvider Interface

The contract for providing authentication services.

**File:** `Core/Core-api/src/main/java/it/water/core/api/security/AuthenticationProvider.java`

```java
public interface AuthenticationProvider extends Service {
    Authenticable login(String username, String password);
    Collection<String> issuersNames();
}
```

**Key Points:**
- `login()`: Validates credentials and returns an `Authenticable` or throws `UnauthorizedException`
- `issuersNames()`: Returns the list of issuer names this provider handles
- Multiple providers can coexist, each handling different issuer names
- The Authentication module selects the right provider by matching issuer names

### EncryptionUtil Interface

Provides cryptographic operations including password hashing.

**File:** `Core/Core-api/src/main/java/it/water/core/api/security/EncryptionUtil.java`

Key methods for authentication:
```java
public interface EncryptionUtil extends Service {
    byte[] hashPassword(byte[] salt, String password)
        throws NoSuchAlgorithmException, InvalidKeySpecException;
    byte[] generate16BytesSalt();
    KeyPair getServerKeyPair();           // Used for JWT signing
    String getServerKeystoreFilePath();
    String getServerKeystorePassword();
    String getServerKeyPassword();
    String getServerKeystoreAlias();
    // ... RSA, AES, and SSL operations
}
```

**Password hashing** uses PBKDF2 algorithm by default.

---

## 4. Authentication Module Deep Dive

The Authentication module is the central orchestrator for login operations.

### AuthenticationApi (Public API)

**File:** `Authentication/Authentication-api/src/main/java/it/water/authentication/api/AuthenticationApi.java`

```java
public interface AuthenticationApi extends BaseApi {
    Authenticable login(String username, String password);
    String generateToken(Authenticable authenticable);
}
```

### AuthenticationSystemApi (System API)

**File:** `Authentication/Authentication-api/src/main/java/it/water/authentication/api/AuthenticationSystemApi.java`

```java
public interface AuthenticationSystemApi extends BaseSystemApi {
    Authenticable login(String username, String password);
    Authenticable login(String username, String password, String authProviderFilter);
    String generateToken(Authenticable authenticable);
}
```

The SystemApi adds an overload that accepts a specific `authProviderFilter` (issuer name)
for explicitly selecting the AuthenticationProvider.

### AuthenticationServiceImpl (Api Implementation)

**File:** `Authentication/Authentication-service/src/main/java/it/water/authentication/service/AuthenticationServiceImpl.java`

```java
@FrameworkComponent
public class AuthenticationServiceImpl extends BaseServiceImpl implements AuthenticationApi {
    @Inject private AuthenticationSystemApi systemService;

    @Override
    public Authenticable login(String username, String password) {
        return systemService.login(username, password);
    }

    @Override
    public String generateToken(Authenticable authenticable) {
        return systemService.generateToken(authenticable);
    }
}
```

Simple delegation pattern: Api -> SystemApi.

### AuthenticationSystemServiceImpl (Core Logic)

**File:** `Authentication/Authentication-service/src/main/java/it/water/authentication/service/AuthenticationSystemServiceImpl.java`

```java
@FrameworkComponent
public class AuthenticationSystemServiceImpl extends BaseSystemServiceImpl
    implements AuthenticationSystemApi {

    @Inject private ComponentRegistry componentRegistry;
    @Inject private AuthenticationOption authenticationOption;
    @Inject private JwtTokenService jwtTokenService;

    @Override
    public Authenticable login(String username, String password) {
        String issuerName = authenticationOption.getIssuerName();
        return login(username, password, issuerName);
    }

    @Override
    public Authenticable login(String username, String password, String issuerName) {
        // 1. Find all registered AuthenticationProviders
        Collection<AuthenticationProvider> providers =
            componentRegistry.findComponents(AuthenticationProvider.class, null);
        // 2. Filter by issuer name
        Optional<AuthenticationProvider> providerOpt = providers.stream()
            .filter(p -> p.issuersNames().contains(issuerName))
            .findFirst();
        // 3. Throw if no provider found
        if (providerOpt.isEmpty())
            throw new UnauthorizedException("No authentication provider found for " + issuerName);
        // 4. Delegate to the matched provider
        return providerOpt.get().login(username, password);
    }

    @Override
    public String generateToken(Authenticable authenticable) {
        return jwtTokenService.generateJwtToken(authenticable);
    }
}
```

**Critical Flow:**
1. Read the configured issuer name from `AuthenticationOption`
2. Scan all registered `AuthenticationProvider` components
3. Find the one whose `issuersNames()` contains the configured issuer
4. Delegate `login()` to that provider
5. On success, generate JWT token via `JwtTokenService`

### AuthenticationRestApi (REST Endpoint)

**File:** `Authentication/Authentication-api/src/main/java/it/water/authentication/api/rest/AuthenticationRestApi.java`

```java
@Path("/authentication")
@FrameworkRestApi
public interface AuthenticationRestApi extends RestApi {
    @POST
    @Path("/login")
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    @Produces(MediaType.APPLICATION_JSON)
    Map<String, String> login(
        @FormParam("username") String username,
        @FormParam("password") String password
    );
}
```

**Key Points:**
- Endpoint: `POST /authentication/login`
- Content-Type: `application/x-www-form-urlencoded`
- Parameters: `username` and `password` as form params
- Response: JSON `{"token": "<jwt_token_string>"}`
- No `@LoggedIn` annotation (login is unauthenticated)

### AuthenticationRestControllerImpl

**File:** `Authentication/Authentication-service/src/main/java/it/water/authentication/service/rest/AuthenticationRestControllerImpl.java`

```java
@FrameworkRestController(referredRestApi = AuthenticationRestApi.class)
public class AuthenticationRestControllerImpl implements AuthenticationRestApi {
    @Inject private AuthenticationApi authenticationApi;

    @Override
    public Map<String, String> login(String username, String password) {
        Authenticable authenticable = authenticationApi.login(username, password);
        String token = authenticationApi.generateToken(authenticable);
        Map<String, String> response = new HashMap<>();
        response.put("token", token);
        return response;
    }
}
```

**Two-step process:**
1. `authenticationApi.login()` - validates credentials, returns Authenticable
2. `authenticationApi.generateToken()` - creates JWT token
3. Return token in response map

---

## 5. AuthenticationProvider Pattern

The `AuthenticationProvider` pattern allows pluggable authentication backends.

### How It Works

```
                  AuthenticationSystemServiceImpl
                            │
                            │ findComponents(AuthenticationProvider.class)
                            v
            ┌───────────────┼───────────────┐
            │               │               │
   UserAuthProvider    DeviceAuthProvider  CustomAuthProvider
   issuer: "User"     issuer: "Device"    issuer: "MyApp"
```

Each provider:
1. Registers as `@FrameworkComponent(services = AuthenticationProvider.class)`
2. Implements `issuersNames()` returning its supported issuers
3. Implements `login(username, password)` with its validation logic

### Creating a Custom AuthenticationProvider

```java
@FrameworkComponent(services = AuthenticationProvider.class)
public class MyDeviceAuthenticationProvider implements AuthenticationProvider {

    @Inject
    private MyDeviceSystemApi deviceSystemApi;

    @Inject
    private EncryptionUtil encryptionUtil;

    @Override
    public Authenticable login(String deviceId, String apiKey) {
        MyDevice device = deviceSystemApi.findByDeviceId(deviceId);
        if (device == null || apiKey == null || apiKey.isBlank())
            throw new UnauthorizedException("Invalid device credentials");
        // Validate API key
        if (!device.getApiKey().equals(apiKey))
            throw new UnauthorizedException("Invalid device credentials");
        return device; // MyDevice must implement Authenticable
    }

    @Override
    public Collection<String> issuersNames() {
        return List.of("com.myapp.model.Device");
    }
}
```

### Registering the Issuer

Configure in `water-application.properties`:
```properties
water.authentication.service.issuer=com.myapp.model.Device
```

---

## 6. User Module Overview

The User module provides a complete user management system including registration,
activation, deactivation, password management, and deletion. It also provides the
default `AuthenticationProvider` for user-based login.

### Module Structure

```
User/
├── User-api/
│   ├── UserApi.java              # Permission-checked API
│   ├── UserSystemApi.java        # System-level API (bypasses permissions)
│   ├── UserRepository.java       # Repository interface
│   ├── rest/
│   │   └── UserRestApi.java      # REST endpoints
│   └── options/
│       └── UserOptions.java      # Configuration interface
├── User-model/
│   ├── WaterUser.java            # JPA entity
│   └── UserConstants.java        # Property key constants
└── User-service/
    ├── UserServiceImpl.java      # Api implementation
    ├── UserSystemServiceImpl.java # System service implementation
    ├── UserAuthenticationProvider.java  # Login implementation
    ├── UserOptionsImpl.java      # Options implementation
    └── UserRepositoryImpl.java   # Repository implementation
```

### UserApi Capabilities

```java
public interface UserApi extends BaseEntityApi<WaterUser> {
    WaterUser findByUsername(String username);
    void register(WaterUser user);
    void unregisterAccountRequest();
    void unregister(String email, String deletionCode);
    PaginableResult<WaterUser> findAllDeleted(int delta, int page, Query filter, QueryOrder queryOrder);
    long countAllDeleted(Query filter);
    void activate(String email, String activationCode);
    void activate(long userId);
    void deactivate(long userId);
    void passwordResetRequest(String email);
    void resetPassword(String email, String resetCode, String password, String passwordConfirm);
    WaterUser changePassword(long userId, String oldPassword, String newPassword, String passwordConfirm);
    WaterUser updateAccountInfo(WaterUser user);
}
```

### UserSystemApi Capabilities

```java
public interface UserSystemApi extends BaseEntitySystemApi<WaterUser> {
    WaterUser register(WaterUser user);
    void activateUser(String email, String activationCode);
    void activateUser(long userId);
    void deactivateUser(long userId);
    void unregister(String email, String deletionCode);
    PaginableResult<WaterUser> findAllDeleted(int delta, int page, Query filter, QueryOrder queryOrder);
    long countAllDeleted(Query filter);
    WaterUser findByUsername(String username);
    WaterUser findByEmail(String email);
    WaterUser changeDeletionCode(String deletionCode);
    WaterUser changePassword(WaterUser user, String password, String passwordConfirm);
}
```

---

## 7. WaterUser Entity Model

**File:** `User/User-model/src/main/java/it/water/user/model/WaterUser.java`

```java
@Entity
@Table(name = "w_user", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"username"}),
    @UniqueConstraint(columnNames = {"email"})
})
@AccessControl(
    availableActions = {
        CrudActions.SAVE, CrudActions.REMOVE, CrudActions.FIND,
        CrudActions.FIND_ALL, CrudActions.UPDATE,
        UserActions.IMPERSONATE, UserActions.ACTIVATE, UserActions.DEACTIVATE
    },
    rolesPermissions = {
        @DefaultRoleAccess(roleName = "userManager",
            actions = {SAVE, REMOVE, FIND, FIND_ALL, UPDATE, IMPERSONATE, ACTIVATE, DEACTIVATE}),
        @DefaultRoleAccess(roleName = "userViewer",
            actions = {FIND, FIND_ALL}),
        @DefaultRoleAccess(roleName = "userEditor",
            actions = {SAVE, FIND, FIND_ALL, UPDATE})
    }
)
public class WaterUser extends AbstractJpaExpandableEntity
    implements ProtectedEntity, User {
```

### Entity Fields

| Field | Type | Constraints | JSON View | Notes |
|-------|------|-------------|-----------|-------|
| `name` | String | @NotEmpty, @Size(max=500), @NoMalitiusCode | Extended | |
| `lastname` | String | @NotEmpty, @Size(max=500), @NoMalitiusCode | Extended | |
| `username` | String | @NotEmpty, @Size(max=400), @Pattern(alphanumeric+.-_) | Extended | Unique |
| `password` | String | @NotNull, @ValidPassword, @NoMalitiusCode | WRITE_ONLY | Hashed |
| `salt` | String | | @JsonIgnore | Base64-encoded |
| `admin` | boolean | default false | Compact (READ_ONLY) | |
| `email` | String | @Email, @NotEmpty, @NoMalitiusCode | Extended | Unique |
| `active` | boolean | default false | Internal | Must be true for login |
| `deleted` | boolean | default false | Internal | Logical deletion |
| `imagePath` | String | @NoMalitiusCode | Extended | |
| `passwordConfirm` | String | @Transient | Internal | Registration only |
| `passwordResetCode` | String | @JsonIgnore | - | For password reset flow |
| `activateCode` | String | @JsonIgnore | - | For registration activation |
| `deletionCode` | String | @JsonIgnore | - | For account deletion |
| `roles` | Set<Role> | @Transient, @JsonIgnore | - | Loaded at login time |

### Constants

```java
public static final String DEFAULT_MANAGER_ROLE = "userManager";
public static final String DEFAULT_VIEWER_ROLE = "userViewer";
public static final String DEFAULT_EDITOR_ROLE = "userEditor";
public static final String WATER_USER_ISSUER = User.class.getName();
// = "it.water.core.api.model.User"
```

### Authenticable Implementation

```java
@Override
public Long getLoggedEntityId() { return getId(); }

@Override
public String getIssuer() { return WATER_USER_ISSUER; }
// "it.water.core.api.model.User"

@Override
public String getScreenName() { return User.super.getScreenName(); }
// delegates to User default -> getUsername()

@Override
public String getScreenNameFieldName() { return User.super.getScreenNameFieldName(); }
// returns "username"
```

### Password Management Methods

```java
public void updateAccountInfo(String name, String lastname, String email, String username) {
    this.name = name;
    this.lastname = lastname;
    this.email = email;
    this.username = username;
}

public void updatePassword(String salt, String password, String passwordConfirm) {
    if (password != null && passwordConfirm != null
        && !password.isBlank() && !passwordConfirm.isBlank()
        && password.equals(passwordConfirm)) {
        this.salt = salt;
        this.password = password;
        this.passwordConfirm = passwordConfirm;
        return;
    }
    throw new ValidationException(
        Collections.singletonList(new ValidationError("Password do not match or invalid", "password", "-"))
    );
}
```

---

## 8. UserAuthenticationProvider Implementation

**File:** `User/User-service/src/main/java/it/water/user/service/UserAuthenticationProvider.java`

```java
@FrameworkComponent(services = AuthenticationProvider.class)
public class UserAuthenticationProvider implements AuthenticationProvider {
    public static final String WRONG_USER_OR_PWD_MESSAGE = "username or password incorrect!";

    @Inject private UserSystemApi userSystemApi;
    @Inject private EncryptionUtil encryptionUtil;
    @Inject private RoleManager roleManager;

    @Override
    public Authenticable login(String username, String password) {
        // 1. Find user by username
        WaterUser u = userSystemApi.findByUsername(username);
        if (u == null || password == null || password.isBlank())
            throw new UnauthorizedException(WRONG_USER_OR_PWD_MESSAGE);

        // 2. Hash the provided password with the user's stored salt
        try {
            byte[] salt = Base64.getDecoder().decode(u.getSalt());
            password = new String(encryptionUtil.hashPassword(salt, password));
        } catch (NoSuchAlgorithmException | InvalidKeySpecException e) {
            throw new UnauthorizedException(WRONG_USER_OR_PWD_MESSAGE);
        }

        // 3. Compare hashed passwords
        if (!u.getPassword().equals(password))
            throw new UnauthorizedException(WRONG_USER_OR_PWD_MESSAGE);

        // 4. Load user roles
        if (roleManager != null) {
            u.setRoles(roleManager.getUserRoles(u.getId()));
        } else {
            u.setRoles(Collections.emptySet());
        }
        return u;
    }

    @Override
    public Collection<String> issuersNames() {
        return List.of(WaterUser.WATER_USER_ISSUER);
        // ["it.water.core.api.model.User"]
    }
}
```

### Login Validation Steps

1. **Find user** by username via `UserSystemApi.findByUsername()`
2. **Null/empty checks** on user and password
3. **Decode salt** from Base64 stored in user record
4. **Hash password** using `EncryptionUtil.hashPassword(salt, password)` (PBKDF2)
5. **Compare** hashed password with stored hash
6. **Load roles** via `RoleManager.getUserRoles(userId)`
7. **Return** the WaterUser as Authenticable (with roles populated)

### Security Considerations

- The error message is intentionally generic ("username or password incorrect!")
  to prevent username enumeration
- Salt is stored Base64-encoded alongside the user record
- PBKDF2 is used for password hashing (configurable via EncryptionUtil)
- Roles are loaded at login time and embedded in the JWT token

---

## 9. Login Flow End-to-End

### Complete Request Flow

```
Client                     REST Layer              Auth Module           User Module         JWT Service
  │                           │                       │                     │                   │
  │ POST /authentication/login│                       │                     │                   │
  │ username=admin&password=X │                       │                     │                   │
  │──────────────────────────>│                       │                     │                   │
  │                           │                       │                     │                   │
  │                           │ login(admin, X)       │                     │                   │
  │                           │──────────────────────>│                     │                   │
  │                           │                       │                     │                   │
  │                           │                       │ getIssuerName()     │                   │
  │                           │                       │ = "User"            │                   │
  │                           │                       │                     │                   │
  │                           │                       │ findComponents(     │                   │
  │                           │                       │   AuthProvider)     │                   │
  │                           │                       │ filter by issuer    │                   │
  │                           │                       │                     │                   │
  │                           │                       │ provider.login()    │                   │
  │                           │                       │────────────────────>│                   │
  │                           │                       │                     │                   │
  │                           │                       │                     │ findByUsername()   │
  │                           │                       │                     │ hashPassword()     │
  │                           │                       │                     │ comparePasswords() │
  │                           │                       │                     │ loadRoles()        │
  │                           │                       │                     │                   │
  │                           │                       │  Authenticable      │                   │
  │                           │                       │<────────────────────│                   │
  │                           │                       │                     │                   │
  │                           │ generateToken(auth)   │                     │                   │
  │                           │──────────────────────>│                     │                   │
  │                           │                       │ generateJwtToken()  │                   │
  │                           │                       │────────────────────────────────────────>│
  │                           │                       │                     │                   │
  │                           │                       │                     │   JWT Token       │
  │                           │                       │<────────────────────────────────────────│
  │                           │                       │                     │                   │
  │                           │   {"token": "eyJ..."}  │                     │                   │
  │<──────────────────────────│                       │                     │                   │
```

### Step-by-Step

1. Client sends `POST /authentication/login` with form-encoded `username` and `password`
2. `AuthenticationRestControllerImpl.login()` delegates to `AuthenticationApi.login()`
3. `AuthenticationServiceImpl` delegates to `AuthenticationSystemServiceImpl`
4. System service reads configured issuer from `AuthenticationOption.getIssuerName()`
5. Scans registry for `AuthenticationProvider` matching the issuer
6. Calls `UserAuthenticationProvider.login(username, password)`
7. Provider finds user by username, hashes password, compares, loads roles
8. Returns `WaterUser` as `Authenticable`
9. Back in REST controller: `authenticationApi.generateToken(authenticable)`
10. `JwtTokenService.generateJwtToken()` creates RSA256-signed JWT
11. Token returned as `{"token": "eyJ..."}`

---

## 10. JWT Token System

### JwtTokenService Interface

**File:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/security/jwt/JwtTokenService.java`

```java
public interface JwtTokenService extends Service {
    String generateJwtToken(Authenticable authenticable);
    boolean validateToken(List<String> validIssuers, String jwtStr);
    Set<Principal> getPrincipals(String jwtToken);
    String getJWK();
}
```

### NimbusJwtTokenService Implementation

**File:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/NimbusJwtTokenService.java`

Uses the **Nimbus JOSE+JWT** library for JWT operations.

#### Token Generation

```java
public String generateJwtToken(Authenticable authenticable) {
    // 1. Get RSA private key from server keystore
    RSAPrivateKey privateKey = (RSAPrivateKey) encryptionUtil.getServerKeyPair().getPrivate();

    // 2. Create RSA signer
    JWSSigner signer = new RSASSASigner(privateKey);

    // 3. Build JWT claims
    JWTClaimsSet claimsSet = new JWTClaimsSet.Builder()
        .subject(authenticable.getScreenName())           // username
        .claim("roles", roleNames)                        // role names list
        .claim("isAdmin", authenticable.isAdmin())        // admin flag
        .claim("loggedEntityId", authenticable.getLoggedEntityId())  // entity ID
        .issuer(authenticable.getIssuer())                // e.g., "it.water.core.api.model.User"
        .expirationTime(new Date(now + durationMillis))   // expiration
        .issueTime(new Date(now))                         // issued at
        .build();

    // 4. Create signed JWT with RS256 algorithm
    SignedJWT signedJWT = new SignedJWT(
        new JWSHeader.Builder(JWSAlgorithm.RS256)
            .keyID(jwtSecurityOptions.jwtKeyId())
            .build(),
        claimsSet
    );
    signedJWT.sign(signer);

    // 5. Optionally encrypt the token
    if (jwtSecurityOptions.encryptJWTToken()) {
        // JWE encryption with RSA-OAEP and A256GCM
    }

    return signedJWT.serialize();
}
```

#### JWT Claims Structure

| Claim | Source | Description |
|-------|--------|-------------|
| `sub` (subject) | `authenticable.getScreenName()` | Username or device name |
| `roles` | Role names from `getRoles()` | List of role name strings |
| `isAdmin` | `authenticable.isAdmin()` | Boolean admin flag |
| `loggedEntityId` | `authenticable.getLoggedEntityId()` | Entity primary key |
| `iss` (issuer) | `authenticable.getIssuer()` | Provider issuer string |
| `exp` (expiration) | Current time + duration | Token expiry timestamp |
| `iat` (issued at) | Current time | Token creation timestamp |

#### Token Validation

```java
public boolean validateToken(List<String> validIssuers, String jwtStr) {
    // 1. Parse the signed JWT
    SignedJWT signedJWT = SignedJWT.parse(jwtStr);

    // 2. Verify signature
    if (jwtSecurityOptions.validateJwtWithJwsUrl()) {
        // Validate using external JWS URL (for distributed systems)
        // Fetches public key from remote endpoint
    } else {
        // Validate using local server public key
        RSAPublicKey publicKey = (RSAPublicKey) encryptionUtil.getServerKeyPair().getPublic();
        JWSVerifier verifier = new RSASSAVerifier(publicKey);
        if (!signedJWT.verify(verifier)) return false;
    }

    // 3. Check expiration
    Date expirationTime = signedJWT.getJWTClaimsSet().getExpirationTime();
    if (expirationTime != null && expirationTime.before(new Date())) return false;

    // 4. Check issuer (if validIssuers provided)
    String issuer = signedJWT.getJWTClaimsSet().getIssuer();
    if (validIssuers != null && !validIssuers.isEmpty()) {
        if (!validIssuers.contains(issuer)) return false;
    }

    return true;
}
```

#### Principal Extraction

```java
public Set<Principal> getPrincipals(String jwtTokenStr) {
    SignedJWT signedJWT = SignedJWT.parse(jwtTokenStr);
    JWTClaimsSet claims = signedJWT.getJWTClaimsSet();

    Set<Principal> principals = new HashSet<>();

    // Extract user principal
    String subject = claims.getSubject();
    boolean isAdmin = (Boolean) claims.getClaim("isAdmin");
    long loggedEntityId = ((Number) claims.getClaim("loggedEntityId")).longValue();
    String issuer = claims.getIssuer();

    principals.add(new UserPrincipal(subject, isAdmin, loggedEntityId, issuer));

    // Extract role principals
    List<String> roles = (List<String>) claims.getClaim("roles");
    if (roles != null) {
        roles.forEach(role -> principals.add(new RolePrincipal(role)));
    }

    return principals;
}
```

#### JWK Endpoint

```java
public String getJWK() {
    RSAPublicKey publicKey = (RSAPublicKey) encryptionUtil.getServerKeyPair().getPublic();
    RSAKey jwk = new RSAKey.Builder(publicKey)
        .keyID(jwtSecurityOptions.jwtKeyId())
        .build();
    return jwk.toJSONString();
}
```

Useful for **distributed systems** where other services need to validate tokens
using the issuing server's public key.

---

## 11. JWT Security Filter Chain

### GenericJWTAuthFilter (Base Class)

**File:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/GenericJWTAuthFilter.java`

This abstract class provides the core JWT validation logic shared by all framework-specific filters.

```java
public class GenericJWTAuthFilter {

    // Validate token from Authorization header or cookie
    protected void validateToken(JwtTokenService jwtTokenService,
                                  LoggedIn annotation,
                                  String authorizationHeader,
                                  String cookieValue) {
        String token = getTokenFromRequest(authorizationHeader, cookieValue);
        if (token == null) {
            throw new NotAuthorizedException("No Bearer token found");
        }

        List<String> issuers = Arrays.asList(annotation.issuers());
        if (!jwtTokenService.validateToken(issuers, token)) {
            throw new NotAuthorizedException("Invalid or expired JWT token");
        }
    }

    // Extract Bearer token from header or cookie
    protected String getTokenFromRequest(String authorization, String cookieValue) {
        if (authorization != null && authorization.startsWith("Bearer ")) {
            return authorization.substring(7);  // Remove "Bearer " prefix
        }
        if (cookieValue != null) {
            return cookieValue;
        }
        return null;
    }

    // Search annotation through class hierarchy
    protected Object getAnnotationFromHierarchy(Class annotationClass, Method method) {
        // Traverses the method's declaring class and all its interfaces
        // to find the @LoggedIn annotation
    }
}
```

### Token Extraction Priority

1. **Authorization Header**: `Authorization: Bearer <token>` (preferred)
2. **Cookie**: `HIT-AUTH=<token>` (fallback)

### Validation Steps in Filter

1. Check if JWT validation is enabled (`restOptions.securityOptions().validateJwt()`)
2. Find `@LoggedIn` annotation on the method (traverses class hierarchy)
3. Extract token from Authorization header or cookie
4. Validate token signature, expiration, and issuer
5. Extract principals from token
6. Create SecurityContext and fill it into the Runtime

---

## 12. Framework-Specific Filters (Spring, CXF)

### Spring Filter

**File:** `Rest/Rest-spring-api/src/main/java/it/water/service/rest/spring/security/SpringJwtAuthenticationFilter.java`

```java
public class SpringJwtAuthenticationFilter extends GenericJWTAuthFilter
    implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        if (restOptions.securityOptions().validateJwt()
            && handler instanceof HandlerMethod handlerMethod) {

            Method method = handlerMethod.getMethod();
            LoggedIn annotation = (LoggedIn) getAnnotationFromHierarchy(LoggedIn.class, method);

            if (annotation != null) {
                String authorization = request.getHeader("Authorization");
                String cookie = extractCookie(request, JWTConstants.JWT_COOKIE_NAME);

                JwtTokenService jwtTokenService = componentRegistry.findComponent(JwtTokenService.class, null);
                validateToken(jwtTokenService, annotation, authorization, cookie);

                // Fill SecurityContext
                Runtime runtime = componentRegistry.findComponent(Runtime.class, null);
                runtime.fillSecurityContext(
                    new SpringSecurityContext(
                        jwtTokenService.getPrincipals(getTokenFromRequest(authorization, cookie))
                    )
                );
            }
        }
        return true;
    }
}
```

**Spring-specific**: Uses `HandlerInterceptor.preHandle()`, creates `SpringSecurityContext`.

### CXF (Apache CXF) Filter

**File:** `Rest/Rest-api-manager-apache-cxf/src/main/java/it/water/service/rest/manager/cxf/security/filters/jwt/CxfJwtAuthenticationFilter.java`

```java
@Priority(Priorities.AUTHENTICATION)
@LoggedIn
@Provider
public class CxfJwtAuthenticationFilter extends GenericJWTAuthFilter
    implements ContainerRequestFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) {
        if (!restOptions.securityOptions().validateJwt()) return;

        String authorizationHeader = requestContext.getHeaderString(HttpHeaders.AUTHORIZATION);
        Cookie c = requestContext.getCookies().get(JWTConstants.JWT_COOKIE_NAME);
        String cookieVal = c != null ? c.getValue() : null;

        LoggedIn annotation = info.getResourceMethod().getAnnotation(LoggedIn.class);
        validateToken(jwtTokenService, annotation, authorizationHeader, cookieVal);

        // Create SecurityContext
        String encodedToken = getTokenFromRequest(authorizationHeader, cookieVal);
        SecurityContext securityContext = new JwtSecurityContext(
            jwtTokenService.getPrincipals(encodedToken)
        );
        Runtime runtime = componentRegistry.findComponent(Runtime.class, null);
        runtime.fillSecurityContext(securityContext);
    }
}
```

**CXF-specific**: Uses JAX-RS `ContainerRequestFilter`, `@NameBinding` with `@LoggedIn`,
creates `JwtSecurityContext`. On error, aborts with 401 JSON response.

### Key Differences

| Aspect | Spring | CXF |
|--------|--------|-----|
| Filter type | `HandlerInterceptor` | `ContainerRequestFilter` |
| Annotation binding | Manual hierarchy search | `@NameBinding` with `@LoggedIn` |
| SecurityContext | `SpringSecurityContext` | `JwtSecurityContext` |
| Error handling | Exception propagation | `requestContext.abortWith(401)` |
| Registration | Registered as Spring bean | `@Provider` with `@Priority(AUTHENTICATION)` |

### Spring-Specific REST Controller

For Spring, the REST API must also provide Spring annotations:

```java
// Authentication-service-spring
@RequestMapping("/authentication")
@FrameworkRestApi
public interface AuthenticationSpringRestApi extends AuthenticationRestApi {
    @PostMapping(path = "/login",
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE)
    @Override
    Map<String, String> login(@RequestParam("username") String username,
                               @RequestParam("password") String password);
}

@RestController
public class AuthenticationSpringRestControllerImpl
    extends AuthenticationRestControllerImpl
    implements AuthenticationSpringRestApi {

    @Override
    public Map<String, String> login(String username, String password) {
        return super.login(username, password);
    }
}
```

---

## 13. @LoggedIn Annotation & Endpoint Protection

### The @LoggedIn Annotation

**File:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/security/LoggedIn.java`

```java
@NameBinding
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface LoggedIn {
    String[] issuers() default {"it.water.core.api.model.User"};
}
```

### How to Use

#### Protect an entire REST API (all endpoints):
```java
@Path("/myresource")
@LoggedIn
@FrameworkRestApi
public interface MyResourceRestApi extends RestApi {
    // All methods require authentication
}
```

#### Protect specific endpoints:
```java
@Path("/myresource")
@FrameworkRestApi
public interface MyResourceRestApi extends RestApi {
    @LoggedIn   // This requires authentication
    @GET
    @Path("/{id}")
    MyResource find(@PathParam("id") long id);

    @POST       // This does NOT require authentication
    @Path("/public")
    void publicAction();
}
```

#### Accept multiple issuers:
```java
@LoggedIn(issuers = {
    "it.water.core.api.model.User",
    "com.myapp.model.Device"
})
@GET
MyResource findByDevice(@PathParam("id") long id);
```

### Important Rules

1. `@LoggedIn` with default issuers only accepts tokens from the User issuer
2. Specify custom issuers to accept tokens from other `AuthenticationProvider` implementations
3. The annotation is checked by the JWT filter before the method executes
4. Without `@LoggedIn`, endpoints are publicly accessible (no JWT validation)
5. The annotation works as `@NameBinding` in CXF and is searched via reflection in Spring

---

## 14. SecurityContext & Principal Model

### JwtSecurityContext

**File:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JwtSecurityContext.java`

```java
public class JwtSecurityContext extends WaterAbstractSecurityContext
    implements RsSecurityContext {

    public JwtSecurityContext(Set<Principal> principals) {
        super(principals);
    }

    @Override
    public boolean isSecure() { return true; }

    @Override
    public String getAuthenticationScheme() { return "JWT"; }
}
```

### Principal Types

**UserPrincipal** - Represents the authenticated entity:
```java
public class UserPrincipal implements Principal {
    private String name;          // screen name (username)
    private boolean admin;        // admin flag
    private long loggedEntityId;  // entity ID
    private String issuer;        // issuer name
}
```

**RolePrincipal** - Represents an assigned role:
```java
public class RolePrincipal implements Principal {
    private String name;  // role name
}
```

### SecurityContext Flow

```
JWT Token claims → getPrincipals() → Set<Principal> → SecurityContext → Runtime.fillSecurityContext()
```

After `fillSecurityContext()`, the security context is available throughout the request
for permission checks via `PermissionUtil`.

---

## 15. JWT Configuration Properties

### JWTConstants

**File:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JWTConstants.java`

```java
// JWT Claim Names
public static final String JWT_CLAIM_ROLES = "roles";
public static final String JWT_CLAIM_LOGGED_ENTITY_ID = "loggedEntityId";
public static final String JWT_CLAIM_IS_ADMIN = "isAdmin";

// Property Keys
public static final String JWT_PROP_VALIDATION_ENABLED = "it.water.rest.security.jwt.validate";
public static final String JWT_PROP_VALIDATE_BY_JWS = "it.water.rest.security.jwt.validate.by.jws";
public static final String JWT_PROP_ENCRYPT_JWT_TOKEN = "it.water.rest.security.jwt.encrypt";
public static final String JWT_PROP_JWS_URL = "it.water.rest.security.jwt.jws.url";
public static final String JWT_PROP_JWT_DURATION_MILLIS = "it.water.rest.security.jwt.duration.millis";

// Cookie Name
public static final String JWT_COOKIE_NAME = "HIT-AUTH";
```

### JwtSecurityOptions Interface

**File:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/options/JwtSecurityOptions.java`

```java
public interface JwtSecurityOptions {
    boolean validateJwt();           // Enable/disable JWT validation
    boolean validateJwtWithJwsUrl(); // Validate via external JWS URL
    String jwtKeyId();               // Key ID in JWK
    boolean encryptJWTToken();       // Enable JWE encryption
    String jwsURL();                 // External JWS endpoint URL
    long jwtTokenDurationMillis();   // Token duration in ms
}
```

### JwtSecurityOptionsImpl Defaults

**File:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JwtSecurityOptionsImpl.java`

| Property | Default | Description |
|----------|---------|-------------|
| `it.water.rest.security.jwt.validate` | `true` | JWT validation enabled |
| `it.water.rest.security.jwt.validate.by.jws` | `false` | Local key validation |
| `it.water.rest.security.jwt.encrypt` | `false` | No JWE encryption |
| `it.water.rest.security.jwt.jws.url` | `""` | No external JWS URL |
| `it.water.rest.security.jwt.duration.millis` | `3600000` (1 hour) | Token lifetime |

### Complete Properties Reference for JWT

```properties
# water-application.properties

# Enable/disable JWT validation (default: true)
it.water.rest.security.jwt.validate=true

# Validate tokens using an external JWS URL instead of local key (default: false)
it.water.rest.security.jwt.validate.by.jws=false

# URL of the external JWS endpoint (required if validate.by.jws=true)
it.water.rest.security.jwt.jws.url=https://auth-server.example.com/.well-known/jwks.json

# Encrypt JWT tokens with JWE (default: false)
it.water.rest.security.jwt.encrypt=false

# JWT token duration in milliseconds (default: 3600000 = 1 hour)
it.water.rest.security.jwt.duration.millis=3600000

# Keystore configuration (used for JWT signing)
it.water.jwt.config.keystore.path=/path/to/keystore.jks
it.water.jwt.config.keystore.password=keystorePassword
it.water.jwt.config.key.password=keyPassword
it.water.jwt.config.keystore.alias=waterkey
```

---

## 16. User Registration & Activation Flow

### Registration Flow

```
1. Client: POST /users/register (no @LoggedIn, public endpoint)
   Body: { name, lastname, username, password, email, passwordConfirm }

2. Server: UserSystemApi.register(user)
   a. Check registration enabled (UserOptions.isRegistrationEnabled())
   b. Generate salt via EncryptionUtil.generate16BytesSalt()
   c. Hash password via EncryptionUtil.hashPassword(salt, password)
   d. Generate activateCode (UUID)
   e. Set active = false
   f. Persist user
   g. Send activation email (if EmailNotificationService available)

3. Response: 200 OK with WaterUser (active=false)
```

### Activation Flow

```
1. Client receives activation email with link containing email + activationCode

2. Client: PUT /users/activate?email=...&activationCode=...
   (No @LoggedIn, public endpoint)

3. Server: UserSystemApi.activateUser(email, activationCode)
   a. Find user by email
   b. Verify activationCode matches stored value
   c. Set active = true
   d. Clear activateCode
   e. Persist

4. Response: 200 OK
```

### Admin Activation/Deactivation

```
# Activate by admin (requires @LoggedIn + ACTIVATE permission)
PUT /users/{id}/activate

# Deactivate by admin (requires @LoggedIn + DEACTIVATE permission)
PUT /users/{id}/deactivate
```

### Unregistration Flow

```
1. Logged user: POST /users/unregisterRequest
   -> Generates deletionCode, sends confirmation email

2. User confirms: DELETE /users/unregister?email=...&deletionCode=...
   -> Marks user as deleted (logical deletion by default)
   -> Physical deletion if UserOptions.isPhysicalDeletionEnabled() = true
```

---

## 17. Password Management

### Password Storage

Passwords are **never stored in plain text**. The storage flow:

```
1. Generate 16-byte random salt: EncryptionUtil.generate16BytesSalt()
2. Encode salt as Base64: Base64.getEncoder().encodeToString(salt)
3. Hash password: EncryptionUtil.hashPassword(salt, password)  // PBKDF2
4. Store: user.salt = base64Salt, user.password = hashedPassword
```

### Password Validation Rules

The `@ValidPassword` annotation enforces:
- Minimum length requirements
- Must contain uppercase, lowercase, digits, and special characters
- No malicious code (`@NoMalitiusCode`)

Example valid password: `"Password1_."`

### Password Reset Flow

```
1. Client: PUT /users/resetPasswordRequest?email=user@example.com
   -> Finds user by email
   -> Generates passwordResetCode (UUID)
   -> Sends email with reset code

2. Client: (via form with reset code)
   PUT /users/resetPassword?email=...&resetCode=...&password=...&passwordConfirm=...
   -> Verifies resetCode matches stored value
   -> Validates new password
   -> Hashes with new salt
   -> Clears passwordResetCode (one-time use)
   -> Persists
```

### Password Change (Logged User)

```java
// UserApi
WaterUser changePassword(long userId, String oldPassword,
                          String newPassword, String passwordConfirm);
```

- Requires the user to be logged in
- Validates old password matches
- Only the user themselves can change their own password
- Validates new password meets requirements

---

## 18. User Module Configuration Properties

### UserConstants

**File:** `User/User-model/src/main/java/it/water/user/model/UserConstants.java`

```java
public static final String USER_OPT_REGISTRATION_ENABLED =
    "it.water.user.registration.enabled";
public static final String USER_OPT_ACTIVATION_URL =
    "it.water.user.activation.url";
public static final String USER_OPT_PASSWORD_RESET_URL =
    "it.water.user.password.reset.url";
public static final String USER_OPT_DEFAULT_ADMIN_PWD =
    "it.water.user.admin.default.password";
public static final String USER_OPT_PHYSICAL_DELETION_ENABLED =
    "it.water.user.physical.deletion.enabled";
public static final String USER_OPT_REGISTRATION_EMAIL_TEMPLATE_NAME =
    "it.water.user.registration.email.template.name";
```

### UserOptions Interface & Implementation

| Property | Method | Default | Description |
|----------|--------|---------|-------------|
| `it.water.user.registration.enabled` | `isRegistrationEnabled()` | `false` | Enable self-registration |
| `it.water.user.activation.url` | `getUserActivationUrl()` | `localhost:8080/water/users/activation` | Activation link URL |
| `it.water.user.password.reset.url` | `getPasswordResetUrl()` | `localhost:8080/water/users/password-reset` | Password reset URL |
| `it.water.user.admin.default.password` | `defaultAdminPwd()` | `"admin"` | Default admin password |
| `it.water.user.physical.deletion.enabled` | `isPhysicalDeletionEnabled()` | `false` | Physical vs logical delete |
| `it.water.user.registration.email.template.name` | `getUserRegistrationEmailTemplateName()` | `null` | Custom email template |

### Complete User Properties

```properties
# water-application.properties

# Enable user self-registration (default: false)
it.water.user.registration.enabled=true

# URL for user activation link in emails
it.water.user.activation.url=https://myapp.com/activate

# URL for password reset link in emails
it.water.user.password.reset.url=https://myapp.com/reset-password

# Default admin password (CHANGE IN PRODUCTION!)
it.water.user.admin.default.password=MySecureAdminPwd1_.

# Enable physical deletion instead of logical (default: false)
it.water.user.physical.deletion.enabled=false

# Custom email template name for registration emails
it.water.user.registration.email.template.name=custom-registration-template
```

---

## 19. Authentication Module Configuration

### AuthenticationOption

**File:** `Authentication/Authentication-api/src/main/java/it/water/authentication/api/options/AuthenticationOption.java`

```java
public interface AuthenticationOption extends Service {
    String getIssuerName();
}
```

### AuthenticationOptionImpl

**File:** `Authentication/Authentication-service/src/main/java/it/water/authentication/service/AuthenticationOptionImpl.java`

```java
@FrameworkComponent
public class AuthenticationOptionImpl implements AuthenticationOption {
    @Inject private ApplicationProperties applicationProperties;

    @Override
    public String getIssuerName() {
        String value = (String) applicationProperties
            .getProperty(AuthenticationConstants.AUTHENTICATION_ISSUER_NAME);
        if (value == null)
            throw new NoIssuerNameDefinedException();
        return value;
    }
}
```

### AuthenticationConstants

```java
public static final String AUTHENTICATION_ISSUER_NAME =
    "water.authentication.service.issuer";
```

### Required Configuration

```properties
# REQUIRED: Defines which AuthenticationProvider to use for login
# Must match an issuer from a registered AuthenticationProvider
water.authentication.service.issuer=it.water.core.api.model.User
```

**If this property is not set**, a `NoIssuerNameDefinedException` is thrown at login time.

### Multiple Issuers Example

If your application has both User and Device authentication:

```properties
# Use User authentication as default
water.authentication.service.issuer=it.water.core.api.model.User
```

For device-specific login, use the overloaded `login()` method:
```java
authenticationSystemApi.login(deviceId, apiKey, "com.myapp.model.Device");
```

---

## 20. JAAS Integration

The Authentication module includes a JAAS (Java Authentication and Authorization Service)
`LoginModule` for environments that use JAAS (e.g., OSGi/Karaf).

**File:** `Authentication/Authentication-service/src/main/java/it/water/authentication/service/jaas/AuthenticationModule.java`

```java
public abstract class AuthenticationModule implements LoginModule {

    @Override
    public boolean login() throws LoginException {
        // 1. Get username/password from callbacks
        Callback[] callbacks = new Callback[2];
        callbacks[0] = new NameCallback("Username: ");
        callbacks[1] = new PasswordCallback("Password: ", false);
        callbackHandler.handle(callbacks);

        // 2. Authenticate
        this.loggedUser = getAuthenticationApi().login(user, password);

        // 3. Check active status
        if (loggedUser == null || !loggedUser.isActive())
            return false;

        // 4. Load roles
        roles = getRoles(loggedUser);
        postAuthentication(loggedUser);
        return true;
    }

    @Override
    public boolean commit() throws LoginException {
        if (loginSucceeded) {
            // Add UserPrincipal
            principals.add(new UserPrincipal(
                loggedUser.getPassword(),
                loggedUser.isAdmin(),
                loggedUser.getLoggedEntityId(),
                loggedUser.getIssuer()
            ));

            // Add RolePrincipals
            for (Role role : roles) {
                principals.add(new RolePrincipal(role.getName()));
            }

            // Allow subclasses to add more principals
            setAdditionalPrincipals(principals);

            subject.getPrincipals().addAll(principals);
        }
        return loginSucceeded;
    }

    // Abstract methods for framework-specific implementations
    protected abstract void postAuthentication(Authenticable authenticated);
    protected abstract void setAdditionalPrincipals(Set<Principal> principals);
    protected abstract List<Role> getRoles(Authenticable authenticable);
    protected abstract AuthenticationApi getAuthenticationApi();
}
```

### JAAS vs JWT

| Aspect | JAAS | JWT |
|--------|------|-----|
| Use case | Server-side sessions (OSGi/Karaf) | Stateless REST APIs |
| State | Server-side Subject | Client-side token |
| Login flow | LoginModule.login() + commit() | REST endpoint + token generation |
| Session | Subject with Principals | Token with claims |

---

## 21. Testing Authentication

### Test Setup Pattern

```java
@ExtendWith(WaterTestExtension.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class MyAuthTest implements Service {
    @Inject private ComponentRegistry componentRegistry;
    @Inject private EncryptionUtil encryptionUtil;
    @Inject private UserApi userApi;
    @Inject private UserSystemApi userSystemApi;
    @Inject private AuthenticationProvider authenticationProvider;
    @Inject private RoleManager roleManager;
    @Inject private UserManager userManager;
    @Inject private Runtime runtime;

    private User adminUser;

    @BeforeAll
    void beforeAll() {
        // Get admin user
        adminUser = userManager.findUser("admin");
        // Start with admin permissions
        TestRuntimeUtils.impersonateAdmin(componentRegistry);
    }
}
```

### Creating Test Users

```java
private WaterUser createUser(int seed) {
    String salt = new String(encryptionUtil.generate16BytesSalt());
    WaterUser user = new WaterUser(
        "name" + seed,
        "lastname" + seed,
        "username" + seed,
        "Password_" + seed,    // Must match @ValidPassword rules
        salt,
        false,                 // not admin
        "mail" + UUID.randomUUID() + "@mail.com"
    );
    user.setPasswordConfirm(user.getPassword());
    return user;
}
```

### Testing Login

```java
@Test
void assertLoginWorks() {
    WaterUser user = createUser(604);
    String username = user.getUsername();
    // Save user as admin
    runAs(adminUser, () -> userApi.save(user));

    // Successful login
    assertDoesNotThrow(() ->
        authenticationProvider.login(username, "Password_604"));

    // Failed login attempts
    assertThrows(UnauthorizedException.class, () ->
        authenticationProvider.login(username, "WrongPassword"));
    assertThrows(UnauthorizedException.class, () ->
        authenticationProvider.login("wrongUsername", "WrongPassword"));
    assertThrows(UnauthorizedException.class, () ->
        authenticationProvider.login(username, ""));
    assertThrows(UnauthorizedException.class, () ->
        authenticationProvider.login(username, null));
    assertThrows(UnauthorizedException.class, () ->
        authenticationProvider.login(null, null));
}
```

### Testing Impersonation

```java
// Impersonate a specific user
TestRuntimeInitializer.getInstance().impersonate(someUser, runtime);

// Run code as a specific user
runAs(someUser, () -> userApi.save(entity));

// Get result as a specific user
WaterUser result = getAs(adminUser, () -> userApi.findByUsername("test"));

// Clear security context (anonymous)
runtime.fillSecurityContext(null);

// Impersonate admin
TestRuntimeUtils.impersonateAdmin(componentRegistry);
```

### Testing Registration Flow

```java
@Test
void userRegistrationSuccess() {
    // Clear context (anonymous)
    runtime.fillSecurityContext(null);
    WaterUser registeredUser = createUser(102);

    // Register
    userApi.register(registeredUser);
    assertTrue(registeredUser.getId() > 0);
    assertFalse(registeredUser.isActive());
    assertNotNull(registeredUser.getActivateCode());

    // Unregister flow
    runAs(registeredUser, () -> userApi.unregisterAccountRequest());
    String deletionCode = getAs(adminUser, () ->
        userApi.findByUsername(registeredUser.getUsername()).getDeletionCode());
    runAs(registeredUser, () ->
        userApi.unregister(registeredUser.getEmail(), deletionCode));
}
```

### Karate REST Test

**File:** `Authentication/Authentication-service/src/test/resources/karate/login.feature`

```gherkin
Feature: Login test

  Scenario: User gives right credentials and logs in succesfully
    Given header Content-Type = 'application/x-www-form-urlencoded'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/authentication/login'
    And request 'username=admin&password=admin'
    When method POST
    Then status 200
    And match response == { "token": #string }

  Scenario: User gives wrong credentials and gets 401
    Given header Content-Type = 'application/x-www-form-urlencoded'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/authentication/login'
    And request 'username=admin&password=wrong'
    When method POST
    Then status 401
```

---

## 22. Best Practices & Anti-Patterns

### Best Practices

1. **Always configure the issuer property**
   ```properties
   water.authentication.service.issuer=it.water.core.api.model.User
   ```
   Without this, login will throw `NoIssuerNameDefinedException`.

2. **Change default admin password in production**
   ```properties
   it.water.user.admin.default.password=YourStrongP@ssw0rd!
   ```

3. **Use `@LoggedIn` on all non-public endpoints**
   ```java
   @LoggedIn
   @GET @Path("/{id}")
   MyResource find(@PathParam("id") long id);
   ```

4. **Specify issuers when accepting non-User tokens**
   ```java
   @LoggedIn(issuers = {"it.water.core.api.model.User", "com.myapp.model.Device"})
   ```

5. **Hash passwords using EncryptionUtil**, never store plain text
   ```java
   byte[] salt = encryptionUtil.generate16BytesSalt();
   String hashed = new String(encryptionUtil.hashPassword(salt, password));
   ```

6. **Load roles at login time** to populate JWT claims correctly
   ```java
   if (roleManager != null) {
       user.setRoles(roleManager.getUserRoles(user.getId()));
   }
   ```

7. **Use generic error messages** to prevent information leakage
   ```java
   throw new UnauthorizedException("username or password incorrect!");
   // NOT: "user not found" or "wrong password"
   ```

8. **Enable registration only when needed**
   ```properties
   it.water.user.registration.enabled=true  # Only if self-registration is required
   ```

9. **Configure JWT duration appropriately for your use case**
   ```properties
   # Short-lived for high security (15 minutes)
   it.water.rest.security.jwt.duration.millis=900000
   # Standard (1 hour, default)
   it.water.rest.security.jwt.duration.millis=3600000
   ```

10. **Use JWS URL validation in distributed systems** where multiple services
    need to validate tokens issued by a central auth server
    ```properties
    it.water.rest.security.jwt.validate.by.jws=true
    it.water.rest.security.jwt.jws.url=https://auth-server/jwk
    ```

### Anti-Patterns

1. **Don't store passwords in plain text** - Always use `EncryptionUtil.hashPassword()`

2. **Don't bypass the AuthenticationProvider pattern** - Don't query users and validate
   passwords directly in controllers. Use the provider pattern.

3. **Don't hardcode issuer names** - Use `AuthenticationOption` and properties

4. **Don't skip `@LoggedIn` annotation** on protected endpoints - The JWT filter only
   activates when the annotation is present

5. **Don't create users without passwordConfirm** - The validation will fail
   ```java
   user.setPasswordConfirm(user.getPassword()); // Required!
   ```

6. **Don't assume roles are loaded** - They are only populated during `login()`.
   After finding a user by ID, roles won't be set unless explicitly loaded.

7. **Don't disable JWT validation in production**
   ```properties
   it.water.rest.security.jwt.validate=true  # NEVER false in production
   ```

---

## 23. Decision Trees

### "How to Add Authentication to My Module"

```
START: Do you need user-based login?
│
├── YES: Is the User module already included?
│   ├── YES: Just configure the issuer property
│   │   water.authentication.service.issuer=it.water.core.api.model.User
│   │   └── Add @LoggedIn to your REST endpoints
│   │
│   └── NO: Add User module dependency
│       └── Then configure as above
│
└── NO: You need a custom Authenticable entity
    │
    ├── 1. Make your entity implement Authenticable
    │   - getLoggedEntityId(), getIssuer(), getScreenName(),
    │     getPassword(), getSalt(), isAdmin(), isActive(), getRoles()
    │
    ├── 2. Create AuthenticationProvider implementation
    │   @FrameworkComponent(services = AuthenticationProvider.class)
    │   class MyProvider implements AuthenticationProvider { ... }
    │
    ├── 3. Configure issuer
    │   water.authentication.service.issuer=com.myapp.model.MyEntity
    │
    └── 4. Add @LoggedIn(issuers={"com.myapp.model.MyEntity"})
        to your REST endpoints
```

### "How to Protect a REST Endpoint"

```
START: Is the endpoint public (no authentication needed)?
│
├── YES: Don't add @LoggedIn
│   Example: POST /authentication/login
│   Example: POST /users/register
│
└── NO: The endpoint requires authentication
    │
    ├── Does it accept only User tokens?
    │   └── YES: @LoggedIn (uses default issuer)
    │
    ├── Does it accept tokens from specific issuers?
    │   └── YES: @LoggedIn(issuers = {"issuer1", "issuer2"})
    │
    └── Does it also need permission checks?
        └── YES: @LoggedIn + @AllowPermissions or @AllowRoles
            (See authorization-knowledge skill)
```

### "How to Debug Authentication Failures"

```
ERROR: 401 Unauthorized on login
│
├── Check: Is water.authentication.service.issuer configured?
│   └── NO → Add it to water-application.properties
│
├── Check: Is an AuthenticationProvider registered for this issuer?
│   └── NO → Register one with @FrameworkComponent(services = AuthenticationProvider.class)
│
├── Check: Does the user exist in the database?
│   └── NO → Register the user first
│
├── Check: Is the user active?
│   └── NO → Activate the user (activate endpoint or admin)
│
├── Check: Is the password correctly hashed?
│   └── Compare: hashPassword(decode(user.salt), inputPassword) == user.password
│
└── Check: JWT validation properties
    └── it.water.rest.security.jwt.validate must be true

ERROR: 401 on protected endpoint (with valid login)
│
├── Check: Is @LoggedIn annotation present on the method/class?
│   └── NO → Add @LoggedIn
│
├── Check: Is the token in the Authorization header?
│   └── Format: "Authorization: Bearer <token>"
│
├── Check: Is the token expired?
│   └── Check duration: it.water.rest.security.jwt.duration.millis
│
├── Check: Does the token issuer match @LoggedIn(issuers)?
│   └── Default issuers = {"it.water.core.api.model.User"}
│
└── Check: Is the keystore configured correctly?
    └── JWT signing requires a valid RSA key in the server keystore
```

---

## 24. Quick Reference Tables

### Authentication Interfaces

| Interface | Module | Purpose |
|-----------|--------|---------|
| `Authenticable` | Core-api | Contract for entities that can authenticate |
| `User` | Core-api | Extends Authenticable with user fields |
| `AuthenticationProvider` | Core-api | Pluggable login implementation |
| `EncryptionUtil` | Core-api | Cryptographic operations, password hashing |
| `AuthenticationApi` | Authentication-api | Public authentication API |
| `AuthenticationSystemApi` | Authentication-api | System-level auth (bypasses permissions) |
| `AuthenticationOption` | Authentication-api | Issuer configuration |
| `AuthenticationRestApi` | Authentication-api | REST login endpoint |
| `JwtTokenService` | Rest-api | JWT generation, validation, principal extraction |
| `JwtSecurityOptions` | Rest-api | JWT configuration options |
| `RsSecurityContext` | Rest-api | Bridge between Water and JAX-RS SecurityContext |

### REST Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/authentication/login` | No | Login and get JWT token |
| POST | `/users` | @LoggedIn | Create user (admin) |
| PUT | `/users` | @LoggedIn | Update user |
| GET | `/users/{id}` | @LoggedIn | Find user by ID |
| GET | `/users` | @LoggedIn | List all users |
| DELETE | `/users/{id}` | @LoggedIn | Delete user |
| POST | `/users/register` | No | Self-registration |
| PUT | `/users/activate` | No | Activate via email+code |
| PUT | `/users/{id}/activate` | @LoggedIn | Admin activate |
| PUT | `/users/{id}/deactivate` | @LoggedIn | Admin deactivate |
| POST | `/users/unregisterRequest` | @LoggedIn | Request account deletion |
| DELETE | `/users/unregister` | @LoggedIn | Confirm account deletion |
| PUT | `/users/resetPasswordRequest` | No | Request password reset |

### Configuration Properties Summary

| Property | Default | Module | Required |
|----------|---------|--------|----------|
| `water.authentication.service.issuer` | - | Authentication | **YES** |
| `it.water.rest.security.jwt.validate` | `true` | Rest-security | No |
| `it.water.rest.security.jwt.validate.by.jws` | `false` | Rest-security | No |
| `it.water.rest.security.jwt.jws.url` | `""` | Rest-security | If JWS=true |
| `it.water.rest.security.jwt.encrypt` | `false` | Rest-security | No |
| `it.water.rest.security.jwt.duration.millis` | `3600000` | Rest-security | No |
| `it.water.user.registration.enabled` | `false` | User | No |
| `it.water.user.activation.url` | `localhost:8080/...` | User | No |
| `it.water.user.password.reset.url` | `localhost:8080/...` | User | No |
| `it.water.user.admin.default.password` | `"admin"` | User | No |
| `it.water.user.physical.deletion.enabled` | `false` | User | No |
| `it.water.user.registration.email.template.name` | `null` | User | No |

### Default Roles

| Role | Permissions |
|------|-------------|
| `userManager` | SAVE, REMOVE, FIND, FIND_ALL, UPDATE, IMPERSONATE, ACTIVATE, DEACTIVATE |
| `userViewer` | FIND, FIND_ALL |
| `userEditor` | SAVE, FIND, FIND_ALL, UPDATE |

### Key Classes Quick Lookup

| Class | Type | Location |
|-------|------|----------|
| `WaterUser` | JPA Entity | `User/User-model` |
| `UserAuthenticationProvider` | @FrameworkComponent | `User/User-service` |
| `AuthenticationSystemServiceImpl` | @FrameworkComponent | `Authentication/Authentication-service` |
| `AuthenticationRestControllerImpl` | @FrameworkRestController | `Authentication/Authentication-service` |
| `NimbusJwtTokenService` | @FrameworkComponent | `Rest/Rest-security` |
| `GenericJWTAuthFilter` | Abstract base | `Rest/Rest-security` |
| `SpringJwtAuthenticationFilter` | HandlerInterceptor | `Rest/Rest-spring-api` |
| `CxfJwtAuthenticationFilter` | ContainerRequestFilter | `Rest/Rest-api-manager-apache-cxf` |
| `JwtSecurityContext` | SecurityContext | `Rest/Rest-security` |
| `UserPrincipal` | Principal | `Core/Core-security` |
| `RolePrincipal` | Principal | `Core/Core-security` |