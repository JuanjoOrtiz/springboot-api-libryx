---
paths:
  - "src/main/resources/application*"
  - "src/test/resources/application*"
  - "src/main/java/**/config/**"
  - "compose.yaml"
  - ".env.example"
---

# Configuration rules

Full property catalog and environment variables: `docs/configuration.md`. Read it before adding or
changing a property, a profile or a variable.

| Profile | Use | Notes |
|---|---|---|
| `local` | Development | Docker Compose, Mailpit, Ollama, seed data, Swagger UI, `DEBUG` for `com.libryx` |
| `test` | Tests | Testcontainers, scheduled jobs disabled, AI doubles, no seed data |
| `prod` | Deployment | Secrets from Docker Secrets or Secrets Manager, JSON logs, Swagger UI off by default |

- `application.yml` holds every default and no secret. Each `application-<profile>.yml` contains only
  what differs.
- All project configuration hangs from `libryx.*` and is read through a `@ConfigurationProperties`
  record annotated with `@Validated`: an invalid configuration prevents startup.
- Services inject the record; never `@Value` or `Environment`.
- Durations carry a unit (`15m`, `48h`, `7d`) and bind to `Duration`.
- Environment variables use the standard Spring names (`SPRING_DATASOURCE_URL`,
  `LIBRYX_SECURITY_JWT_SECRET`); do not invent aliases.
- Secrets: `.env` locally (git-ignored; `.env.example` is versioned), Docker Secrets imported with
  `configtree` in production, Secrets Manager on AWS. Sensitive values are `${VARIABLE}` with no default.
- When adding a property or variable, update `application.yml`, `docs/configuration.md`,
  `.env.example` and `compose.yaml` in the same change.
- Do not create new profiles, and never use `@Profile` in business logic.
- Starting without an active profile must fail fast, not start with development values.
