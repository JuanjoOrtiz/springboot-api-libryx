---
paths:
  - "src/main/java/**/job/**"
---

# Scheduled job rules

Frequencies are properties under `libryx.jobs.*`, never fixed in the annotation.

| Job | Frequency | Module |
|---|---|---|
| Send pending notifications (outbox) | every minute | `notification` |
| Expire `READY` requests after 48 hours | every 15 minutes | `loan` |
| Mark overdue loans (`OVERDUE`) | daily, 00:05 | `loan` |
| Expire finished sanctions | daily, 00:10 | `sanction` |
| Notify loans about to be due (`DUE_SOON`) | daily, 09:00 | `loan` |
| Delete expired verification and recovery codes | daily, 03:00 | `auth` |

- Daily jobs run in the library time zone (`libryx.time-zone`, `Europe/Madrid`), set in the `cron`'s `zone`.
- A job class lives in the `job/` package of the module that owns the data and **only delegates to
  that module's service interface**. No business logic in the scheduled class.
- Every job is **idempotent**: running it twice in a row never duplicates sanctions, notices or state changes.
- Process in batches, one transaction per batch. One failing record does not stop the rest.
- Each run logs its start and end at `INFO` with the number of records processed, uses its own
  correlation id, and records the `libryx.jobs.execution` timer.
- The application is assumed to run as **a single instance**. Scaling to several needs a distributed
  lock first: flag it, do not add one on your own.
- Jobs are disabled in tests by property (`libryx.jobs.enabled=false`) and tested by calling the
  service with a fixed `Clock`.
