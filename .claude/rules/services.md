---
paths:
  - "src/main/java/**/service/**"
---

# Service layer rules

Metric catalog, log levels and Actuator details: `docs/observability.md`.

## Interface and implementation

- Every service is an interface in `service/` (`LoanService`) with one implementation in
  `service/impl/` (`LoanServiceImpl`).
- Controllers, jobs and other modules inject the **interface**. Nothing depends on an `Impl` class,
  except its own unit test.
- The interface exposes only what callers need. Javadoc goes on the interface.
- Split by business case when a service grows past ~300 lines or ~7 dependencies
  (`LoanService`, `LoanRenewalService`, `LoanRequestService`).
- A complex eligibility check goes to its own class (`LoanEligibilityPolicy`), injected into the service.
  It is a plain `@Component` in `service/`, without an interface: it is not a service.

## Business logic lives here

- Entities are anemic: all rules, state transitions and invariants are enforced in the service.
  A service never leaves an entity in an invalid state.
- Change entity state through one service method per transition (`markReturned`, `lift`), so each
  transition has a single place where its rules are checked.
- Services take and return DTOs or ids. Entities never leave the service layer, not even to the
  module's own controller: the service maps them with the module's MapStruct mapper.
- Query methods do not modify state. A method that modifies says so in its name.

## Errors

- A broken business rule throws a business exception from `shared` with its `ErrorCode` and
  arguments (`new LoanLimitExceededException(memberId, max)`), never a message text.
- Never catch an exception just to log it and rethrow it, and never return `null` to signal an error.

## Authorization

- Role checks (`@PreAuthorize`) go on the controller methods. The service receives the acting
  user's id as a parameter and enforces ownership itself.
- Scheduled jobs call services without a security context, so service methods never depend on one.

## Transactions

- `@Transactional` on the implementation, never on controllers or repositories.
- Class-level `@Transactional(readOnly = true)`, plus `@Transactional` on methods that write.
- `@Transactional` and `@Cacheable` do not apply on self-invocation: call through another bean.
- Never send email or call an external service inside a business transaction (see `notifications.md`).

## Time and configuration

- Inject `Clock`. Business dates (`due_date`, sanction start and end, "due today") are computed in
  the library time zone, `libryx.time-zone` (`Europe/Madrid`); instants are stored in UTC.
- Business limits come from `@ConfigurationProperties` records, never from literals.

## Logging and metrics

- `ERROR` for unexpected failures (with the exception), `WARN` for anomalies the system recovers from,
  `INFO` for business events and jobs, `DEBUG` for diagnostics.
- Log each business event once, here, at `INFO`, with identifiers. Do not log and rethrow.
- Never log passwords, tokens, codes or personal data.
- Every relevant business event (loan, return, sanction, notification, export, login, rate-limit
  rejection, job run) records a Micrometer metric named `libryx.<area>.<event>`, through one
  component per module (`LoanMetrics`). Low-cardinality tags only: enums and booleans, never ids or emails.
