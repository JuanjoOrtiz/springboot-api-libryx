# Configuration and profiles

Reference for `.claude/rules/configuration.md`. Read it before adding or changing a property, a
profile or an environment variable.

Names of `spring.*`, `management.*` and `springdoc.*` properties must be checked against the Spring
Boot 4 documentation when you write them; the `libryx.*` names are the ones in this document.

## 1. Profiles

| Profile | Use | Infrastructure | Notes |
|---|---|---|---|
| `local` | Development on your machine | Docker Compose: MariaDB, Redis, Mailpit. The machine's Ollama for chat | Seed data, Swagger UI on, readable console logs, `DEBUG` for `com.libryx` |
| `test` | Automated tests | Testcontainers: MariaDB and Redis | Scheduled jobs off, fixed `Clock`, test secrets, no seed data, AI replaced by test doubles |
| `prod` | Deployment | Managed services or containers | Secrets from Docker Secrets or Secrets Manager, JSON logs, Swagger UI off by default, `INFO` level, chat with Claude |

- There is no implicit default profile: starting without an active profile must fail fast for lack of
  mandatory configuration, not start with development values.
- Do not create new profiles (`dev`, `staging`, `docker`) without asking.
- No `@Profile` in business logic. Profiles change configuration and infrastructure, not rules.

## 2. File layout

```
src/main/resources/
├── application.yml          # shared by all profiles: safe defaults, no secrets
├── application-local.yml    # only what differs locally
├── application-prod.yml     # only what differs in production
├── messages.properties      # user-facing text, in Spanish
└── db/
    ├── migration/           # versioned Flyway migrations
    └── seed/local/          # seed data, local profile only
src/test/resources/
└── application-test.yml     # only what differs in tests
```

- `application.yml` contains **every** `libryx.*` default. A profile only overrides.
- No versioned file contains secrets. Sensitive values are always `${VARIABLE}` with no default.
- Order inside each file: `spring` → `management` → `springdoc` → `logging` → `libryx`.
- Durations carry a unit (`15m`, `48h`, `7d`) and bind to `java.time.Duration`; never bare integers.

## 3. Typed properties

- One `record` with `@ConfigurationProperties` per group (`LoanProperties`, `JwtProperties`,
  `RateLimitProperties`, `CacheProperties`, `JobProperties`), annotated with `@Validated`.
- Bean Validation constraints on the fields (`@Positive`, `@NotBlank`, `@NotNull`): an invalid
  configuration **prevents startup**.
- Services inject the record; never `@Value` or `Environment`.
- A new property is added to `application.yml` with its default **and** to the catalog below, in the same change.

## 4. `libryx.*` property catalog

### General

| Property | Default | Description |
|---|---|---|
| `libryx.time-zone` | `Europe/Madrid` | Library time zone for business dates and daily jobs |
| `libryx.mail.from` | — | Sender address for emails |
| `management.server.port` | `8081` | Actuator port, separate from the API (Spring property; see `observability.md`) |

### Business rules

| Property | Default | Description |
|---|---|---|
| `libryx.loan.duration-days` | `14` | Loan duration |
| `libryx.loan.max-active-per-member` | `3` | Simultaneous active loans per member |
| `libryx.loan.max-renewals` | `2` | Renewals per loan |
| `libryx.loan.renewal-days` | `14` | Days added by each renewal |
| `libryx.loan.due-soon-days` | `2` | Lead time of the due-soon notice |
| `libryx.loan-request.max-open-per-member` | `3` | Simultaneous open requests per member |
| `libryx.loan-request.pickup-window` | `48h` | Time to pick up a ready request |
| `libryx.sanction.late-return-days` | `21` | Duration of the late-return sanction |

### Security

| Property | Default | Description |
|---|---|---|
| `libryx.security.jwt.secret` | — (secret) | HS256 key, at least 256 bits |
| `libryx.security.jwt.access-token-ttl` | `15m` | Access token lifetime |
| `libryx.security.jwt.refresh-token-ttl` | `7d` | Refresh token lifetime |
| `libryx.security.cors.allowed-origins` | — | Allowed origins (frontend) |
| `libryx.security.password.min-length` | `8` | Minimum password length |
| `libryx.security.password.max-length` | `64` | Maximum password length |
| `libryx.security.lockout.max-failed-attempts` | `5` | Consecutive failed logins that lock the account |
| `libryx.security.lockout.duration` | `15m` | Lock duration |
| `libryx.security.email-verification.code-ttl` | `15m` | Validity of the email verification code |
| `libryx.security.email-verification.max-attempts` | `5` | Check attempts per verification code |
| `libryx.security.password-reset.code-ttl` | `15m` | Validity of the recovery code |
| `libryx.security.password-reset.max-attempts` | `5` | Check attempts per recovery code |
| `libryx.security.code-resend-cooldown` | `60s` | Minimum wait between two codes sent to the same email |

The password composition rule (one upper-case letter, one lower-case letter and one digit) is fixed in
code as a validation constraint, not a property.

### Rate limiting

Each rule has `limit` and `window` under `libryx.rate-limit.rules.<rule>`.

| Rule | `limit` | `window` | Key |
|---|---|---|---|
| `login-email` | `5` | `15m` | email |
| `login-ip` | `20` | `15m` | IP |
| `register-ip` | `5` | `1h` | IP |
| `code-request-email` | `3` | `1h` | email |
| `code-request-ip` | `10` | `1h` | IP |
| `token-refresh` | `10` | `1m` | user |
| `api-default` | `120` | `1m` | user |
| `catalog-search` | `30` | `1m` | user |
| `export` | `10` | `1h` | user |
| `assistant-minute` | `10` | `1m` | user |
| `assistant-day` | `100` | `24h` | user |
| `rag-ingest` | `5` | `1h` | user |

