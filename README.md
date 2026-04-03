# FocusGuard Backend

> REST API backend for FocusGuard — the context-aware productivity timer for Android.

[![Kotlin](https://img.shields.io/badge/Kotlin-2.x-purple.svg)](https://kotlinlang.org)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.x-green.svg)](https://spring.io/projects/spring-boot)
[![JDK](https://img.shields.io/badge/JDK-24-blue.svg)](https://openjdk.org)
[![PostgreSQL](https://img.shields.io/badge/DB-PostgreSQL-316192.svg)](https://www.postgresql.org)

FocusGuard Backend is the server-side counterpart to the [FocusGuard Android app](https://github.com/castrofacu/FocusGuard). It is built with Kotlin and Spring Boot and exposes a REST API for storing and retrieving focus sessions. The project is in its early stages — currently open to unauthenticated requests — with a clear roadmap toward a production-ready service.

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [API Reference](#api-reference)
- [Getting Started](#getting-started)
- [Running Tests](#running-tests)
- [Key Design Decisions](#key-design-decisions)
- [Roadmap](#roadmap)

---

## Features

- **Session Storage** — Create and retrieve focus sessions via a clean REST API
- **Input Validation** — Bean Validation on all incoming request bodies with structured error responses
- **Centralized Error Handling** — `@RestControllerAdvice` maps exceptions to consistent `{ status, message }` responses
- **PostgreSQL** — Persistent relational database; data survives application restarts
- **Layered Test Suite** — Slice tests (`@WebMvcTest`, `@DataJpaTest`) and pure unit tests with MockK cover each layer independently

---

## Architecture

The project follows a classic **Layered (N-Tier) Architecture** with a strict top-down dependency flow and **Spring Data JPA** for the repository pattern.

```
┌──────────────────────────────────────────────────┐
│                   Controller                     │
│         FocusSessionController (/sessions)       │
└────────────────────┬─────────────────────────────┘
                     │ Service Interface
┌────────────────────▼─────────────────────────────┐
│                    Service                       │
│   FocusSessionService (i/f)                      │
│   FocusSessionServiceImpl                        │
└────────────────────┬─────────────────────────────┘
                     │ Repository
┌────────────────────▼─────────────────────────────┐
│                  Repository                      │
│   FocusSessionRepository (JpaRepository)         │
└────────────────────┬─────────────────────────────┘
                     │ Entity
┌────────────────────▼─────────────────────────────┐
│                   Domain                         │
│   FocusSession (@Entity → focus_sessions table)  │
└──────────────────────────────────────────────────┘
           ↕ DTO + Mapper (FocusSessionDto / toDto / toEntity)
           ↕ Exception Handling (GlobalExceptionHandler)
```

### Package Structure

```
src/main/kotlin
└── com.facucastro.focusguard
    ├── FocusguardApplication.kt          ← Spring Boot entry point
    ├── controller/
    │   └── FocusSessionController.kt     ← REST endpoints
    ├── service/
    │   ├── FocusSessionService.kt        ← Service interface
    │   └── FocusSessionServiceImpl.kt    ← Business logic
    ├── repository/
    │   └── FocusSessionRepository.kt     ← Spring Data JPA
    ├── domain/
    │   └── FocusSession.kt               ← JPA entity
    ├── dto/
    │   ├── FocusSessionDto.kt            ← API contract + validation
    │   └── FocusSessionMapper.kt         ← toDto() / toEntity() extensions
    └── exception/
        ├── SessionNotFoundException.kt   ← Custom 404 exception
        └── GlobalExceptionHandler.kt     ← Unified error responses

src/test/kotlin
└── com.facucastro.focusguard
    ├── fixtures/FocusSessionFixtures.kt          ← Shared test data
    ├── controller/FocusSessionControllerTest.kt  ← @WebMvcTest + MockMvc
    ├── dto/FocusSessionMapperTest.kt             ← Pure unit tests
    ├── repository/FocusSessionRepositoryTest.kt  ← @DataJpaTest + H2
    └── service/FocusSessionServiceImplTest.kt    ← Pure unit tests + MockK
```

---

## Tech Stack

| Category | Technology |
|----------|------------|
| Language | Kotlin 2.x |
| Framework | Spring Boot 4.x |
| JDK | Java 24 |
| Web | Spring MVC (`spring-boot-starter-webmvc`) |
| Persistence | Spring Data JPA + Hibernate |
| Database | PostgreSQL |
| Validation | Jakarta Bean Validation (`spring-boot-starter-validation`) |
| Serialization | Jackson Kotlin Module |
| Testing | JUnit 5, MockK, `@WebMvcTest`, `@DataJpaTest`, H2 (test scope only) |

---

## API Reference

Base URL: `http://localhost:8080`

All request and response bodies are JSON. Errors follow the format:
```json
{ "status": <http_code>, "message": "<description>" }
```

### Create a session

```
POST /sessions
```

**Request body:**

| Field | Type | Constraints |
|-------|------|-------------|
| `id` | `Long` | Positive |
| `startTime` | `Long` | Positive (Unix epoch ms) |
| `durationSeconds` | `Int` | Minimum 1 |
| `distractionCount` | `Int` | Minimum 0 |

**Example:**
```json
{
  "id": 1,
  "startTime": 1700000000000,
  "durationSeconds": 1500,
  "distractionCount": 2
}
```

**Responses:**
- `201 Created` — session created; `Location: /sessions/{id}` header included
- `400 Bad Request` — validation failed

---

### Get all sessions

```
GET /sessions
```

**Responses:**
- `200 OK` — array of session objects (may be empty)

---

### Get session by ID

```
GET /sessions/{id}
```

**Responses:**
- `200 OK` — session object
- `404 Not Found` — session does not exist

---

## Getting Started

### Prerequisites

- JDK 24
- PostgreSQL instance (local install or Docker)
- Gradle (or use the included `./gradlew` wrapper)

### 1. Clone the repository

```bash
git clone https://github.com/facucastro/FocusGuardBackend.git
cd FocusGuardBackend
```

### 2. Create the database

```bash
psql -U postgres -c "CREATE DATABASE focusguarddb;"
```

Or with Docker:

```bash
docker run --name focusguard-db \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  -d postgres:17
```

### 3. Configure credentials

The application reads database credentials from environment variables. Set them before running:

```bash
export DB_USERNAME=postgres
export DB_PASSWORD=yourpassword
```

If the variables are not set, the application falls back to `postgres` / `postgres` — suitable for local development only.

### 4. Run the application

```bash
./gradlew bootRun
```

The server starts on `http://localhost:8080`. On first run, Hibernate automatically creates the `focus_sessions` table (`ddl-auto: update`).

---

## Running Tests

```bash
./gradlew test
```

Tests run without any external infrastructure. Repository tests use an in-memory H2 database (PostgreSQL compatibility mode) so no running PostgreSQL instance or Docker is required.

The test suite covers each layer independently:

| Test class | Strategy | Scope |
|---|---|---|
| `FocusSessionControllerTest` | `@WebMvcTest` + MockK | Web layer only |
| `FocusSessionServiceImplTest` | Pure unit test + MockK | Service only |
| `FocusSessionRepositoryTest` | `@DataJpaTest` + H2 (profile `test`) | JPA layer only |
| `FocusSessionMapperTest` | Pure unit test | DTO mapping |

---

## Key Design Decisions

**Interface-based service layer**  
`FocusSessionService` is an interface, with `FocusSessionServiceImpl` as its implementation. This decouples the web layer from business logic and makes the service trivially mockable in `@WebMvcTest` slices without loading the full application context.

**DTO mapping via Kotlin extension functions**  
Rather than introducing a mapping library, `FocusSessionMapper.kt` provides `toDto()` and `toEntity()` as idiomatic Kotlin extension functions. This keeps the mapping close to the data types while remaining easy to test in isolation.

**Client-provided IDs**  
`FocusSession` uses a client-supplied `Long` as its primary key (no auto-generation). This lets the Android app assign IDs locally (e.g. from Room) and sync them to the backend without a round-trip to discover the server-generated ID.

**Slice tests over full integration tests**  
Each layer is tested in isolation using Spring's test slices (`@WebMvcTest`, `@DataJpaTest`) and pure unit tests with MockK. This keeps tests fast and failures easy to localize.

**Credentials via environment variables**  
Database credentials are never hardcoded. `application.yaml` uses `${DB_USERNAME:postgres}` / `${DB_PASSWORD:postgres}` — the value after `:` is a local dev fallback only. Production deployments must supply the real values through environment variables.

---

## Roadmap

- [ ] Add authentication and authorization (Spring Security + JWT or Firebase token verification)
- [ ] Enforce per-user data isolation — sessions scoped to the authenticated user
- [ ] Add pagination and filtering to `GET /sessions`
- [ ] Add endpoint to delete or update a session
- [ ] Introduce Docker + `docker-compose` for local development
- [ ] Migrate repository tests from H2 to Testcontainers once Docker is available
- [ ] Add request logging and structured application logging
