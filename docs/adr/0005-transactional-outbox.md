# ADR-0005: Transactional outbox for notifications

- **Status:** Accepted
- **Date:** 2026-10-09

## Context

Business events (request ready, overdue loan, sanction…) must notify the user by email and in the app.
Sending an email inside the business transaction can notify about a change that is then rolled back,
or block the transaction on a slow SMTP server. During a deployment two API tasks coexist briefly.

## Decision

- The `notifications` row is inserted **in the same transaction** as the event; it stores the data
  (`template_data`), not the rendered email.
- A scheduled job sends pending `EMAIL` notices after commit, claiming rows with
  `SELECT … FOR UPDATE SKIP LOCKED`, retrying up to `libryx.notifications.max-attempts`.
- A deterministic `dedup_key` plus `channel` is unique, so a retry never creates a second notice.
- Codes (verification, recovery, account setup) are the exception: they travel only in memory and are
  sent after commit, so the code is never stored in clear.

## Alternatives considered

| Alternative | Why it was discarded |
|---|---|
| Send inside the transaction | Emails for rolled-back changes, and SMTP latency inside the transaction |
| `@TransactionalEventListener` only | A crash after commit loses the email, with no retry |
| Message broker (RabbitMQ, SQS) | Another piece of infrastructure for a single application |

## Consequences

- **Gains:** no lost or phantom notices, retries, and a history of every notice sent.
- **Costs:** emails leave up to a minute after the event; the table needs cleanup over time.
- **Watch for:** a growing `PENDING` backlog (`libryx.notifications.pending`).
