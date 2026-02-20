---
name: test-generation
description: Triggers when user asks to generate, complete, or improve tests for Water Framework modules. Enforces 80% code coverage and SonarQube compliance.
allowed-tools: Bash, Read, Glob, Grep, Edit, Write
---

You are an expert test engineer for the **Water Framework**. Your role is to generate, complete, and verify tests for Water modules, enforcing **80% minimum code coverage** (line + branch) and **SonarQube compliance**.

---

## Step 0: Analysis Phase

Before writing any test, analyze the target module thoroughly.

### 0.1 Identify the module structure

```bash
# Find all source classes that need coverage
find <Module>/<SubModule>/src/main/java -name "*.java" | sort

# Find existing tests
find <Module>/<SubModule>/src/test/java -name "*.java" | sort
```

### 0.2 Detect technology stack

Read `build.gradle` and `.yo-rc.json` at the module root to determine:
- **Water native** (`projectTechnology: water`) - uses `WaterTestExtension`
- **Spring Boot** (`projectTechnology: spring`) - uses `@SpringBootTest`
- **OSGi** (`projectTechnology: osgi`) - uses OSGi test bundles

### 0.3 Detect module capabilities

Check for:
- **REST services**: Look for `RestApi` interfaces in `<Module>-api/src/main/java/**/api/rest/`
- **Protected entities**: Check if entity implements `ProtectedEntity` or has `@AllowPermissions`
- **Owned entities**: Check if entity implements `OwnedResource`
- **Repository layer**: Look for `Repository` interfaces extending `BaseRepository`
- **SystemApi**: Look for `SystemApi` interfaces

### 0.4 Check current coverage

```bash
# If JaCoCo reports exist from a previous build
cat <Module>/<SubModule>/build/reports/jacoco/test/html/index.html 2>/dev/null
# Or the XML report
cat <Module>/<SubModule>/build/reports/jacoco/test/jacocoTestReport.xml 2>/dev/null
```

---

## Step 1: Test Structure Rules

### 1.1 Water Native Unit Test (canonical pattern)

This is the **primary test pattern** for all Water modules. Use `WaterTestExtension` for component injection.

```java
package <package>;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import it.water.core.api.registry.ComponentRegistry;
import it.water.core.api.service.Service;
import it.water.core.interceptors.annotations.Inject;
import it.water.core.testing.utils.junit.WaterTestExtension;
import it.water.core.testing.utils.runtime.TestRuntimeUtils;
import it.water.core.testing.utils.bundle.TestRuntimeInitializer;
import lombok.Setter;

@ExtendWith(WaterTestExtension.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class <Entity>ApiTest implements Service {

    @Inject
    @Setter
    private ComponentRegistry componentRegistry;

    @Inject
    @Setter
    private <Entity>Api entityApi;

    @Inject
    @Setter
    private <Entity>SystemApi entitySystemApi;

    @Inject
    @Setter
    private <Entity>Repository entityRepository;

    @Inject
    @Setter
    private Runtime runtime;

    // For protected entities - inject role/user managers
    @Inject
    @Setter
    private RoleManager roleManager;

    @Inject
    @Setter
    private UserManager userManager;

    // Test users (for permission testing)
    private it.water.core.api.model.User adminUser;
    private it.water.core.api.model.User managerUser;
    private it.water.core.api.model.User viewerUser;
    private it.water.core.api.model.User editorUser;

    @BeforeAll
    void beforeAll() {
        // Set up roles if entity is protected
        // Role managerRole = roleManager.getRole(<Entity>.DEFAULT_MANAGER_ROLE);
        // adminUser = userManager.findUser("admin");
        // managerUser = userManager.addUser("manager", ...);
        // roleManager.addRole(managerUser.getId(), managerRole);
        TestRuntimeUtils.impersonateAdmin(componentRegistry);
    }

    // Tests follow in @Order sequence...
}
```

### 1.2 Spring Boot REST Integration Test

For modules with `-service-spring` submodule:

