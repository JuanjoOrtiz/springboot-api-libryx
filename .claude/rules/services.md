---
paths:
  - "src/main/java/**/service/**"
---

# Service layer rules

Metric catalog, log levels and Actuator details: `docs/observability.md`.

## Interface and implementation

- Every service is an interface in `service/` (`LoanService`) with one implementation in
  `service/impl/` (`LoanServiceImpl`).
- Controllers, jobs and other modules inject the **interface**. Nothing depends on an `Impl` class.
- The interface exposes only what callers need. Javadoc goes on the interface.
- Split by business case when a service grows past ~300 lines or ~7 dependencies
  (`LoanService`, `LoanRenewalService`, `LoanRequestService`).
- A complex eligibility check goes to its own class (`LoanEligibilityPolicy`), injected into the service.

## Business logic lives here

- Entities are anemic: all rules, state transitions and invariants are enforced in the service.
  A service never leaves an entity in an invalid state.
- Change entity state through one service method per transition (`markReturned`, `lift`), so each
  transition has a single place where its rules are checked.
- Services receive and return DTOs or ids at the module boundary; entities stay inside the module.
- Query methods do not modify state. A method that modifies says so in its name.

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
