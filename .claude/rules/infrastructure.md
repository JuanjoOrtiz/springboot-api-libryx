---
paths:
  - "infra/**"
  - ".github/**"
  - "Dockerfile"
  - "compose.yaml"
  - "monitoring/**"
---

# Infrastructure rules

Architecture, secrets and Terraform conventions: `docs/deployment-aws.md`.

## Continuous integration

GitHub Actions, in `.github/workflows/ci.yml`.

- Runs on every pull request and on every push to `develop` and `main`.
- Steps: JDK 25, Maven cache and `./mvnw -B clean verify` (unit and integration tests with
  Testcontainers, plus the coverage check). Publishes the test and JaCoCo reports.
- When `infra/` changes, a second job runs `terraform fmt -check` and `terraform validate`.
- CI **does not deploy**, uses no AWS credentials or AI keys, and needs no secret.
- A pull request is not merged with CI red. Reproduce a CI failure locally with `./mvnw clean verify`
  and fix the cause.
- Do not edit the workflow to make it pass: no `-DskipTests`, no `continue-on-error`, no disabled tests.

## Docker

- `compose.yaml` defines `mariadb` (11.8), `redis`, `mailpit` and `flyway` (one-shot), plus `prometheus`
  and `grafana` for local dashboards (configuration in `monitoring/`, see `docs/observability.md`). If a service
  name changes, update the commands in `CLAUDE.md`.
- The application `Dockerfile` is multi-stage, runs on a Java 25 JRE as a non-root user, and includes
  the embedding model files.
- Ollama is not part of Compose: the `local` profile uses the one installed on the machine.

## AWS

- Deployment exists **as code and documentation** in `infra/terraform/`. It is not deployed.
- Architecture: ALB with HTTPS → one **ECS Fargate** service → **RDS MariaDB 11.8** and **ElastiCache**.
  Secrets in Secrets Manager, mail through SES, logs in CloudWatch.
- **Never run `terraform apply`, `destroy` or `import`**, or any AWS CLI command that creates or
  changes resources. The code must pass `terraform fmt -check` and `terraform validate`.
- No real account ids, ARNs, domains or secrets in the code: variables and placeholders.
- The application constrains the infrastructure:
  - **One API task**, no autoscaling: scheduled jobs assume a single instance.
  - **Migrate before deploying**: Flyway runs as a one-off task and the API only validates the schema.
  - Migrations are compatible with the previous API version; an incompatible change takes two deployments.
  - The load balancer health check is `/actuator/health/readiness` on the management port (8081).
  - API and frontend share the same registrable domain, because of the `SameSite=Strict` refresh cookie.
- A change to environment variables, health, logs or startup is mirrored in `infra/` and its document.
