# AGENTS.md — Ticket Management and Notification Service

## 1. Stack

| Technology | Role |
|---|---|
| **Java 21** | Primary backend language for core business logic |
| **Spring Boot 3.x** | Application framework (REST, DI, lifecycle management) |
| **Spring Data JPA** | ORM and repository abstraction over relational database |
| **Spring Security** | Authentication/authorisation integration with user auth system |
| **Spring Mail** | Email notification dispatch |
| **Hibernate** | JPA provider |
| **PostgreSQL** | Primary relational datastore |
| **Flyway** | Database schema versioning and migrations |
| **Node.js 20 LTS** | SMS notification sidecar service runtime |
| **Express 4.x** | HTTP framework for the Node.js notification sidecar |
| **Axios** | HTTP client within the Node.js sidecar for outbound calls |
| **Jest** | Unit and integration testing for Node.js sidecar |
| **JUnit 5** | Unit and integration testing for Spring Boot service |
| **Mockito** | Mocking framework for Java tests |
| **Testcontainers** | Spin up real PostgreSQL instances for integration tests |
| **Maven** | Java build and dependency management |
| **Docker / Docker Compose** | Containerisation and local orchestration |
| **GitHub Actions** | CI pipeline |

---

## 2. Project Structure

```
ticket-management-service/
├── AGENTS.md                          # This file
├── tasks.md                           # Agent-generated task breakdown (created before coding)
├── docker-compose.yml                 # Local orchestration (app + sidecar + postgres)
├── .env.example                       # Environment variable template
├── .github/
│   └── workflows/
│       └── ci.yml                     # GitHub Actions CI pipeline
│
├── ticket-service/                    # Java / Spring Boot module
│   ├── pom.xml                        # Maven project descriptor
│   ├── Dockerfile                     # Multi-stage Java image
│   └── src/
│       ├── main/
│       │   ├── java/com/company/ticketservice/
│       │   │   ├── TicketServiceApplication.java      # Spring Boot entry point
│       │   │   ├── config/
│       │   │   │   ├── SecurityConfig.java            # Spring Security configuration
│       │   │   │   ├── JpaConfig.java                 # JPA/datasource configuration
│       │   │   │   └── MailConfig.java                # JavaMailSender bean configuration
│       │   │   ├── controller/
│       │   │   │   └── TicketController.java          # REST endpoints for ticket CRUD
│       │   │   ├── service/
│       │   │   │   ├── TicketService.java             # Interface defining ticket operations
│       │   │   │   ├── TicketServiceImpl.java         # Business logic implementation
│       │   │   │   └── NotificationDispatcher.java    # Calls sidecar + mail for notifications
│       │   │   ├── repository/
│       │   │   │   └── TicketRepository.java          # Spring Data JPA repository
│       │   │   ├── domain/
│       │   │   │   ├── Ticket.java                    # JPA entity
│       │   │   │   ├── TicketStatus.java              # Enum: OPEN, IN_PROGRESS, RESOLVED, CLOSED
│       │   │   │   └── TicketPriority.java            # Enum: LOW, MEDIUM, HIGH, CRITICAL
│       │   │   ├── dto/
│       │   │   │   ├── TicketCreateRequest.java       # Inbound creation payload
│       │   │   │   ├── TicketUpdateRequest.java       # Inbound update payload
│       │   │   │   └── TicketResponse.java            # Outbound response DTO
│       │   │   ├── mapper/
│       │   │   │   └── TicketMapper.java              # Entity <-> DTO conversion
│       │   │   ├── exception/
│       │   │   │   ├── TicketNotFoundException.java   # 404 domain exception
│       │   │   │   ├── InvalidStatusTransitionException.java  # 422 domain exception
│       │   │   │   └── GlobalExceptionHandler.java    # @RestControllerAdvice handler
│       │   │   └── client/
│       │   │       └── SmsNotificationClient.java     # HTTP client to Node.js sidecar
│       │   └── resources/
│       │       ├── application.yml                    # Base application configuration
│       │       ├── application-local.yml              # Local dev overrides
│       │       ├── application-test.yml               # Test profile configuration
│       │       └── db/migration/
│       │           ├── V1__create_tickets_table.sql   # Initial schema migration
│       │           └── V2__add_priority_column.sql    # Example subsequent migration
│       └── test/
│           └── java/com/company/ticketservice/
│               ├── controller/
│               │   └── TicketControllerTest.java      # MockMvc slice tests
│               ├── service/
│               │   ├── TicketServiceImplTest.java     # Unit tests with Mockito
│               │   └── NotificationDispatcherTest.java
│               ├── repository/
│               │   └── TicketRepositoryTest.java      # @DataJpaTest slice tests
│               └── integration/
│                   └── TicketIntegrationTest.java     # Full context + Testcontainers
│
└── notification-sidecar/              # Node.js / Express SMS sidecar
    ├── package.json
    ├── package-lock.json
    ├── Dockerfile                     # Node.js production image
    ├── .env.example                   # Sidecar env variable template
    ├── jest.config.js                 # Jest configuration
    ├── src/
    │   ├── app.js                     # Express app factory (no listen call)
    │   ├── server.js                  # Entry point: imports app, calls listen
    │   ├── routes/
    │   │   └── notificationRoutes.js  # POST /notify/sms endpoint registration
    │   ├── controllers/
    │   │   └── notificationController.js  # Request validation and response shaping
    │   ├── services/
    │   │   └── smsService.js          # SMS provider integration logic (Twilio/SNS)
    │   ├── middleware/
    │   │   ├── authMiddleware.js      # Validates shared secret / JWT from Java service
    │   │   └── errorHandler.js        # Centralised Express error handler
    │   └── config/
    │       └── index.js               # Reads and validates env vars (throws on missing)
    └── tests/
        ├── unit/
        │   ├── smsService.test.js     # Unit tests for SMS provider calls
        │   └── notificationController.test.js
        └── integration/
            └── notificationRoutes.test.js  # Supertest end-to-end route tests
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder steps.

### Step 1 — Read All Specifications
- Read every spec document provided in the repository before writing any code.
- Identify all ticket lifecycle states, notification triggers, authentication requirements, and external provider contracts.
- Note every ambiguity; resolve it conservatively (strictest interpretation) and log the decision in `tasks.md`.

### Step 2 — Create `tasks.md`
- Create `tasks.md` in the repository root before touching any source file.
- Structure it as a checklist with three sections: **Pending**, **In Progress**, **Done**.
- Each task must map to a single file or a tightly cohesive set of changes.
- Example task format:
  ```
  - [ ] Implement TicketServiceImpl#updateStatus with valid transition guard
  - [ ] Write TicketServiceImplTest covering all invalid transition combinations
  ```
- Move tasks from **Pending** → **In Progress** when work starts, → **Done** when tests pass.

### Step 3 — Scaffold Infrastructure First
1. Create `docker-compose.yml` and both `Dockerfile`s.
2. Create Flyway migration `V1__create_tickets_table.sql`.
3. Verify `docker compose up` starts PostgreSQL and both services without errors before proceeding.

### Step 4 — Implement Domain Layer (Java)
- Create enums `TicketStatus` and `TicketPriority`.
- Create `Ticket` JPA entity with all required fields.
- Create `TicketRepository`.

### Step 5 — Implement Service Layer (Java)
- Implement `TicketServiceImpl` with full status-transition validation.
- Implement `NotificationDispatcher` (email via Spring Mail, SMS via `SmsNotificationClient`).
- Implement `SmsNotificationClient` using `RestTemplate` or `WebClient` targeting the sidecar.

### Step 6 — Implement Controller and Exception Handling (Java)
- Implement `TicketController` with `@Valid` on all request bodies.
- Implement `GlobalExceptionHandler` covering all custom exceptions and `MethodArgumentNotValidException`.

### Step 7 — Implement Notification Sidecar (Node.js)
- Implement `config/index.js` first; it must throw on missing required env vars at startup.
- Implement `smsService.js`, then `notificationController.js`, then `notificationRoutes.js`.
- Wire everything in `app.js`; keep `server.js` as the sole file that calls `app.listen`.

### Step 8 — Write All Tests
- Write unit tests alongside each implementation file (do not batch all tests at the end).
- Write integration tests after all units are complete.
- Run the full test suite and confirm it passes before moving to Step 9.

### Step 9 — Validate Coverage
- Java: run `mvn verify` and confirm JaCoCo reports ≥ 90% line and branch coverage.
- Node.js: run `npm test -- --coverage` and confirm Jest reports ≥ 90% line and branch coverage.
- Fix any gaps before proceeding.

### Step 10 — Final Checklist
- [ ] `docker compose up --build` starts all services cleanly.
- [ ] All migrations run automatically on startup.
- [ ] All tests pass with ≥ 90% coverage.
- [ ] No hardcoded secrets anywhere in source code.
- [ ] `tasks.md` shows all tasks in **Done**.

---

## 4. Coding Conventions

### Java / Spring Boot

- **Package naming:** `com.company.ticketservice.<layer>` — never mix layers in one package.
- **Class naming:** `PascalCase`; suffix by role: `Controller`, `Service`, `ServiceImpl`, `Repository`, `Config`, `Exception`, `Client`, `Mapper`.
- **Method naming:** `camelCase`; use verb-noun form (`createTicket`, `updateStatus`, `dispatchEmailNotification`).
- **Constants:** `UPPER_SNAKE_CASE` in dedicated `static final` fields or a `Constants` class.
- **DTOs:** use Java Records for `TicketCreateRequest`, `TicketUpdateRequest`, and `TicketResponse` — they are immutable by design.
- **Validation:** annotate all DTO fields with Bean Validation (`@NotNull`, `@NotBlank`, `@Size`, etc.); never validate in the service layer what can be validated at the boundary.
- **Entity rules:** `Ticket` must use `UUID` as its primary key (`@GeneratedValue(strategy = GenerationType.UUID)`); never expose the entity directly from a controller.
- **Service interface pattern:** always program to the `TicketService` interface; inject the interface, not `TicketServiceImpl`.
- **Transaction boundaries:** `@Transactional` lives on `ServiceImpl` methods, never on controllers or repositories.
- **Exception handling:** throw domain exceptions from the service layer; catch and translate only in `GlobalExceptionHandler`.
- **Logging:** use SLF4J with `LoggerFactory.getLogger(ClassName.class)`; log at `INFO` for lifecycle events, `WARN` for recoverable issues, `ERROR` for unhandled failures. Never log sensitive data.
- **Configuration:** all tunable values go in `application.yml`; inject with `@ConfigurationProperties` beans, not raw `@Value` fields.
- **No Lombok:** write explicit constructors, getters, and builders — reduces magic and improves readability for agents.

### Node.js / Express

- **File naming:** `camelCase.js` for all source files; `PascalCase` only for class-based modules (avoid classes; prefer plain functions and modules).
- **Exports:** use named exports (`module.exports = { functionName }`); avoid default exports for testability.
- **Async style:** `async/await` exclusively; no raw Promise chains or callbacks.
- **Error propagation:** always `throw` errors from `smsService.js`; let `errorHandler.js` middleware catch and format the response.
- **Config validation:** `config/index.js` must call `process.exit(1)` with a clear message if any required env var is absent.
- **No global state:** do not attach state to the `app` object; pass dependencies explicitly.
- **Route handlers:** keep controllers thin — validate input, call service, return response. No business logic in controllers.
- **HTTP status codes:** `201` for created resources, `200` for success, `400` for validation errors, `401`/`403` for auth failures, `500` for unhandled errors.

### Shared

- **No magic numbers:** every numeric constant must be named.
- **Secrets:** read exclusively from environment variables; `.env.example` documents all required keys with placeholder values.
- **Git commits:** one logical change per commit; message format `type(scope): description` (e.g., `feat(ticket): add priority escalation logic`).

---

## 5. Testing

### Java Testing Strategy

| Layer | Tool | Annotation / Approach |
|---|---|---|
| Unit — Service | JUnit 5 + Mockito | Plain `@ExtendWith(MockitoExtension.class)` |
| Unit — Controller | JUnit 5 + MockMvc | `@WebMvcTest(TicketController.class)` |
| Unit — Repository | JUnit 5 + H2 / DataJpaTest | `@DataJpaTest` |
| Integration | JUnit 5 + Testcontainers | `@SpringBootTest` + `PostgreSQLContainer` |

**Rules:**
- Every public method in `TicketServiceImpl` must have a corresponding test covering the happy path and all error/edge paths.
- Every invalid status transition must be asserted to throw `InvalidStatusTransitionException`.
- `NotificationDispatcher` tests must mock `JavaMailSender` and `SmsNotificationClient` — never send real email or SMS in tests.
- Use `@ActiveProfiles("test")` and `application-test.yml` to prevent test code from touching real infrastructure.
- JaCoCo configuration goes in `pom.xml` under the `verify` phase; fail the build if line coverage < 90% or branch coverage < 90%.

**Running Java tests:**
```bash
cd ticket-service
mvn test                  # unit tests only
mvn verify                # unit + integration + coverage report
# Report location: ticket-service/target/site/jacoco/index.html
```

### Node.js Testing Strategy

| Layer | Tool | Approach |
|---|---|---|
| Unit — Service | Jest | Mock SMS provider SDK with `jest.mock()` |
| Unit — Controller | Jest | Mock `smsService` module |
| Integration — Routes | Jest + Supertest | Start real Express app; mock only external HTTP |

**Rules:**
- `smsService.js` must be tested with the SMS provider SDK fully mocked — no real network calls.
- Integration route tests must set the required auth header; assert `401` when it is absent.
- Use `beforeEach` / `afterEach` to reset mocks; never share mutable state between tests.
- Coverage thresholds in `jest.config.js`:
  ```js
  coverageThreshold: {
    global: { lines: 90, branches: 90, functions: 90, statements: 90 }
  }
  ```

**Running Node.js tests