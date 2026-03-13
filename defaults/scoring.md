# PR Review Scoring Criteria

Reference for the Oracle agent. Defines score format, category criteria, decision matrix, and output template.

---

## SCORE_JSON Format

Every review MUST end with this block:

```
=== SCORE_JSON_START ===
{
  "architecture_design": <1-5>,
  "http_standards": <1-5>,
  "language_standards": <1-5>,
  "database_standards": <1-5>,
  "testing_standards": <1-5>,
  "security_standards": <1-5>,
  "monitoring_standards": <1-5>,
  "documentation_standards": <1-5>,
  "deployment_standards": <1-5>,
  "memory_safety": <1-5>,
  "requirements_alignment": <0-100 or -1>,
  "critical_issues": ["issue1", "issue2"],
  "warnings": ["warning1", "warning2"]
}
=== SCORE_JSON_END ===
```

Notes:
- Categories set to -1 are excluded from average calculation.

---

## Scoring Scale (1-5)

| Score | Label | Description |
|-------|-------|-------------|
| 5 | EXCELLENT | Fully meets or exceeds all standards |
| 4 | GOOD | Meets most standards with minor improvements needed |
| 3 | ACCEPTABLE | Meets basic requirements, some standards missing |
| 2 | NEEDS_WORK | Several standards violations, improvements required |
| 1 | REJECT | Critical standards violated, MR should be rejected |

---

## Category Definitions

### 1. Architecture & Design (architecture_design)
- **5**: DDD followed, Clean Code, proper layer separation (controllers/usecases/services/repository)
- **4**: Good architecture, minor layer separation issues
- **3**: Basic structure present, some concerns mixed
- **2**: Poor separation of concerns, tangled dependencies
- **1**: No clear architecture, monolithic/spaghetti code

### 2. HTTP Standards (http_standards)
- **5**: All routes start with "/", profiler middleware present, all endpoints idempotent
- **4**: Routes correct, profiler present, minor idempotency issues
- **3**: Most routes correct, basic profiler, some idempotency missing
- **2**: Several routes without "/", no profiler, idempotency issues
- **1**: Routes broken, no profiler, non-idempotent PUT (REJECT)

### 3. Language Standards (language_standards)

**Node.js:**
- **5**: All async/await, no callbacks, proper error handling with try/catch
- **4**: Mostly async/await, minor callback patterns, good error handling
- **3**: Mix of patterns, some callbacks, basic error handling
- **2**: Callback patterns, poor error handling, some blocking operations
- **1**: Callback hell, missing error handling, blocking operations (REJECT)

**Go:**
- **5**: Idiomatic Go, interfaces for all layers, context propagation, proper error wrapping (%w), goroutine-safe
- **4**: Good Go patterns, mostly interface-based, context used, good error handling
- **3**: Adequate Go, some concrete types, basic context usage, errors checked but not wrapped
- **2**: Non-idiomatic Go, ignored errors, missing context, race conditions possible
- **1**: Panic in handlers, ignored errors, no interfaces, data races (REJECT)

### 4. Database Standards (database_standards)
- **5**: Parameterized queries, explicit columns (NO SELECT *), health check present
- **4**: Parameterized queries, mostly explicit columns, health check
- **3**: Some parameterized queries, occasional SELECT *, basic health check
- **2**: Raw SQL in some places, SELECT * used, no health check
- **1**: SQL injection risk, widespread SELECT *, no health check (REJECT)

### 5. Testing Standards (testing_standards)
- **5**: 100% unit test coverage, comprehensive integration tests
- **4**: 90%+ coverage, good integration tests
- **3**: 70%+ coverage, basic integration tests
- **2**: 50%+ coverage, minimal tests
- **1**: No tests or <50% coverage (REJECT)

### 6. Security Standards (security_standards)
- **5**: Whitelist validation, proper retries with backoff, idempotency keys
- **4**: Mostly whitelist, good retry logic, idempotency present
- **3**: Mix of whitelist/blacklist, basic retry, some idempotency
- **2**: Blacklist validation, poor retry logic, missing idempotency
- **1**: Blacklist only, no retries, no idempotency handling (REJECT)

### 7. Monitoring Standards (monitoring_standards)
- **5**: Request-ID in all logs, APM registered, proper logging format
- **4**: Request-ID mostly present, APM configured, good logging
- **3**: Some request-ID, basic APM, standard logging
- **2**: Inconsistent request-ID, partial APM, poor logging
- **1**: No request-ID, no APM, no structured logging

### 8. Documentation Standards (documentation_standards)
- **5**: Comprehensive README.md, API docs (OpenAPI/Postman), architecture diagrams
- **4**: Good README.md, API docs present
- **3**: Basic README.md, some API documentation
- **2**: Minimal README.md, no API docs
- **1**: No README.md (REJECT)

### 9. Deployment Standards (deployment_standards)
- **5**: Feature flags for all new features, canary release ready, Docker/Helm configured
- **4**: Feature flags present, deployment scripts ready
- **3**: Basic feature flags, standard deployment
- **2**: No feature flags, manual deployment
- **1**: No feature flags for new features (REJECT for new features)

