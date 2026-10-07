# Deployment on AWS

Reference for `.claude/rules/infrastructure.md`. Read it before touching `infra/` or anything that
constrains deployment (environment variables, health, logs, migrations, scheduled jobs).

**Scope:** the deployment exists as code and documentation for the portfolio. **It is not deployed.**
Nothing in this repository may be run against a real AWS account unless explicitly asked.

## 1. Architecture

```
                    Internet
                       │
               ┌───────▼────────┐
               │ ALB (HTTPS,ACM)│            public subnets (2 AZs)
               └───────┬────────┘
                       │
        ┌──────────────▼───────────────┐
        │ ECS Fargate — API service    │     private subnets (2 AZs)
        │ (1 task)                     │──── NAT ──► Anthropic, SES, ECR
        └───┬──────────────┬───────────┘
            │              │
   ┌────────▼─────┐  ┌─────▼───────────┐
   │ RDS MariaDB  │  │ ElastiCache     │     private data subnets
   │ 11.8         │  │ (Redis)         │
   └──────────────┘  └─────────────────┘
```

| Piece | AWS service | Notes |
|---|---|---|
| API image | ECR | Tagged with the commit SHA; never `latest` in a task definition |
| API | ECS Fargate | One service, in private subnets, behind the ALB |
| Migrations | ECS Fargate, one-off task | The same one-shot Flyway task as in Docker Compose |
| Ingress | Application Load Balancer + ACM | HTTPS only; port 80 redirects to 443 |
| Database | RDS for MariaDB 11.8 | 11.8 is required for the assistant's `VECTOR` type |
| Redis | ElastiCache | Cache, refresh tokens, denylist and rate limiting |
| Secrets | Secrets Manager | Injected into the task as environment variables |
| Mail | Amazon SES (SMTP interface) | Replaces Mailpit; same `spring.mail.*` properties |
| Logs | CloudWatch Logs | The application already writes JSON to standard output |
| Outbound internet | NAT Gateway | For ECR, SES and the Anthropic API |

The Angular frontend is deployed from its own repository (S3 + CloudFront) and is outside this document.

## 2. What the application requires from the infrastructure

These points come from rules in `CLAUDE.md` and `.claude/rules/`. Do not break them when changing
the infrastructure.

- **A single API task** (`desired_count = 1`, no autoscaling). Scheduled jobs assume one instance;
  with more they would run twice. Scaling needs a distributed lock first.
- **Migrate before deploying.** Every deployment runs the Flyway task and waits for it to succeed;
  only then is the service updated. The API starts with `ddl-auto=validate` and does not migrate.
- **Health check:** the ALB calls `/actuator/health/readiness`. It depends on MariaDB and Redis, not on mail.
- **Same site for API and frontend.** The refresh token travels in a `SameSite=Strict` cookie: the API
  and the frontend must hang from the same registrable domain (`api.<domain>` and `app.<domain>`).
- **`prod` profile** set in the task definition, with `LIBRYX_SECURITY_CORS_ALLOWED_ORIGINS` pointing
  at the frontend domain.
- **Assistant:** in `prod` the chat uses Claude through its API (`spring.ai.model.chat=anthropic`);
  Ollama is for `local` only and is not deployed. Embeddings are computed by an ONNX model inside the
  task itself: its files ship **inside the image** and the task memory is sized with it in mind.
- **Time zone:** container and database in UTC. The business time zone comes from `libryx.time-zone`.
- **Encryption in transit** to RDS and ElastiCache, and at rest in both.
- **No public access to data:** RDS and ElastiCache accept connections only from the API's security group.

## 3. Secrets and configuration

| Data | Lives in | Reaches the application as |
|---|---|---|
| Database password | Secrets Manager | `SPRING_DATASOURCE_PASSWORD` |
| JWT signing key | Secrets Manager | `LIBRYX_SECURITY_JWT_SECRET` |
| Anthropic key | Secrets Manager | `SPRING_AI_ANTHROPIC_API_KEY` |
| SES SMTP credentials | Secrets Manager | `SPRING_MAIL_USERNAME`, `SPRING_MAIL_PASSWORD` |
| Everything else | Task definition | Plain environment variables |

- No secret in `.tf` files, versioned `.tfvars`, task definitions or Terraform outputs.
- The task execution role can read only Libryx's secrets, not every secret in the account.
- The task role follows least privilege: no administrative permissions.

## 4. Infrastructure code

Terraform, in `infra/terraform/`.

```
infra/terraform/
├── modules/
│   ├── network/      # VPC, subnets, NAT, security groups
│   ├── registry/     # ECR
│   ├── database/     # RDS MariaDB
│   ├── cache/        # ElastiCache
│   ├── secrets/      # Secrets Manager
│   ├── service/      # ECS: cluster, task definitions (API and Flyway), service, roles
│   └── ingress/      # ALB, certificate, rules
└── envs/
    └── prod/         # module composition, variables and state backend
```

Conventions:

- One module per piece, with `variables.tf`, `main.tf` and `outputs.tf`. Variables have `description` and `type`.
- Nothing hardcoded: region, domain, instance sizes and account ids are variables.
- Remote state with locking. **Never** version a `terraform.tfstate` or a `.tfvars` with real values;
  version `terraform.tfvars.example`.
- Terraform and provider versions are pinned.
- Every resource carries the tags `Project = "libryx"` and `Environment`.
- Defaults aim at minimum cost (small instances, a single zone for data), with variables to harden
  them (`multi_az`, `deletion_protection`).
- The code passes `terraform fmt -check` and `terraform validate`, which need no credentials.

### What is not done

- **Do not run `terraform apply`, `destroy` or `import`**, or AWS CLI commands that create or modify
  resources. `plan` only when asked and credentials exist.
- Do not write real account ids, ARNs, domains or addresses: use variables or placeholders.
- Do not add AWS services that are not in the architecture table without an ADR.

## 5. Deployment flow (documented, not automated)

1. Continuous integration builds and tests (`./mvnw clean verify`).
2. The image is built and pushed to ECR with the commit SHA.
3. The Flyway task runs and must finish successfully.
4. A new task definition revision is registered with that image.
5. The ECS service is updated and the new task must pass the health check.
6. If it does not, the service rolls back to the previous revision (deployment circuit breaker with rollback enabled).

During step 5 the old task and the new one coexist for a moment. That is why scheduled jobs must be
idempotent even though the service has a single task.

Migrations must be **backward compatible** with the previous API version: during step 5 the new
schema and the old code coexist. An incompatible change is split across two deployments.
