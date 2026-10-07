# Logging and observability

Reference for the logging rules in `CLAUDE.md` and `.claude/rules/services.md`. Read it before adding
logs or metrics, or touching the Actuator configuration.

## 1. Logging

SLF4J through `@Slf4j`. Messages in English, on one line, with `{}` placeholders.

```java
log.info("Loan created loanId={} copyId={} memberId={}", loan.getId(), copyId, memberId);
```

### Levels

| Level | When | Examples |
|---|---|---|
| `ERROR` | Unexpected failure that needs attention. Always with the exception | Unhandled error (500), a scheduled job that aborts, a notification out of retries |
| `WARN` | Anomaly the system recovers from by itself | Redis unavailable and the read served from the database, a mail retry, an account locked after failed attempts, a refresh token reused |
| `INFO` | Business and lifecycle events | Loan created, returned or renewed; sanction created or lifted; request ready; export generated; start and end of a job with its record count |
| `DEBUG` | Detail for diagnosis. Off in `prod` | The outcome of a rule, a cache key, a built query |

### Rules

- A fact is logged **once**, in the layer that knows it: business events in the service, errors in
  the global handler. Do not log and rethrow the same exception.
- Business exceptions (4xx) are logged without a stack trace: `WARN` for 401, 403 and 429; `INFO` or
  nothing for 400, 404, 409 and 422. Only 500s carry a stack trace, at `ERROR`.
- **Never** in a log: passwords, tokens, verification or recovery codes, `Authorization` or `Cookie`
  headers, or personal data (name, email, document id, phone, address). Log **identifiers**.
- No string concatenation or `String.format` in the log call.
- No `System.out`, no `printStackTrace()`, no logging inside loops over large collections.
- Do not log full request or response bodies.

### Format

| Profile | Format | Root level | `com.libryx` |
|---|---|---|---|
| `local` | Readable console text | `INFO` | `DEBUG` |
| `test` | Readable console text | `WARN` | `INFO` |
| `prod` | Structured JSON (`logging.structured.format.console`) | `INFO` | `INFO` |

Always to standard output; the container collects it. The application writes no log files.

## 2. Correlation id

Every request carries an identifier that appears in all its logs and in the response.

- A `OncePerRequestFilter` in `shared`, first in the chain:
  1. Reads the `X-Request-Id` header. If it is missing or invalid (more than 64 characters, or
     characters outside `[A-Za-z0-9-]`), it generates a UUID.
  2. Stores it in the MDC under `requestId`.
  3. Returns it in the `X-Request-Id` response header.
  4. Clears the MDC in a `finally` block.
- After authentication, the JWT filter adds `userId` to the MDC. Never the email.
- The global handler includes `requestId` in every `ProblemDetail`, so a user can quote it.
- Scheduled jobs generate their own id per run and add `job` to the MDC.
- When work moves to another thread (`@Async`, after-commit listeners), the MDC is propagated with a
  `TaskDecorator`; the `requestId` is not lost.
- The log pattern includes `requestId` and `userId` in every profile.

## 3. Metrics

Micrometer. Technical metrics (HTTP, JVM, connection pool, cache) come from auto-configuration; this
section defines the **business** ones.

### Conventions

- Lower-case, dot-separated names: `libryx.<area>.<event>`.
- **Low-cardinality** tags: enum values or booleans. Never `userId`, `bookId`, an email, an ISBN or
  anything that grows with the data.
- Metrics are recorded in the service, next to the event, through one component per module
  (`LoanMetrics`) that wraps the `MeterRegistry`. Services do not build counters by hand.
- A new metric is added to this table in the same change.

### Catalog

| Metric | Type | Tags | Measures |
|---|---|---|---|
| `libryx.auth.registrations` | counter | — | Member registrations |
| `libryx.auth.email.verifications` | counter | `result` = `success` \| `failure` | Email verification attempts |
| `libryx.auth.logins` | counter | `result` = `success` \| `failure` \| `locked` \| `unverified` | Login attempts |
| `libryx.auth.password.resets` | counter | `step` = `requested` \| `completed` \| `failed` | Password recoveries |
| `libryx.loans.created` | counter | — | Loans handed over |
| `libryx.loans.returned` | counter | `late` = `true` \| `false` | Returns |
| `libryx.loans.renewed` | counter | — | Renewals |
| `libryx.loans.overdue` | gauge | — | Loans overdue right now |
| `libryx.loan.requests.created` | counter | — | Requests created |
| `libryx.loan.requests.closed` | counter | `status` = `FULFILLED` \| `CANCELLED` \| `REJECTED` \| `EXPIRED` | Requests closed |
| `libryx.sanctions.created` | counter | `reason` | Sanctions applied |
| `libryx.sanctions.lifted` | counter | — | Sanctions lifted by hand |
| `libryx.notifications.sent` | counter | `type`, `channel`, `result` = `sent` \| `failed` | Notification sends |
| `libryx.notifications.pending` | gauge | — | Size of the outbox queue |
| `libryx.exports.generated` | counter | `type`, `format` | Reports exported |
| `libryx.ratelimit.rejected` | counter | `rule` | Requests rejected with 429 |
| `libryx.assistant.requests` | timer | `result` = `success` \| `error` | Assistant queries and their duration |
| `libryx.rag.documents.ingested` | counter | `source`, `result` | Documents indexed |
| `libryx.jobs.execution` | timer | `job`, `result` = `success` \| `error` | Scheduled job runs |

## 4. Actuator

Expose only what is needed: `management.endpoints.web.exposure.include=health,info,metrics,prometheus`.

| Endpoint | Access | Notes |
|---|---|---|
| `/actuator/health` | Public | No details for anonymous callers (`show-details=when-authorized`). `liveness` and `readiness` probes on |
| `/actuator/info` | Public | Version and build data. No environment information |
| `/actuator/metrics` | `ADMIN` only | Technical and business metrics |
| `/actuator/prometheus` | Never public: `ADMIN` or the internal monitoring network | The same metrics in Prometheus format (`micrometer-registry-prometheus`) |
| Anything else | **Not exposed** | `env`, `configprops`, `beans`, `heapdump`, `threaddump`, `loggers`, `mappings`, `shutdown` |

- Access rules are declared in `SecurityConfig`, not left to defaults.
- `readiness` depends on MariaDB and Redis. The mail server is **not** part of the application's
  health: an SMTP outage delays notifications (the outbox retries), it does not take the API down.
- Actuator endpoints are outside the general rate limit and the OpenAPI documentation.
- Do not expose a new endpoint or add a metrics registry besides Prometheus without asking.
