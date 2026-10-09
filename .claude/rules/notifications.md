---
paths:
  - "src/main/java/**/notification/**"
---

# Notification rules

- **Outbox pattern**: the `notifications` row is inserted in the **same transaction** as the event that
  causes it. Sending happens later, in the outbox job or a
  `@TransactionalEventListener(phase = AFTER_COMMIT)`. Never send email inside a business transaction.
- Other modules ask for a notification through this module's service interface; they never write to
  the table or call the mail sender themselves.
- Store the **data** (`template_data`, JSON), not the rendered email. Subject and body are produced by
  the template class of each type at send time, with texts from `messages.properties`
  (`email.<type>.subject`, `email.<type>.body`).
- One template class per notification type, selected by type (Strategy). Adding a type is adding a class.
- `dedup_key` + `channel` is unique: use a deterministic key (`REQUEST_READY:loan_request:42`) so a
  retry never duplicates a notice.
- **Codes are never stored in clear.** `EMAIL_VERIFICATION`, `PASSWORD_RESET` and `ACCOUNT_SETUP`
  carry the 6-digit code only in memory, inside the event, and are sent after commit. Their
  `notifications` row records the send **without the code**. They are not retried: if the email fails,
  the user asks for a new code.
- `ACCOUNT_SETUP` is the email an `ADMIN`-created account receives to set its first password.

## Channels

| Types | Channels |
|---|---|
| `EMAIL_VERIFICATION`, `PASSWORD_RESET`, `ACCOUNT_SETUP` | `EMAIL` only |
| Every other type | `EMAIL` and `IN_APP` |

- `IN_APP` notices are visible as soon as they are committed. The outbox job processes only `EMAIL`.
- A user lists their own notices and marks them read (`read_at`); never anyone else's.

## Retries and cancellation

- The outbox job claims its rows with `SELECT … FOR UPDATE SKIP LOCKED`: during a deployment two tasks
  coexist briefly, and they must never send the same email twice.

- Failed sends are retried and record `attempts` and `last_error`, up to
  `libryx.notifications.max-attempts`. Then the notice becomes `FAILED` and is logged at `ERROR`.
- A notice that no longer makes sense becomes `CANCELLED` instead of being sent: a pending `DUE_SOON`
  for a loan already returned, or any notice for a deleted or inactive account.
- Adding a notification type requires a migration that updates the `CHECK` on `notifications.type`.
- Locally all mail goes to Mailpit. The mail server is not part of the application's health.
- Record `libryx.notifications.sent` with `type`, `channel` and `result` tags.
