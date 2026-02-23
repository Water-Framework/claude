---
name: water-generate
description: Use when the user wants to generate Water Framework code - new projects, entities, REST services, modules, or extensions using the yo water generator. Also use when the user asks about available generators, project scaffolding, or how to create microservices with Water/Spring/OSGi/Quarkus.
allowed-tools: Bash, Read, Glob, Grep
---

You are an expert assistant for the **Water Framework code generator** (`generator-water`), a Yeoman-based scaffolding tool for Java microservices.

> **CRITICAL RULE — GENERATOR FIRST**: Before writing, editing, or deleting ANY file in a Water Framework project, you MUST evaluate whether a generator command can perform that operation. Manual code changes are only permitted when no generator command covers the required operation. This rule applies to: creating projects, adding entities, adding REST layers, adding modules, building, and publishing.

---

## GENERATOR-FIRST PROTOCOL

Every time the user asks to create, modify, or build anything in a Water Framework project, run this evaluation:

```
1. READ .yo-rc.json in the project root to understand generator state
2. MATCH the requested operation to the generator command table below
3. IF a generator command exists → run it, then guide manual customization
4. IF no generator command exists → proceed with manual implementation
5. DOCUMENT any manual changes that were necessary (so they survive future regeneration)
```

### Quick Operation → Generator Command Mapping

| User asks to... | Check first | Generator command |
|----------------|-------------|-------------------|
| Create a new project/module | Always | `yo water:new-project` |
| Add a new entity with CRUD | Always | `yo water:add-entity` |
| Add REST API to a project | Always | `yo water:add-rest-services` |
| Add a custom Gradle submodule | Always | `yo water:new-empty-module` |
| Extend an entity from another module | Always | `yo water:new-entity-extension` |
| Build one or more projects | Always | `yo water:build --projects=X,Y` |
| Build everything | Always | `yo water:build-all` |
| Publish to Maven | Always | `yo water:publish` / `yo water:publish-all` |
| Analyze code quality | Always | `yo water:stabilityMetrics` |
| Add fields to an entity | **No generator** → manual edit | — |
| Add custom business logic | **No generator** → manual edit | — |
| Add configuration properties | **No generator** → manual edit (Options pattern) | — |
| Add custom REST endpoints | **No generator** → manual edit | — |
| Add dependencies to build.gradle | **No generator** → manual edit | — |
| Add a Redis/HTTP/custom integration | **No generator** → manual edit | — |

### When Manual Changes Are Required

If the generator cannot handle the operation, proceed manually but follow these rules:
1. Never edit `.yo-rc.json` manually — it is generator metadata, not user config
2. Follow Water Framework patterns (Options pattern for properties, `@FrameworkComponent`, `@Inject`)
3. Place files in the correct submodule (model/api/service) as the generator would
4. Use the same package structure the generator defines in `.yo-rc.json`

---

## READING .yo-rc.json

Before any operation, read `.yo-rc.json` in the project root to understand its configuration:

```bash
cat <ProjectName>/.yo-rc.json
```

Key fields and their meaning:

| Field | Description | Example |
|-------|-------------|---------|
| `projectName` | Project name (PascalCase) | `"ApiGateway"` |
| `projectGroupId` | Maven group ID | `"com.apigateway"` |
| `projectTechnology` | Target runtime | `"spring"`, `"water"`, `"osgi"`, `"quarkus"` |
| `applicationType` | Entity or service | `"entity"`, `"service"` |
| `modelName` | Primary entity name | `"Route"` |
| `hasRestServices` | REST layer enabled | `true`, `false` |
| `isProtectedEntity` | Permission system enabled | `true`, `false` |
| `isOwnedEntity` | Ownership semantics | `true`, `false` |
| `projectServicePath` | Path to service module | `"ApiGateway/ApiGateway-service"` |
| `apiPackage` | Java package for API interfaces | `"com.apigateway.api"` |
| `servicePackage` | Java package for implementations | `"com.apigateway.service"` |
| `entities` | List of all generated entities | `[{modelName, isProtectedEntity, isOwnedEntity}]` |
| `parent-project` | Root project marker | `true` |
| `inner-project` | Submodule marker | `true` |

Use this data to derive correct package names and paths before writing any manual code.

---

## Step 0: Prerequisites Check

Before running any generator command, run this single prerequisite script that auto-fixes Node version via NVM if needed:

