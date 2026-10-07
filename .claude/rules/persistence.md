---
paths:
  - "src/main/java/**/entity/**"
  - "src/main/java/**/repository/**"
  - "src/main/resources/db/**"
---

# Persistence rules

## Flyway owns the schema

- Migrations in `src/main/resources/db/migration`, named `V{n}__description_in_snake_case.sql`.
- **Never edit an applied migration**: add a new one.
- `spring.jpa.hibernate.ddl-auto=validate` and `spring.jpa.open-in-view=false`.
- A new or changed entity ships with its migration in the same change.
- `V1__libryx_schema.sql` is the source of truth for tables and enum values.

## Schema conventions

- Tables and columns in English, `snake_case`, plural table names. InnoDB, `utf8mb4_unicode_ci`.
- Primary keys `BIGINT AUTO_INCREMENT` → `Long`. Exceptions: `roles.id` (`SMALLINT`) and `vector_store.id` (`UUID`).
- Enums are `VARCHAR` + `CHECK` → `@Enumerated(EnumType.STRING)`. Adding an enum value requires a
  migration that updates the `CHECK`.
- `DATETIME(6)` in UTC → `Instant`; `DATE` → `LocalDate`. Set `hibernate.jdbc.time_zone=UTC`.
- A `version` column → `@Version` (optimistic locking) on `users`, `books`, `book_copies`, `loans`, `loan_requests`.
- Constraint name prefixes: `pk_`, `fk_`, `uk_`, `ck_`, `ix_`, `ft_`.

## Entities

- Anemic: fields, JPA mappings, getters and setters. No business logic, no validation of business rules.
- `LAZY` relations by default. Solve N+1 with `@EntityGraph` or `JOIN FETCH`, never with `EAGER`.
- `equals`/`hashCode` based on the identifier, never on collections.
- Never use Lombok `@Data`, `@EqualsAndHashCode` or `@ToString` on an entity.

## Points that are easy to get wrong

- `loans.open_flag` and `loan_requests.open_flag` are **generated columns**: if mapped, use
  `insertable = false, updatable = false`. Their `UNIQUE` constraints guarantee in the database that a
  copy has one open loan at most and a member has one open request per work.
- `users` uses **soft delete** (`deleted_at`) to keep loan history. Never issue a physical `DELETE`.
- **Work ≠ copy**: `books` is the title; `book_copies` is what gets lent. Requests target the work,
  loans target the copy.
- Credentials (`users`) are separate from personal data (`user_profiles`). See `security.md`.
- Catalog search uses the `FULLTEXT` index `ft_books_search` (title, subtitle, synopsis).
- `vector_store` belongs to Spring AI: created by Flyway, never by the application
  (`spring.ai.vectorstore.mariadb.initialize-schema=false`).
- The global handler turns `OptimisticLockingFailureException` and `DataIntegrityViolationException`
  into a 409 with a clear message. Do not replace database constraints with in-memory checks.

## Seed data (`local` profile only)

- Data needed in every environment (roles) goes in versioned migrations.
- Demo data goes in `src/main/resources/db/seed/local` as repeatable migrations `R__seed_*.sql`. That
  location is added to Flyway only in `local`: never in `test` or `prod`.
- Seeds are idempotent and use fictional people and public-domain works. **Never real personal data.**

| User | Role | State |
|---|---|---|
| `admin@libryx.local` | `ADMIN` | — |
| `librarian@libryx.local` | `LIBRARIAN` | — |
| `member@libryx.local` | `MEMBER` | Active loans and a queued request |
| `member.sanctioned@libryx.local` | `MEMBER` | An active late-return sanction |

- All four are email-verified and share a development password documented in the README; the seed
  stores only its hash.
- The seed catalog covers every state: works with and without free copies, an overdue loan, one
  `PENDING` and one `READY` request.
- Tests do not use the seeds: each test builds its own data.
