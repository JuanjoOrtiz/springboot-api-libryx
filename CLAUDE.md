# Libryx Backend

REST API for library management: catalog, members, loans, loan requests, sanctions, notifications,
reports and a RAG-based librarian assistant. It is a portfolio project, so code quality, tests and
documentation matter as much as features.

- **Backend only**: a stateless REST API. The Angular frontend lives in a separate repository.
- **Layered monolith organized by module** (`controller → service → repository`). It is not hexagonal
  or Clean Architecture: do not introduce ports, adapters or use-case classes.
- **Bootstrap in progress**: the repository started from a Spring Boot 3.5 / Java 21 skeleton. While
  `docs/bootstrap.md` has unchecked items, read it before any task; the changes it lists are pre-approved.
- If an instruction here conflicts with what you find in the code, ask before "fixing" either side.

## Stack

| Area | Technology |
|---|---|
| Language | Java 25, no preview features |
| Framework | Spring Boot 4.x (Spring Framework 7, Jakarta EE 11, Jackson 3) |
| Persistence | Spring Data JPA on MariaDB 11.8, Flyway migrations |
| Redis | Catalog cache (Spring Cache), refresh tokens, JWT denylist, rate limiting |
| Security | Spring Security 7, JWT (JJWT), role checks with `@PreAuthorize` |
| Mapping / boilerplate | MapStruct, Lombok |
| API docs | SpringDoc OpenAPI 3.x |
| Mail | Spring Mail; Mailpit locally |
| Exports | iText (PDF), Apache POI (XLSX) |
| AI | Spring AI 2.0.x: Claude in `prod`, Ollama in `local`, in-app ONNX embeddings, MariaDB vector store |
| Observability | Micrometer, Actuator, Prometheus registry |
| Tests | JUnit Jupiter, Mockito, AssertJ, Testcontainers |
| Build / CI | Maven wrapper (`./mvnw`), GitHub Actions |
| Infrastructure | Docker Compose; AWS with Terraform, as code only |

`pom.xml` is the source of truth for versions. Do not change versions or add dependencies without asking.

## Commands

```bash
docker compose up -d mariadb redis mailpit      # local infrastructure
docker compose run --rm flyway                  # apply migrations (one-shot container)
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
./mvnw test                                     # unit tests
./mvnw test -Dtest=LoanServiceImplTest          # one class (append #method for one test)
./mvnw clean verify                             # unit + integration tests (needs Docker) + coverage
```

Local URLs: API `http://localhost:8080/api`, Swagger UI `/swagger-ui.html`, health `/actuator/health`,
Mailpit `http://localhost:8025`. The assistant uses the Ollama installed on the machine (port 11434).

## Where the rules live

This file holds what applies everywhere. Area rules in `.claude/rules/` load automatically when you
touch matching files. Reference documents in `docs/` are **not** loaded: read the relevant one before
working in that area, and update it in the same change when your work alters it.

| Working on | Rule file (auto-loaded) | Reference to read |
|---|---|---|
| Controllers, DTOs, errors, user messages | `api.md` | `docs/api-contract.md` |
| Service interfaces and implementations | `services.md` | `docs/observability.md` |
| Entities, repositories, migrations | `persistence.md` | — |
| Auth, users, security configuration | `security.md` | — |
| Loans, loan requests, sanctions | `business-rules.md` | — |
| Catalog and its cache | `catalog.md` | — |
| Scheduled jobs | `scheduled-jobs.md` | — |
| Notifications / exports / assistant | `notifications.md`, `exports.md`, `assistant.md` | — |
| Tests | `testing.md` | — |
| Properties, profiles, environment | `configuration.md` | `docs/configuration.md` |
| Docker, CI, Terraform | `infrastructure.md` | `docs/deployment-aws.md` |

## Package layout

Root package `com.libryx`, split first by **business module**, then by **layer**.

```
com.libryx
├── config/         # security, Redis, OpenAPI and typed properties
├── security/       # JWT filter, token service, denylist, UserDetailsService
├── shared/         # base exceptions, global error handler, common DTOs, request-id filter
├── auth/  user/  catalog/  loan/  sanction/
└── notification/  export/  assistant/  dashboard/

loan/
├── controller/     # @RestController: HTTP only
├── service/        # service interfaces
│   └── impl/       # implementations: business logic and transactions
├── job/            # scheduled jobs, only if the module has any
├── repository/     # Spring Data JPA interfaces
├── entity/         # JPA entities and enums
├── dto/            # request and response records
└── mapper/         # MapStruct mappers
```

- `controller → service → repository`. A controller never injects a repository.
- JPA entities never leave the service layer: controllers receive and return DTOs.
- A module uses another **through its service interface**, never through its repository or entities.
- `shared`, `config` and `security` do not depend on any business module.
- No circular dependencies between modules: decouple with an application event, not with `@Lazy`.

## Build order

Build one module at a time, in this order. Do not start work that belongs to a later module.

0. **Bootstrap** (`docs/bootstrap.md`) · 1. **Auth** · 2. **Members** · 3. **Catalog** · 4. **Loans** ·
5. **Sanctions** · 6. **Loan requests** · 7. **Dashboard / reports** · 8. **Infrastructure, testing, docs**

Notifications are cross-cutting. The RAG assistant is added last, as its own module.

## Code conventions

- **Language**: code, comments, logs, commits and documentation in **English**. Text shown to end
  users (validation and error messages, emails) in **Spanish**, always in `messages.properties`.
- **Java 25**: DTOs are `record`s. Pattern matching and `sealed` types where they make code clearer.
  `var` only when the type is obvious on the same line. No preview features.
- **Injection**: constructor injection with `final` fields. Never `@Autowired` on fields.
- **Services**: every service is an interface (`BookService`) plus one implementation
  (`BookServiceImpl` in `service/impl/`). Everything else depends on the interface.