```bash
# ── 1. Java >= 17 ──────────────────────────────────────────────────────────
java --version

# ── 2. Gradle >= 7.0 ───────────────────────────────────────────────────────
gradle --version

# ── 3. Node.js >= 18 (auto-fix via NVM if needed) ──────────────────────────
NODE_MAJOR=$(node --version 2>/dev/null | sed 's/v\([0-9]*\).*/\1/')
if [ -z "$NODE_MAJOR" ] || [ "$NODE_MAJOR" -lt 18 ]; then
  echo "Node < 18 detected (${NODE_MAJOR:-none}). Attempting NVM fix..."
  if [ -s "$HOME/.nvm/nvm.sh" ]; then
    source "$HOME/.nvm/nvm.sh"
    # Pick the highest installed version >= 18, or install LTS if none found
    TARGET=$(nvm list --no-colors 2>/dev/null \
              | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+' \
              | sed 's/v//' \
              | awk -F. '$1>=18' \
              | sort -t. -k1,1n -k2,2n -k3,3n \
              | tail -1)
    if [ -n "$TARGET" ]; then
      nvm use "$TARGET"
    else
      echo "No NVM version >= 18 found. Installing Node LTS..."
      nvm install --lts
      nvm use --lts
    fi
    echo "Node version now: $(node --version)"
  else
    echo "ERROR: NVM not found and Node < 18. Install NVM or Node >= 18 manually."
    exit 1
  fi
else
  echo "Node $(node --version) OK (>= 18)"
fi

# ── 4. yo (Yeoman) ─────────────────────────────────────────────────────────
yo --version 2>/dev/null || echo "yo not installed"

# ── 5. generator-water ─────────────────────────────────────────────────────
yo --generators 2>/dev/null | grep water || echo "generator-water not installed"
```

If `yo` or `generator-water` are missing after the Node check:
```bash
npm install -g yo generator-water --registry https://nexus.acsoftware.it/nexus/repository/npm-acs-public-repo
```

| Tool | Minimum | Check command |
|------|---------|---------------|
| Java | >= 17 | `java --version` |
| Gradle | >= 7.0 | `gradle --version` |
| Node.js | >= 18 | auto-checked and fixed by script above |
| yo | any | `yo --version` |
| generator-water | any | `yo --generators \| grep water` |

---

## Step 1: Understand the user's intent

| Operation | Command | When to use |
|-----------|---------|-------------|
| **New Project** | `yo water:new-project` | Brand new microservice (model + API + service layers) |
| **Add Entity** | `yo water:add-entity` | New JPA entity with full CRUD in existing project |
| **Add REST Services** | `yo water:add-rest-services` | REST layer on existing project without REST |
| **New Empty Module** | `yo water:new-empty-module` | Custom Gradle submodule in existing project |
| **New Entity Extension** | `yo water:new-entity-extension` | Extend entity from another module (e.g., WaterUser) |
| **Build** | `yo water:build` | Build selected projects (respects dependency order) |
| **Build All** | `yo water:build-all` | Build all workspace projects |
| **Publish** | `yo water:publish` | Publish selected projects to Maven |
| **Publish All** | `yo water:publish-all` | Publish all workspace projects |
| **Project Order** | `yo water:projects-order` | Define build/deploy precedence |
| **Show Order** | `yo water:projects-order-show` | Display current build order |
| **Stability Metrics** | `yo water:stabilityMetrics` | Code quality analysis |
| **Help** | `yo water:help --fulltext` | Full generator documentation |

---

## Step 2: Gather configuration for new-project

### Technology (`--projectTechnology`)
- **`water`** — Native Water Framework. Technology-agnostic. Supports Spring adapter.
- **`spring`** — Spring Boot 3.x. Full Spring Data JPA. Choose Spring (`--springRepository=true`) or Water repositories.
- **`osgi`** — OSGi/Karaf. Generates features XML for Karaf distribution.
- **`quarkus`** — Cloud-native. Ultra-fast startup, GraalVM compatible.

### Application Type (`--applicationType`)
- **`entity`** — Persistence application. Full CRUD: Model (JPA) + API + Service. Always has a model.
- **`service`** — Integration application. Business logic without owning entities. Optionally has model (`--hasModel=true`).

### Entity options (entity type only)
- `--modelName` — Entity class name in PascalCase (`Product`, `Order`, `Device`)
- `--isProtectedEntity` — Enable Permission System access control
- `--isOwnedEntity` — Enable ownership semantics (entities belong to users)

### REST Services (`--hasRestServices`)
- `true` → generates REST controllers, REST API interfaces, Karate test files
- `--restContextRoot` — REST base path (e.g., `/products`)
- `--hasAuthentication` — Add `@Login` for automatic authentication

### Additional Modules (`--moreModules` + `--modules`)
Available (comma-separated):
- `user-integration` — Remote user service querying
- `role-integration` — Remote role service querying
- `permission` — Local permission management
- `shared-entity-integration` — Remote shared entity querying

### Other options
- `--hasSonarqubeIntegration` — Adds Sonarqube properties for CI/CD
- `--publishModule` — Repository URL/name/credentials for Maven deployment
- `--springRepository` — Use Spring Data repositories (spring technology only)

