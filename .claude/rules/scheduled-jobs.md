---
paths:
  - "src/main/java/**/job/**"
---

# Scheduled job rules

Frequencies are properties under `libryx.jobs.*`, never fixed in the annotation.

| Job | Frequency | Module |
|---|---|---|
| Send pending notifications (outbox) | every minute | `notification` |
| Expire `READY` requests after 48 hours and set the copy aside for the next in the queue | every 15 minutes | `loan` |
| Mark overdue loans (`OVERDUE`) and send the `OVERDUE` notice | daily, 00:05 | `loan` |
| Expire finished sanctions | daily, 00:10 | `sanction` |
| Notify loans about to be due (`DUE_SOON`) | daily, 09:00 | `loan` |
| Delete expired verification and recovery codes | daily, 03:00 | `auth` |

- Daily jobs run in the library time zone (`libryx.time-zone`, `Europe/Madrid`), set in the `cron`'s `zone`.
- A job class lives in the `job/` package of the module that owns the data and **only delegates to
  that module's service interface**. No business logic in the scheduled class.
- Every job is **idempotent**: running it twice in a row never duplicates sanctions, notices or state
  changes. Notices use a deterministic `dedup_key` (`DUE_SOON:loan:42`, `OVERDUE:loan:42`).
- **Correctness never depends on a job having run.** Services decide by date, not only by the stored
  status: a loan past its `due_date` already counts as overdue, and a sanction whose `end_date` has
  arrived no longer blocks. Jobs only bring the stored status up to date and send the notices.
- Jobs select by condition (`due_date < today`), never by "yesterday", so a missed run is caught up by
  the next one.
- Process in batches of `libryx.jobs.batch-size` records, one transaction per batch. One failing record
  does not stop the rest.
- Each run logs its start and end at `INFO` with the number of records processed, uses its own
  correlation id, and records the `libryx.jobs.execution` timer.
- The application is assumed to run as **a single instance**. Scaling to several needs a distributed
  lock first: flag it, do not add one on your own.
- Jobs are disabled in tests by property (`libryx.jobs.enabled=false`) and tested by calling the
  service with a fixed `Clock`.
