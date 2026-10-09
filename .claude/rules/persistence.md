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
- The application migrates only in `test`, against the Testcontainers database. In `local` and `prod`
  it does not (`spring.flyway.enabled=false`): the one-shot Flyway container does.
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
- `equals` is true only when both entities have an id and the ids match; `hashCode` returns a constant
  per class. Never base them on collections or on mutable fields.
- Never use Lombok `@Data`, `@EqualsAndHashCode` or `@ToString` on an entity.
- `created_at` and `updated_at` are filled by the database: map them read-only
  (`insertable = false, updatable = false`). Re-read the entity if the value is needed after saving.

## Repositories

- Spring Data interfaces. Derived queries for simple lookups, `@Query` in JPQL otherwise, native SQL
  only for MariaDB features: the catalog `FULLTEXT` search (JPQL has no `MATCH ... AGAINST`) and vectors.
- Single results return `Optional`; list queries take a `Pageable` and return a page or a slice.

## Points that are easy to get wrong

- `loans.open_flag`, `loan_requests.open_flag` and `sanctions.open_flag` are **generated columns**: if
  mapped, use `insertable = false, updatable = false`. Their `UNIQUE` constraints guarantee in the
  database that a copy has one open loan at most, a user has one open request per work, and a user has
  one active sanction at most.
- `users` uses **soft delete** (`deleted_at`) to keep loan history. Never issue a physical `DELETE`.
  The `User` entity filters deleted rows with `@SQLRestriction("deleted_at IS NULL")`.
- Deleting a user **frees the email**: in the same transaction it is replaced by a unique anonymous
  value (`deleted-<id>@libryx.invalid`), so the address can register again. What happens to the
  `user_profiles` row on deletion is **not defined yet: ask**.
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
| `user@libryx.local` | `USER` | Active loans and a queued request |
| `user.sanctioned@libryx.local` | `USER` | An active late-return sanction |

- All four are email-verified (`email_verified = true`) and `ACTIVE`, and share the development password `Libryx2026!`, also documented in the
  README. It is valid only in `local`; the seed stores only its hash.
- The seed catalog covers every state: works with and without free copies, an overdue loan, one
  `PENDING` and one `READY` request.
- Tests do not use the seeds: each test builds its own data.
