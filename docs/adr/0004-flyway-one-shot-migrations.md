# ADR-0004: Flyway migrations as a one-shot step before deployment

- **Status:** Accepted
- **Date:** 2026-10-09

## Context

The schema relies on MariaDB features (generated columns, `CHECK`, `FULLTEXT`, `VECTOR`), so it must be
written in SQL and versioned. Migrating from inside the application at startup ties a schema change to
a running instance and gives the API user DDL permissions.

## Decision

- Flyway runs as a **one-shot container** (`docker compose run --rm flyway` locally, a one-off ECS task
  on AWS) **before** the API is deployed. The API starts with `spring.flyway.enabled=false` and
  `ddl-auto=validate`.
- Only the `test` profile migrates from the application, embedded, against Testcontainers.
- Migrations are backward compatible with the previous API version; an incompatible change is split
  across two deployments.

## Alternatives considered

| Alternative | Why it was discarded |
|---|---|
| Flyway at application startup | Couples migration and startup, and the API needs DDL rights |
| Hibernate `ddl-auto=update` | No history, no review and no control over MariaDB-specific DDL |
| Liquibase | No advantage over plain SQL migrations for a single database engine |

## Consequences

- **Gains:** a failed migration never starts a broken API; the runtime database user can be limited.
- **Costs:** one more step in local setup and in the deployment flow.
- **Watch for:** forgetting the migration step, which shows up as a schema validation error at startup.