---

## Step 3: Generate the command

### Non-interactive mode (PREFERRED — always use for automation)

Pass all parameters via `--inlineArgs` to skip prompts:

```bash
yo water:new-project --inlineArgs \
  --projectName=<name> \
  --projectTechnology=<water|spring|osgi|quarkus> \
  --applicationType=<entity|service> \
  --modelName=<EntityName> \
  --hasRestServices=<true|false> \
  --restContextRoot=<path> \
  --hasAuthentication=<true|false> \
  [other options...]
```

### Interactive mode (for first-time users only)
```bash
yo water:new-project
```

### Build commands (ALWAYS use generator, fallback to Gradle only if generator unavailable)

```bash
# Preferred: build specific projects via generator
yo water:build --projects=ProjectA,ProjectB

# Preferred: build all
yo water:build-all

# Fallback only (if yo is unavailable): build via Gradle directly
./gradlew :ProjectName-service:build -x test

# Skip tests (acceptable for local builds)
yo water:build --projects=ProjectA --skipTests=true
```

### Add entity to existing project

```bash
# Interactive (if yo-rc.json exists in target project directory)
yo water:add-entity

# With inline args
yo water:add-entity --inlineArgs \
  --modelName=<EntityName> \
  --isProtectedEntity=<true|false> \
  --isOwnedEntity=<true|false>
```

### Add REST services to existing project

```bash
yo water:add-rest-services --inlineArgs \
  --restContextRoot=<path> \
  --hasAuthentication=<true|false>
```

### Add empty custom module

```bash
yo water:new-empty-module --inlineArgs \
  --moduleName=<ModuleName>
```

### Extend entity from another module

```bash
yo water:new-entity-extension --inlineArgs \
  --sourceProject=<SourceProjectName> \
  --modelName=<EntityName>
```

---

## Step 4: Generated project structure

After generation, the project has this structure:

```
<ProjectName>/
  build.gradle                    # Parent build config
  settings.gradle                 # Module includes
  .yo-rc.json                     # Generator state (DO NOT edit manually)
  <ProjectName>-model/
    src/main/java/<package>/
      model/<Entity>.java          # JPA entity (add fields here)
  <ProjectName>-api/
    src/main/java/<package>/
      api/<Entity>Api.java          # Service interface
      api/<Entity>SystemApi.java    # System service interface
      api/<Entity>Repository.java   # Repository interface
      api/rest/<Entity>RestApi.java (if REST)
      api/options/<Module>Options.java  # (manual: Options pattern)
  <ProjectName>-service/
    src/main/java/<package>/
      service/<Entity>ServiceImpl.java      # Business logic (customize here)
      service/<Entity>SystemServiceImpl.java
      repository/<Entity>RepositoryImpl.java
      service/rest/<Entity>RestControllerImpl.java (if REST)
    src/test/java/<package>/
      <Entity>ApiTest.java
      <Entity>RestApiTest.java (if REST)
    src/test/resources/karate/
      <Entity>-crud.feature (if REST — Karate integration tests)
    src/main/resources/
      application.properties        # Config (add properties here)
```

---

## Step 5: Post-generation customization guide

After the generator runs, guide the user on what to customize manually:

| What to add | Where | Pattern to follow |
|-------------|-------|-------------------|
| Entity fields + JPA annotations | `<Entity>.java` in model | Jakarta Persistence (`@Column`, `@NotEmpty`) |
| Custom business methods | `<Entity>Api.java` interface + impl | Water Service interface pattern |
| Configuration properties | Options interface + impl | **Options pattern** (see properties-knowledge skill) |
| Extra dependencies | submodule `build.gradle` | Follow existing dependency style |
| Custom REST endpoints | REST controller impl | Water REST patterns (see rest-knowledge skill) |
| Custom queries | Repository impl | QueryBuilder fluent API (see persistence-knowledge skill) |
| Test coverage | `<Entity>ApiTest.java` | Water test extension pattern |

---

## Step 6: Build and publish workflow

```bash
# 1. Build a specific project (preferred)
yo water:build --projects=MyProject

# 2. Build with dependency chain (when multiple projects)
yo water:build --projects=Core,Repository,MyProject

# 3. Publish to local Maven (for inter-project dependencies)
yo water:publish --projects=MyProject
# OR directly via Gradle (faster for local dev):
./gradlew :MyProject-model:publishToMavenLocal :MyProject-api:publishToMavenLocal :MyProject-service:publishToMavenLocal -x test

# 4. Publish all
yo water:publish-all
```

---

## Decision tree: should I use the generator?