`code-request-*` covers both email verification codes and password recovery codes.

### Cache

| Property | Default | Description |
|---|---|---|
| `libryx.cache.book-detail-ttl` | `10m` | One work by id |
| `libryx.cache.catalog-reference-ttl` | `1h` | Categories, authors and publishers |
| `libryx.cache.catalog-search-ttl` | `2m` | Search results and listings |

### Assistant (Spring AI properties)

These are not `libryx.*`, but they are always set explicitly in each profile.

| Property | `local` | `test` | `prod` |
|---|---|---|---|
| `spring.ai.model.chat` | `ollama` | `none` | `anthropic` |
| `spring.ai.model.embedding` | `transformers` | `none` | `transformers` |
| `spring.ai.ollama.base-url` | `http://localhost:11434` | — | — |
| `spring.ai.ollama.chat.model` | chosen local model | — | — |
| `spring.ai.anthropic.api-key` | — | — | secret |
| `spring.ai.anthropic.chat.model` | — | — | chosen Claude model |
| `spring.ai.embedding.transformer.onnx.model-uri` | path to the ONNX model | — | path inside the image |
| `spring.ai.embedding.transformer.tokenizer.uri` | path to the tokenizer | — | path inside the image |
| `spring.ai.vectorstore.mariadb.initialize-schema` | `false` | `false` | `false` |
| `spring.ai.vectorstore.mariadb.dimensions` | `384` | `384` | `384` |

The embedding model is the same in `local` and `prod` (`multilingual-e5-small`, 384 dimensions).

### Scheduled jobs

| Property | Default | Description |
|---|---|---|
| `libryx.jobs.enabled` | `true` (`false` in `test`) | Turns all jobs on or off |
| `libryx.jobs.outbox.fixed-delay` | `1m` | Send pending notifications |
| `libryx.jobs.request-expiry.fixed-delay` | `15m` | Expire requests that were not picked up |
| `libryx.jobs.overdue.cron` | `0 5 0 * * *` | Mark overdue loans |
| `libryx.jobs.sanction-expiry.cron` | `0 10 0 * * *` | Expire finished sanctions |
| `libryx.jobs.due-soon.cron` | `0 0 9 * * *` | Notify loans about to be due |
| `libryx.jobs.code-cleanup.cron` | `0 0 3 * * *` | Delete expired verification and recovery codes |

`cron` expressions are evaluated in `libryx.time-zone`.

## 5. Environment variables

Standard Spring names (relaxed binding), with no invented aliases.

| Variable | Required | Secret | Description |
|---|---|---|---|
| `SPRING_PROFILES_ACTIVE` | yes | no | `local`, `test` or `prod` |
| `SPRING_DATASOURCE_URL` | yes | no | MariaDB JDBC URL |
| `SPRING_DATASOURCE_USERNAME` | yes | no | Database user |
| `SPRING_DATASOURCE_PASSWORD` | yes | **yes** | Database password |
| `SPRING_DATA_REDIS_HOST` | yes | no | Redis host |
| `SPRING_DATA_REDIS_PORT` | no | no | Redis port (6379) |
| `SPRING_DATA_REDIS_PASSWORD` | in `prod` | **yes** | Redis password |
| `SPRING_MAIL_HOST` | yes | no | SMTP server (Mailpit locally) |
| `SPRING_MAIL_PORT` | yes | no | SMTP port (1025 locally) |
| `SPRING_MAIL_USERNAME` | in `prod` | no | SMTP user |
| `SPRING_MAIL_PASSWORD` | in `prod` | **yes** | SMTP password |
| `LIBRYX_SECURITY_JWT_SECRET` | yes | **yes** | JWT signing key |
| `LIBRYX_SECURITY_CORS_ALLOWED_ORIGINS` | yes | no | Frontend origin |
| `LIBRYX_MAIL_FROM` | yes | no | Sender address for emails |
| `GRAFANA_ADMIN_USER` | only in `local` | no | Grafana admin user (read by Compose, not by the application) |
| `GRAFANA_ADMIN_PASSWORD` | only in `local` | **yes** | Grafana admin password (read by Compose, not by the application) |
| `SPRING_AI_ANTHROPIC_API_KEY` | only in `prod` | **yes** | Claude API key for the assistant's chat |
| `SPRING_AI_ANTHROPIC_CHAT_MODEL` | only in `prod` | no | Claude model used by the assistant |
| `SPRING_AI_OLLAMA_BASE_URL` | no | no | Ollama URL locally (`http://localhost:11434`) |
| `SPRING_AI_OLLAMA_CHAT_MODEL` | only in `local` | no | Ollama model used by the assistant |

The embedding model runs inside the application and needs no key.

### Secrets

- **Local**: a `.env` file read by Docker Compose. `.env` is in `.gitignore`; `.env.example` is
  versioned with every variable and fake values.
- **Production**: Docker Secrets mounted at `/run/secrets`, imported with
  `spring.config.import: optional:configtree:/run/secrets/` and referenced as `${secret_name}`.
- **AWS**: Secrets Manager injects each secret into the task as an environment variable (see `deployment-aws.md`).
- **Tests**: fixed test values in `application-test.yml`; never a real secret.
- When adding a variable, update this table, `.env.example` and `compose.yaml` in the same change.
