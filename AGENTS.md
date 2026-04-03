# AGENTS.md — FocusGuard Backend

Coding-agent reference for this repository. Read this before making any changes.

---

## Project Overview

Kotlin + Spring Boot 4 REST API. Layered architecture (Controller → Service → Repository → Domain).
Root package: `com.facucastro.focusguard`. JDK 24. Gradle wrapper 9.4.0.

---

## Build & Run Commands

Always use the Gradle wrapper (`./gradlew`), never a system-installed Gradle.

```bash
# Run the application
./gradlew bootRun

# Build (compile + test)
./gradlew build

# Clean build outputs
./gradlew clean

# Run all tests
./gradlew test

# Run a single test class
./gradlew test --tests "com.facucastro.focusguard.service.FocusSessionServiceImplTest"

# Run a single test method (use the full backtick method name)
./gradlew test --tests "com.facucastro.focusguard.service.FocusSessionServiceImplTest.given a valid session dto, when createSession is called, then it returns the saved dto"

# Clean then full build
./gradlew clean build
```

There is no lint or static-analysis task configured (no ktlint, no detekt).

CI runs `./gradlew test` on every push/PR to `main`.

---

## Architecture

```
Controller  →  Service (interface + impl)  →  Repository  →  Domain (JPA Entity)
                            ↕
                     DTO + Mapper (extension functions)
                            ↕
                    GlobalExceptionHandler (@RestControllerAdvice)
```

### Layer rules

- **Controller** — handles HTTP only; works exclusively with DTOs; delegates all logic to the service interface.
- **Service interface** — defines the contract; DTO in, DTO out.
- **Service impl** — only layer allowed to call the mapper; converts DTOs to entities before persisting, converts back before returning.
- **Repository** — plain `JpaRepository` extension; no custom queries yet.
- **Domain** — `@Entity` class with `var` fields (Hibernate requirement). No validation or JSON annotations here.
- **DTO** — `data class` with `val` fields and Bean Validation annotations. This is the API contract.
- **Mapper** — top-level Kotlin extension functions (`toDto()` / `toEntity()`) in `dto/FocusSessionMapper.kt`.

---

## Package Structure

```
src/main/kotlin/com/facucastro/focusguard/
├── FocusguardApplication.kt
├── controller/FocusSessionController.kt
├── service/FocusSessionService.kt          ← interface
├── service/FocusSessionServiceImpl.kt      ← @Service
├── repository/FocusSessionRepository.kt
├── domain/FocusSession.kt                  ← @Entity
├── dto/FocusSessionDto.kt
├── dto/FocusSessionMapper.kt
└── exception/
    ├── SessionNotFoundException.kt
    └── GlobalExceptionHandler.kt

src/test/kotlin/com/facucastro/focusguard/
├── fixtures/FocusSessionFixtures.kt        ← shared test data (Kotlin object)
├── controller/FocusSessionControllerTest.kt
├── dto/FocusSessionMapperTest.kt
├── repository/FocusSessionRepositoryTest.kt
└── service/FocusSessionServiceImplTest.kt
```

Test packages mirror main packages exactly. New source files go in the matching test package.

---

## Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Classes | `PascalCase` | `FocusSessionServiceImpl` |
| Service interface | No `I` prefix | `FocusSessionService` |
| Service implementation | `Impl` suffix | `FocusSessionServiceImpl` |
| DTO | `Dto` suffix (not `DTO`) | `FocusSessionDto` |
| Exception | Descriptive + `Exception` | `SessionNotFoundException` |
| Exception handler | `Handler` suffix | `GlobalExceptionHandler` |
| Test class | Mirrors subject + `Test` | `FocusSessionServiceImplTest` |
| Test fixtures | Kotlin `object` + `Fixtures` | `FocusSessionFixtures` |
| Functions / variables | `camelCase` | `createSession`, `savedEntity` |
| Injected dependencies | Role-based, not type-based | `private val service` (not `focusSessionService`) |
| DB table names | `snake_case`, plural | `focus_sessions` |
| Packages | Lowercase, no underscores | `controller`, `dto`, `exception` |
| Extension functions | Verb form | `toDto()`, `toEntity()` |

---

## Code Style

### Formatting

- **Indentation:** 4 spaces, no tabs.
- **Braces:** opening brace on the same line; closing brace on its own line.
- **Line length:** keep under 120 characters.
- **Expression body** (`=`) for single-expression functions; block body (`{ }`) for multi-statement functions.
- **Method chains:** break before `.` when wrapping:
  ```kotlin
  return ResponseEntity
      .created(URI.create("/sessions/${created.id}"))
      .body(created)
  ```
- **Multiple annotations:** one per line, above the declaration.
- **Constructor parameters:** one per line when there are multiple, indented 4 spaces.

