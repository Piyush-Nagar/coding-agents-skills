# Package Structure

Our backend services follow **Package by Feature** organized across four sibling modules. This document is the source of truth for where new code goes.

> **Language note.** The examples below are written in Java/Spring Boot because that is our primary backend stack, but **the structure itself is language-agnostic and mandatory across every backend language we use** (Kotlin, Go, Python, Node, etc.). Translate the file extensions and framework annotations; keep the folder layout, naming suffixes (`*Db`, `*Repo`, `*RepoImpl`, `*Dao`, `*Controller`, `*ControllerImpl`), and the interface/impl split unchanged.

## Core principle

**Group by feature, not by technical layer.** A change to the "orders" feature should touch files that live next to each other, not files scattered across `controllers/`, `services/`, `repositories/` packages. Technical layering happens at the *module* boundary (`db`, `client`, `capabilities`, `starter`), and **package-by-feature happens inside each module**.

## The four modules

Every service is composed of these four sibling modules:

```
<service>/
├── capabilities/   # feature/business logic — the brain
├── db/             # persistence: entities, repositories, migrations
├── client/        # outbound integrations with external systems
└── starter/        # Spring Boot starter — wires everything and runs
```

### `capabilities/` — business logic

Each feature gets its own package, **and inside each feature there is one sub-package per API endpoint** (`getById`, `create`, `updateById`, `getByQuery`, …). Each sub-feature owns three sibling folders:

- `service/` — the single service class for that API
- `dto/` — `request/` and `response/` (whichever applies)
- `mapper/` — translates `*Db` entities ↔ DTOs (both directions) for this sub-feature only

The **controller for the feature lives at the root of the feature package**, split into an interface and an implementation just like repositories:

- `capabilities/<feature>/<Feature>Controller.java` — interface; declares routes, HTTP verbs, request/response types, API docs annotations
- `capabilities/<feature>/<Feature>ControllerImpl.java` — implementation; carries the framework annotation (`@RestController`) and delegates each route to the matching sub-feature service

Shared across all of that feature's sub-features. See [Controllers](#controllers).