- **Entities are anemic**: fields, mappings, getters and setters. Business rules, state transitions and
  invariants live in services, never in entities, controllers or mappers.
- **Time**: inject `Clock`; never call `Instant.now()` or `LocalDate.now()` without it.
- **Configuration**: typed `@ConfigurationProperties` records under `libryx.*`. No magic numbers.
- **Logging**: `@Slf4j`, English, `{}` placeholders. Log each fact once, with ids. **Never** log
  passwords, tokens, codes, `Authorization`/`Cookie` headers or personal data.
- **Lombok**: `@Getter`, `@Setter`, `@RequiredArgsConstructor`, `@Builder`, `@Slf4j`. On entities never
  use `@Data`, `@EqualsAndHashCode` or `@ToString` (they break with lazy relations).
- **MapStruct**: one mapper per aggregate, `componentModel = "spring"`,
  `unmappedTargetPolicy = ReportingPolicy.ERROR`, no business logic.
- **Naming**: `BookController`, `BookService`, `BookServiceImpl`, `BookRepository`, `Book`, `BookMapper`;
  DTOs `BookCreateRequest`, `BookUpdateRequest`, `BookResponse`, `BookSummaryResponse`; exceptions
  `BookNotFoundException`, `LoanLimitExceededException`.

### Spring Boot 4, not 3

Do not copy Boot 3 idioms, imports or starters.

- Starters: `spring-boot-starter-webmvc` (was `-web`), `-aspectj` (was `-aop`), and Flyway needs
  `spring-boot-starter-flyway`. Each technology has its own test starter (`-webmvc-test`, `-data-jpa-test`,
  `-security-test`): a missing test annotation usually means a missing test starter.
- Jackson 3: packages `tools.jackson.*` (annotations stay in `com.fasterxml.jackson.annotation`).
  Customize with `JsonMapper` or `JsonMapperBuilderCustomizer`, not an `ObjectMapper` bean.
- Tests: `@MockitoBean` and `@MockitoSpyBean` replace `@MockBean` and `@SpyBean`. `@SpringBootTest`
  needs `@AutoConfigureMockMvc` to get a `MockMvc`.
- Nullability with JSpecify annotations. Only `jakarta.*`, never `javax.*`.
- When unsure about an API, check the Boot 4 documentation before using it.

## Design principles

Criteria for deciding, not an excuse to add abstractions. When in doubt, the simplest solution wins.

- **Single responsibility**: controllers translate HTTP, services apply the rules of one aggregate,
  mappers convert. Split a service that passes ~300 lines or ~7 dependencies by business case.
- **Open/closed**: resolve variants with polymorphism (one `ReportExporter` per format, one template
  per notification type), not with a growing `if`/`switch`. An exhaustive `switch` on an enum is fine.
- **Liskov**: every implementation honours the full contract. No `UnsupportedOperationException`.
- **Interface segregation**: small interfaces and case-specific DTOs instead of one with nullable fields.
- **Dependency inversion**: depend on interfaces at every boundary; always inject through the constructor.
- **KISS, YAGNI, DRY**: build what the task asks. Each business rule lives in one place. Wait for the
  third repetition before abstracting.
- **Fail fast**: validate at the edge and check preconditions first. Specific exceptions; never swallow one.
- **No `null` results**: lookups return `Optional`, collections come back empty. `Optional` is never a
  parameter or a field. Prefer immutable values (`record`, `final`, `List.copyOf`).
- **Clean code**: intention-revealing names, short methods, early returns, no boolean flag parameters,
  comments that explain why, no dead or commented-out code.

## Workflow

- Conventional Commits in English: `feat(loan): add renewal endpoint`. Branches `feature/<module>-<topic>`,
  `fix/<topic>`, `chore/<topic>`. Do not commit or push unless asked.
- Branches start from `develop` and merge back into it. `main` only receives `develop` once it builds
  and its tests pass. Never commit directly to `main` or `develop`.
- Small changes focused on one module. No unrequested refactors outside the task.
- Documentation changes travel with the code they describe. The README must be enough to run the
  project from scratch. Endpoints live in OpenAPI and the data model in migrations: do not duplicate them.
- Record architecture, dependency, security-model or data-model decisions as an ADR in
  `docs/adr/NNNN-kebab-case-title.md`, from `docs/adr/0000-template.md`. Propose it before implementing.
  An accepted ADR is never edited: a new one supersedes it.

## Definition of done

- [ ] `./mvnw clean verify` is green
- [ ] Flyway migration included when the model changed
- [ ] DTOs are validated records mapped with MapStruct; no entity exposed
- [ ] `@PreAuthorize` on every new operation, with tests for an allowed and a denied role
- [ ] Business rules covered by tests, including boundary cases
- [ ] New endpoints documented in OpenAPI
- [ ] New properties, error codes and metrics added to their document in `docs/`
- [ ] User-facing text in `messages.properties`, in Spanish
- [ ] No secrets, no personal data in logs, no magic numbers

## Never

- Edit an applied Flyway migration, or set `ddl-auto` to anything but `validate`.
- Return a JPA entity from a controller, or inject a repository into one.
- Put business logic in entities, controllers or mappers.
- Expose personal data outside the `ADMIN` endpoints, or write it to logs, exports without audit, or the assistant.
- Hardcode business limits, secrets, URLs or CORS origins.
- Call a real AI provider from a test, or run `terraform apply` / `destroy`.
- Lower the coverage threshold, skip CI, disable a test or change a business rule to make a build pass.
- Use Boot 3 APIs, imports or starters (`@MockBean`, `com.fasterxml.jackson.databind`, `spring-boot-starter-web`).
- Introduce layers, patterns or dependencies this guide does not cover without asking first.