```java
package <package>;

import com.intuit.karate.junit5.Karate;
import it.water.core.api.registry.ComponentRegistry;
import it.water.core.testing.utils.runtime.TestRuntimeUtils;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.TestPropertySource;

@SpringBootTest(classes = <Module>Application.class,
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
        "water.rest.security.jwt.validate.by.jws=false",
        "water.rest.security.jwt.validate=false",
        "water.testMode=true"
})
public class <Entity>RestSpringApiTest {
    @Autowired
    private ComponentRegistry componentRegistry;

    @LocalServerPort
    private int serverPort;

    @BeforeEach
    void impersonateAdmin() {
        TestRuntimeUtils.impersonateAdmin(componentRegistry);
    }

    @Karate.Test
    Karate restInterfaceTest() {
        return Karate.run("../<Module>-service/src/test/resources/karate")
                .systemProperty("webServerPort", String.valueOf(serverPort))
                .systemProperty("host", "localhost")
                .systemProperty("protocol", "http");
    }
}
```

### 1.3 Karate REST Test Runner (Water native)

For the `-service` submodule with REST:

```java
package <package>;

import com.intuit.karate.junit5.Karate;
import it.water.core.api.registry.ComponentRegistry;
import it.water.core.api.service.Service;
import it.water.core.interceptors.annotations.Inject;
import it.water.core.testing.utils.bundle.TestRuntimeInitializer;
import it.water.core.testing.utils.junit.WaterTestExtension;
import it.water.core.testing.utils.runtime.TestRuntimeUtils;
import lombok.Setter;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestInstance;
import org.junit.jupiter.api.extension.ExtendWith;

@ExtendWith(WaterTestExtension.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class <Entity>RestApiTest implements Service {

    @Inject
    @Setter
    private ComponentRegistry componentRegistry;

    @BeforeEach
    void beforeEach() {
        TestRuntimeUtils.impersonateAdmin(componentRegistry);
    }

    @Karate.Test
    Karate restInterfaceTest() {
        return Karate.run("classpath:karate")
                .systemProperty("webServerPort", TestRuntimeInitializer.getInstance().getRestServerPort())
                .systemProperty("host", "localhost")
                .systemProperty("protocol", "http");
    }
}
```

### 1.4 Karate Feature File Template

Place in `src/test/resources/karate/<entity>-crud.feature`:

```gherkin
# Generated with Water Generator
# The Goal of feature test is to ensure the correct format of json responses
# If you want to perform functional test please refer to ApiTest
Feature: Check <Entity> Rest Api Response

  Scenario: <Entity> CRUD Operations

    Given header Content-Type = 'application/json'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/<entities>'
    And request
    """
      {
        "entityVersion": 1,
        <json fields for creation>
      }
    """
    When method POST
    Then status 200
    And match response ==
    """
      {
        "id": #number,
        "entityVersion": 1,
        "entityCreateDate": '#number',
        "entityModifyDate": '#number',
        <expected json fields>
      }
    """
    * def entityId = response.id

    # --------------- UPDATE -----------------------------
    Given header Content-Type = 'application/json'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/<entities>'
    And request
    """
      {
        "id": #(entityId),
        "entityVersion": 1,
        <updated json fields>
      }
    """
    When method PUT
    Then status 200
    And match response ==
    """
      {
        "id": #number,
        "entityVersion": 2,
        "entityCreateDate": '#number',
        "entityModifyDate": '#number',
        <expected updated fields>
      }
    """

    # --------------- FIND -----------------------------
    Given header Content-Type = 'application/json'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/<entities>/'+entityId
    When method GET
    Then status 200

    # --------------- FIND ALL -------------------------
    Given header Content-Type = 'application/json'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/<entities>'
    When method GET
    Then status 200
    And match response.results contains
    """
    {
      "id": #(entityId),
      <expected fields>
    }
    """

    # --------------- DELETE ---------------------------
    Given header Content-Type = 'application/json'
    And header Accept = 'application/json'
    Given url serviceBaseUrl+'/water/<entities>/'+entityId
    When method DELETE
    Then status 204
```

---

## Step 2: Coverage Requirements (80% Minimum)

Every test class MUST cover the following scenarios to reach the 80% threshold.

### 2.1 Component Injection Verification

```java
@Test
@Order(1)
void componentsInstantiatedCorrectly() {
    Assertions.assertNotNull(componentRegistry.findComponent(<Entity>Api.class, null));
    Assertions.assertNotNull(componentRegistry.findComponent(<Entity>SystemApi.class, null));
    Assertions.assertNotNull(componentRegistry.findComponent(<Entity>Repository.class, null));
}
```

### 2.2 CRUD Operations (Happy Path)

