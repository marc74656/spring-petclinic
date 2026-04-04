# CLAUDE.md — AI Assistant Guide for Spring PetClinic

This file provides context for AI assistants (Claude and others) working in this repository.

## Project Overview

Spring PetClinic is a sample Spring Boot application demonstrating enterprise Java best practices. It manages a pet clinic's owners, pets, veterinarians, and visits. It is intentionally kept simple to serve as a reference for learning Spring Boot patterns.

- **Framework:** Spring Boot 4.0.3
- **Java Version:** 17 (minimum)
- **Build Tools:** Both Maven and Gradle are supported
- **Default Database:** H2 in-memory (MySQL and PostgreSQL also supported)
- **Template Engine:** Thymeleaf
- **License:** Apache 2.0

---

## Repository Structure

```
src/
├── main/
│   ├── java/org/springframework/samples/petclinic/
│   │   ├── PetClinicApplication.java       # Main entry point (@SpringBootApplication)
│   │   ├── PetClinicRuntimeHints.java      # GraalVM native image hints
│   │   ├── model/                          # Shared base domain classes
│   │   │   ├── BaseEntity.java             # Abstract base with auto-generated ID
│   │   │   ├── NamedEntity.java            # Adds name field
│   │   │   └── Person.java                 # Adds firstName, lastName
│   │   ├── owner/                          # Owner/Pet/Visit domain
│   │   │   ├── Owner.java                  # Owner entity (extends Person)
│   │   │   ├── Pet.java                    # Pet entity (extends NamedEntity)
│   │   │   ├── PetType.java                # Reference data (cat, dog, etc.)
│   │   │   ├── Visit.java                  # Vet visit record
│   │   │   ├── OwnerRepository.java        # JpaRepository with pagination
│   │   │   ├── PetTypeRepository.java      # JpaRepository for pet types
│   │   │   ├── OwnerController.java        # Owner CRUD routes
│   │   │   ├── PetController.java          # Pet management routes
│   │   │   ├── VisitController.java        # Visit management routes
│   │   │   ├── PetValidator.java           # Custom validator for Pet
│   │   │   └── PetTypeFormatter.java       # Spring Formatter for PetType
│   │   ├── vet/                            # Vet/Specialty domain
│   │   │   ├── Vet.java                    # Vet entity (extends Person)
│   │   │   ├── Specialty.java              # Specialty reference data
│   │   │   ├── VetRepository.java          # Cached repository (Repository<Vet, Integer>)
│   │   │   └── VetController.java          # HTML + JSON REST endpoints
│   │   └── system/                         # Infrastructure/configuration
│   │       ├── CacheConfiguration.java     # JCache + Caffeine "vets" cache
│   │       ├── WebConfiguration.java       # i18n + locale configuration
│   │       ├── WelcomeController.java      # GET / -> welcome.html
│   │       └── CrashController.java        # Demo error handling
│   └── resources/
│       ├── application.properties          # Main application config
│       ├── application-mysql.properties    # MySQL profile config
│       ├── application-postgres.properties # PostgreSQL profile config
│       ├── banner.txt                      # Custom startup banner
│       ├── db/
│       │   ├── h2/                         # H2 schema + sample data
│       │   ├── mysql/                      # MySQL schema + setup
│       │   └── postgres/                   # PostgreSQL schema + setup
│       ├── messages/                       # i18n properties (9 languages)
│       ├── templates/                      # Thymeleaf HTML templates
│       └── static/                         # CSS, fonts, images (Bootstrap 5, Font Awesome 4)
└── test/
    ├── java/org/springframework/samples/petclinic/
    │   ├── owner/                          # Controller + repository slice tests
    │   ├── vet/                            # Vet controller tests
    │   ├── system/                         # Crash controller tests
    │   └── service/                        # ClinicServiceTests (DataJpaTest)
    └── jmeter/                             # JMeter performance test plans
```

---

## Build & Run Commands

### Maven (preferred)

```bash
# Run the application (H2 default)
./mvnw spring-boot:run

# Full build with tests
./mvnw clean verify

# Run tests only
./mvnw test

# Run a specific test class
./mvnw test -Dtest=OwnerControllerTests

# Build without tests
./mvnw clean package -DskipTests

# Compile SCSS to CSS (requires css profile)
./mvnw package -P css

# Build a container image
./mvnw spring-boot:build-image
```

### Gradle

```bash
./gradlew bootRun          # Run the application
./gradlew build            # Full build with tests
./gradlew test             # Run tests only
./gradlew check            # Run all checks (checkstyle, formatting)
./gradlew bootBuildImage   # Build container image
./gradlew format           # Apply code formatting
```

---

## Database Configuration

### Default: H2 (in-memory)

No setup required. H2 Console available at `http://localhost:8080/h2-console` (UUID printed at startup).

### MySQL

