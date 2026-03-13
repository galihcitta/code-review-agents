# Go Review Standards

Industry-standard checklist for reviewing Go codebases. Each section maps to a scoring category. Items marked **REJECT** are critical issues that warrant MR rejection. Items marked **WARNING** should be flagged but don't block merge.

---

## 1. Architecture & Design

- DDD / Clean Code architecture: proper layer separation (handlers → usecases/services → repository)
- Single Responsibility Principle per file/package
- No business logic in handlers — delegate to service/usecase layer
- No data access in handlers — delegate to repository layer
- Dependencies initialized in one place and injected via constructor
- No circular dependencies between packages
- Configuration separated from business logic (12-factor)
- Consistent error propagation across layers (wrap with context, don't swallow)

---

## 2. Language Standards

- All layers MUST communicate through interfaces — **REJECT** if concrete types at layer boundaries
- `context.Context` as first parameter for all I/O functions — **REJECT** if missing in handlers/services
- Error wrapping with `fmt.Errorf("context: %w", err)` — never ignore errors
- No `panic()` in HTTP handlers or library code — **REJECT**
- No ignored errors (using `_` to discard error returns) — **REJECT**
- Goroutine safety: shared state protected with mutex/channels/sync.Map
- No unprotected shared map access — **REJECT** (data race)
- Proper struct tags (`json`, `validate`) on all API types
- Follow Effective Go and Uber Go Style Guide
- Use `errors.Is`/`errors.As` for error checking, not string comparison

**REJECT if:** panic in handlers, ignored errors, no interfaces at boundaries, data races

---

## 3. HTTP Standards

- All routes MUST start with "/" — **REJECT** if missing
- RESTful naming: nouns not verbs in URLs
- pprof endpoints MANDATORY — **REJECT** if missing:
  - `/profiler/pprof/*` endpoints plus `/profiler/metrics`
- All dependencies initialized in ONE file via constructor injection
- Proper HTTP status codes
- PUT MUST be idempotent (no increment/append operations) — **REJECT** if not
- GET MUST be read-only — **WARNING** if modifying data
- POST/PATCH SHOULD use `Idempotency-Key` header — **WARNING** if missing
- DELETE must return same result on repeat calls
- Graceful shutdown with signal handling (SIGTERM/SIGINT)
- Allowed frameworks: net/http, gorilla/mux, chi, gin, echo

**REJECT if:** routes without "/", missing pprof, non-idempotent PUT
**WARNING if:** GET modifies data, POST/PATCH without Idempotency-Key

---

## 4. Database Standards

- All queries MUST use `?` placeholders — **REJECT** raw SQL with string concatenation
- `SELECT *` is PROHIBITED (exception: `COUNT(*)`) — **REJECT**
- Explicit column selection with `Scan()`
- `rows.Close()` with `defer` immediately after query — **REJECT** if missing
- `rows.Err()` must be checked after iteration
- Health check endpoint must test all DB/cache/queue dependencies
- Slow query threshold: 10 seconds
- Migration tools: golang-migrate, goose, or atlas
- Migrations must be additive only; no destructive changes in same MR
- Connection pool configured: `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`
- Transaction boundaries where multiple related writes occur

**REJECT if:** SQL injection risk, SELECT *, missing rows.Close()

---

## 5. Security Standards

- Use whitelist (explicit allowed values map) not blacklist for validation — **REJECT** blacklist-only
- Input validation and sanitization at handler boundary using struct tags + validator
- No hardcoded secrets, API keys, or passwords in source code
- OWASP top 10: SQL injection, XSS, SSRF, path traversal, command injection
- `defer resp.Body.Close()` after all HTTP client calls — **REJECT** if missing
- Proper authentication/authorization checks on all endpoints
- Missing rate limiting on sensitive endpoints
- HTTP client timeout MUST be set (default: 30 seconds) — no infinite timeouts
- Repository/HTTP clients MUST implement retry with exponential backoff
  - Retriable status codes: 408, 429, 500, 502, 503, 504

**REJECT if:** SQL injection, hardcoded secrets, blacklist-only, missing resp.Body.Close()

---

## 6. Testing Standards

- Unit tests MUST be present for new/changed code — **REJECT** if missing
- Coverage target: 100% for new code
- Table-driven tests MANDATORY for multi-scenario coverage
- Integration tests with `httptest` or testcontainers covering happy paths
- `go test -race -count=1` MANDATORY in CI — **REJECT** if race detection skipped
- Test file naming: `*_test.go`
- Mocks via testify/mock or mockgen
- Edge case coverage: nil, empty, boundary values
- No test code in production files

**REJECT if:** no unit tests, <50% coverage, race detection disabled

---

## 7. Monitoring Standards

- APM tool MUST be registered for all services with distributed tracing enabled
- Request-ID / trace-ID MUST be in all log statements
  - Middleware: falls back to X-Trace-ID header, then generates UUID
- Structured logging ONLY — use `zap` or `zerolog`
- No `fmt.Println` or `log.Println` in production code
- Proper log levels: error for errors, warn for recoverable, info for business events
- No sensitive data in logs (PII, tokens, passwords)
- Alerts: Apdex < 0.9 = critical, error rate > 5% = critical, slow query > 10s = warning
- Logs centralized to observability platform

**WARNING if:** inconsistent logging, missing request-ID, fmt.Println in handlers

---

## 8. Memory & Resource Safety

- Goroutine leaks: `go func()` MUST have context cancellation or done channel — **REJECT** if leaking
  - Use `errgroup` for managed goroutine lifecycles
- `defer cancel()` immediately after `context.WithCancel`/`WithTimeout`/`WithDeadline` — **REJECT** if missing
- `defer resp.Body.Close()` after all HTTP calls — **REJECT** if missing
- `defer rows.Close()` after all DB queries — **REJECT** if missing
- `defer file.Close()` after `os.Open`/`os.Create`
- DB pool limits configured: `SetMaxOpenConns`, `SetMaxIdleConns`
- Unbuffered channels must have guaranteed reader/writer — **REJECT** potential deadlock
- Unbounded maps: maps don't shrink in Go — recreate periodically or use bounded alternatives — **REJECT** unbounded growth
- Slice retention: copy subslices to avoid retaining large backing arrays
- All SELECT queries must have `LIMIT` — **REJECT** unbounded queries
- Use cursor-based pagination; hard upper bound (e.g., 500 rows)

**REJECT if:** goroutine leaks, missing defer Close/cancel, unbounded maps/queries, unbuffered channels without guaranteed consumer

---

## 9. Event & Queue Patterns

- All messages MUST retry with exponential backoff (default: 3 retries, 100ms initial, 2x factor, 5s max)
- Failed messages MUST go to Dead Letter Queue (DLQ) after max retries
- Consumers MUST be idempotent — use cache-based idempotency key with 24h TTL
- Context propagation through message handlers
- Proper message acknowledgment required
- Error handling in all consumers — no silent failures
- No unbounded message processing — batch with limits
- Queue naming convention: `service_name.domain_name.use_case`

---

## 10. Deployment Standards

- Feature flags required for all new production features — **REJECT** if missing
  - Support: canary percentage, targeted user IDs, on/off toggle
  - OPTIONAL for POC/testing/unreleased/internal tools
- Canary release ready
- Docker configuration: multi-stage build (build in golang-alpine, run in alpine), non-root user
- Helm charts for Kubernetes deployment
- Docker image tagged with commit SHA
- No breaking changes without migration path
- Graceful shutdown handling

**REJECT if:** missing feature flags for new production features

---

## 11. Documentation & CI/CD

**Documentation:**
- README.md MUST be present and comprehensive — **REJECT** if missing
  - Sections: overview, architecture, tech stack, getting started, API docs link, config reference
- API documentation in OpenAPI v3 or Postman Collection format
- Architecture diagrams (Mermaid or PlantUML) for structural changes
- Godoc comments on exported types and functions

**CI/CD:**
- Self-hosted runners only
- Pipeline stages: lint, test, build, deploy
- `golangci-lint` for static analysis
- `go test -race` in CI
- `go vet` in CI
- No skipped tests or disabled checks
- Required Makefile targets: `build`, `run`, `test`, `lint`, `fmt`, `clean`, `docker`

**Configuration:**
- New env vars MUST appear in `.env.example` — **REJECT** if missing
- Configuration follows 12-factor app principles via `godotenv`/`viper`/env struct tags
- No hardcoded environment-specific values
- `UPPER_SNAKE_CASE` for env vars; `PascalCase` for Go constants

**REJECT if:** missing README.md, new env vars not in .env.example
