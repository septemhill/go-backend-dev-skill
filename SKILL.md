---
name: Golang Backend Development
description: Architectural standards and coding practices for the Go backend.
---

# Golang Backend Development Standards

This skill defines the architectural requirements, coding standards, and best practices for the Golang backend. AI agents must adhere to these guidelines to ensure consistency, maintainability, and extensibility.

## 1. Project Organization

- `backend/cmd/`: Application entry points. Keep `main.go` slim; use it for configuration and component initialization.
- `backend/internal/`: Project-internal code.
    - `handler/`: HTTP layer. Uses `Handler` generic wrapper. Defers logic to `service`.
    - `service/`: Domain logic layer. Orchestrates business rules and repository calls.
    - `repository/`: Data layer. Handles raw SQL queries via `pgxpool`.
    - `models/`: Data structures (DB models, DTOs, request/response types).
- `backend/pkg/`: Shared utility packages (e.g., custom `statemachine`).
- `backend/migrations/`: SQL migration scripts.

## 2. Core Architectural Patterns

### Layered Responsibility
- **Handlers** must NOT contain business logic. They decode requests, call services, and encode responses.
- **Services** are the source of truth for business rules. They must be agnostic to the delivery method (HTTP).
- **Repositories** bridge the domain models and the database.
- **Interface Design**: Keep interfaces small with minimal methods to provide maximum flexibility for implementers.
- **Return Structs, Accept Interfaces**: Functions should return concrete types even if they implement an interface. Parameters should be interfaces to reduce coupling.

### The `Handler` Wrapper
Every HTTP endpoint should be wrapped using the `Handler[REQ, RESP]` found in `internal/handler/base.go`. This ensures consistent:
- Request decoding (`HandleRequestFunc`, `DefaultDecoder`)
- Error handling (`DefaultErrorHandler`)
- Response encoding (`HandleResponseFunc`, `DefaultEncoder`)

### Dependency Injection
Dependencies must be injected via constructors:
```go
repo := repository.NewRepository(dbPool)
svc := service.NewService(repo, config.JWTSecret)
h := handler.NewAuthHandler(svc.Auth)
```

## 3. Coding Standards & Tooling

### Standard Library First
- Always prioritize using the Go standard library to implement functionality unless strictly necessary. Keep external dependencies to a minimum.

### Context First
- Pass `context.Context` to all `service` and `repository` methods to support cancellation and timeouts.

### Database: Raw SQL with `pgx` or `sqlx`
- Prefer raw SQL over ORMs for performance and transparency.
- Use `github.com/jackc/pgx/v5` and `pgxpool`, or `github.com/jmoiron/sqlx`.
- Scan results into structs defined in `internal/models`.

### HTTP Middleware
Apply cross-cutting concerns via middleware:
- `LoggingMiddleware`: Records full request/response details for debugging.
- `AuthMiddleware`: Validates JWT and extracts `userID`.
- `CORSMiddleware`: Standard CORS headers.

### Design Principles
- **KISS (Keep It Simple and Stupid)**: Prioritize simplicity over complexity. Avoid over-engineering.
- **SOLID**: Follow SOLID principles to ensure maintainable and scalable code, but always prioritize KISS.

### Testing & Coverage
- All service, repository, and package implementations must have test case coverage.
- **Mocking**: Use the [mockery](https://github.com/vektra/mockery) library to generate mock files for interfaces. This facilitates consistent and easy unit testing.
- Prioritize testability in all implementations.

### 4. Code Quality & Linting
- **Configuration**: If `.golangci.yml` does not exist in the project root, create one using the reference configuration from [golangci-lint reference](https://raw.githubusercontent.com/golangci/golangci-lint/master/.golangci.reference.yml).
- **Enforcement**: Run `golangci-lint run ./...` on the entire project after making changes or fixing issues to ensure code quality and consistency.

## 4. Error Handling & Return Values
- Return clear, typed errors from services (e.g., `ErrUnauthorized`).
- Use `uuid.UUID` for all resource identifiers.
- Ensure all JSON-serialized fields have appropriate `json` tags.

## 5. Extensibility Goal
- When implementing new features, consider how side effects (notifications, logs, state transitions) can be hooked into the existing flow without tightly coupling the core business logic.
