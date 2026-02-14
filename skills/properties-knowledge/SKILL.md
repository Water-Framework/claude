---
name: properties-knowledge
description: Knowledge base for Water Framework module properties management, the Options pattern, ApplicationProperties, runtime property loading, environment variable resolution, and component configuration. Use when designing, implementing, or reviewing configurable properties for Water modules.
allowed-tools: Read, Glob, Grep
---

You are an expert Water Framework configuration architect with deep knowledge of the Options pattern, ApplicationProperties, runtime property loading, and component configuration. You use this knowledge to guide correct property design, help analyze configurable properties of modules, and define dedicated Options components following the framework conventions.

---

# WATER FRAMEWORK PROPERTIES & CONFIGURATION KNOWLEDGE BASE

## Table of Contents

1. [Properties Architecture Overview](#1-properties-architecture-overview)
2. [ApplicationProperties Interface](#2-applicationproperties-interface)
3. [Implementation-specific Property Loaders](#3-implementation-specific-property-loaders)
4. [The Options Pattern](#4-the-options-pattern)
5. [Options Pattern Step-by-Step Guide](#5-options-pattern-step-by-step-guide)
6. [Property Naming Conventions](#6-property-naming-conventions)
7. [Environment Variable Resolution](#7-environment-variable-resolution)
8. [ComponentConfiguration & Registry](#8-componentconfiguration--registry)
9. [FrameworkComponent Properties](#9-frameworkcomponent-properties)
10. [Runtime Property Override](#10-runtime-property-override)
11. [Existing Module Properties Catalog](#11-existing-module-properties-catalog)
12. [Testing Properties](#12-testing-properties)
13. [Best Practices & Anti-Patterns](#13-best-practices--anti-patterns)
14. [Decision Trees](#14-decision-trees)
15. [Quick Reference Tables](#15-quick-reference-tables)

---

## 1. Properties Architecture Overview

Water Framework uses a **technology-agnostic property management system** where:

- **ApplicationProperties** interface abstracts property loading across Spring, OSGi, and test environments
- **Options pattern** provides type-safe, module-scoped property accessors via dedicated `@FrameworkComponent` classes
- **Environment variable resolution** supports `${env:VAR_NAME:-default}` syntax
- **Runtime reloading** allows dynamic property changes without restart

### Layer Architecture

```
Property Source (file, env vars, Spring Environment, OSGi ConfigAdmin)
  |
  v
ApplicationProperties (technology-specific loader)
  |
  v
Options Component (@FrameworkComponent, injects ApplicationProperties)
  |  getPropertyOrDefault("key", defaultValue)
  v
Service / Api Layer (injects Options interface)
  |  options.getMyProperty()
  v
Business Logic
```

### Key Principle

Services NEVER read properties directly from `ApplicationProperties`. They inject an **Options interface** that provides typed, documented accessors with sensible defaults.

---

## 2. ApplicationProperties Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/bundle/ApplicationProperties.java`

```java
public interface ApplicationProperties {
    // Lifecycle
    void setup();

    // Core access
    Object getProperty(String key);
    boolean containsKey(String key);

    // Dynamic loading
    void loadProperties(File file);
    void loadProperties(Properties props);
    void unloadProperties(File file);
    void unloadProperties(Properties props);

    // Default values (with type overloads)
    default String getPropertyOrDefault(String propName, String defaultValue);
    default long getPropertyOrDefault(String propName, long defaultValue);
    default boolean getPropertyOrDefault(String propName, boolean defaultValue);

    // Environment variable resolution
    default String resolvePropertyValue(String propertyValue);
}
```

### getPropertyOrDefault Pattern

All overloads follow the same logic:
1. Try `getProperty(key)`
2. If not null → convert to target type and return
3. If null or exception → return `defaultValue`

```java
// String
String val = applicationProperties.getPropertyOrDefault("my.key", "fallback");

// long
long val = applicationProperties.getPropertyOrDefault("my.timeout", 3600000L);

// boolean
boolean val = applicationProperties.getPropertyOrDefault("my.feature.enabled", false);
```

---

## 3. Implementation-specific Property Loaders

### 3.1 Spring Implementation

**File:** `Implementation/Implementation-spring/src/main/java/it/water/implementation/spring/bundle/SpringApplicationProperties.java`

- Uses Spring's `Environment` bean
- Properties from `application.properties`, `application.yml`, environment variables, and Spring profiles
- `setup()` is no-op (Spring manages lifecycle)
- File-based `loadProperties()` NOT supported (uses Spring property sources)

**Property file:** `src/main/resources/application.properties`
```properties
water.rest.services.url=http://localhost:8080
it.water.user.registration.enabled=true
water.connectors.hadoop.url=hdfs://cluster:8020
```

### 3.2 OSGi Implementation

**File:** `Implementation/Implementation-osgi/src/main/java/it/water/implementation/osgi/bundle/OsgiApplicationProperties.java`

- Loads from `etc/it.water.application` (Karaf configuration directory)
- Integrates with OSGi `ConfigurationAdmin`
- Supports bundle-specific properties via `it.water.application.properties` resource
- Dynamic property loading via `loadBundleProperties()`
- Warns on property conflicts between bundles

**Property file:** `etc/it.water.application`
```properties
water.rest.services.url=http://localhost:8080
it.water.user.registration.enabled=true
```

### 3.3 Test Implementation

**File:** `Core/Core-testing-utils/src/main/java/it/water/core/testing/utils/bundle/TestApplicationProperties.java`

- Loads from `src/test/resources/it.water.application.properties`
- Supports runtime override via `override(String key, Object value)`
- Used in test environment initialization

**Property file:** `src/test/resources/it.water.application.properties`
```properties
water.testMode=true
water.keystore.password=water.
water.keystore.alias=server-cert
water.rest.security.jwt.validate=false
```

---

## 4. The Options Pattern

The Options pattern is the **standard way** to expose module configuration in Water Framework. Every configurable module follows this exact three-layer structure.

### Structure

```
module-api/
  └── src/main/java/.../api/options/
      └── MyModuleOptions.java          (Interface extends Service)

module-model/  (optional, for constants)
  └── src/main/java/.../model/
      └── MyModuleConstants.java         (Property key constants)

module-service/
  └── src/main/java/.../service/
      └── MyModuleOptionsImpl.java       (@FrameworkComponent implementing Options)
```

### Complete Example: Hadoop Module

**1. Options Interface** (`HadoopConnector-api`):

```java
// File: HadoopConnector-api/src/main/java/it/water/connectors/hadoop/api/options/HadoopOptions.java
package it.water.connectors.hadoop.api.options;

import it.water.core.api.service.Service;

public interface HadoopOptions extends Service {
    String getHadoopUrl();
}
```

**2. Options Implementation** (`HadoopConnector-service`):

```java
// File: HadoopConnector-service/src/main/java/it/water/connectors/hadoop/service/HadoopOptionsImpl.java
package it.water.connectors.hadoop.service;

import it.water.connectors.hadoop.api.options.HadoopOptions;
import it.water.core.api.bundle.ApplicationProperties;
import it.water.core.interceptors.annotations.FrameworkComponent;
import it.water.core.interceptors.annotations.Inject;
import lombok.Setter;

@FrameworkComponent
public class HadoopOptionsImpl implements HadoopOptions {
    @Inject
    @Setter
    private ApplicationProperties applicationProperties;

    @Override
    public String getHadoopUrl() {
        return applicationProperties.getPropertyOrDefault(
            "water.connectors.hadoop.url", "hdfs://localhost:8020");
    }
}
```

**3. Usage in Service**:

```java
@FrameworkComponent
public class HadoopConnectorServiceImpl implements HadoopConnectorApi {
    @Inject @Setter
    private HadoopOptions hadoopOptions;

    public void connect() {
        String url = hadoopOptions.getHadoopUrl();
        // Use the configured URL
    }
}
```

---

## 5. Options Pattern Step-by-Step Guide

### Step 1: Define Constants (in module-model)

```java
package it.water.mymodule.model;

public abstract class MyModuleConstants {
    // Naming: it.water.<module>.<section>.<property>
    public static final String PROP_BASE_URL = "it.water.mymodule.base.url";
    public static final String PROP_TIMEOUT = "it.water.mymodule.timeout";
    public static final String PROP_RETRY_ENABLED = "it.water.mymodule.retry.enabled";
    public static final String PROP_MAX_RETRIES = "it.water.mymodule.max.retries";

    private MyModuleConstants() {} // prevent instantiation
}
```

### Step 2: Define Options Interface (in module-api)

```java
package it.water.mymodule.api.options;

import it.water.core.api.service.Service;

public interface MyModuleOptions extends Service {
    String getBaseUrl();
    long getTimeout();
    boolean isRetryEnabled();
    int getMaxRetries();
}
```

**Rules:**
- Interface MUST extend `Service` (for component registry)
- Methods return typed values (String, long, boolean, int)
- Method names follow Java getter conventions
- No setter methods (read-only)

### Step 3: Implement Options (in module-service)

```java
package it.water.mymodule.service;

import it.water.mymodule.api.options.MyModuleOptions;
import it.water.mymodule.model.MyModuleConstants;
import it.water.core.api.bundle.ApplicationProperties;
import it.water.core.interceptors.annotations.FrameworkComponent;
import it.water.core.interceptors.annotations.Inject;
import lombok.Setter;

@FrameworkComponent
public class MyModuleOptionsImpl implements MyModuleOptions {
    @Inject
    @Setter
    private ApplicationProperties applicationProperties;

    @Override
    public String getBaseUrl() {
        return applicationProperties.getPropertyOrDefault(
            MyModuleConstants.PROP_BASE_URL, "http://localhost:8080");
    }

    @Override
    public long getTimeout() {
        return applicationProperties.getPropertyOrDefault(
            MyModuleConstants.PROP_TIMEOUT, 30000L);
    }

    @Override
    public boolean isRetryEnabled() {
        return applicationProperties.getPropertyOrDefault(
            MyModuleConstants.PROP_RETRY_ENABLED, true);
    }

    @Override
    public int getMaxRetries() {
        return (int) applicationProperties.getPropertyOrDefault(
            MyModuleConstants.PROP_MAX_RETRIES, 3L);
    }
}
```

### Step 4: Inject and Use in Services

```java
@FrameworkComponent
public class MyModuleServiceImpl implements MyModuleApi {
    @Inject @Setter
    private MyModuleOptions options;

    public void doWork() {
        String url = options.getBaseUrl();
        long timeout = options.getTimeout();
        if (options.isRetryEnabled()) {
            // retry logic with options.getMaxRetries()
        }
    }
}
```

### Step 5: Configure Properties

In `it.water.application.properties` (or `application.properties` for Spring):
```properties
it.water.mymodule.base.url=https://production.example.com
it.water.mymodule.timeout=60000
it.water.mymodule.retry.enabled=true
it.water.mymodule.max.retries=5
```

---

## 6. Property Naming Conventions

### Standard Patterns

| Scope | Pattern | Example |
|-------|---------|---------|
| Framework core | `water.core.*` | `water.core.api.service.cluster.node.id` |
| REST module | `water.rest.*` | `water.rest.services.url` |
| Security/JWT | `water.rest.security.*` | `water.rest.security.jwt.validate` |
| Keystore | `water.keystore.*` | `water.keystore.password` |
| Email module | `it.water.mail.*` | `it.water.mail.smtp.host` |
| User module | `it.water.user.*` | `it.water.user.registration.enabled` |
| Connectors | `water.connectors.<name>.*` | `water.connectors.hadoop.url` |
| Custom modules | `it.water.<module>.*` | `it.water.mymodule.base.url` |

### Naming Rules

1. Use lowercase dot-separated notation
2. Group by module, then section, then specific property
3. Boolean properties should end with `.enabled` or use `is` prefix in getter
4. URL properties should end with `.url`
5. Timeout properties should end with `.timeout` or `.duration.millis`
6. Password/secret properties should end with `.password` or `.secret`

---

## 7. Environment Variable Resolution

**Pattern:** `${env:VAR_NAME}` or `${env:VAR_NAME:-defaultValue}`

### How It Works

```java
// In ApplicationProperties.resolvePropertyValue():
Pattern ENV_PATTERN = Pattern.compile("\\$\\{env:([^:}]+)(:-([^}]+))?}");

// Resolution logic:
// 1. Match ${env:VAR_NAME:-default} pattern
// 2. Try System.getenv(VAR_NAME)
// 3. If env var not set -> use default value
// 4. If no default and env var not set -> empty string
```

### Usage in Properties Files

```properties
# Direct environment variable
water.connectors.hadoop.url=${env:HADOOP_URL}

# With default fallback
water.connectors.hadoop.url=${env:HADOOP_URL:-hdfs://localhost:8020}

# Database configuration
javax.persistence.jdbc.url=${env:DB_URL:-jdbc:postgresql://localhost:5432/waterdb}
javax.persistence.jdbc.user=${env:DB_USER:-water}
javax.persistence.jdbc.password=${env:DB_PASSWORD:-water123}

# Mixed with static text
it.water.mail.smtp.host=${env:SMTP_HOST:-smtp.gmail.com}
```

### Multiple Env Vars in One Property

```properties
# Multiple placeholders in same value
water.app.url=${env:PROTOCOL:-https}://${env:APP_HOST:-localhost}:${env:APP_PORT:-8080}
```

---

## 8. ComponentConfiguration & Registry

### ComponentConfiguration Interface

**File:** `Core/Core-api/src/main/java/it/water/core/api/registry/ComponentConfiguration.java`

```java
public interface ComponentConfiguration {
    int getPriority();
    boolean isPrimary();
    Dictionary<String, Object> getConfiguration();
    Dictionary<String, Object> getConfigurationAsDictionary();
    void addProperty(String key, Object value);
    void removeProperty(String key);
    boolean hasProperty(String key, Object value);
}
```

### PropertiesComponentConfiguration

**File:** `Core/Core-registry/src/main/java/it/water/core/registry/model/PropertiesComponentConfiguration.java`

Stores component-level properties (separate from application-level properties). Used for component filtering and identification.

### ComponentConfigurationFactory

**File:** `Core/Core-registry/src/main/java/it/water/core/registry/model/ComponentConfigurationFactory.java`

Fluent builder for creating component configurations:

```java
ComponentConfiguration config = ComponentConfigurationFactory
    .createNewComponentPropertyFactory()
    .withPriority(2)
    .withProp("implementation", "custom")
    .withProp("region", "eu-west-1")
    .build();

componentRegistry.registerComponent(MyService.class, myImpl, config);
```

---

## 9. FrameworkComponent Properties

**File:** `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/FrameworkComponent.java`

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@IndexAnnotated
public @interface FrameworkComponent {
    String[] properties() default {};     // Component properties (key=value)
    Class<?>[] services() default {};     // Exposed interfaces
    int priority() default 1;             // Higher = overrides lower
}
```

### Properties Attribute Usage

Component-level properties are used for **component filtering**, NOT application configuration:

```java
// Register a specific PermissionManager implementation
@FrameworkComponent(
    properties = {"implementation=default"},
    services = {PermissionManager.class}
)
public class PermissionManagerDefault implements PermissionManager { ... }

// Register a custom PermissionManager (overrides default)
@FrameworkComponent(
    priority = 2,
    properties = {"implementation=custom"},
    services = {PermissionManager.class}
)
public class CustomPermissionManager implements PermissionManager { ... }
```

### Priority System

- Default priority: `1` (framework components)
- Higher priority overrides lower when resolving via `ComponentRegistry.findComponent()`
- Use `priority = 2` or higher for application-level overrides

### Component Filtering

```java
// Find component by property filter
ComponentFilter filter = componentRegistry.getComponentFilterBuilder()
    .createFilter("implementation", "custom");
PermissionManager pm = componentRegistry.findComponent(PermissionManager.class, filter);
```

---

## 10. Runtime Property Override

### Dynamic Property Loading

```java
@Inject private ApplicationProperties applicationProperties;

// Load from Properties object
Properties newProps = new Properties();
newProps.put("it.water.mymodule.base.url", "https://new-server.com");
newProps.put("it.water.mymodule.timeout", "60000");
applicationProperties.loadProperties(newProps);

// Options getters IMMEDIATELY reflect new values
// (because they read from ApplicationProperties on each call)

// Unload properties (remove overrides)
applicationProperties.unloadProperties(newProps);
```

### Component Override at Runtime

Override an Options implementation with higher priority:

```java
// Create custom options implementation
MyModuleOptions customOptions = new MyModuleOptions() {
    @Override public String getBaseUrl() { return "https://test.example.com"; }
    @Override public long getTimeout() { return 5000L; }
    @Override public boolean isRetryEnabled() { return false; }
    @Override public int getMaxRetries() { return 0; }
};

// Register with higher priority
componentRegistry.registerComponent(
    MyModuleOptions.class,
    customOptions,
    ComponentConfigurationFactory.createNewComponentPropertyFactory()
        .withPriority(2)
        .build()
);
```

### Test Property Override

```java
// In test setup
TestApplicationProperties testProps =
    (TestApplicationProperties) applicationProperties;
testProps.override("it.water.mymodule.base.url", "http://test-server:9090");
```

---

## 11. Existing Module Properties Catalog

### Email Module

**Constants:** `EMail/EMail-model/src/main/java/it/water/email/model/EMailConstants.java`
**Options:** `EMail/EMail-api/src/main/java/it/water/email/api/EMailOptions.java`
**Impl:** `EMail/EMail-service/src/main/java/it/water/email/service/EMailOptionsImpl.java`

| Property Key | Method | Default | Description |
|-------------|--------|---------|-------------|
| `it.water.mail.sender.name` | `systemSenderName()` | `"Water-Application"` | Sender display name |
| `it.water.mail.smtp.host` | `smtpHostname()` | `null` | SMTP server hostname |
| `it.water.mail.smtp.port` | `smtpPort()` | `null` | SMTP server port |
| `it.water.mail.smtp.username` | `smtpUsername()` | `null` | SMTP authentication username |
| `it.water.mail.smtp.password` | `smtpPassword()` | `null` | SMTP authentication password |
| `it.water.mail.smtp.auth.enabled` | `isAuthEnabled()` | `null` | Enable SMTP authentication |
| `it.water.mail.smtp.start-ttls.enabled` | `isStartTTLSEnabled()` | `null` | Enable STARTTLS |

### User Module

**Constants:** `User/User-model/src/main/java/it/water/user/model/UserConstants.java`
**Options:** `User/User-api/src/main/java/it/water/user/api/options/UserOptions.java`
**Impl:** `User/User-service/src/main/java/it/water/user/service/UserOptionsImpl.java`

| Property Key | Method | Default | Description |
|-------------|--------|---------|-------------|
| `it.water.user.registration.enabled` | `isRegistrationEnabled()` | `true` | Enable user self-registration |
| `it.water.user.activation.url` | `getActivationUrl()` | (from REST) | URL for activation link |
| `it.water.user.password.reset.url` | `getPasswordResetUrl()` | (from REST) | URL for password reset |
| `it.water.user.admin.default.password` | `getAdminDefaultPassword()` | `"Water1!"` | Default admin password |
| `it.water.user.physical.deletion.enabled` | `isPhysicalDeletionEnabled()` | `true` | Hard-delete vs soft-delete |
| `it.water.user.registration.email.template.name` | `getRegistrationEmailTemplateName()` | `null` | Email template name |

### REST Module

**Constants:** `Rest/Rest-service/src/main/java/it/water/service/rest/RestConstants.java`
**Options:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/options/RestOptions.java`
**Impl:** `Rest/Rest-service/src/main/java/it/water/service/rest/RestOptionsImpl.java`

| Property Key | Method | Default | Description |
|-------------|--------|---------|-------------|
| `water.rest.services.url` | `getRestServicesUrl()` | `"http://localhost:8080"` | Backend services URL |
| `water.rest.frontend.url` | `getFrontendUrl()` | `""` | Frontend application URL |
| `water.rest.root.context` | `getRootContext()` | `"/water"` | REST root context path |
| `water.rest.uploadFolder.path` | `getUploadFolderPath()` | `""` | File upload directory |
| `water.rest.uploadFolder.maxFileSize` | `getUploadMaxFileSize()` | `10485760` | Max upload size (bytes) |

### JWT Security Module

**Constants:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JWTConstants.java`
**Options:** `Rest/Rest-api/src/main/java/it/water/service/rest/api/options/JwtSecurityOptions.java`
**Impl:** `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JwtSecurityOptionsImpl.java`

| Property Key | Method | Default | Description |
|-------------|--------|---------|-------------|
| `water.rest.security.jwt.validate` | `jwtValidationActive()` | `true` | Enable JWT validation |
| `water.rest.security.jwt.validate.by.jws` | `jwtValidationByJWS()` | `false` | Validate using JWS |
| `water.service.rest.security.jwt.validate.by.jws.key.id` | `jwtValidationJWSKeyId()` | `""` | JWS key identifier |
| `water.rest.security.jwt.encrypt` | `encryptJWTToken()` | `false` | Encrypt JWT tokens |
| `water.rest.security.jwt.jws.url` | `jwsUrl()` | `""` | JWS endpoint URL |
| `water.rest.security.jwt.duration.millis` | `jwtTokenDurationMillis()` | `3600000` | Token expiration (ms) |

### Hadoop Connector

**Options:** `HadoopConnector/HadoopConnector-api/src/main/java/it/water/connectors/hadoop/api/options/HadoopOptions.java`
**Impl:** `HadoopConnector/HadoopConnector-service/src/main/java/it/water/connectors/hadoop/service/HadoopOptionsImpl.java`

| Property Key | Method | Default | Description |
|-------------|--------|---------|-------------|
| `water.connectors.hadoop.url` | `getHadoopUrl()` | `"hdfs://localhost:8020"` | HDFS cluster URL |

### Cluster Node Options

**Options+Constants:** `Core/Core-api/src/main/java/it/water/core/api/service/cluster/ClusterNodeOptions.java`
**Impl:** `Core/Core-service/src/main/java/it/water/core/service/cluster/ClusterNodeOptionsImpl.java`

| Property Key | Method | Default | Description |
|-------------|--------|---------|-------------|
| `water.core.api.service.cluster.node.id` | `getNodeId()` | `"water-node-0"` | Node identifier |
| `water.core.api.service.cluster.node.layer.id` | `getLayerId()` | `"water-layer-0"` | Layer identifier |
| `water.core.api.service.cluster.node.ip` | `getNodeIp()` | `"127.0.0.1"` | Node IP address |
| `water.core.api.service.cluster.node.host` | `getNodeHost()` | `"localhost"` | Node hostname |
| `water.core.api.service.cluster.node.useIp` | `useIp()` | `false` | Use IP instead of hostname |
| `water.core.api.service.cluster.mode.enabled` | `clusterModeEnabled()` | `false` | Enable cluster mode |

### Keystore Properties (cross-module)

| Property Key | Default | Description |
|-------------|---------|-------------|
| `water.keystore.password` | `"water."` | Keystore password |
| `water.keystore.alias` | `"server-cert"` | Certificate alias |
| `water.keystore.file` | (varies) | Keystore file path |
| `water.private.key.password` | `"water."` | Private key password |

---

## 12. Testing Properties

### Test Property File

Create `src/test/resources/it.water.application.properties`:

```properties
# Test mode
water.testMode=true

# Keystore (for JWT)
water.keystore.password=water.
water.keystore.alias=server-cert
water.keystore.file=src/test/resources/certs/server.keystore
water.private.key.password=water.

# Disable JWT validation in tests
water.rest.security.jwt.validate=false
water.rest.security.jwt.duration.millis=3600000

# Module-specific test values
it.water.mymodule.base.url=http://test-server:9090
it.water.mymodule.timeout=5000
```

### Testing Options Components

```java
@ExtendWith(WaterTestExtension.class)
public class MyModuleOptionsTest {
    @Inject private MyModuleOptions options;
    @Inject private ApplicationProperties applicationProperties;

    @Test
    void testDefaultValues() {
        // Without properties set, defaults should be used
        assertNotNull(options.getBaseUrl());
        assertEquals("http://localhost:8080", options.getBaseUrl());
    }

    @Test
    void testDynamicPropertyLoading() {
        // Load new properties at runtime
        Properties props = new Properties();
        props.put("it.water.mymodule.base.url", "https://dynamic.example.com");
        applicationProperties.loadProperties(props);

        // Options immediately reflect new values
        assertEquals("https://dynamic.example.com", options.getBaseUrl());

        // Cleanup
        applicationProperties.unloadProperties(props);
    }
}
```

### Overriding Options in Tests

```java
@BeforeAll
void setup() {
    // Create test-specific Options with higher priority
    MyModuleOptions testOptions = new MyModuleOptionsImpl() {
        @Override public String getBaseUrl() { return "http://test:1234"; }
    };

    componentRegistry.registerComponent(
        MyModuleOptions.class,
        testOptions,
        ComponentConfigurationFactory.createNewComponentPropertyFactory()
            .withPriority(2)  // Override default (priority 1)
            .build()
    );
}
```

---

## 13. Best Practices & Anti-Patterns

### Best Practices

1. **Always use the Options pattern.** Never read ApplicationProperties directly in services.

2. **Define constants for property keys.** Use a `Constants` class in module-model to avoid string duplication.

3. **Always provide sensible defaults.** Use `getPropertyOrDefault()` with a meaningful fallback.

4. **Options interface MUST extend Service.** This enables injection via `@Inject` and component registry.

5. **Keep Options interfaces in module-api.** This allows other modules to depend on Options without depending on service implementation.

6. **One Options class per module.** Don't scatter property access across multiple classes.

7. **Use environment variable resolution for sensitive values.**
   ```properties
   it.water.mail.smtp.password=${env:SMTP_PASSWORD}
   ```

8. **Document property keys with comments in Constants class.** Future maintainers need to know what each property controls.

9. **Test with both default and overridden values.** Verify that defaults work AND that property loading overrides them correctly.

10. **Use `@FrameworkComponent` properties for component identification, not application configuration.** Component properties are for filtering/selection, not business settings.

### Anti-Patterns

1. **Reading ApplicationProperties directly in service code.**
   ```java
   // BAD
   String url = applicationProperties.getPropertyOrDefault("my.url", "default");

   // GOOD
   String url = myModuleOptions.getBaseUrl();
   ```

2. **Hardcoding property keys as inline strings.**
   ```java
   // BAD
   applicationProperties.getProperty("it.water.mymodule.url");

   // GOOD
   applicationProperties.getProperty(MyModuleConstants.PROP_BASE_URL);
   ```

3. **Putting Options implementation in module-api.** Implementation belongs in module-service.

4. **Forgetting @Setter on @Inject fields.** Without Lombok `@Setter`, the framework cannot inject dependencies.

5. **Using `getProperty()` without null check.** Always prefer `getPropertyOrDefault()`.

6. **Mixing component properties with application properties.** `@FrameworkComponent(properties = {...})` is for component identity, not configuration values.

---

## 14. Decision Trees

### How to Add Configurable Properties to a Module

```
Need to add configuration to a module?
  |
  +-- Does an Options interface already exist?
  |     |
  |     +-- YES: Add method to existing interface + implementation
  |     |
  |     +-- NO: Create the full Options pattern:
  |           1. Constants class (module-model)
  |           2. Options interface (module-api, extends Service)
  |           3. OptionsImpl class (module-service, @FrameworkComponent)
  |
  +-- Where to define the property value?
  |     |
  |     +-- Static value --> it.water.application.properties
  |     +-- Environment-dependent --> ${env:VAR_NAME:-default}
  |     +-- Spring-specific --> application.properties
  |     +-- OSGi-specific --> etc/it.water.application
  |     +-- Test only --> src/test/resources/it.water.application.properties
  |
  +-- How to consume?
        |
        +-- In a service --> @Inject MyModuleOptions options
        +-- In a test --> @Inject MyModuleOptions options (same pattern)
        +-- Override at runtime --> applicationProperties.loadProperties(props)
        +-- Override in test --> componentRegistry.registerComponent(..., priority=2)
```

### How to Analyze Module Properties

```
Want to understand a module's configurable properties?
  |
  1. Look for *Options.java in module-api/src/main/java/.../api/options/
  |
  2. Look for *Constants.java in module-model/src/main/java/.../model/
  |
  3. Look for *OptionsImpl.java in module-service/src/main/java/.../service/
  |
  4. Check getPropertyOrDefault() calls for keys and defaults
  |
  5. Check it.water.application.properties in test resources for example values
```

---

## 15. Quick Reference Tables

### Options Pattern Structure

| Layer | Location | Class | Purpose |
|-------|----------|-------|---------|
| Constants | `module-model` | `MyModuleConstants` | Property key strings |
| Interface | `module-api` | `MyModuleOptions extends Service` | Typed property accessors |
| Implementation | `module-service` | `@FrameworkComponent MyModuleOptionsImpl` | Reads from ApplicationProperties |

### ApplicationProperties Methods

| Method | Description |
|--------|-------------|
| `setup()` | Initialize at startup |
| `getProperty(key)` | Get raw value (may be null) |
| `containsKey(key)` | Check existence |
| `getPropertyOrDefault(key, String)` | Get String with fallback |
| `getPropertyOrDefault(key, long)` | Get long with fallback |
| `getPropertyOrDefault(key, boolean)` | Get boolean with fallback |
| `loadProperties(Properties)` | Load properties dynamically |
| `unloadProperties(Properties)` | Remove loaded properties |
| `resolvePropertyValue(value)` | Resolve `${env:VAR:-default}` |

### Property Loaders by Technology

| Technology | Class | Property Source |
|-----------|-------|----------------|
| Spring | `SpringApplicationProperties` | `application.properties`, Spring Environment |
| OSGi | `OsgiApplicationProperties` | `etc/it.water.application`, ConfigAdmin |
| Test | `TestApplicationProperties` | `src/test/resources/it.water.application.properties` |

### Key File Paths

| Component | Path |
|-----------|------|
| ApplicationProperties interface | `Core/Core-api/src/main/java/it/water/core/api/bundle/ApplicationProperties.java` |
| Spring loader | `Implementation/Implementation-spring/src/main/java/it/water/implementation/spring/bundle/SpringApplicationProperties.java` |
| OSGi loader | `Implementation/Implementation-osgi/src/main/java/it/water/implementation/osgi/bundle/OsgiApplicationProperties.java` |
| Test loader | `Core/Core-testing-utils/src/main/java/it/water/core/testing/utils/bundle/TestApplicationProperties.java` |
| ComponentConfiguration | `Core/Core-api/src/main/java/it/water/core/api/registry/ComponentConfiguration.java` |
| ComponentConfigurationFactory | `Core/Core-registry/src/main/java/it/water/core/registry/model/ComponentConfigurationFactory.java` |
| @FrameworkComponent | `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/FrameworkComponent.java` |
| @Inject | `Core/Core-interceptors/src/main/java/it/water/core/interceptors/annotations/Inject.java` |
| Hadoop Options (reference) | `HadoopConnector/HadoopConnector-api/src/main/java/it/water/connectors/hadoop/api/options/HadoopOptions.java` |
| Hadoop Options Impl | `HadoopConnector/HadoopConnector-service/src/main/java/it/water/connectors/hadoop/service/HadoopOptionsImpl.java` |
| Email Constants | `EMail/EMail-model/src/main/java/it/water/email/model/EMailConstants.java` |
| Email Options | `EMail/EMail-api/src/main/java/it/water/email/api/EMailOptions.java` |
| User Constants | `User/User-model/src/main/java/it/water/user/model/UserConstants.java` |
| User Options | `User/User-api/src/main/java/it/water/user/api/options/UserOptions.java` |
| Rest Constants | `Rest/Rest-service/src/main/java/it/water/service/rest/RestConstants.java` |
| JWT Constants | `Rest/Rest-security/src/main/java/it/water/service/rest/security/jwt/JWTConstants.java` |
| Cluster Options | `Core/Core-api/src/main/java/it/water/core/api/service/cluster/ClusterNodeOptions.java` |