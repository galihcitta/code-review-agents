# Node.js Review Standards

Industry-standard checklist for reviewing Node.js codebases. Each section maps to a scoring category. Items marked **REJECT** are critical issues that warrant MR rejection. Items marked **WARNING** should be flagged but don't block merge.

---

## 1. Language Standards

- All async operations MUST use async/await — **REJECT** callback hell patterns
- No blocking synchronous operations in request handlers (fs.readFileSync, crypto.pbkdf2Sync, etc.)
- All DB operations must be Promise-based with async/await
- Proper error handling: try/catch/finally — no swallowed errors, no empty catch blocks
- Error wrapping with context: throw new Error(`Failed to process order ${id}: ${err.message}`)
- No `new Promise()` where async/await works directly
- Use `Map` over plain objects for dynamic key collections and caches
- Concurrent code must be tested with `--detectOpenHandles`
- Follow [goldbergyoni/nodebestpractices](https://github.com/goldbergyoni/nodebestpractices)

**REJECT if:** callback hell, missing error handling, blocking operations in handlers

---

## 2. HTTP Standards

- All routes MUST start with "/" — **REJECT** if missing
- RESTful naming: nouns not verbs in URLs (e.g., `/users` not `/getUsers`)
- Profiler endpoints MANDATORY — **REJECT** if missing:
  - `/profiler/start`, `/profiler/stop`, `/profiler/heap-snapshot`, `/metrics`
- All dependencies initialized in ONE file, passed via constructor injection
- Proper HTTP status codes (201 for creation, 204 for no content, etc.)
- PUT MUST be idempotent (no increment/append operations) — **REJECT** if not
- GET MUST be read-only — **WARNING** if modifying data
- POST/PATCH SHOULD use `Idempotency-Key` header — **WARNING** if missing
- DELETE must return same result on repeat calls
- PATCH operations must be set-based, not delta-based

**REJECT if:** routes without "/", missing profiler, non-idempotent PUT
**WARNING if:** GET modifies data, POST/PATCH without Idempotency-Key

---

## 3. Database Standards

- All queries MUST use parameterized/prepared statements — **REJECT** raw SQL with string interpolation
- `SELECT *` is PROHIBITED (exception: `COUNT(*)`) — **REJECT**
- Always use explicit column selection
- Health check endpoint must test all DB/cache/queue dependencies
- Slow query threshold: 10 seconds — flag queries likely to exceed this
- DDL migrations must be additive only; no destructive changes in the same MR
- Connection pool configuration required; connections cleaned up in finally blocks
- Transaction boundaries where multiple related writes occur
- Schemas should be denormalized where appropriate for read performance

**REJECT if:** SQL injection risk, SELECT *, missing health check

---

## 4. Security Standards

- Use whitelist (explicit allowed values) not blacklist for validation — **REJECT** blacklist-only
- Input validation and sanitization at handler boundary
- No hardcoded secrets, API keys, or passwords in source code
- OWASP top 10: SQL injection, XSS, SSRF, path traversal, command injection, insecure deserialization
- Proper authentication/authorization checks on all endpoints
- Missing rate limiting on sensitive endpoints (login, password reset, OTP)
- Repository/HTTP clients MUST implement retry with exponential backoff
  - Retriable status codes: 408, 429, 500, 502, 503, 504
  - Default timeout: 30 seconds
  - POST/PATCH retry only with `Idempotency-Key` header

**REJECT if:** SQL injection, XSS, hardcoded secrets, blacklist-only validation

---

## 5. Testing Standards

- Unit tests MUST be present for new/changed code — **REJECT** if missing
- Coverage target: 100% for new code (branches, functions, lines, statements)
- Integration tests for API endpoints covering happy paths
- Edge case coverage: null, empty, boundary values
- Test naming follows convention: `*_test.js` or `*.test.js`
- Mocks/stubs used appropriately — don't test implementation details
- No test code in production files
- Load tests for critical paths (target: 1000 concurrent users, <5s response)

**REJECT if:** no unit tests, <50% coverage

---

## 6. Monitoring Standards

- APM tool MUST be registered for all services (first require in app entry)
- Request-ID / trace-ID MUST be in all log statements
  - Middleware injects UUID, creates child logger with context
- Structured logging required (e.g., a structured logger like Pino) — no console.log in production
- Proper log levels: error for errors, warn for recoverable issues, info for business events
- No sensitive data in logs (PII, tokens, passwords)
- Alerts configured: Apdex < 0.9 = critical, error rate > 5% = critical, slow query > 10s = warning
- Logs centralized to observability platform

**WARNING if:** inconsistent logging, missing request-ID

---

## 7. Memory & Resource Safety

**Backend:**
- Event listener leaks: `.on()` must have matching `.off()` or use `.once()`/`AbortController` — **REJECT** if accumulating
- Timer leaks: `setInterval`/`setTimeout` must store IDs and clear on destroy/SIGTERM — **REJECT** if leaking
- Stream/connection leaks: all resources closed in `finally` blocks — **REJECT** unclosed DB connections
- Unbounded caches: Map/Set/Object must have max size + TTL (use `lru-cache`) — **REJECT** unbounded growth
- Closures must not capture large objects (request objects, buffers) — extract needed values
- Circular references: use `WeakRef` for back-references — **REJECT** circular refs without WeakRef/destroy()
- Unresolved Promises: all must resolve/reject on all paths with timeouts — **WARNING** in hot paths
- Puppeteer/JSDOM: always `page.close()` / `dom.window.close()`

**Frontend (.tsx/.jsx/.vue):**
- `useEffect` MUST return cleanup function when using timers/listeners/fetch — **REJECT** if missing
- Use `AbortController` for fetch requests, abort on unmount
- Clear intervals/timeouts on unmount
- Vue: pair `onMounted` with `onUnmounted`
- Angular: implement `OnDestroy`
- No state updates after unmount

**REJECT if:** unclosed connections, goroutine leaks, event listener accumulation, unbounded caches, useEffect without cleanup

---

## 8. Event & Queue Patterns

- All messages MUST retry with exponential backoff (default: 3 retries, 100ms initial, 2x factor, 5s max)
- Failed messages MUST go to Dead Letter Queue (DLQ) after max retries
- Consumers MUST be idempotent — use cache-based idempotency key with 24h TTL
- Proper message acknowledgment required
- Error handling in all consumers — no silent failures
- No unbounded message processing — batch with limits
- Queue naming convention: `service_name.domain_name.use_case`
- Always send callback notification on completion

---

## 9. Deployment Standards

- Feature flags required for all new production features — **REJECT** if missing for new features
  - Support: canary percentage, targeted user IDs, on/off toggle
  - OPTIONAL for POC/testing/unreleased/internal tools
- Canary release ready
- Docker configuration: multi-stage build, alpine base, `dumb-init` for signal handling, non-root user
- Helm charts for Kubernetes deployment
- Docker image tagged with commit SHA
- No breaking changes without migration path
- Rollback plan considered

**REJECT if:** missing feature flags for new production features

---

## 10. Documentation & CI/CD

**Documentation:**
- README.md MUST be present and comprehensive — **REJECT** if missing
  - Sections: overview, architecture, tech stack, getting started, API docs link, config reference
- API documentation in OpenAPI v3 or Postman Collection format
- Architecture diagrams (Mermaid or PlantUML) for structural changes

**CI/CD:**
- Self-hosted runners only (not public runners)
- `npm ci` for deterministic installs
- Pipeline stages: lint, test, build, deploy
- No skipped tests or disabled checks
- Required npm scripts: `start`, `dev`, `test`, `test:unit`, `test:integration`, `test:coverage`, `lint`

**Configuration:**
- New `process.env.*` variables MUST appear in `.env.example` — **REJECT** if missing
- Configuration follows 12-factor app principles
- No hardcoded environment-specific values
- Node engine `>=18.0.0` in package.json

**REJECT if:** missing README.md, new env vars not in .env.example