```bash
# Start MySQL via Docker
docker run -e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic \
  -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=petclinic \
  -p 3306:3306 mysql:9.6

# Run app with MySQL profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=mysql
```

Environment variables: `MYSQL_URL`, `MYSQL_USER`, `MYSQL_PASS`

### PostgreSQL

```bash
# Start PostgreSQL via Docker
docker run -e POSTGRES_USER=petclinic -e POSTGRES_PASSWORD=petclinic \
  -e POSTGRES_DB=petclinic -p 5432:5432 postgres:18.3

# Run app with PostgreSQL profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=postgres
```

Environment variables: `POSTGRES_URL`, `POSTGRES_USER`, `POSTGRES_PASS`

### Docker Compose (MySQL + PostgreSQL together)

```bash
docker compose up
```

---

## Architecture & Design Patterns

### Layered Architecture

```
HTTP Request → Controller → Repository → Database
                  ↕              ↕
              Thymeleaf       Entity/Model
              Templates
```

- **No service layer** — business logic lives in repositories and entities
- **No DTOs** — domain entities are used directly in controllers and views
- **Repositories** extend `JpaRepository<Entity, Integer>` or `Repository<Entity, Integer>`
- **Controllers** use `@Controller` (not `@RestController`) returning view names

### Key Patterns

| Pattern | Implementation |
|---------|----------------|
| Repository | Spring Data JPA (`JpaRepository`) |
| Dependency Injection | Constructor injection preferred |
| Caching | `@Cacheable("vets")` with Caffeine via JCache |
| Transactions | `@Transactional(readOnly=true)` on read queries |
| Validation | Jakarta Bean Validation + custom `PetValidator` |
| Pagination | `Pageable` parameter in repository methods |
| i18n | `SessionLocaleResolver` + `LocaleChangeInterceptor` |

### Entity Hierarchy

```
BaseEntity (id)
├── NamedEntity (name)
│   ├── PetType
│   └── Pet → has List<Visit>
│       └── Specialty
└── Person (firstName, lastName)
    ├── Owner → has List<Pet>
    └── Vet → has List<Specialty>
```

### URL Routing Overview

| Method | URL | Controller | Action |
|--------|-----|------------|--------|
| GET | `/` | WelcomeController | Welcome page |
| GET | `/owners/find` | OwnerController | Search form |
| GET | `/owners` | OwnerController | Paginated list |
| GET | `/owners/{id}` | OwnerController | Owner detail |
| GET/POST | `/owners/new` | OwnerController | Create owner |
| GET/POST | `/owners/{id}/edit` | OwnerController | Edit owner |
| GET/POST | `/owners/{id}/pets/new` | PetController | Add pet |
| GET/POST | `/owners/{id}/pets/{pid}/edit` | PetController | Edit pet |
| GET/POST | `/owners/{id}/pets/{pid}/visits/new` | VisitController | Add visit |
| GET | `/vets.html` | VetController | Vet list (HTML) |
| GET | `/vets` | VetController | Vet list (JSON) |

---

## Testing

### Test Layers

| Annotation | Scope | Example |
|------------|-------|---------|
| `@SpringBootTest` | Full context | `PetClinicIntegrationTests` |
| `@WebMvcTest` | Controller slice | `OwnerControllerTests` |
| `@DataJpaTest` | JPA slice | `ClinicServiceTests` |
| Testcontainers | Real DB | `MySqlIntegrationTests`, `PostgresIntegrationTests` |

### Running Tests

```bash
# All tests
./mvnw test

# Specific class
./mvnw test -Dtest=ClinicServiceTests

# Integration tests (includes verify phase)
./mvnw verify

# With coverage report
./gradlew test jacocoTestReport
```

### Test Data

- Tests use the H2 in-memory database by default
- Data loaded from `src/main/resources/db/h2/data.sql`
- JPA tests are `@Transactional` and auto-rollback

### Notes for Writing Tests

- Use `@WebMvcTest(OwnerController.class)` for controller unit tests — mock repositories with `@MockitoBean`
- Use `@DataJpaTest` for repository tests — gets a real in-memory database
- Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` for full integration tests
- Tests annotated `@DisabledInNativeImage` are skipped during GraalVM native image builds

---

## Code Style & Conventions

### Formatting

- **Java/XML:** 4-space tab indentation
- **HTML/SQL/CSS:** 2-space indentation
- **Gradle:** 2-space indentation
- **Charset:** UTF-8, LF line endings, trailing newline required

Formatting is enforced by the **Spring Java Format** plugin. The build fails if formatting is violated.

```bash
# Check formatting
./mvnw validate