| Order | Test Name | What it covers |
|-------|-----------|----------------|
| 2 | `saveOk` | Create entity, verify version=1 and id>0 |
| 3 | `updateShouldWork` | Find by query, modify field, verify version increments |
| 4 | `updateShouldFailWithWrongVersion` | Optimistic locking - stale version throws `WaterRuntimeException` |
| 5 | `findAllShouldWork` | Unpaginated findAll, verify count |
| 6 | `findAllPaginatedShouldWork` | Insert multiple, test pagination (pages, nextPage) |
| 7 | `removeAllShouldWork` | Delete all, verify countAll==0 |

### 2.3 Validation Failures

| Order | Test Name | What it covers |
|-------|-----------|----------------|
| 8 | `saveShouldFailOnDuplicatedEntity` | `DuplicateEntityException` on unique constraint violation |
| 9 | `saveShouldFailOnValidationFailure` | `ValidationException` on invalid input (e.g., XSS injection `<script>`) |
| 10 | `saveShouldFailWithNullRequiredFields` | `ValidationException` when mandatory fields are null |

### 2.4 Permission Scenarios (for Protected Entities)

| Order | Test Name | What it covers |
|-------|-----------|----------------|
| 11 | `managerCanDoEverything` | Manager role: save, update, find, remove all succeed |
| 12 | `viewerCannotSaveOrUpdateOrRemove` | Viewer role: find works, save/update/remove throw `UnauthorizedException` |
| 13 | `editorCannotRemove` | Editor role: save/update/find work, remove throws `UnauthorizedException` |
| 14 | `unauthorizedUserCannotAccess` | No role: all operations throw `UnauthorizedException` |

Use this pattern for permission impersonation:

```java
// Switch to a specific user
TestRuntimeInitializer.getInstance().impersonate(targetUser, runtime);

// Switch back to admin
TestRuntimeUtils.impersonateAdmin(componentRegistry);
```

### 2.5 Edge Cases

| Order | Test Name | What it covers |
|-------|-----------|----------------|
| 15 | `findByIdShouldFailForNonExistentEntity` | `find(9999L)` throws exception or returns null |
| 16 | `removeShouldFailForNonExistentEntity` | `remove(9999L)` throws exception |
| 17 | `findAllOnEmptyTableShouldReturnEmptyResults` | Empty result set handling |

### 2.6 SystemApi Tests (if applicable)

```java
@Test
@Order(20)
void systemApiSaveBypassesPermissions() {
    // SystemApi operates without permission checks
    runtime.fillSecurityContext(null); // clear security context
    <Entity> entity = createEntity(9001);
    Assertions.assertDoesNotThrow(() -> entitySystemApi.save(entity));
}
```

### 2.7 Repository-specific Tests (if custom queries exist)

Test every custom query method defined in the Repository interface with both valid and invalid inputs.

---

## Step 3: SonarQube Compliance Rules

All generated tests MUST comply with these rules. **Violations will cause SonarQube quality gate failure.**

### Mandatory Rules

1. **Every `@Test` method MUST have at least one assertion.** No empty tests.
   ```java
   // BAD
   @Test void testSomething() { entityApi.save(entity); }

   // GOOD
   @Test void saveOk() {
       Entity saved = entityApi.save(entity);
       Assertions.assertTrue(saved.getId() > 0);
   }
   ```

2. **Use `assertThrows` / `assertDoesNotThrow` for exception testing.** Never use try-catch.
   ```java
   // BAD
   @Test void testFail() {
       try { entityApi.save(null); fail(); } catch (ValidationException e) { /* ok */ }
   }

   // GOOD
   @Test void saveShouldFailWithNull() {
       Assertions.assertThrows(ValidationException.class, () -> entityApi.save(null));
   }
   ```

3. **No duplicate test code.** Extract entity factory methods:
   ```java
   private <Entity> createEntity(int seed) {
       return new <Entity>("name" + seed, "field" + seed, ...);
   }
   ```

4. **Test method names describe behavior.** Use pattern: `<action>Should<ExpectedResult>` or `<role>Can<Action>`.
   - `saveOk`, `updateShouldWork`, `saveShouldFailOnDuplicatedEntity`
   - `managerCanDoEverything`, `viewerCannotSaveOrUpdateOrRemove`

5. **No `@SuppressWarnings` without documented justification.**

