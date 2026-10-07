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
- Every protected endpoint has a test for an allowed role and at least one denied role.
- Every rate limit has a test that checks the 429.
- Scheduled jobs are disabled (`libryx.jobs.enabled=false`) and tested through their service.
- AI is replaced by doubles of `ChatModel` and `EmbeddingModel`: no provider call, no model load.
- Each test builds its own data with builders or fixtures. Tests never use the local seed data.
- Coverage with JaCoCo in `./mvnw verify`: **at least 80 % of lines in `service` packages**. Below
  that the build fails. Do not add exclusions or lower the threshold to reach it.
- Never disable, delete or weaken a test to make the build pass: fix the cause.

## Spring Boot 4

- `@MockitoBean` and `@MockitoSpyBean`; `@MockBean` and `@SpyBean` no longer exist.
- `@SpringBootTest` does not provide `MockMvc` by itself: add `@AutoConfigureMockMvc`.
- Each technology has its own test starter (`spring-boot-starter-webmvc-test`,
  `spring-boot-starter-data-jpa-test`, `spring-boot-starter-security-test`). `@WithMockUser` needs the
  security test starter.
- Test-slice annotations moved packages in Boot 4: let the compiler resolve the import instead of
  copying a Boot 3 one.