### 10. Memory & Resource Safety (memory_safety)
- **5**: All resources properly cleaned up; bounded caches; all queries paginated with LIMIT; language-specific lifecycle patterns correctly applied; no unbounded growth
- **4**: Good resource management with minor gaps (e.g., one missing cleanup path, cache without TTL but with max size, one query missing LIMIT in non-critical path)
- **3**: Basic cleanup present but inconsistent: some cleanup blocks missing, some unbounded collections, some queries without LIMIT, most resources managed
- **2**: Multiple leak patterns present: missing pagination on list endpoints, unbounded queries in request handlers, uncleaned resources
- **1**: Critical leaks: unbounded queries in hot paths, unclosed DB connections, goroutine leaks, event listener accumulation (REJECT)

---

## Requirements Alignment (0-100)

- Compare code changes against linked spec
- Score based on percentage of requirements covered
- Set to -1 if no spec linked (excluded from decision)

---

## Critical Issues vs Warnings

### Critical Issues (REJECT if present)
1. Missing unit tests
2. SQL injection vulnerabilities
3. Routes without leading "/"
4. Non-idempotent PUT operations
5. Callback hell patterns (Node.js)
6. Missing error handling
7. SELECT * queries (except COUNT(*))
8. Missing profiler middleware
9. Missing README.md
10. Missing feature flags for new features
11. Unclosed database connections without finally/defer
12. Event listeners accumulating without cleanup (Node.js)
13. Goroutine leaks without context cancellation (Go)
14. Unbounded in-memory cache/collection growth
15. Missing defer resp.Body.Close() after HTTP calls (Go)
16. Circular references without WeakRef or destroy() (Node.js)
17. useEffect without cleanup return when using timers/listeners/fetch (React)
18. Component lifecycle setup without matching teardown (Vue/Angular)
19. Unbounded query results — SELECT/findAll/Find without LIMIT

### Warnings (flag but don't reject)
1. Unresolved Promises in hot paths (flag for author to confirm intentional)
2. GET endpoints modifying data
3. POST without Idempotency-Key
4. PATCH without Idempotency-Key
5. Inconsistent logging
6. Missing integration tests

---

## Decision Matrix

| Average Score | Label | Decision |
|---------------|-------|----------|
| 4.5 - 5.0 | EXCELLENT | Approve immediately |
| 3.5 - 4.4 | GOOD | Approve with minor suggestions |
| 2.5 - 3.4 | ACCEPTABLE | Request changes before merge |
| 1.5 - 2.4 | NEEDS_WORK | Major improvements required |
| 1.0 - 1.4 | REJECT | Reject MR |

**Override Rule:** Any single category score of 1 = REJECT regardless of average.

**N/A categories:** If a category is -1 (e.g., language_standards for Python), exclude from average. Divide by count of scored categories, not always 10.

---

## Output Format Template

```markdown
# MR Review Summary

**MR Title:** {branch name or first commit message}
**Author:** {from git log if available}
**Reviewed by:** Claude Code Agent Team
**Date:** {today's date}

## Overview
{1-2 sentence description of what the MR accomplishes}

## Approval Status
{Approved|Approved with comments|Changes requested|Changes requested — major improvements required|Rejected} ({EXCELLENT|GOOD|ACCEPTABLE|NEEDS_WORK|REJECT})

## Key Findings

### Finding 1: {title}
**File:** `{file}` (Lines {line})
**Severity:** {critical/warning/suggestion}
**Issue:** {description}
**Suggested fix:**
\`\`\`{language}
{code fix}
\`\`\`
**Standard:** {reference}

{repeat for each validated finding, ordered by severity}

## Action Items
- [ ] {action item with file reference}
{one checkbox per critical/warning finding}

## Category Scores

| Category | Score | Label |
|----------|-------|-------|
| Architecture & Design | {1-5} | {label} |
| HTTP Standards | {1-5} | {label} |
| Language Standards | {1-5} | {label} |
| Database Standards | {1-5} | {label} |
| Testing Standards | {1-5} | {label} |
| Security Standards | {1-5} | {label} |
| Monitoring Standards | {1-5} | {label} |
| Documentation Standards | {1-5} | {label} |
| Deployment Standards | {1-5} | {label} |
| Memory Safety | {1-5} | {label} |
| **Average** | {avg} | **{decision}** |
| Requirements Alignment | {0-100 or -1} | |

=== SCORE_JSON_START ===
{
  "architecture_design": <1-5>,
  "http_standards": <1-5>,
  "language_standards": <1-5>,
  "database_standards": <1-5>,
  "testing_standards": <1-5>,
  "security_standards": <1-5>,
  "monitoring_standards": <1-5>,
  "documentation_standards": <1-5>,
  "deployment_standards": <1-5>,
  "memory_safety": <1-5>,
  "requirements_alignment": <0-100 or -1>,
  "critical_issues": ["issue1", "issue2"],
  "warnings": ["warning1", "warning2"]
}
=== SCORE_JSON_END ===
```
