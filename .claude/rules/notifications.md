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
- Channels `EMAIL` and `IN_APP`. Failed sends are retried and record `attempts` and `last_error`.
- Adding a notification type requires a migration that updates the `CHECK` on `notifications.type`.
- Locally all mail goes to Mailpit. The mail server is not part of the application's health.
- Record `libryx.notifications.sent` with `type`, `channel` and `result` tags.