### Kotlin Idioms

- Prefer `val` over `var` everywhere except JPA entity fields (Hibernate requires `var`).
- Use `data class` for DTOs and small value types. Do **not** use `data class` for JPA entities.
- Use named arguments when constructing objects with multiple parameters.
- Use string templates: `"Session with id $id not found"`, `"/sessions/${dto.id}"`.
- Use `map`, `joinToString`, and other stdlib collection functions; avoid explicit loops where idiomatic alternatives exist.
- Use Kotlin extension functions for mapping logic; do not add mapping methods to domain or DTO classes.
- Use `orElseThrow { CustomException(...) }` (lambda form) when unwrapping `Optional`.
- No coroutines — the project is fully synchronous Spring MVC.

### Imports

- No wildcard (`*`) imports.
- Order: project-internal imports first, then third-party, then standard library.
- One import per line.

---

## Dependency Injection

- **Constructor injection** in all production code — never field injection.
- Inject the **interface**, never the concrete implementation.
- `@Autowired` on `lateinit var` is acceptable **only** in test classes.

```kotlin
// Correct
@RestController
class FocusSessionController(
    private val service: FocusSessionService
)

// Wrong — do not do this
@Autowired
lateinit var service: FocusSessionServiceImpl
```

---

## Validation & Error Handling

- Bean Validation on DTO fields uses the explicit `@field:` target prefix:
  ```kotlin
  @field:Positive(message = "id must be positive")
  val id: Long
  ```
- Custom exceptions extend `RuntimeException` directly. Message is built in the superclass call; no extra properties:
  ```kotlin
  class SessionNotFoundException(id: Long) :
      RuntimeException("Session with id $id not found")
  ```
- No `try/catch` in production code — let exceptions propagate to `GlobalExceptionHandler`.
- `GlobalExceptionHandler` maps exceptions to `ErrorResponse(status: Int, message: String)` and the appropriate HTTP status.

---

## Testing

### Test Strategy

| Layer | Annotation | Scope |
|---|---|---|
| Controller | `@WebMvcTest` | Web slice only; service mocked |
| Service impl | Plain unit test | No Spring context; repository mocked |
| Repository | `@DataJpaTest` + `@ActiveProfiles("test")` | JPA + H2 in-memory (test profile); no web layer |
| Mapper | Plain unit test | No Spring context |

### Test Method Naming

Backtick strings following Given/When/Then:
```kotlin
@Test
fun `given a valid session dto, when createSession is called, then it returns the saved dto`() { ... }
```

### Test Body Structure

Always use `// Given`, `// When`, `// Then` comments. Collapse `// When` and `// Then` only for MockMvc calls where stubbing and assertions are in the same expression.

### Fixtures

All shared test data lives in `FocusSessionFixtures` (Kotlin `object`). Add new factory functions there rather than duplicating data across test classes.
```kotlin
val entity = FocusSessionFixtures.defaultEntity()        // override specific fields as needed
val dto    = FocusSessionFixtures.defaultDto(id = 99L)
```

### MockK Patterns

- **Pure unit tests:** create mocks as class-level `private val` properties using `mockk()`.
- **Spring slice tests (`@WebMvcTest`):** provide the mock via an inner `@TestConfiguration` class with a `@Bean` method; `@Autowired` the mock into the test; call `clearMocks(mock)` in `@BeforeEach`.
- Stub with `every { ... } returns ...` or `every { ... } throws ...`.
- Verify with `verify(exactly = 1) { ... }`.

### Assertions

Use `kotlin.test` assertions (`assertEquals`, `assertNotNull`, `assertTrue`, `assertFailsWith`). No AssertJ or Hamcrest.

---

## Configuration Notes

- `application.yaml` (not `.properties`) is the config file format.
- Database: PostgreSQL (`jdbc:postgresql://localhost:5432/focusguarddb`). `ddl-auto: update` — Hibernate updates the schema automatically; data persists between restarts.
- DB credentials are read from environment variables `DB_USERNAME` and `DB_PASSWORD` with fallback defaults (`postgres` / `postgres`) for local development only. Never commit real credentials; always supply them via env vars in production.
- Repository tests use H2 in-memory (profile `test`): no Docker or running PostgreSQL instance required. H2 runs in PostgreSQL compatibility mode (`MODE=PostgreSQL`). When Docker is added to the project, this will be migrated to Testcontainers.
- `show-sql: true` and `format_sql: true` are on — SQL is printed to stdout during development.
- Compiler flags: `-Xjsr305=strict` (strict null safety for JSR-305 annotations), `-Xannotation-default-target=param-property`.
- `allOpen` is configured for `@Entity`, `@MappedSuperclass`, and `@Embeddable` (Kotlin classes are `final` by default; this is required for Hibernate proxying).