6. **No `Thread.sleep()`.** Use framework synchronization or test utilities.

7. **No hardcoded file paths.** Use classpath resources:
   ```java
   // BAD
   File f = new File("/tmp/test-data.json");

   // GOOD
   InputStream is = getClass().getResourceAsStream("/test-data.json");
   ```

8. **No commented-out test methods.** Delete or fix them.

9. **Max cognitive complexity per method: 15.** Split complex test logic into helpers.

10. **Each test should test one behavior** (single assertion concept, though multiple related assertions are acceptable within one concept).

---

## Step 4: Test Generation Workflow

When generating tests for a module, follow this exact workflow:

### 4.1 Generate Missing Unit Tests

For each source class in `src/main/java` without a corresponding test:

1. **Entity model classes** -> Test validation rules, getters/setters, equality, toString
2. **Api interfaces** -> Test through the service implementation (ApiTest class)
3. **SystemApi interfaces** -> Test system-level operations without permission context
4. **Repository interfaces** -> Test custom queries if any
5. **ServiceImpl classes** -> Covered through Api/SystemApi tests
6. **RestControllerImpl** -> Covered through Karate feature files

### 4.2 Complete Existing Tests

If tests already exist but coverage is below 80%:

1. Run tests and read the JaCoCo report to identify uncovered lines
2. Add tests specifically targeting uncovered branches/lines
3. Pay special attention to:
   - Exception handling paths
   - Null checks and guard clauses
   - Switch/if branches
   - Validation logic

### 4.3 Helper Method: Entity Factory

Always create a private factory method in the test class to avoid test code duplication:

```java
private <Entity> createEntity(int seed) {
    <Entity> entity = new <Entity>();
    entity.setName("testName" + seed);
    entity.setDescription("testDescription" + seed);
    // Set all required fields with seed-based unique values
    return entity;
}
```

For entities with constructor parameters:

```java
private <Entity> createEntity(int seed, long roleId, long userId) {
    return new <Entity>("name" + seed, <other constructor args using seed>);
}
```

---

## Step 5: Verification

After generating all tests, verify coverage meets the 80% threshold.

### 5.1 Build and Run Tests

```bash
# Using the Water generator (preferred)
yo water:build --projects=<ModuleName>

# Or using Gradle directly
cd <ModulePath> && ./gradlew test jacocoTestReport
```

### 5.2 Check Coverage Report

```bash
# Read JaCoCo HTML report summary
cat <SubModule>/build/reports/jacoco/test/html/index.html | grep -A 5 "Total"

# Or read the XML report for precise numbers
cat <SubModule>/build/reports/jacoco/test/jacocoTestReport.xml
```

### 5.3 Coverage Gap Analysis

If coverage is below 80%:

1. Identify uncovered classes/methods from the JaCoCo report
2. Trace which source lines are not covered (red lines in HTML report)
3. Generate targeted tests for those specific paths
4. Re-run and verify

### 5.4 Checklist Before Completion

- [ ] All `@Test` methods have assertions
- [ ] No `@SuppressWarnings` without justification
- [ ] No `Thread.sleep()` in test code
- [ ] No commented-out tests
- [ ] No try-catch for exception testing (use `assertThrows`)
- [ ] Entity factory method extracts common creation logic
- [ ] Test method names follow naming convention
- [ ] Line coverage >= 80%
- [ ] Branch coverage >= 80%
- [ ] All CRUD operations tested (save, update, find, findAll, remove)
- [ ] Validation failure scenarios tested
- [ ] Permission scenarios tested (if entity is protected)
- [ ] Edge cases tested (non-existent IDs, empty results, boundary values)
- [ ] Karate feature file covers REST CRUD + response format validation (if REST module)

---

## Important Notes

- **Always read existing tests first** before generating new ones to avoid duplication
- **Never overwrite existing passing tests** - only add missing coverage
- **Use `@Order` annotations** to maintain test execution sequence (CRUD operations depend on data created in earlier tests)
- **Clean up test data** at the end of permission test blocks by switching back to admin: `TestRuntimeUtils.impersonateAdmin(componentRegistry)`
- **For Spring modules**: use `@Autowired` instead of `@Inject`/`@Setter` and `@SpringBootTest` instead of `@ExtendWith(WaterTestExtension.class)`
- **Karate feature files** test REST response format, not business logic - functional tests belong in the `ApiTest` class