---
paths:
  - "src/test/**"
---

# Testing rules

| Type | Tools | Covers |
|---|---|---|
| Unit | JUnit Jupiter, Mockito, AssertJ | Service implementations: every business rule and its boundaries |
| Web | `@WebMvcTest` + Spring Security Test | Controllers: validation, status codes, role authorization |
| Persistence | `@DataJpaTest` + Testcontainers | Repositories, custom queries, database constraints |
| Integration | `@SpringBootTest` + Testcontainers | Full flows: request → loan → return → sanction |

- **No H2.** The schema depends on MariaDB (`VECTOR`, generated columns, `CHECK`, `FULLTEXT`).
  Persistence and integration tests run against MariaDB 11.8 and Redis with Testcontainers.
- Unit tests are `*Test`; integration tests are `*IT` and run in `verify` through Failsafe.
- Test the implementation class (`LoanServiceImplTest`), mocking the interfaces it depends on.
- Method names `shouldResult_whenCondition`. Structure given / when / then.
- Control dates with a fixed `Clock`, never with sleeps or the real time.
- Every business rule has a test for the valid case, the boundary and the rejection.
- Every protected endpoint has tests for an allowed role, a denied role (403), an anonymous request
  (401) and, when it has an owner, another user's resource (404).
- Web tests run with the real security configuration and mock only the token service. Never disable
  the filters (`addFilters = false`) to make a test pass.
- Every rate limit has a test that checks the 429.
- Scheduled jobs are disabled (`libryx.jobs.enabled=false`) and tested through their service.
- AI is replaced by doubles of `ChatModel` and `EmbeddingModel`: no provider call, no model load.
- Each test builds its own data. Builders live in one class per module, in
  `src/test/java/com/libryx/<module>/testdata/` (`BookTestData.aBook()`). Tests never use the local seed data.
- Coverage with JaCoCo in `./mvnw verify`: **at least 80 % of lines in `service` packages**. Below
  that the build fails. Do not add exclusions or lower the threshold to reach it.
- Never disable, delete or weaken a test to make the build pass: fix the cause.

## Test database

- The schema is created by Flyway when the test context starts, with the same migrations as
  production. No seed data, no `ddl-auto` other than `validate`.
- Containers start **once per test run** and are shared: static containers in one shared
  configuration, connected with `@ServiceConnection`. Never one container per test class.
- `@DataJpaTest` must use the container, never an embedded database: if Spring tries to replace the
  datasource, stop it with `@AutoConfigureTestDatabase(replace = NONE)`.
- Tests do not depend on each other's data: each one cleans up or runs in a transaction that rolls back.

## Spring Boot 4

- `@MockitoBean` and `@MockitoSpyBean`; `@MockBean` and `@SpyBean` no longer exist.
- `@SpringBootTest` does not provide `MockMvc` by itself: add `@AutoConfigureMockMvc`.
- Each technology has its own test starter (`spring-boot-starter-webmvc-test`,
  `spring-boot-starter-data-jpa-test`, `spring-boot-starter-security-test`). `@WithMockUser` needs the
  security test starter.
- Test-slice annotations moved packages in Boot 4: let the compiler resolve the import instead of
  copying a Boot 3 one.