- Depends on: `db` (only via `*Repo` interfaces — see [DB usage rule](#db-usage-rule)), `client`
- Never depends on: `starter`
- Contains no `main` class, no `@SpringBootApplication`
- Owns: HTTP controllers, API request/response DTOs, mappers, services

### `db/` — persistence

Organized **one folder per table**. Each table folder has exactly three sub-folders:

```
db/<tableName>/
├── model/
│   ├── <EntityName>Db.java          # main JPA entity
│   └── <Other>Db.java               # sub-models (embedded types, child entities, projections)
├── dao/
│   └── <EntityName>Dao.java         # low-level data access (Criteria/JPQL/native queries)
└── repository/
    ├── <EntityName>Repo.java        # PUBLIC interface — the only type capabilities sees
    └── impl/
        └── <EntityName>RepoImpl.java # composes Spring Data + Dao behind the interface
```

#### Naming convention

| Layer | Suffix | Example |
|---|---|---|
| JPA entity | `Db` | `TestcaseDb` |
| Repository interface | `Repo` | `TestcaseRepo` |
| Repository implementation | `RepoImpl` | `TestcaseRepoImpl` |
| Data access helper | `Dao` | `TestcaseDao` |

#### DB usage rule

**Capabilities must depend on `*Repo` interfaces only.** `*RepoImpl` and `*Dao` are internal — never injected or referenced from outside the `db/<tableName>/` package. This is what keeps the persistence layer swappable and capability tests focused on the contract.

- Depends on: nothing in this service
- Exposes: `*Repo` interfaces and `*Db` model types
- Never contains: business logic, transformation to API DTOs

### `client/` — outbound integrations

Holds the code that calls **other services this service depends on**. Primarily one sub-folder per **dependent microservice** in our own platform (`paymentService`, `userService`, `executionService`, …). Third-party SDK wrappers or message producers may live here too, but the dominant pattern — and the reason this module exists — is dependent-microservice clients.

```
client/<microserviceName>/
├── <Microservice>Client.java       # interface — what capabilities depends on
├── impl/
│   └── <Microservice>ClientImpl.java   # impl: HTTP/gRPC/SDK call, retries, timeouts
├── dto/                            # request/response shapes for THIS microservice's API
│   ├── request/
│   └── response/
└── mapper/                         # optional: translate between their shapes and ours
```

#### Rules

- **One sub-folder per dependent microservice**, named after the service. Do not lump multiple microservices into one package even if they are "related."
- **Capabilities depend on `*Client` interfaces only.** `*ClientImpl` is internal — same rule as `*Repo` / `*RepoImpl`.
- The client interface exposes the microservice's API in terms of its own DTOs; capabilities can adapt those further if needed.
- Cross-cutting concerns (retries, circuit breaking, request/response logging, auth headers) belong inside `impl/` — capabilities should never see them.

- Depends on: nothing in this service
- Exposes: `*Client` interfaces and their request/response DTOs
- Never contains: business logic, capability orchestration

### `starter/` — composition root

The only module that runs. Contains `@SpringBootApplication`, global REST exception handlers, security config, filters, and `application.yml`. **No controllers** — those live in `capabilities/`.

- Depends on: `capabilities` (and transitively `db`, `client`)
- Owns: the deployable artifact, cross-cutting web config (security, CORS, error handling)
- Never owns: feature controllers or feature DTOs

## Dependency rule

```
starter → capabilities → { db, client }
```

Arrows go one way. `db` and `client` never import from `capabilities` or `starter`. `capabilities` never imports from `starter`.

## Controllers

There is **one controller per feature**, not per sub-feature, and it lives **at the root of the feature package inside `capabilities/`**. It is split into two files, mirroring the `*Repo` / `*RepoImpl` split:

- `capabilities/<feature>/<Feature>Controller.java` — **interface**. Declares the routes, HTTP verbs, path/query params, request/response DTO types, and API doc annotations (OpenAPI/Swagger). Anyone reading the interface should see the whole HTTP contract.
- `capabilities/<feature>/<Feature>ControllerImpl.java` — **implementation**. Carries the framework annotation (`@RestController` in Spring), is the injection target for sub-feature services, and contains one short method per route that calls the matching `service/` and returns the result. No logic beyond delegation.

Why the split? The interface is the API contract — stable, documented, testable in isolation. The impl is wiring — it can change without changing the contract. Same reasoning as `*Repo` vs `*RepoImpl`.

The controller delegates each HTTP route to the matching sub-feature service in `capabilities/<feature>/<sub-feature>/service/`. Request/response DTOs are owned by the sub-feature (`capabilities/<feature>/<sub-feature>/dto/`); the controller is a thin HTTP adapter only.

Why inside `capabilities/` and not `starter/`? The controller, its service, and its DTOs all change together when an API changes — keeping them in one module preserves the package-by-feature promise of "one feature, one place." `starter/` is reserved for cross-cutting wiring (security, error handlers, app entry point), not per-feature web code.

## Worked example: a `testcase` feature

The tree below uses `.java` extensions for illustration. Substitute the file extension and framework idioms of your language — Kotlin `.kt`, Go packages, Python modules, etc. — but **keep the folder layout and naming suffixes identical**.

```
testcase-service/
├── capabilities/
│   └── com/example/testcase/capabilities/
│       └── testcase/                          # main feature
│           ├── TestcaseController.java        # interface — HTTP contract, route declarations, API docs
│           ├── TestcaseControllerImpl.java    # impl — @RestController, delegates to sub-feature services
│           ├── getById/                       # sub-feature = one API
│           │   ├── service/
│           │   │   └── GetTestcaseByIdService.java
│           │   ├── dto/
│           │   │   └── response/
│           │   │       └── GetTestcaseByIdResponse.java
│           │   └── mapper/
│           │       └── GetTestcaseByIdMapper.java     # TestcaseDb → response
│           ├── create/
│           │   ├── service/
│           │   │   └── CreateTestcaseService.java
│           │   ├── dto/
│           │   │   ├── request/
│           │   │   │   └── CreateTestcaseRequest.java
│           │   │   └── response/
│           │   │       └── CreateTestcaseResponse.java
│           │   └── mapper/
│           │       └── CreateTestcaseMapper.java      # request → TestcaseDb, TestcaseDb → response
│           ├── updateById/
│           │   ├── service/
│           │   │   └── UpdateTestcaseByIdService.java
│           │   ├── dto/
│           │   │   ├── request/
│           │   │   │   └── UpdateTestcaseByIdRequest.java
│           │   │   └── response/
│           │   │       └── UpdateTestcaseByIdResponse.java
│           │   └── mapper/
│           │       └── UpdateTestcaseByIdMapper.java
│           └── getByQuery/
│               ├── service/
│               │   └── GetTestcaseByQueryService.java
│               ├── dto/
│               │   ├── request/
│               │   │   └── GetTestcaseByQueryRequest.java
│               │   └── response/
│               │       └── GetTestcaseByQueryResponse.java
│               └── mapper/
│                   └── GetTestcaseByQueryMapper.java
│
├── db/                                        # one folder per table
│   └── com/example/testcase/db/
│       └── testcase/                          # table name
│           ├── model/
│           │   ├── TestcaseDb.java            # JPA entity — main
│           │   └── TestcaseStepDb.java        # sub-model (child/embedded)
│           ├── dao/
│           │   └── TestcaseDao.java           # query/criteria helper, package-private use
│           └── repository/
│               ├── TestcaseRepo.java          # interface — capabilities depends on THIS
│               └── impl/
│                   └── TestcaseRepoImpl.java  # composes Spring Data + TestcaseDao
│
├── client/                                   # one folder per dependent microservice
│   └── com/example/testcase/client/
│       ├── executionService/                 # dependent microservice we call
│       │   ├── ExecutionServiceClient.java        # interface
│       │   ├── impl/
│       │   │   └── ExecutionServiceClientImpl.java  # HTTP/gRPC call, retries, timeouts
│       │   ├── dto/
│       │   │   ├── request/
│       │   │   │   └── RunTestcaseRequest.java
│       │   │   └── response/
│       │   │       └── RunTestcaseResponse.java
│       │   └── mapper/
│       │       └── ExecutionServiceMapper.java
│       └── userService/                      # another dependent microservice
│           ├── UserServiceClient.java
│           └── impl/
│               └── UserServiceClientImpl.java
│
└── starter/                                  # cross-cutting wiring only — no controllers
    └── com/example/testcase/
        ├── TestcaseServiceApplication.java
        ├── config/
        │   ├── SecurityConfig.java
        │   └── WebConfig.java
        ├── error/
        │   └── GlobalExceptionHandler.java   # @ControllerAdvice — cross-cutting, not per feature
        └── resources/application.yml
```

## Rules of thumb

- **Adding a new API to an existing feature?** Create a new sub-feature package inside `capabilities/<feature>/` with its own `service/`, `dto/{request,response}/`, and `mapper/`, then add the route declaration to `<Feature>Controller.java` (interface) and the delegating method to `<Feature>ControllerImpl.java`. Do not create a second controller.
- **Adding a brand new feature?** Create the feature package in `capabilities/` (controller at root, sub-features per API), and the matching table folder in `db/<tableName>/` with `model/`, `dao/`, `repository/{impl/}`. Add a client only if you call an external system. Do not touch `starter/`.
- **Always inject `*Repo`, never `*RepoImpl` or `*Dao`.** Capabilities depend on the interface only. `*RepoImpl` and `*Dao` are internal to `db/<tableName>/` — they exist to let you swap query strategy or backing store without changing capability code.
- **Always inject `*Client`, never `*ClientImpl`.** Same rule for dependent-microservice calls. Capabilities depend on the interface; the transport (HTTP, gRPC, SDK) lives behind `impl/`.
- **One folder per dependent microservice inside `client/`.** Name the folder after the microservice. Do not group "all internal services" into one package.
- **Sub-feature granularity stops at `capabilities/`.** `db/` is organized per table; `client/` per external system. Splitting either per API would scatter cohesive code.
- **Request/response DTOs and their mapper are per-sub-feature, never shared.** If `create` and `updateById` look similar today, keep two DTOs and two mappers. Sharing couples two APIs that will drift independently.
- **Mappers go entity ↔ DTO directly.** No intermediate domain model layer. `<SubFeature>Mapper` converts `*Db` ↔ that sub-feature's request/response DTOs.
- **Three sub-features have similar code?** Resist the urge to extract a shared service or mapper until you have the same pattern in *four or more* places.
- **Cross-feature reuse?** Put shared utilities in a `common/` package inside the relevant module. Never reach across features.
- **Two data shapes, one boundary:** JPA entities (`*Db`) in `db/<table>/model/`, API DTOs in `capabilities/<feature>/<sub-feature>/dto/`. Mapping happens in the sub-feature `mapper/`. This separation is non-negotiable.
- **No business logic in controllers.** Controllers in `capabilities/<feature>/` translate HTTP ↔ sub-feature service call only.
- **No JPA annotations outside `db/<table>/model/`.** If you see `@Entity` on a DTO, that's a bug.

## Anti-patterns

- A `services/`, `dto/`, `mapper/`, or `controllers/` package at the top of any module — that's package-by-layer leaking back in.
- A controller in `starter/` — controllers belong at the root of their feature inside `capabilities/`.
- A `*Controller` class that is not an interface, or a `*ControllerImpl` without a matching interface — the interface/impl split is mandatory.
- The framework annotation (`@RestController`, etc.) on the interface instead of the impl — interface is the contract, impl carries the wiring.
- One controller per sub-feature — controllers are shared at the feature level.
- A single `XxxService` handling every API of a feature — split per sub-feature (`GetXxxByIdService`, `CreateXxxService`, …).
- Sharing a `XxxRequest` DTO or mapper between `create` and `update` sub-features — they will drift; keep them separate.
- **Capability code injecting `XxxRepoImpl`, `XxxDao`, or `XxxClientImpl` directly** — always go through the `XxxRepo` / `XxxClient` interface. The impl, dao, and concrete transport are internal.
- A single `client/integrations/` or `client/external/` folder holding multiple microservices' code — every dependent microservice gets its own named folder.
- Putting a JPA entity (`*Db`) directly into a DTO field, or returning it from a controller — go through the sub-feature mapper.
- Adding a feature-level `model/` package in `capabilities/<feature>/` to "share types between sub-features" — sub-features should each define their own DTOs and translate via mapper.
- `capabilities` importing from `starter` — composition flows one way.
- A "shared" module that everything depends on — prefer duplication until a pattern is proven.
