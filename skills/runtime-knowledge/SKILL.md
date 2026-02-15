---
name: runtime-knowledge
description: Comprehensive knowledge base for Water Framework runtime environment, including ComponentRegistry, component lifecycle (@OnActivate/@OnDeactivate), service discovery, @FrameworkComponent annotation, @Inject dependency injection, proxy/interceptor mechanisms, OSGi/Spring runtime differences, and testing utilities. Use when designing, implementing, reviewing, or debugging component registration, lifecycle management, dependency injection, or runtime configuration in Water modules.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework runtime architect with deep knowledge of the ComponentRegistry, component lifecycle management, service discovery, dependency injection via @Inject, proxy/interceptor patterns, and multi-platform runtime abstraction (OSGi/Spring/Quarkus). You use this knowledge to guide correct component design, lifecycle implementation, and runtime configuration in Water modules.

---

# WATER FRAMEWORK RUNTIME KNOWLEDGE BASE

## Table of Contents

1. [Runtime Architecture Overview](#1-runtime-architecture-overview)
2. [Module Dependency Map](#2-module-dependency-map)
3. [Runtime Interface](#3-runtime-interface)
4. [ComponentRegistry](#4-componentregistry)
5. [@FrameworkComponent Annotation](#5-frameworkcomponent-annotation)
6. [Component Lifecycle: @OnActivate & @OnDeactivate](#6-component-lifecycle-onactivate--ondeactivate)
7. [Dependency Injection: @Inject](#7-dependency-injection-inject)
8. [ComponentFilter & Filtering](#8-componentfilter--filtering)
9. [ComponentConfiguration](#9-componentconfiguration)
10. [ApplicationConfiguration & Properties](#10-applicationconfiguration--properties)
11. [Service Discovery & ClassIndex](#11-service-discovery--classindex)
12. [Initialization Pipeline](#12-initialization-pipeline)
13. [Proxy & Interceptor Mechanisms](#13-proxy--interceptor-mechanisms)
14. [OSGi Runtime Implementation](#14-osgi-runtime-implementation)
15. [Spring Runtime Implementation](#15-spring-runtime-implementation)
16. [Testing Patterns](#16-testing-patterns)
17. [Best Practices & Anti-Patterns](#17-best-practices--anti-patterns)
18. [Decision Trees](#18-decision-trees)
19. [Quick Reference Tables](#19-quick-reference-tables)

---

## 1. Runtime Architecture Overview

Water Framework implements a **technology-agnostic runtime abstraction** that allows the same application code to run on OSGi, Spring, and Quarkus through a unified ComponentRegistry and lifecycle model.

### Layer Architecture

```
Application Components (@FrameworkComponent)
  |  annotated with lifecycle (@OnActivate, @OnDeactivate)
  |  injected via @Inject
  v
ComponentRegistry (abstract service registry)
  |  register, find, unregister components
  |  invoke lifecycle methods
  v
Runtime (thread-local security context + application properties)
  |
  +--- OSGi: OsgiComponentRegistry → BundleContext/ServiceRegistry
  +--- Spring: SpringComponentRegistry → ApplicationContext/BeanFactory
  +--- Test: TestComponentRegistry → In-memory HashMap
```

### Key Principles

- **Core-api** defines `Runtime`, `ComponentRegistry`, `ComponentFilter`, `ComponentConfiguration`, `@OnActivate`, `@OnDeactivate` (technology-agnostic interfaces)
- **Core-registry** provides `AbstractComponentRegistry` base implementation
- **Core-bundle** provides `WaterRuntime`, `RuntimeInitializer`, `ApplicationInitializer`, `AbstractInitializer`
- **Core-interceptors** provides `WaterAbstractInterceptor`, `WaterComponentsInjector`, `@Inject`
- **Implementation-osgi** provides `OsgiComponentRegistry`, `OsgiServiceInterceptor`, `WaterBundleActivator`
- **Implementation-spring** provides `SpringComponentRegistry`, `SpringServiceInterceptor`, `BaseSpringInitializer`
- **Core-testing-utils** provides `TestComponentRegistry`, `WaterTestRuntime`, `TestRuntimeUtils`

---

## 2. Module Dependency Map

```
Core-api (interfaces: Runtime, ComponentRegistry, ComponentFilter, ComponentConfiguration)
  |
  +--- Core-registry (AbstractComponentRegistry, filter implementations)
  |       depends on: Core-api
  |
  +--- Core-interceptors (WaterAbstractInterceptor, WaterComponentsInjector, @Inject)
  |       depends on: Core-api
  |
  +--- Core-bundle (WaterRuntime, RuntimeInitializer, ApplicationInitializer, AbstractInitializer)
  |       depends on: Core-api, Core-registry, Core-interceptors
  |
  +--- Implementation-osgi (OsgiComponentRegistry, OsgiServiceInterceptor, WaterBundleActivator)
  |       depends on: Core-api, Core-registry, Core-bundle
  |
  +--- Implementation-spring (SpringComponentRegistry, SpringServiceInterceptor, BaseSpringInitializer)
  |       depends on: Core-api, Core-registry, Core-bundle
  |
  +--- Core-testing-utils (TestComponentRegistry, WaterTestRuntime, TestRuntimeUtils)
          depends on: Core-api, Core-registry, Core-bundle
```

---

## 3. Runtime Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/bundle/Runtime.java`

```java
public interface Runtime extends Service {
    SecurityContext getSecurityContext();
    void fillSecurityContext(SecurityContext securityContext);
    ApplicationProperties getApplicationProperties();
}
```

### Methods

| Method | Purpose |
|--------|---------|
| `getSecurityContext()` | Returns thread-local SecurityContext for current request |
| `fillSecurityContext(ctx)` | Sets thread-local security context for current thread |
| `getApplicationProperties()` | Returns application-wide configuration properties |

### WaterRuntime (Production Implementation)

**File:** `Core/Core-bundle/src/main/java/it/water/core/bundle/WaterRuntime.java`

```java
@FrameworkComponent(services = Runtime.class)
public class WaterRuntime implements Runtime {
    private ThreadLocal<SecurityContext> securityContext = new ThreadLocal<>();

    @Inject @Setter
    private ApplicationProperties applicationProperties;

    @Override
    public SecurityContext getSecurityContext() {
        return securityContext.get();
    }

    @Override
    public void fillSecurityContext(SecurityContext ctx) {
        securityContext.set(ctx);
    }

    @Override
    public ApplicationProperties getApplicationProperties() {
        return applicationProperties;
    }
}
```

### Key Characteristics

- Uses `ThreadLocal<SecurityContext>` for thread-safe per-request security
- Injected with `ApplicationProperties` at startup
- Priority 1 (default) — can be overridden by higher-priority implementations
- Security context is null when no user is authenticated

---

## 4. ComponentRegistry

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/ComponentRegistry.java`

```java
public interface ComponentRegistry extends Service {
    // Find all components matching type and optional filter
    <T> List<T> findComponents(Class<T> componentClass, ComponentFilter filter);

    // Find single highest-priority component
    <T> T findComponent(Class<T> componentClass, ComponentFilter filter);

    // Register a new component
    <T, K> ComponentRegistration<T, K> registerComponent(
        Class<T> componentClass,
        T component,
        ComponentConfiguration configuration);

    // Unregister a component
    <T> boolean unregisterComponent(ComponentRegistration<?, ?> registration);
    <T> boolean unregisterComponent(T component);

    // Lifecycle method invocation
    <T> void invokeLifecycleMethod(LifecyclePhase phase, T component);
}
```

### Component Resolution

- **`findComponent(Class, filter)`** returns the component with the **highest priority** (lowest number)
- **`findComponents(Class, filter)`** returns all matching components, sorted by priority
- If no component is found, returns `null` (not an exception)

### Priority System

| Priority Value | Meaning |
|---------------|---------|
| 0 | Highest priority (custom override) |
| 1 | Default framework priority |
| 2 | Lower priority (test/fallback) |
| Higher numbers | Lower priority |

When multiple components implement the same interface, `findComponent()` returns the one with the lowest priority number.

### AbstractComponentRegistry

**File:** `Core/Core-registry/src/main/java/it/water/core/registry/AbstractComponentRegistry.java`

Base implementation providing:
- Lifecycle method invocation via reflection
- `@OnActivate` / `@OnDeactivate` method discovery
- Parameter resolution from the registry itself
- Proxy-aware method matching

---

## 5. @FrameworkComponent Annotation

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/FrameworkComponent.java` (via ClassIndex)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated  // Atteo ClassIndex for compile-time discovery
public @interface FrameworkComponent {
    Class<?>[] services() default {};  // Interfaces to register as
    int priority() default 1;         // Component priority
}
```

### Usage

```java
// Register as specific service interface
@FrameworkComponent(services = MyService.class)
public class MyServiceImpl implements MyService { ... }

// Register as multiple interfaces
@FrameworkComponent(services = {MyService.class, AnotherService.class})
public class MultiServiceImpl implements MyService, AnotherService { ... }

// Register with custom priority (overrides default implementation)
@FrameworkComponent(services = MyService.class, priority = 0)
public class HighPriorityServiceImpl implements MyService { ... }
```

### Key Points

- `services` parameter determines which interfaces the component is registered under
- If `services` is empty, the component is registered but not discoverable by interface
- `priority` determines resolution order when multiple implementations exist
- `@IndexAnnotated` ensures compile-time indexing (no classpath scanning at runtime)

---

## 6. Component Lifecycle: @OnActivate & @OnDeactivate

### @OnActivate

**File:** `Core/Core-api/src/main/java/it/water/core/api/interceptors/OnActivate.java`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OnActivate {
}
```

### @OnDeactivate

**File:** `Core/Core-api/src/main/java/it/water/core/api/interceptors/OnDeactivate.java`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OnDeactivate {
}
```

### Lifecycle Timing

```
Component Instantiation (new MyServiceImpl())
  |
  v
Field Injection (@Inject fields resolved and set)
  |
  v
Registry Registration (component stored in registry)
  |
  v
@OnActivate method invoked    <---- ACTIVATION
  |
  v
Component is live and serving requests
  |
  ... (component lifetime) ...
  |
  v
@OnDeactivate method invoked  <---- DEACTIVATION
  |
  v
Registry Unregistration
```

### Parameter Injection in Lifecycle Methods

The framework automatically resolves method parameters from the ComponentRegistry:

```java
@FrameworkComponent(services = MyService.class)
public class MyServiceImpl implements MyService {

    // No-argument activation
    @OnActivate
    public void onActivate() {
        log.info("Component activated");
    }

    // With parameter injection from registry
    @OnActivate
    public void onActivate(ComponentRegistry registry, ApplicationProperties props) {
        // Parameters resolved at runtime from ComponentRegistry
        this.someConfig = props.getProperty("my.config.key");
    }

    @OnDeactivate
    public void onDeactivate() {
        // Cleanup resources
        this.cache.clear();
    }
}
```

### Parameter Resolution Process

1. Inspect `@OnActivate` method signature for parameter types
2. For each parameter, call `componentRegistry.findComponent(parameterType, null)`
3. Pass resolved components to lifecycle method
4. Handle null parameters gracefully (with warning log)

### Important Notes

- **`@Inject` fields ARE available** during `@OnActivate` (they are injected before activation)
- Only **one** `@OnActivate` method per class is invoked
- If the component is behind a proxy, the framework searches for matching methods on the actual implementation class
- Parameters that cannot be resolved are passed as `null`

---

## 7. Dependency Injection: @Inject

### @Inject Annotation

**File:** `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/Inject.java`

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Inject {
    boolean injectOnceAtStartup() default true;
}
```

### Injection Modes

| Mode | `injectOnceAtStartup` | Behavior |
|------|----------------------|----------|
| Startup | `true` (default) | Inject once when component is registered |
| Runtime | `false` | Re-inject on every method call |

### Usage

```java
@FrameworkComponent(services = MyService.class)
public class MyServiceImpl implements MyService {

    @Inject @Setter    // Setter required for injection
    private AnotherService anotherService;

    @Inject @Setter
    private ComponentRegistry componentRegistry;

    @Inject(injectOnceAtStartup = false) @Setter  // Re-injected per call
    private DynamicService dynamicService;
}
```

### WaterComponentsInjector

**File:** `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/implementation/WaterComponentsInjector.java`

```java
@FrameworkComponent(services = WaterComponentsInjector.class)
public class WaterComponentsInjector implements BeforeMethodFieldInterceptor<Inject> {

    public static <S extends Service> void inject(
        ComponentRegistry componentRegistry, S destination, List<Field> fields) {
        fields.forEach(annotatedField -> {
            Object service = componentRegistry.findComponent(annotatedField.getType(), null);
            if (service != null) {
                Method setter = findSetterMethod(destination, annotatedField);
                setter.invoke(destination, service);
            }
        });
    }
}
```

### Setter Convention

- Field name: `myField`
- Required setter: `setMyField(FieldType value)`
- Use Lombok `@Setter` for automatic setter generation
- The setter must be accessible (public or protected)

### Injection Order

```
1. @FrameworkComponent classes instantiated
2. @Inject fields with injectOnceAtStartup=true are resolved and set
3. Components registered in ComponentRegistry
4. @OnActivate methods invoked
5. (On method calls) @Inject fields with injectOnceAtStartup=false are re-injected
```

---

## 8. ComponentFilter & Filtering

### ComponentFilter Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/filter/ComponentFilter.java`

```java
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
```

### ComponentFilterBuilder

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/filter/ComponentFilterBuilder.java`

```java
public interface ComponentFilterBuilder {
    ComponentFilter createFilter(String name, String value);
}
```

### Platform-Specific Implementations

| Platform | Builder | Filter Format |
|----------|---------|---------------|
| OSGi | `OSGiComponentFilterBuilder` | LDAP syntax: `(property=value)` |
| Spring | `SpringComponentFilterBuilder` | Properties matching |
| Test | `TestComponentFilterBuilder` | In-memory matching |

### Usage

```java
ComponentFilterBuilder filterBuilder = componentRegistry
    .findComponent(ComponentFilterBuilder.class, null);

// Simple filter
ComponentFilter filter = filterBuilder.createFilter("type", "sensor");
List<DeviceService> sensors = componentRegistry
    .findComponents(DeviceService.class, filter);

// Composite filter
ComponentFilter filter = filterBuilder.createFilter("priority", "HIGH")
    .and("status", "active")
    .or(filterBuilder.createFilter("name", "legacy"));

// Find with filter
MyService service = componentRegistry.findComponent(MyService.class, filter);
```

---

## 9. ComponentConfiguration

### ComponentConfiguration Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/ComponentConfiguration.java`

```java
public interface ComponentConfiguration {
    int getPriority();
    boolean isPrimary();
    Properties getConfiguration();
    Dictionary<String, Object> getConfigurationAsDictionary();
    void addProperty(String name, Object value);
    void removeProperty(String name);
    boolean hasProperty(String name);
}
```

### Creating Configuration

```java
ComponentConfiguration config = ComponentConfigurationFactory
    .createNewComponentPropertyFactory()
    .withPriority(2)
    .addProperty("region", "eu-west")
    .addProperty("maxRetries", 3)
    .build();

// Register component with configuration
componentRegistry.registerComponent(
    MyService.class,
    new MyServiceImpl(),
    config
);
```

### Finding by Configuration Properties

```java
// Find components with specific properties
ComponentFilter filter = filterBuilder.createFilter("region", "eu-west");
List<MyService> euServices = componentRegistry
    .findComponents(MyService.class, filter);
```

---

## 10. ApplicationConfiguration & Properties

### ApplicationConfiguration Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/ApplicationConfiguration.java`

```java
public interface ApplicationConfiguration {
    Properties getConfiguration();
}
```

### ApplicationProperties Interface

```java
public interface ApplicationProperties extends Service {
    Object getProperty(String key);
    void loadProperties(Properties props);
    void override(String key, Object value);
    boolean containsKey(String key);
}
```

### OSGi Configuration

**File:** `Implementation/Implementation-osgi/.../OsgiApplicationConfiguration.java`

```java
public void loadProperties() {
    Configuration config = confAdmin.getConfiguration("it.water.application");
    Dictionary<String, Object> dict = config.getProperties();
    // Convert to Properties
}
```

- Source: OSGi ConfigurationAdmin
- File: `etc/it.water.application.cfg` (in Karaf)
- Format: Key=value pairs

### Spring Configuration

**File:** `Implementation/Implementation-spring/.../SpringApplicationConfiguration.java`

```java
public void loadProperties() {
    for (PropertySource<?> source : environment.getPropertySources()) {
        if (source instanceof EnumerablePropertySource) {
            for (String name : ((EnumerablePropertySource<?>) source).getPropertyNames()) {
                props.put(name, environment.getProperty(name));
            }
        }
    }
}
```

- Sources (in priority order):
  1. Command-line arguments
  2. System properties
  3. `application-{profile}.properties`
  4. `application.properties`
  5. Environment variables
  6. Spring defaults

---

## 11. Service Discovery & ClassIndex

### Atteo ClassIndex

The framework uses **Atteo ClassIndex** for compile-time annotation indexing:

```java
@IndexAnnotated  // On annotations like @FrameworkComponent, @FrameworkRestApi
```

At compile time, an annotation processor generates index files in `META-INF/services/`. At runtime, discovery is instant (no classpath scanning).

### Discovery Process

```java
// In AbstractInitializer
protected void initializeFrameworkComponents() {
    // Get all @FrameworkComponent-annotated classes from ClassIndex
    Iterable<Class<?>> annotatedClasses = getAnnotatedClasses(FrameworkComponent.class);
    this.setupFrameworkComponents(annotatedClasses);
}
```

### Benefits over Classpath Scanning

| Aspect | ClassIndex | Classpath Scan |
|--------|-----------|----------------|
| Speed | O(1) lookup | O(n) classes |
| Reflection | Zero at discovery | Full scan |
| Cross-JAR | Works across JARs | May miss JARs |
| Startup time | Milliseconds | Seconds |

---

## 12. Initialization Pipeline

### AbstractInitializer

**File:** `Core/Core-bundle/src/main/java/it/water/core/bundle/AbstractInitializer.java`

```
initializeFrameworkComponents()
  |
  v
getAnnotatedClasses(FrameworkComponent.class)  -- ClassIndex lookup
  |
  v
setupFrameworkComponents(classes)
  |
  for each @FrameworkComponent class:
  |
  +-- 1. Instantiate via default constructor
  |
  +-- 2. Read @FrameworkComponent annotation (services, priority)
  |
  +-- 3. Create ComponentConfiguration (priority, properties)
  |
  +-- 4. Inject @Inject fields (injectOnceAtStartup=true)
  |       -> WaterComponentsInjector.inject(registry, component, fields)
  |
  +-- 5. Register component in ComponentRegistry
  |       -> registerComponent(serviceClass, component, config)
  |
  +-- 6. Invoke @OnActivate method
          -> invokeLifecycleMethod(ACTIVATE, component)
```

### ApplicationInitializer

**File:** `Core/Core-bundle/src/main/java/it/water/core/bundle/ApplicationInitializer.java`

Extends AbstractInitializer with additional application-level setup:

```
ApplicationInitializer.start()
  |
  +-- 1. initializeFrameworkComponents()
  |       (all @FrameworkComponent registration)
  |
  +-- 2. setupApplicationProperties()
  |       (load configuration)
  |
  +-- 3. initializeResourcePermissionsAndActions()
  |       (register permission actions for entities)
  |
  +-- 4. setupClusterMode()
  |       (if cluster support is available)
  |
  +-- 5. startRestApis()
          (start REST server via RestApiManager)
```

### RuntimeInitializer

**File:** `Core/Core-bundle/src/main/java/it/water/core/bundle/RuntimeInitializer.java`

```java
public abstract class RuntimeInitializer<T, K> extends ApplicationInitializer<T, K> {
    protected void initializeFrameworkComponents(boolean registerRegistry) {
        if (registerRegistry)
            registerComponentRegistryAsComponent();  // Registry registers itself
        super.initializeFrameworkComponents();
    }
}
```

### OSGi Initialization

**File:** `Implementation/Implementation-osgi/.../WaterBundleActivator.java`

```
start(BundleContext context):
  1. startFrameworkComponents()
  2. setupApplicationProperties()
  3. activateComponents()
  4. initializeResourcePermissionsAndActions()
  5. setupClusterMode()
  6. startRestApis()

stop(BundleContext context):
  1. stopRestApis()
  2. shutDownClusterMode()
  3. unregisterFrameworkComponents()
```

### Spring Initialization

**File:** `Implementation/Implementation-spring/.../BaseSpringInitializer.java`

Two-phase initialization:

```
Phase 1 - BeanFactoryPostProcessor:
  -> Register @FrameworkComponent classes as Spring beans
  -> Must happen before @Autowired validation

Phase 2 - ApplicationReadyEvent:
  -> setupApplicationProperties()
  -> initializeResourcePermissionsAndActions()
  -> setupClusterMode()
  -> startRestApis()
```

**Why two phases?** Spring validates `@Autowired` fields after BeanFactoryPostProcessor. Framework components must be registered before this validation, so @FrameworkComponents can be injected via `@Autowired` in Spring beans.

---

## 13. Proxy & Interceptor Mechanisms

### WaterAbstractInterceptor

**File:** `Core/Core-interceptors/src/main/java/it/water/core/interceptors/WaterAbstractInterceptor.java`

Base class for all interceptor chains:

```java
public abstract class WaterAbstractInterceptor<S extends Service> {

    protected <T extends Service> void executeInterceptorBeforeMethod(
        T service, Method method, Object[] args) {
        // 1. Find all BeforeMethodInterceptor components
        // 2. Execute each in priority order
        // 3. Find annotated fields → execute BeforeMethodFieldInterceptor
    }

    protected <T extends Service> void executeInterceptorAfterMethod(
        T service, Method method, Object[] args, Object result) {
        // 1. Find all AfterMethodInterceptor components
        // 2. Execute each in priority order
    }
}
```

### Interceptor Types

| Interface | Phase | Purpose |
|-----------|-------|---------|
| `BeforeMethodInterceptor<A>` | Before method call | Permission checks, validation |
| `BeforeMethodFieldInterceptor<A>` | Before, per-field | Field injection (@Inject) |
| `AfterMethodInterceptor<A>` | After method call | Auditing, post-processing |
| `AfterMethodFieldInterceptor<A>` | After, per-field | Return value transformation |

### OSGi Proxy Pattern

**File:** `Implementation/Implementation-osgi/.../OSGiUtil.java`

```java
public static <S extends Service> ServiceRegistration<S> registerProxyService(
    Bundle bundle, String[] interfaces, Dictionary<String, Object> config,
    ClassLoader cl, S service, ComponentRegistry registry) {

    // 1. Create interceptor wrapping the service
    OsgiServiceInterceptor<S> interceptor = new OsgiServiceInterceptor<>(service, registry);

    // 2. Create JDK dynamic proxy
    Object proxy = Proxy.newProxyInstance(cl, interfaceClasses, interceptor);

    // 3. Mark as proxy
    config.put("it.water.core.api.interceptors.isProxy", true);

    // 4. Register proxy (not original) in OSGi
    return bundleContext.registerService(interfaces, proxy, config);
}
```

### OsgiServiceInterceptor

```java
public class OsgiServiceInterceptor<S extends Service>
    extends WaterAbstractInterceptor<S> implements InvocationHandler {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        executeInterceptorBeforeMethod(getService(), method, args);
        Object result = method.invoke(getService(), args);
        executeInterceptorAfterMethod(getService(), method, args, result);
        return result;
    }
}
```

### Spring AOP Interceptor

**File:** `Implementation/Implementation-spring/.../SpringServiceInterceptor.java`

```java
@Aspect
@Component
public class SpringServiceInterceptor extends WaterAbstractInterceptor<Service> {

    @Pointcut("execution(* *(..)) && target(it.water.core.api.service.Service+)")
    public void waterServicesPointcut() { }

    @Before("waterServicesPointcut()")
    public void beforeServiceExecution(JoinPoint joinPoint) {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        Object[] args = joinPoint.getArgs();
        Service target = (Service) joinPoint.getTarget();
        executeInterceptorBeforeMethod(target, method, args);
    }

    @AfterReturning(value = "waterServicesPointcut()", returning = "result")
    public void afterServiceExecution(JoinPoint joinPoint, Object result) {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        Object[] args = joinPoint.getArgs();
        Service target = (Service) joinPoint.getTarget();
        executeInterceptorAfterMethod(target, method, args, result);
    }
}
```

### Key Difference

| Aspect | OSGi Proxy | Spring AOP |
|--------|-----------|------------|
| Mechanism | JDK Dynamic Proxy | Spring AOP (@Aspect) |
| Scope | Per-service proxy | Pointcut on all Service+ |
| Registration | Proxy registered in BundleContext | Spring manages beans |
| Interception | InvocationHandler.invoke() | @Before / @AfterReturning |

---

## 14. OSGi Runtime Implementation

### OsgiComponentRegistry

**File:** `Implementation/Implementation-osgi/.../OsgiComponentRegistry.java`

```java
public class OsgiComponentRegistry extends AbstractComponentRegistry {
    private BundleContext bundleContext;

    @Override
    public <T> T findComponent(Class<T> componentClass, ComponentFilter filter) {
        String filterStr = (filter != null) ? filter.getFilter() : null;
        ServiceReference<T>[] refs = bundleContext.getServiceReferences(componentClass, filterStr);
        // Sort by priority, return highest priority
        Arrays.sort(refs, priorityComparator);
        return bundleContext.getService(refs[0]);
    }

    @Override
    public <T, K> ComponentRegistration<T, K> registerComponent(
        Class<T> componentClass, T component, ComponentConfiguration config) {
        // Register as OSGi service (possibly with proxy)
        if (component instanceof Service) {
            return OSGiUtil.registerProxyService(bundle, interfaces, dict, cl, component, this);
        }
        return bundleContext.registerService(componentClass, component, dict);
    }
}
```

### OSGi-Specific Features

- Components registered as OSGi services
- Priority stored as service property: `it.water.component.priority`
- Filtering uses LDAP filter syntax
- Proxies created automatically for `Service` implementations
- Bundle lifecycle tied to OSGi start/stop

### OSGi Configuration

```
etc/it.water.application.cfg:
  my.property.key=value
  another.property=123
```

---

## 15. Spring Runtime Implementation

### SpringComponentRegistry

**File:** `Implementation/Implementation-spring/.../SpringComponentRegistry.java`

```java
public class SpringComponentRegistry extends AbstractComponentRegistry {
    private ConfigurableListableBeanFactory beanFactory;

    @Override
    public <T> T findComponent(Class<T> componentClass, ComponentFilter filter) {
        Map<String, T> beans = beanFactory.getBeansOfType(componentClass);
        // Filter and sort by priority
        TreeMap<Integer, T> sorted = new TreeMap<>(Comparator.reverseOrder());
        for (T bean : beans.values()) {
            if (filter == null || filter.matches(beanProperties(bean))) {
                sorted.put(getPriority(bean), bean);
            }
        }
        return sorted.isEmpty() ? null : sorted.lastEntry().getValue();
    }

    @Override
    public <T, K> ComponentRegistration<T, K> registerComponent(
        Class<T> componentClass, T component, ComponentConfiguration config) {
        String beanName = generateBeanName(componentClass);
        beanFactory.registerSingleton(beanName, component);
        if (config.isPrimary()) {
            // Mark as primary bean for @Autowired resolution
        }
        return new SpringComponentRegistration<>(beanName, component);
    }
}
```

### Spring-Specific Features

- Components registered as Spring singleton beans
- Uses `ConfigurableListableBeanFactory` for dynamic registration
- Priority managed via TreeMap ordering
- `isPrimary()` maps to Spring `@Primary` for `@Autowired` resolution
- Interception via Spring AOP (no explicit proxies needed)

### @EnableWaterFramework

```java
@EnableWaterFramework  // On Spring Boot main class or config
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### Spring Configuration

```yaml
# application.properties or application.yml
it.water.application.property.key=value
spring.profiles.active=dev
```

---

## 16. Testing Patterns

### TestComponentRegistry

**File:** `Core/Core-testing-utils/.../TestComponentRegistry.java`

In-memory registry for unit tests:

```java
public class TestComponentRegistry extends AbstractComponentRegistry {
    private Map<Class<?>, List<ComponentRegistration<?, ?>>> components = new HashMap<>();

    @Override
    public <T> T findComponent(Class<T> componentClass, ComponentFilter filter) {
        List<ComponentRegistration<?, ?>> regs = components.get(componentClass);
        if (regs == null || regs.isEmpty()) return null;
        // Sort by priority, return highest
        return (T) regs.get(0).getComponent();
    }
}
```

### WaterTestRuntime

**File:** `Core/Core-testing-utils/.../WaterTestRuntime.java`

```java
@FrameworkComponent(priority = 2, services = Runtime.class)
public class WaterTestRuntime implements Runtime {
    private SecurityContext securityContext;  // Field-based (not ThreadLocal)

    @Override
    public SecurityContext getSecurityContext() {
        return securityContext;
    }

    @Override
    public void fillSecurityContext(SecurityContext ctx) {
        this.securityContext = ctx;
    }
}
```

- Priority 2 (lower than production WaterRuntime which is 1)
- Uses field-based security context (not thread-local) for simpler testing
- Allows direct context manipulation

### WaterTestExtension (JUnit 5)

```java
@ExtendWith({MockitoExtension.class, WaterTestExtension.class})
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class MyServiceTest implements Service {

    @Inject @Setter
    private MyService myService;

    @Inject @Setter
    private ComponentRegistry componentRegistry;

    @Test
    void testServiceIsRegistered() {
        assertNotNull(myService);
    }
}
```

### TestRuntimeUtils

**File:** `Core/Core-testing-utils/.../TestRuntimeUtils.java`

```java
public class TestRuntimeUtils {

    // Execute code as admin user
    public static void impersonateAdmin(ComponentRegistry registry) {
        Runtime runtime = registry.findComponent(Runtime.class, null);
        SecurityContext adminCtx = createAdminSecurityContext();
        runtime.fillSecurityContext(adminCtx);
    }

    // Execute code as specific user
    public static void runAs(User user, Runnable operation) {
        Runtime runtime = getRuntime();
        SecurityContext ctx = createUserSecurityContext(user);
        runtime.fillSecurityContext(ctx);
        try {
            operation.run();
        } finally {
            runtime.fillSecurityContext(null);
        }
    }

    // Execute and return result as specific user
    public static <R> R getAs(User user, Supplier<R> supplier) {
        Runtime runtime = getRuntime();
        SecurityContext ctx = createUserSecurityContext(user);
        runtime.fillSecurityContext(ctx);
        try {
            return supplier.get();
        } finally {
            runtime.fillSecurityContext(null);
        }
    }

    // Assert validation exception with specific invalid fields
    public static void assertValidationException(Runnable r, String... invalidFields) {
        try {
            r.run();
            fail("Expected ValidationException");
        } catch (ValidationException e) {
            for (String field : invalidFields) {
                assertTrue(e.getViolations().stream()
                    .anyMatch(v -> v.getField().equals(field)));
            }
        }
    }
}
```

### BundleTestInitializer

```java
@BeforeAll
void setup() {
    // Create test initializer with mock registry
    BundleTestInitializer initializer = new BundleTestInitializer(
        testComponentRegistry, testRuntime);
    initializer.start();
}

@Test
void testComponentRegistration() {
    MyService service = componentRegistry.findComponent(MyService.class, null);
    assertNotNull(service);
}
```

---

## 17. Best Practices & Anti-Patterns

### Best Practices

1. **Always use `@FrameworkComponent` to register components.** Don't register manually unless you need dynamic registration.

2. **Specify `services` in `@FrameworkComponent`.** Without it, the component won't be discoverable by interface.

3. **Use `@Inject` with `@Setter` (Lombok).** The setter is required for the injection mechanism.

4. **Use `@OnActivate` for initialization that depends on other components.** Don't put complex logic in constructors.

5. **Use `@OnDeactivate` for cleanup.** Close connections, clear caches, unregister listeners.

6. **Use priority to override default implementations.**
   ```java
   @FrameworkComponent(services = MyService.class, priority = 0)
   public class CustomMyServiceImpl implements MyService { ... }
   ```

7. **Use `ComponentFilter` for conditional component selection.** Don't iterate through `findComponents()` manually.

8. **Test with `WaterTestExtension` and `TestRuntimeUtils`.** Use `impersonateAdmin()` for admin operations, `runAs()` for user-specific tests.

9. **Keep components stateless when possible.** Thread-local state should be in the `Runtime` security context.

10. **Use `injectOnceAtStartup = false` sparingly.** Runtime injection adds overhead per method call.

### Anti-Patterns

1. **Putting initialization logic in constructors.** The ComponentRegistry may not be ready during construction.

2. **Using `new` to create service instances.** Use the ComponentRegistry or `@Inject` — manual instantiation bypasses interceptors.

3. **Depending on registration order.** Components can be registered in any order. Use `@OnActivate` with parameter injection for dependencies.

4. **Ignoring priority.** If two implementations exist, the wrong one may be resolved.

5. **Using Spring `@Autowired` instead of Water `@Inject` in @FrameworkComponent classes.** Use `@Inject` for portable code that works on both OSGi and Spring.

6. **Accessing `Runtime.getSecurityContext()` outside request threads.** The ThreadLocal is null in background threads.

7. **Registering components with empty `services` array and expecting them to be findable by interface.** The `services` parameter is required for interface-based lookup.

---

## 18. Decision Trees

### How to Create a Component

```
Need a new framework component?
  |
  +-- Implements a service interface?
  |     --> @FrameworkComponent(services = MyService.class)
  |     --> Implement the interface
  |     --> Add @Inject @Setter for dependencies
  |
  +-- Needs initialization after dependencies are ready?
  |     --> Add @OnActivate method
  |
  +-- Needs cleanup on shutdown?
  |     --> Add @OnDeactivate method
  |
  +-- Needs to override an existing implementation?
  |     --> Set priority = 0 (higher than default 1)
  |
  +-- Needs configuration properties?
        --> Use @Inject ApplicationProperties
        --> Or accept ComponentConfiguration in registration
```

### How to Find a Component

```
Need to find a component?
  |
  +-- By interface (single best match)?
  |     --> componentRegistry.findComponent(MyService.class, null)
  |
  +-- By interface with filter?
  |     --> ComponentFilter filter = filterBuilder.createFilter("key", "value")
  |     --> componentRegistry.findComponent(MyService.class, filter)
  |
  +-- All implementations?
  |     --> componentRegistry.findComponents(MyService.class, null)
  |
  +-- In a @FrameworkComponent class?
        --> Use @Inject @Setter on a field of the desired type
```

### How to Test a Component

```
Need to test a component?
  |
  +-- Unit test (no server)?
  |     --> @ExtendWith({MockitoExtension.class, WaterTestExtension.class})
  |     --> implements Service
  |     --> @Inject @Setter for dependencies
  |
  +-- Need admin context?
  |     --> TestRuntimeUtils.impersonateAdmin(registry)
  |
  +-- Need specific user context?
  |     --> TestRuntimeUtils.runAs(user, () -> { ... })
  |
  +-- Need to validate exceptions?
        --> TestRuntimeUtils.assertValidationException(
              () -> service.save(invalidEntity), "fieldName")
```

---

## 19. Quick Reference Tables

### Key Annotations

| Annotation | Target | Purpose |
|-----------|--------|---------|
| `@FrameworkComponent` | TYPE | Register class as framework component |
| `@OnActivate` | METHOD | Called after component registration |
| `@OnDeactivate` | METHOD | Called before component unregistration |
| `@Inject` | FIELD | Inject dependency from ComponentRegistry |
| `@IndexAnnotated` | ANNOTATION_TYPE | Enable ClassIndex discovery |

### ComponentRegistry Methods

| Method | Returns | Purpose |
|--------|---------|---------|
| `findComponent(Class, filter)` | Single T | Best-priority match |
| `findComponents(Class, filter)` | List\<T\> | All matches sorted |
| `registerComponent(Class, T, config)` | Registration | Register new component |
| `unregisterComponent(registration)` | boolean | Unregister component |
| `invokeLifecycleMethod(phase, T)` | void | Invoke @OnActivate/@OnDeactivate |

### Platform Comparison

| Aspect | OSGi | Spring | Test |
|--------|------|--------|------|
| Registry | `OsgiComponentRegistry` | `SpringComponentRegistry` | `TestComponentRegistry` |
| Backing | `BundleContext` | `BeanFactory` | `HashMap` |
| Proxy | `OsgiServiceInterceptor` | Spring AOP `@Aspect` | `TestServiceProxy` |
| Config | `ConfigurationAdmin` | `Environment` | Manual `Properties` |
| Filter | LDAP syntax | Properties matching | In-memory |
| Init | `WaterBundleActivator` | `BaseSpringInitializer` | `BundleTestInitializer` |
| Runtime | `WaterRuntime` (ThreadLocal) | `WaterRuntime` (ThreadLocal) | `WaterTestRuntime` (field) |

### Key File Paths

| Component | Path |
|-----------|------|
| Runtime interface | `Core/Core-api/src/main/java/it/water/core/api/bundle/Runtime.java` |
| WaterRuntime | `Core/Core-bundle/src/main/java/it/water/core/bundle/WaterRuntime.java` |
| ComponentRegistry | `Core/Core-api/src/main/java/it/water/core/api/registry/ComponentRegistry.java` |
| AbstractComponentRegistry | `Core/Core-registry/src/main/java/it/water/core/registry/AbstractComponentRegistry.java` |
| @OnActivate | `Core/Core-api/src/main/java/it/water/core/api/interceptors/OnActivate.java` |
| @OnDeactivate | `Core/Core-api/src/main/java/it/water/core/api/interceptors/OnDeactivate.java` |
| @Inject | `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/Inject.java` |
| WaterComponentsInjector | `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/implementation/WaterComponentsInjector.java` |
| WaterAbstractInterceptor | `Core/Core-interceptors/src/main/java/it/water/core/interceptors/WaterAbstractInterceptor.java` |
| ComponentFilter | `Core/Core-api/src/main/java/it/water/core/api/registry/filter/ComponentFilter.java` |
| ComponentConfiguration | `Core/Core-api/src/main/java/it/water/core/api/registry/ComponentConfiguration.java` |
| ApplicationConfiguration | `Core/Core-api/src/main/java/it/water/core/api/registry/ApplicationConfiguration.java` |
| AbstractInitializer | `Core/Core-bundle/src/main/java/it/water/core/bundle/AbstractInitializer.java` |
| RuntimeInitializer | `Core/Core-bundle/src/main/java/it/water/core/bundle/RuntimeInitializer.java` |
| OsgiComponentRegistry | `Implementation/Implementation-osgi/src/main/java/it/water/implementation/osgi/registry/OsgiComponentRegistry.java` |
| OsgiServiceInterceptor | `Implementation/Implementation-osgi/src/main/java/it/water/implementation/osgi/interceptors/OsgiServiceInterceptor.java` |
| WaterBundleActivator | `Implementation/Implementation-osgi/src/main/java/it/water/implementation/osgi/bundle/WaterBundleActivator.java` |
| SpringComponentRegistry | `Implementation/Implementation-spring/src/main/java/it/water/implementation/spring/registry/SpringComponentRegistry.java` |
| SpringServiceInterceptor | `Implementation/Implementation-spring/src/main/java/it/water/implementation/spring/interceptors/SpringServiceInterceptor.java` |
| BaseSpringInitializer | `Implementation/Implementation-spring/src/main/java/it/water/implementation/spring/bundle/BaseSpringInitializer.java` |
| TestComponentRegistry | `Core/Core-testing-utils/src/main/java/it/water/core/testing/utils/registry/TestComponentRegistry.java` |
| WaterTestRuntime | `Core/Core-testing-utils/src/main/java/it/water/core/testing/utils/bundle/WaterTestRuntime.java` |
| TestRuntimeUtils | `Core/Core-testing-utils/src/main/java/it/water/core/testing/utils/runtime/TestRuntimeUtils.java` |