```
User requests a change to a Water project
  |
  +-- Is it creating a NEW project or module?
  |     YES → yo water:new-project (always)
  |
  +-- Is it adding a NEW entity with CRUD?
  |     YES → yo water:add-entity (always)
  |
  +-- Is it adding REST to a project without REST?
  |     YES → yo water:add-rest-services (always)
  |
  +-- Is it adding a new Gradle submodule?
  |     YES → yo water:new-empty-module (always)
  |
  +-- Is it extending an entity from another module?
  |     YES → yo water:new-entity-extension (always)
  |
  +-- Is it a BUILD operation?
  |     YES → yo water:build --projects=X (preferred)
  |           fallback: ./gradlew if yo is unavailable
  |
  +-- Is it a PUBLISH operation?
  |     YES → yo water:publish --projects=X
  |
  +-- Does it involve modifying entity fields?
  |     NO generator → edit <Entity>.java in model submodule
  |
  +-- Does it involve adding configuration properties?
  |     NO generator → implement Options pattern manually
  |                    (follow properties-knowledge skill)
  |
  +-- Does it involve adding custom REST endpoints?
  |     NO generator → edit REST controller in service submodule
  |                    (follow rest-knowledge skill)
  |
  +-- Does it involve adding business logic?
  |     NO generator → edit ServiceImpl in service submodule
  |
  +-- Does it involve adding dependencies?
  |     NO generator → edit submodule build.gradle
  |
  \-- None of the above?
        → Manual implementation following Water patterns
          Always check relevant knowledge skills first:
          - architecture-knowledge
          - persistence-knowledge
          - rest-knowledge
          - properties-knowledge
          - runtime-knowledge
```

---

## Common scenarios & full examples

**Scenario 1: New Spring CRUD microservice with REST and permissions**
```bash
yo water:new-project --inlineArgs \
  --projectName=product-catalog \
  --projectTechnology=spring \
  --applicationType=entity \
  --modelName=Product \
  --hasRestServices=true \
  --restContextRoot=/products \
  --hasAuthentication=true \
  --isProtectedEntity=true \
  --isOwnedEntity=true
```

**Scenario 2: Integration/connector service without persistence**
```bash
yo water:new-project --inlineArgs \
  --projectName=notification-service \
  --projectTechnology=water \
  --applicationType=service \
  --hasModel=false \
  --hasRestServices=true \
  --restContextRoot=/notifications
```

**Scenario 3: OSGi modular project**
```bash
yo water:new-project --inlineArgs \
  --projectName=iot-gateway \
  --projectTechnology=osgi \
  --applicationType=entity \
  --modelName=Device \
  --hasRestServices=true \
  --restContextRoot=/devices
```

**Scenario 4: Quarkus cloud-native project**
```bash
yo water:new-project --inlineArgs \
  --projectName=order-service \
  --projectTechnology=quarkus \
  --applicationType=entity \
  --modelName=Order \
  --hasRestServices=true \
  --restContextRoot=/orders
```

**Scenario 5: Add second entity to existing project**
```bash
cd ApiGateway  # must be in project root where .yo-rc.json lives
yo water:add-entity --inlineArgs \
  --modelName=RateLimitRule \
  --isProtectedEntity=true \
  --isOwnedEntity=false
```

**Scenario 6: Build only selected projects**
```bash
yo water:build --projects=Core,Repository,JpaRepository,MyProject
```

**Scenario 7: Connector/integration project (service type, manual customization after)**
```bash
# Step 1: scaffold with generator
yo water:new-project --inlineArgs \
  --projectName=redis-api-gateway \
  --projectTechnology=spring \
  --applicationType=service \
  --hasModel=true \
  --hasRestServices=true \
  --restContextRoot=/redis

# Step 2: manually add after generation:
# - RedisConstants.java (model submodule)
# - RedisOptions interface + impl (api + service submodules)
# - Redis client dependency in service/build.gradle
# - Connection manager service impl
# - application.properties entries
```

---

## Important rules

- **Workspace required**: All commands except `new-project` require a project with `.yo-rc.json`.
- **Never edit `.yo-rc.json`**: It is generator metadata. Manual edits can break re-generation.
- **Naming conventions**: Project names use kebab-case (`my-project`), entity names use PascalCase (`MyEntity`).
- **Group ID**: Auto-derived from project name if not specified (`my-project` → `com.my.project`).
- **Water version**: Always `project.waterVersion` — managed by WaterWorkspaceGradlePlugin, never hardcoded.
- **Re-generation**: Running `add-entity` or `add-rest-services` on an existing project is safe — generator merges rather than overwrites.
- **Template strategy**: Generator uses fallback: exact version → minor (3.0.X) → major (3.X.Y) → basic.
- **Inline args are preferred**: Always use `--inlineArgs` to avoid interactive prompts in automated contexts.
- **Build order matters**: Always specify dependency chain in `--projects` (e.g., `Core,Repository,MyProject`).