# Apply formatting (Gradle)
./gradlew format
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Packages | `org.springframework.samples.petclinic.*` | `owner`, `vet`, `system` |
| Entities | PascalCase | `Owner`, `PetType` |
| Controllers | `*Controller` | `OwnerController` |
| Repositories | `*Repository` | `OwnerRepository` |
| Methods | camelCase, verb-noun | `findByLastName`, `processCreationForm` |
| Constants | UPPER_SNAKE_CASE | — |
| Variables | camelCase | `firstName`, `birthDate` |

### File Headers

All Java source files must begin with the Apache 2.0 license header:

```java
/*
 * Copyright 2012-2019 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...
 */
```

### Checkstyle

- Uses `src/checkstyle/nohttp-checkstyle.xml`
- Validates no HTTP URLs (all links must use HTTPS)
- Run: `./mvnw validate` (Maven) or `./gradlew check` (Gradle)

---

## Key Configuration

### application.properties (key entries)

```properties
# Database (overridden per profile)
database=h2
spring.sql.init.schema-locations=classpath*:db/${database}/schema.sql
spring.sql.init.data-locations=classpath*:db/${database}/data.sql

# JPA - no DDL auto, snake_case naming
spring.jpa.hibernate.ddl-auto=none
spring.jpa.open-in-view=false
spring.jpa.hibernate.naming.physical-strategy=...PhysicalNamingStrategySnakeCaseImpl
spring.jpa.properties.hibernate.default_batch_fetch_size=16

# Actuator - all endpoints exposed
management.endpoints.web.exposure.include=*

# i18n
spring.messages.basename=messages/messages

# Static resource caching
spring.web.resources.cache.cachecontrol.max-age=12h
```

### JPA Naming

Field names use camelCase in Java (e.g., `firstName`), but the `PhysicalNamingStrategySnakeCaseImpl` maps them to snake_case in the database (e.g., `first_name`). No `@Column` annotations are needed for standard fields.

---

## CI/CD

### GitHub Actions

| Workflow | Trigger | Command |
|----------|---------|---------|
| `maven-build.yml` | push/PR to main | `./mvnw -B verify` |
| `gradle-build.yml` | push/PR to main | `./gradlew build` |
| `deploy-and-test-cluster.yml` | push to main | Kubernetes deployment |

### Commits

This project uses **DCO (Developer Certificate of Origin)** instead of CLA. All commits must include a `Signed-off-by` trailer:

```bash
git commit -s -m "Add feature X"
```

---

## Internationalization

The app supports 9 languages via Spring's message bundle mechanism:

- English (default), German, Spanish, Farsi, Korean, Portuguese, Russian, Turkish

Language files: `src/main/resources/messages/messages_<lang>.properties`

Switch language in UI via `?lang=de` query parameter (configured in `WebConfiguration.java`).

When adding new UI strings, add them to **all** language files. There is a test `I18nPropertiesSyncTest` that verifies all language files have the same keys.

---

## GraalVM Native Image

The app supports AOT compilation to a native binary:

```bash
# Maven
./mvnw -Pnative native:compile

# Gradle
./gradlew nativeCompile
```

Resource patterns and serialization hints are registered in `PetClinicRuntimeHints.java`.

Tests annotated with `@DisabledInNativeImage` are excluded from native image test runs.

---

## Common Development Tasks

### Adding a New Entity

1. Create entity class in appropriate package, extending `BaseEntity`, `NamedEntity`, or `Person`
2. Add JPA annotations (`@Entity`, `@Table`, relationships)
3. Create `*Repository` interface extending `JpaRepository<Entity, Integer>`
4. Add to schema SQL files in `src/main/resources/db/h2/`, `db/mysql/`, `db/postgres/`
5. Add sample data to `data.sql`
6. Create controller and Thymeleaf templates as needed

### Adding a New Controller Route

1. Add method to relevant `*Controller` class
2. Annotate with `@GetMapping` or `@PostMapping`
3. Return view name string (e.g., `"owners/ownerDetails"`) or use `RedirectView`
4. Create corresponding Thymeleaf template in `src/main/resources/templates/`
5. Add `@WebMvcTest` unit test for the new route

### Modifying the Database Schema

- Never use `spring.jpa.hibernate.ddl-auto=update` — schema is managed manually via SQL files
- Update all three schema files: `db/h2/schema.sql`, `db/mysql/schema.sql`, `db/postgres/schema.sql`
- Update `data.sql` files if sample data is affected

---

## What NOT to Do

- Do not add a service layer unnecessarily — keep repositories and entities handling business logic
- Do not use `@RestController` for web UI controllers — use `@Controller`
- Do not add DTOs — domain entities go directly into the view model
- Do not use `spring.jpa.hibernate.ddl-auto=create` or `update` — schema is SQL-managed
- Do not use HTTP URLs in source code — checkstyle will fail (use HTTPS)
- Do not skip code formatting — the build will fail
- Do not add `@Column` annotations for fields that follow the snake_case naming convention
- Do not add `@Service` classes wrapping simple repository calls
