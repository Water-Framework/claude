---
name: rest-knowledge
description: Comprehensive knowledge base for Water Framework REST layer, including RestApi interface hierarchy, REST controller patterns, JAX-RS and Spring MVC dual support, JWT authentication filters, JSON views, exception mapping, Swagger integration, and REST testing. Use when designing, implementing, reviewing, or debugging any REST endpoint, controller, filter, or serialization feature in Water modules.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework REST architect with deep knowledge of the REST abstraction layer, dual JAX-RS/Spring MVC support, JWT authentication, REST controller proxy pattern, JSON serialization views, exception mapping, and REST testing. You use this knowledge to guide correct REST API implementations, security filter design, and endpoint exposure in Water modules.

---

# WATER FRAMEWORK REST KNOWLEDGE BASE

## Table of Contents

1. [REST Architecture Overview](#1-rest-architecture-overview)
2. [Module Dependency Map](#2-module-dependency-map)
3. [RestApi Interface Hierarchy](#3-restapi-interface-hierarchy)
4. [REST Annotations](#4-rest-annotations)
5. [REST Controller Pattern](#5-rest-controller-pattern)
6. [BaseEntityRestApi - CRUD REST](#6-baseentityrestapi---crud-rest)
7. [RestControllerProxy & Interception](#7-restcontrollerproxy--interception)
8. [RestApiManager & RestApiRegistry](#8-restapimanager--restapiregistry)
9. [JWT Authentication & Security Filters](#9-jwt-authentication--security-filters)
10. [JSON Serialization & Views](#10-json-serialization--views)
11. [Exception Mapping](#11-exception-mapping)
12. [REST Options & Configuration](#12-rest-options--configuration)
13. [Spring REST Implementation](#13-spring-rest-implementation)
14. [CXF (JAX-RS) REST Implementation](#14-cxf-jax-rs-rest-implementation)
15. [Swagger / OpenAPI Integration](#15-swagger--openapi-integration)
16. [REST Testing Patterns](#16-rest-testing-patterns)
17. [REST Initialization Flow](#17-rest-initialization-flow)
18. [Best Practices & Anti-Patterns](#18-best-practices--anti-patterns)
19. [Decision Trees](#19-decision-trees)
20. [Quick Reference Tables](#20-quick-reference-tables)

---

## 1. REST Architecture Overview

Water Framework implements a **framework-agnostic REST layer** where a single REST API definition can be deployed on both JAX-RS (Apache CXF) and Spring MVC, using dynamic proxies and interceptors for cross-cutting concerns.

### Layer Architecture

```
REST API Interface (@FrameworkRestApi)
  |  JAX-RS annotations (@Path, @GET, @POST, etc.)
  |
  +--- Spring REST API Interface (optional, adds @RequestMapping)
  |
  v
REST Controller Implementation (@FrameworkRestController)
  |  extends BaseEntityRestApi<T> for entities
  |  delegates to BaseEntityApi / custom service
  v
RestControllerProxy (InvocationHandler)
  |  before/after method interceptors
  |  permission checks, logging, validation
  v
Business Service (Api → SystemApi → Repository)
```

### Key Principles

- **Core-api** defines `RestApi`, `@FrameworkRestApi`, `@FrameworkRestController`, `WaterJsonView` (technology-agnostic)
- **Rest-api** defines `RestOptions`, `@LoggedIn`, `RootApi`, `StatusApi`, `RsSecurityContext`
- **Rest-service** provides `GenericExceptionMapperProvider`, Jackson module, base proxy patterns
- **Rest-jaxrs-api** provides JAX-RS specific implementations
- **Rest-spring-api** provides Spring MVC specific implementations and filters
- **Rest-api-manager-apache-cxf** provides CXF REST API Manager and per-request proxy
- **Rest-security** provides JWT token service, security context, and base auth filters
- **Rest-persistence** provides `BaseEntityRestApi` for entity CRUD

---

## 2. Module Dependency Map

```
Core-api (interfaces: RestApi, FrameworkRestApi, FrameworkRestController, WaterJsonView)
  |
  +--- Rest-api (RestOptions, @LoggedIn, RootApi, StatusApi, RsSecurityContext, JwtSecurityOptions)
  |
  +--- Rest-service (GenericExceptionMapperProvider, WaterJacksonModule, RestControllerProxy)
  |       depends on: Core-api, Rest-api, Repository-entity
  |
  +--- Rest-jaxrs-api (JAX-RS RootApi/StatusApi implementations)
  |       depends on: Rest-api
  |
  +--- Rest-spring-api (Spring RootApi/StatusApi, SpringJwtAuthenticationFilter)
  |       depends on: Rest-api, Rest-security
  |
  +--- Rest-api-manager-apache-cxf (CxfRestApiManager, PerRequestProxyProvider, CxfJwtAuthenticationFilter)
  |       depends on: Rest-service, Rest-security
  |
  +--- Rest-security (GenericJWTAuthFilter, JwtTokenService, JwtSecurityContext)
  |       depends on: Rest-api, Core-api
  |
  +--- Rest-persistence (BaseEntityRestApi)
          depends on: Core-api, Rest-api
```

---

## 3. RestApi Interface Hierarchy

### RestApi (Base Marker)

**File:** `Core/Core-api/src/main/java/it/water/core/api/service/rest/RestApi.java`

```java
public interface RestApi extends Service {
    // Marker interface — identifies classes as REST resources
}
```

### RootApi & StatusApi

**File:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/RootApi.java`

```java
@FrameworkRestApi
@Api
@Path("/")
public interface RootApi extends RestApi {
    @GET
    @ApiOperation(value = "/", response = String.class)
    Response getRootInfo();
}
```

```java
@FrameworkRestApi
@Api
@Path("/status")
public interface StatusApi extends RestApi {
    @GET
    @ApiOperation(value = "/status", response = String.class)
    Response checkModulesStatus();
}
```

### Entity REST API Pattern

A typical entity REST API follows a three-tier structure:

```
1. Generic Interface (JAX-RS annotations)
   └─ @FrameworkRestApi + @Path + @LoggedIn + JAX-RS + Swagger

2. Spring Interface (extends generic, adds Spring annotations)
   └─ @FrameworkRestApi + @RequestMapping + Spring MVC

3. Controller Implementation
   └─ @FrameworkRestController + extends BaseEntityRestApi<T>
```

---

## 4. REST Annotations

### @FrameworkRestApi

**File:** `Core/Core-api/src/main/java/it/water/core/api/service/rest/FrameworkRestApi.java`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated  // Atteo ClassIndex for compile-time discovery
public @interface FrameworkRestApi {
}
```

- Marks interfaces as REST API definitions
- Discovered at startup via ClassIndex (zero-reflection)
- Applied to both generic and Spring-specific interfaces

### @FrameworkRestController

**File:** `Core/Core-api/src/main/java/it/water/core/api/service/rest/FrameworkRestController.java`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface FrameworkRestController {
    Class<? extends RestApi> referredRestApi();
}
```

- Links implementation classes to their RestApi interface
- `referredRestApi` parameter identifies which RestApi this controller implements
- Used by `RestApiRegistry` to map interfaces to implementations

### @LoggedIn

**File:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/security/LoggedIn.java`

```java
@NameBinding
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface LoggedIn {
    String[] issuers() default {"it.water.core.api.model.User"};
}
```

- Marks REST methods/classes requiring JWT authentication
- `issuers` parameter specifies allowed JWT issuer types
- Uses `@NameBinding` for JAX-RS filter binding
- Can be applied at TYPE level (all methods) or METHOD level (specific methods)

---

## 5. REST Controller Pattern

### Defining a REST API Interface (Generic / JAX-RS)

```java
@FrameworkRestApi
@Api(tags = "MyEntity")
@Path("/myentities")
@LoggedIn
public interface MyEntityRestApi extends RestApi {

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @LoggedIn
    @AllowPermissions(actions = {CrudActions.SAVE}, checkById = true, idParamIndex = 0, systemApiRef = "it.water.mymodule.api.MyEntitySystemApi")
    MyEntity saveMyEntity(
        @JsonView(WaterJsonView.Public.class) MyEntity entity);

    @PUT
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @LoggedIn
    @AllowPermissions(actions = {CrudActions.UPDATE}, checkById = true, idParamIndex = 0, systemApiRef = "it.water.mymodule.api.MyEntitySystemApi")
    MyEntity updateMyEntity(
        @JsonView(WaterJsonView.Public.class) MyEntity entity);

    @GET
    @Path("/{id}")
    @Produces(MediaType.APPLICATION_JSON)
    @LoggedIn
    @AllowPermissions(actions = {CrudActions.FIND}, checkById = true, idParamIndex = 0, systemApiRef = "it.water.mymodule.api.MyEntitySystemApi")
    @JsonView(WaterJsonView.Public.class)
    MyEntity findMyEntity(@PathParam("id") long id);

    @DELETE
    @Path("/{id}")
    @Produces(MediaType.APPLICATION_JSON)
    @LoggedIn
    @AllowPermissions(actions = {CrudActions.REMOVE}, checkById = true, idParamIndex = 0, systemApiRef = "it.water.mymodule.api.MyEntitySystemApi")
    void removeMyEntity(@PathParam("id") long id);

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    @LoggedIn
    @AllowGenericPermissions(actions = {CrudActions.FIND_ALL}, resourceName = "it.water.mymodule.model.MyEntity")
    @JsonView(WaterJsonView.Public.class)
    PaginableResult<MyEntity> findAllMyEntity(
        @QueryParam("delta") int delta,
        @QueryParam("page") int page,
        @QueryParam("filter") String filter,
        @QueryParam("order") String order);
}
```

### Defining a Spring REST API Interface

```java
@FrameworkRestApi
@RequestMapping("/myentities")
public interface MyEntitySpringRestApi extends MyEntityRestApi {

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    MyEntity saveMyEntity(@RequestBody MyEntity entity);

    @PutMapping(consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    MyEntity updateMyEntity(@RequestBody MyEntity entity);

    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    @JsonView(WaterJsonView.Public.class)
    MyEntity findMyEntity(@PathVariable("id") long id);

    @DeleteMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    void removeMyEntity(@PathVariable("id") long id);

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    @JsonView(WaterJsonView.Public.class)
    PaginableResult<MyEntity> findAllMyEntity(
        @RequestParam(value = "delta", defaultValue = "10") int delta,
        @RequestParam(value = "page", defaultValue = "1") int page,
        @RequestParam(value = "filter", required = false) String filter,
        @RequestParam(value = "order", required = false) String order);
}
```

### Implementing the Controller

```java
@FrameworkRestController(referredRestApi = MyEntityRestApi.class)
public class MyEntityRestControllerImpl extends BaseEntityRestApi<MyEntity>
    implements MyEntityRestApi, MyEntitySpringRestApi {

    @Inject @Setter
    private MyEntityApi myEntityApi;

    @Override
    protected BaseEntityApi<MyEntity> getEntityService() {
        return myEntityApi;
    }

    // CRUD methods are inherited from BaseEntityRestApi
    // Override only if custom behavior is needed
}
```

---

## 6. BaseEntityRestApi - CRUD REST

**File:** `Rest/Rest-persistence/src/main/java/it/water/service/rest/persistence/BaseEntityRestApi.java`

```java
public abstract class BaseEntityRestApi<T extends BaseEntity> {
    protected abstract BaseEntityApi<T> getEntityService();

    // Default CRUD implementations
    public T save(T entity) {
        return getEntityService().save(entity);
    }

    public T update(T entity) {
        return getEntityService().update(entity);
    }

    public T find(long id) {
        return getEntityService().find(id);
    }

    public void remove(long id) {
        getEntityService().remove(id);
    }

    public PaginableResult<T> findAll(int delta, int page, String filter, String order) {
        // Delegates to service with query parsing
        return getEntityService().findAll(filter, delta, page, order);
    }
}
```

### Key Points

- Provides default implementations for all CRUD operations
- Delegates to `BaseEntityApi<T>` service
- Subclasses only override for custom behavior
- Handles `filter` and `order` string parsing automatically

---

## 7. RestControllerProxy & Interception

**File:** `Rest/Rest-service/src/main/java/it/water/service/rest/RestControllerProxy.java`

```java
public class RestControllerProxy extends WaterAbstractInterceptor<RestApi>
    implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. Execute before-method interceptors
        executeInterceptorBeforeMethod(restApi, method, args);

        // 2. Invoke actual implementation method
        Method implMethod = findMatchingMethod(implementationClass, method);
        Object result = implMethod.invoke(implementation, args);

        // 3. Execute after-method interceptors
        executeInterceptorAfterMethod(restApi, method, args, result);

        return result;
    }
}
```

### Interceptor Chain

Extends `WaterAbstractInterceptor<RestApi>` which provides:

| Interceptor Phase | Interface | Purpose |
|-------------------|-----------|---------|
| Before Method | `BeforeMethodInterceptor<RestApi>` | Permission checks, validation |
| Before Method Field | `BeforeMethodFieldInterceptor<Inject>` | Field injection |
| After Method | `AfterMethodInterceptor<RestApi>` | Post-processing, auditing |
| After Method Field | `AfterMethodFieldInterceptor` | Return value processing |

### Per-Request Instantiation

- Controllers are instantiated per-request for thread safety
- CXF uses `PerRequestProxyProvider` to create proxies
- Spring uses Spring beans with appropriate scope

---

## 8. RestApiManager & RestApiRegistry

### RestApiManager Interface

```java
public interface RestApiManager extends Service {
    void startRestApiServer();
    void stopRestApiServer();
}
```

### RestApiRegistry Interface

```java
public interface RestApiRegistry extends Service {
    void addRestApiService(Class<? extends RestApi> restApiClass);
    void removeRestApiService(Class<? extends RestApi> restApiClass);
    Class<?> getRestApiImplementation(Class<? extends RestApi> restApiClass);
    List<Class<? extends RestApi>> getRegisteredRestApis();
    void sendRestartApiManagerRestartRequest();
}
```

### Registration Flow

1. `@FrameworkRestApi` classes discovered via Atteo ClassIndex
2. `@FrameworkRestController` implementations mapped to their `referredRestApi`
3. `RestApiRegistry` stores the mapping: RestApi interface → Implementation class
4. `RestApiManager` uses the registry to create JAX-RS/Spring endpoints

---

## 9. JWT Authentication & Security Filters

### GenericJWTAuthFilter

**File:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/GenericJWTAuthFilter.java`

Base class for all JWT authentication filters with common logic:

```java
public class GenericJWTAuthFilter {
    // Extract token from Authorization header or JWT cookie
    public String getTokenFromRequest(String authorization, String cookieValue) {
        if (authorization != null && authorization.startsWith("Bearer "))
            return authorization.substring(7);
        if (cookieValue != null)
            return cookieValue.startsWith("Bearer ") ? cookieValue.substring(7) : cookieValue;
        throw new UnauthorizedException();
    }

    // Validate token against allowed issuers
    public JwtSecurityContext validateToken(
        JwtTokenService jwtTokenService,
        LoggedIn loggedInAnnotation,
        String authorization,
        String cookieValue) {
        String token = getTokenFromRequest(authorization, cookieValue);
        List<String> issuers = Arrays.asList(loggedInAnnotation.issuers());
        if (jwtTokenService.validateToken(issuers, token)) {
            Set<Principal> principals = jwtTokenService.getPrincipals(token);
            return new JwtSecurityContext(principals);
        }
        throw new UnauthorizedException();
    }
}
```

### CxfJwtAuthenticationFilter (JAX-RS)

**File:** `Rest/Rest-api-manager-apache-cxf/src/main/java/.../CxfJwtAuthenticationFilter.java`

```java
@Priority(Priorities.AUTHENTICATION)
@LoggedIn
public class CxfJwtAuthenticationFilter extends GenericJWTAuthFilter
    implements ContainerRequestFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        // 1. Find @LoggedIn annotation on resource method/class
        LoggedIn loggedIn = findLoggedInAnnotation(requestContext);
        if (loggedIn == null) return;

        // 2. Extract token from header or cookie
        String authorization = requestContext.getHeaderString(HttpHeaders.AUTHORIZATION);
        String cookie = /* extract JWT cookie */;

        // 3. Validate and create security context
        JwtSecurityContext ctx = validateToken(jwtTokenService, loggedIn, authorization, cookie);
        requestContext.setSecurityContext(ctx);

        // 4. Fill framework Runtime security context
        runtime.fillSecurityContext(ctx);
    }
}
```

### SpringJwtAuthenticationFilter

**File:** `Rest/Rest-spring-api/src/main/java/.../SpringJwtAuthenticationFilter.java`

```java
@Component
public class SpringJwtAuthenticationFilter extends GenericJWTAuthFilter
    implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 1. Check if handler method has @LoggedIn annotation
        LoggedIn loggedIn = findLoggedInAnnotation(handler);
        if (loggedIn == null) return true;

        // 2. Extract token
        String authorization = request.getHeader(HttpHeaders.AUTHORIZATION);
        String cookie = extractJwtCookie(request);

        // 3. Validate and set security context
        JwtSecurityContext ctx = validateToken(jwtTokenService, loggedIn, authorization, cookie);
        runtime.fillSecurityContext(ctx);
        return true;
    }
}
```

### JwtSecurityContext

**File:** `Rest/Rest-security/src/main/java/.../JwtSecurityContext.java`

```java
public class JwtSecurityContext implements RsSecurityContext {
    private Set<Principal> principals;
    private String permissionImplementation;

    // SecurityContext interface
    public Principal getUserPrincipal();     // Returns UserPrincipal
    public boolean isUserInRole(String role); // Checks RolePrincipal set
    public boolean isSecure();               // Always true for JWT
    public String getAuthenticationScheme();  // Returns "JWT"

    // Water extensions
    public boolean isAdmin();                // Checks admin flag
    public String getPermissionImplementation(); // Custom permission impl
    public String getIssuerClassName();      // JWT issuer class
}
```

### JwtTokenService Interface

```java
public interface JwtTokenService extends Service {
    String generateJwtToken(Authenticable user);
    boolean validateToken(List<String> issuers, String token);
    Set<Principal> getPrincipals(String token);
    String getJWK();  // JSON Web Key for external validation
}
```

---

## 10. JSON Serialization & Views

### WaterJsonView

**File:** `Core/Core-api/src/main/java/it/water/core/api/service/rest/WaterJsonView.java`

```java
public interface WaterJsonView {
    interface Compact { }                     // Minimal fields
    interface Extended extends Compact { }    // Standard + detail fields
    interface Public extends Extended { }     // All public fields (default)
    interface Internal { }                    // System-only fields
    interface Secured { }                     // Encrypted/hashed fields
    interface Privacy { }                     // GDPR/privacy fields
}
```

### View Hierarchy

```
Compact (minimal)
  └── Extended (includes Compact)
       └── Public (includes Extended + Compact) — DEFAULT VIEW

Internal  (independent — system internals)
Secured   (independent — encrypted fields)
Privacy   (independent — GDPR-sensitive)
```

### Usage on Entity Fields

```java
@Entity
public class MyEntity extends AbstractJpaEntity {
    @JsonView(WaterJsonView.Compact.class)
    private String name;            // Shown in Compact, Extended, Public

    @JsonView(WaterJsonView.Extended.class)
    private String description;     // Shown in Extended, Public only

    @JsonView(WaterJsonView.Public.class)
    private String fullDetails;     // Shown in Public only

    @JsonView(WaterJsonView.Internal.class)
    private String systemField;     // Never shown to external clients

    @JsonView(WaterJsonView.Secured.class)
    private String encryptedData;   // Only in secured contexts
}
```

### Usage on REST Endpoints

```java
@GET
@Path("/{id}")
@JsonView(WaterJsonView.Public.class)  // Controls which fields are serialized
MyEntity findMyEntity(@PathParam("id") long id);

@GET
@Path("/{id}/compact")
@JsonView(WaterJsonView.Compact.class) // Minimal response payload
MyEntity findMyEntityCompact(@PathParam("id") long id);
```

### WaterJacksonModule

Registers custom serializers/deserializers:
- Custom `WaterJsonDeserializer` handles `ExpandableEntity` extensions
- Looks up `EntityExtensionService` for custom field mapping
- Respects `@JsonView` annotations during serialization

---

## 11. Exception Mapping

**File:** `Rest/Rest-service/src/main/java/it/water/service/rest/GenericExceptionMapperProvider.java`

```java
@Provider
public class GenericExceptionMapperProvider implements ExceptionMapper<Throwable> {
    @Override
    public Response toResponse(Throwable exception) {
        BaseError error = handleException(exception);
        return Response.status(error.getStatusCode()).entity(error).build();
    }
}
```

### Exception → HTTP Status Mapping

| Exception | HTTP Status | Code |
|-----------|------------|------|
| `DuplicateEntityException` | 409 Conflict | Duplicate entity |
| `ValidationException` | 422 Unprocessable Entity | Validation errors |
| `EntityNotFound` | 404 Not Found | Entity not found |
| `NoResultException` | 404 Not Found | No result |
| `UnauthorizedException` | 401 Unauthorized | Auth required |
| `RuntimeException` (generic) | 500 Internal Server Error | Server error |
| `IllegalArgumentException` | 500 Internal Server Error | Bad argument |

### BaseError Response Format

```json
{
  "statusCode": 422,
  "type": "ValidationException",
  "validationErrors": [
    { "field": "name", "message": "must not be blank" },
    { "field": "email", "message": "invalid format" }
  ]
}
```

---

## 12. REST Options & Configuration

### RestOptions Interface

**File:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/options/RestOptions.java`

```java
public interface RestOptions extends Service {
    String frontendUrl();           // Frontend app URL
    String servicesUrl();           // Backend services URL
    String restRootContext();       // REST context path (default: "/water")
    String uploadFolderPath();      // Upload directory path
    long uploadMaxFileSize();       // Max upload file size in bytes
    JwtSecurityOptions securityOptions(); // JWT configuration
}
```

### JwtSecurityOptions

```java
public interface JwtSecurityOptions {
    boolean validateJwt();                // Enable JWT validation
    long jwtTokenDurationMillis();        // Token expiration time
    String jwtKeyId();                    // Key ID for JWK
    boolean validateJwtWithJwsUrl();      // Use external JWS URL
    String jwsURL();                      // External JWS endpoint
}
```

### Configuration Properties

| Property Key | Default | Description |
|-------------|---------|-------------|
| `it.water.rest.frontend.url` | `""` | Frontend URL |
| `it.water.rest.services.url` | `""` | Backend services URL |
| `it.water.rest.upload.path` | `""` | Upload folder path |
| `it.water.rest.upload.maxFileSize` | `1024` | Max file size (bytes) |
| `it.water.rest.security.jwt.validate` | `true` | Validate JWT |
| `it.water.rest.security.jwt.duration` | (configured) | Token duration ms |
| `it.water.rest.security.jwt.validateByJws` | `false` | Use JWS URL |
| `it.water.rest.security.jwt.jwsUrl` | `""` | JWS endpoint URL |

---

## 13. Spring REST Implementation

### Spring-Specific Architecture

```
Spring MVC Auto-Configuration
  |
  +--- @EnableWaterFramework (activates Water Framework in Spring)
  |
  +--- WaterRestSpringConfiguration (configures REST)
  |      - Registers SpringJwtAuthenticationFilter as HandlerInterceptor
  |      - Configures Jackson ObjectMapper with WaterJacksonModule
  |      - Sets up CORS filters
  |
  +--- Spring REST Controllers (implements SpringRestApi interfaces)
         - @RestController or @FrameworkRestController
         - Uses @RequestMapping, @GetMapping, etc.
         - Uses @RequestBody, @PathVariable, @RequestParam
```

### Key Differences from JAX-RS

| Aspect | JAX-RS (CXF) | Spring MVC |
|--------|--------------|------------|
| Annotations | `@Path`, `@GET`, `@POST` | `@RequestMapping`, `@GetMapping` |
| Parameter binding | `@PathParam`, `@QueryParam` | `@PathVariable`, `@RequestParam` |
| Request body | `@Consumes` + parameter | `@RequestBody` |
| Auth filter | `ContainerRequestFilter` | `HandlerInterceptor` |
| Status codes | `Response.status()` | `@ResponseStatus` |
| Context | `@Context` injection | `HttpServletRequest` parameter |

### Spring REST Test Pattern

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@ContextConfiguration(classes = WaterRestSpringConfiguration.class)
@EnableWaterFramework
class MyEntityRestSpringTest {
    @Autowired
    private TestRestTemplate template;
    @Autowired
    private JwtTokenService jwtTokenService;

    @Test
    void testAuthenticatedEndpoint() {
        FakeUser user = new FakeUser();
        String jwt = jwtTokenService.generateJwtToken(user);
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + jwt);
        HttpEntity<?> httpEntity = new HttpEntity<>(headers);
        ResponseEntity<MyEntity> response = template.exchange(
            "/myentities/1", HttpMethod.GET, httpEntity, MyEntity.class);
        assertEquals(200, response.getStatusCodeValue());
    }

    @Test
    void testUnauthorizedAccess() {
        ResponseEntity<String> response = template.getForEntity(
            "/myentities/1", String.class);
        assertEquals(401, response.getStatusCodeValue());
    }
}
```

---

## 14. CXF (JAX-RS) REST Implementation

### CxfRestApiManager

**File:** `Rest/Rest-api-manager-apache-cxf/src/main/java/.../CxfRestApiManager.java`

```java
@FrameworkComponent(services = RestApiManager.class)
public class CxfRestApiManager implements RestApiManager {
    @Inject private RestApiRegistry restApiRegistry;
    @Inject private ComponentRegistry componentRegistry;
    @Inject private RestOptions restOptions;

    @Override
    public void startRestApiServer() {
        JAXRSServerFactoryBean serverFactory = new JAXRSServerFactoryBean();
        serverFactory.setAddress(restOptions.restRootContext());

        // Register REST resources as per-request proxies
        List<Object> providers = new ArrayList<>();
        for (Class<? extends RestApi> restApi : restApiRegistry.getRegisteredRestApis()) {
            Class<?> implClass = restApiRegistry.getRestApiImplementation(restApi);
            providers.add(new PerRequestProxyProvider(restApi, implClass, componentRegistry));
        }
        serverFactory.setServiceBeans(providers);

        // Add JWT filter
        serverFactory.setProviders(List.of(
            getGenericExceptionMapper(),
            getJacksonJsonProvider(),
            getContainerRequestFilters()
        ));

        // Add Swagger feature
        serverFactory.setFeatures(List.of(createSwaggerFeature()));

        server = serverFactory.create();
    }
}
```

### PerRequestProxyProvider

Creates a new proxy for each incoming request:

```java
public class PerRequestProxyProvider implements ResourceProvider {
    @Override
    public Object getInstance(Message m) {
        // 1. Create implementation instance
        Object impl = implClass.getDeclaredConstructor().newInstance();

        // 2. Inject dependencies
        WaterComponentsInjector.inject(componentRegistry, impl, injectFields);

        // 3. Create dynamic proxy with interceptor
        RestControllerProxy proxy = new RestControllerProxy(restApi, impl, componentRegistry);
        return Proxy.newProxyInstance(classLoader, interfaces, proxy);
    }
}
```

---

## 15. Swagger / OpenAPI Integration

### CXF Swagger Configuration

```java
private Swagger2Feature createSwaggerFeature() {
    Swagger2Feature swagger = new Swagger2Feature();
    swagger.setTitle(contextRoot + ": Application Rest Services");
    swagger.setDescription("List of all " + contextRoot + " rest services");
    swagger.setUsePathBasedConfig(true); // Required for OSGi
    swagger.setPrettyPrint(true);
    swagger.setBasePath(contextRoot);
    swagger.setSupportSwaggerUi(true);
    return swagger;
}
```

### Spring OpenAPI (Springdoc)

- Spring uses `springdoc-openapi` via `/v3/api-docs`
- Automatically discovers Spring REST controllers
- Uses Swagger annotations (`@Api`, `@ApiOperation`, `@ApiResponses`)

### Swagger Annotations Usage

```java
@Api(tags = "MyEntity")
@ApiOperation(value = "Find entity by ID", response = MyEntity.class)
@ApiResponses(value = {
    @ApiResponse(code = 200, message = "Entity found"),
    @ApiResponse(code = 404, message = "Entity not found"),
    @ApiResponse(code = 401, message = "Unauthorized")
})
MyEntity findMyEntity(@ApiParam(value = "Entity ID") @PathParam("id") long id);
```

---

## 16. REST Testing Patterns

### Unit Test (No Server)

```java
@ExtendWith({MockitoExtension.class, WaterTestExtension.class})
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class RestServiceTest implements Service {
    @Inject @Setter
    private RestOptions restOptions;

    @Test
    void testExceptionMapper() {
        GenericExceptionMapperProvider mapper = new GenericExceptionMapperProvider();
        Response r = mapper.toResponse(new UnauthorizedException());
        assertEquals(401, r.getStatus());
    }

    @Test
    void testRestOptions() {
        assertNotNull(restOptions.securityOptions());
        assertNotNull(restOptions.restRootContext());
    }
}
```

### JWT Security Test

```java
@ExtendWith({MockitoExtension.class, WaterTestExtension.class})
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class RestSecurityTest implements Service {

    @Test
    void testReleaseJWTWithRoles() {
        Set<TestRole> roles = new HashSet<>();
        roles.add(new TestRole("Role1"));
        roles.add(new TestRole("Role2"));
        TestUser u = new TestUser("fakeUser", roles);
        String token = getJwtTokenService().generateJwtToken(u);
        assertNotNull(token);
        assertTrue(getJwtTokenService().validateToken(
            Collections.singletonList(TestUser.class.getName()), token));
        Set<Principal> principals = getJwtTokenService().getPrincipals(token);
        assertEquals(3, principals.size()); // 1 user + 2 roles
    }

    @Test
    void testJWTSecurityContext() {
        // ... generate token with roles ...
        JwtSecurityContext ctx = new JwtSecurityContext(
            getJwtTokenService().getPrincipals(token));
        assertTrue(ctx.isSecure());
        assertFalse(ctx.isAdmin());
        assertTrue(ctx.isUserInRole("Role1"));
        assertEquals("JWT", ctx.getAuthenticationScheme());
    }

    @Test
    void testTokenManipulation() {
        String token = getJwtTokenService().generateJwtToken(user);
        token += "manipulated";
        assertFalse(getJwtTokenService().validateToken(issuers, token));
        assertEquals(0, getJwtTokenService().getPrincipals(token).size());
    }
}
```

### Integration Test (Spring Boot)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@EnableWaterFramework
class MyEntityRestIntegrationTest {
    @Autowired private TestRestTemplate template;
    @Autowired private JwtTokenService jwtTokenService;

    @Test
    void testCRUDFlow() {
        String jwt = jwtTokenService.generateJwtToken(testUser);
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.AUTHORIZATION, "Bearer " + jwt);
        headers.setContentType(MediaType.APPLICATION_JSON);

        // CREATE
        MyEntity entity = new MyEntity();
        entity.setName("test");
        HttpEntity<MyEntity> createRequest = new HttpEntity<>(entity, headers);
        ResponseEntity<MyEntity> createResponse = template.exchange(
            "/myentities", HttpMethod.POST, createRequest, MyEntity.class);
        assertEquals(200, createResponse.getStatusCodeValue());

        // READ
        long id = createResponse.getBody().getId();
        HttpEntity<?> readRequest = new HttpEntity<>(headers);
        ResponseEntity<MyEntity> readResponse = template.exchange(
            "/myentities/" + id, HttpMethod.GET, readRequest, MyEntity.class);
        assertEquals(200, readResponse.getStatusCodeValue());

        // DELETE
        ResponseEntity<Void> deleteResponse = template.exchange(
            "/myentities/" + id, HttpMethod.DELETE, readRequest, Void.class);
        assertEquals(200, deleteResponse.getStatusCodeValue());
    }
}
```

---

## 17. REST Initialization Flow

```
1. Application Startup
   |
2. ComponentRegistry initialized
   |
3. @FrameworkRestApi classes discovered via ClassIndex
   |
4. @FrameworkRestController implementations found and mapped
   |
5. RestApiRegistry stores interface → implementation mappings
   |
6. RestApiManager.startRestApiServer() called
   |
   +--- CXF: Creates JAXRSServerFactoryBean with:
   |    - PerRequestProxyProvider for each REST API
   |    - CxfJwtAuthenticationFilter
   |    - GenericExceptionMapperProvider
   |    - JacksonJsonProvider (with WaterJacksonModule)
   |    - Swagger2Feature
   |
   +--- Spring: Spring auto-configures with:
        - Spring MVC discovers @RestController beans
        - SpringJwtAuthenticationFilter as HandlerInterceptor
        - WaterJacksonModule on ObjectMapper
        - Springdoc OpenAPI
   |
7. Server ready — accepting requests
   |
8. Request arrives
   → JWT Filter validates token (if @LoggedIn)
   → Security context filled in Runtime
   → RestControllerProxy intercepts (before)
   → Controller method executes
   → RestControllerProxy intercepts (after)
   → Jackson serializes response (with @JsonView)
   → Exception mapper handles errors
```

---

## 18. Best Practices & Anti-Patterns

### Best Practices

1. **Always define a generic JAX-RS interface first.** The Spring interface extends it, ensuring a single source of truth.

2. **Use `@LoggedIn` at TYPE level for full entity APIs.** Apply at METHOD level only for mixed public/authenticated APIs.

3. **Use `@JsonView` to control serialization.** Don't expose `Internal` or `Secured` fields in REST responses.

4. **Combine `@AllowPermissions` with `@LoggedIn`.** Authentication must happen before authorization.

5. **Use `BaseEntityRestApi<T>` for entity CRUD.** Don't rewrite save/update/find/remove logic.

6. **Return proper HTTP status codes.** Use `GenericExceptionMapperProvider` — don't catch and re-throw exceptions in controllers.

7. **Test both authenticated and unauthenticated flows.** Ensure `@LoggedIn` endpoints return 401 without token.

8. **Use `WaterJsonView.Public.class` as default view.** Override with `Compact` for list endpoints needing smaller payloads.

9. **Keep REST controllers thin.** All business logic should be in the Api/SystemApi layer.

10. **Use Swagger annotations for documentation.** They auto-generate OpenAPI docs for both CXF and Spring.

### Anti-Patterns

1. **Putting business logic in REST controllers.** Controllers should only delegate to services.

2. **Bypassing `GenericExceptionMapperProvider`.** Don't catch exceptions and return custom Response objects.

3. **Forgetting `@FrameworkRestApi` on interfaces.** Without it, the framework won't discover the REST API.

4. **Missing `@FrameworkRestController(referredRestApi = ...)` on implementations.** The registry won't find the mapping.

5. **Using `@Secured` or Spring Security instead of `@LoggedIn`.** Water Framework uses its own JWT filter chain.

6. **Returning `Response` objects from entity APIs.** Return typed entities — let the framework handle status codes.

7. **Not defining both JAX-RS and Spring interfaces for dual deployment.** If you only need one, the other platform won't work.

---

## 19. Decision Trees

### How to Define a REST Endpoint

```
Need a REST endpoint?
  |
  +-- Entity CRUD?
  |     |
  |     +-- Standard CRUD operations?
  |     |     --> Extend BaseEntityRestApi<T>
  |     |     --> Use @FrameworkRestApi + @FrameworkRestController
  |     |
  |     +-- Custom entity operations?
  |           --> Add custom methods to RestApi interface
  |           --> Override in controller
  |
  +-- Non-entity endpoint?
  |     --> Create RestApi interface with @FrameworkRestApi
  |     --> Implement with @FrameworkRestController
  |
  +-- Public (no auth) endpoint?
  |     --> Don't add @LoggedIn to the method
  |
  +-- Authenticated endpoint?
        --> Add @LoggedIn with appropriate issuers
        --> Add @AllowPermissions for authorization
```

### How to Handle Security

```
Endpoint security needed?
  |
  +-- No authentication required?
  |     --> Don't use @LoggedIn
  |
  +-- Authentication only (any logged user)?
  |     --> @LoggedIn with default issuers
  |
  +-- Authentication + specific entity permissions?
  |     --> @LoggedIn + @AllowPermissions(actions, checkById)
  |
  +-- Authentication + generic resource permissions?
  |     --> @LoggedIn + @AllowGenericPermissions(actions, resourceName)
  |
  +-- Custom issuer validation?
        --> @LoggedIn(issuers = {"com.example.MyUser"})
```

---

## 20. Quick Reference Tables

### REST Module Files

| Component | Path |
|-----------|------|
| RestApi interface | `Core/Core-api/src/main/java/it/water/core/api/service/rest/RestApi.java` |
| @FrameworkRestApi | `Core/Core-api/src/main/java/it/water/core/api/service/rest/FrameworkRestApi.java` |
| @FrameworkRestController | `Core/Core-api/src/main/java/it/water/core/api/service/rest/FrameworkRestController.java` |
| WaterJsonView | `Core/Core-api/src/main/java/it/water/core/api/service/rest/WaterJsonView.java` |
| @LoggedIn | `Rest/Rest-api/src/main/java/it/water/service/rest/api/security/LoggedIn.java` |
| RestOptions | `Rest/Rest-api/src/main/java/it/water/service/rest/api/options/RestOptions.java` |
| RootApi | `Rest/Rest-api/src/main/java/it/water/service/rest/api/RootApi.java` |
| StatusApi | `Rest/Rest-api/src/main/java/it/water/service/rest/api/StatusApi.java` |
| GenericExceptionMapperProvider | `Rest/Rest-service/src/main/java/it/water/service/rest/GenericExceptionMapperProvider.java` |
| RestControllerProxy | `Rest/Rest-service/src/main/java/it/water/service/rest/RestControllerProxy.java` |
| GenericJWTAuthFilter | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/GenericJWTAuthFilter.java` |
| JwtSecurityContext | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JwtSecurityContext.java` |
| JwtTokenService | `Rest/Rest-security/src/main/java/it/water/service/rest/api/security/jwt/JwtTokenService.java` |
| CxfRestApiManager | `Rest/Rest-api-manager-apache-cxf/src/main/java/.../CxfRestApiManager.java` |
| CxfJwtAuthFilter | `Rest/Rest-api-manager-apache-cxf/src/main/java/.../CxfJwtAuthenticationFilter.java` |
| SpringJwtAuthFilter | `Rest/Rest-spring-api/src/main/java/.../SpringJwtAuthenticationFilter.java` |
| BaseEntityRestApi | `Rest/Rest-persistence/src/main/java/it/water/service/rest/persistence/BaseEntityRestApi.java` |

### HTTP Methods → JAX-RS → Spring

| Operation | JAX-RS | Spring |
|-----------|--------|--------|
| Create | `@POST` | `@PostMapping` |
| Read (single) | `@GET @Path("/{id}")` | `@GetMapping("/{id}")` |
| Read (list) | `@GET` | `@GetMapping` |
| Update | `@PUT` | `@PutMapping` |
| Delete | `@DELETE @Path("/{id}")` | `@DeleteMapping("/{id}")` |

### JsonView Selection Guide

| View | Use Case | Fields Included |
|------|----------|-----------------|
| `Compact` | List endpoints, mobile apps | ID, name, key fields |
| `Extended` | Detail views | Compact + descriptions, metadata |
| `Public` | Default full view | Extended + all non-sensitive fields |
| `Internal` | System-to-system only | System fields (never expose via REST) |
| `Secured` | Admin/encrypted contexts | Encrypted/hashed data |
| `Privacy` | GDPR-compliant views | Personal data (requires consent) |