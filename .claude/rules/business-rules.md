---
paths:
  - "src/main/java/**/loan/**"
  - "src/main/java/**/sanction/**"
---

# Business rules: loans, loan requests and sanctions

Limits are **configurable** properties under `libryx.*` (see `docs/configuration.md`). Never write
them as literals. If a task forces you to change a rule in this file, **stop and ask**.

The domain says "user" (role `USER`); the database columns keep the name `member_id`.

| Rule | Value |
|---|---|
| Loan duration | 14 days |
| Active loans per user | 3 at most (`ACTIVE` and `OVERDUE` both count) |
| Renewals per loan | 2 at most, each adds 14 days |
| Open requests per user | 3 at most, and only one per work |
| Pick-up window for a ready request | 48 hours |
| Sanction for a late return | 21 days without borrowing, no fine |
| Active sanctions per user | 1 at most |
| Due-soon notice | 2 days before the due date |

## Loans

- A librarian creates the loan when handing over the copy. `due_date = loan_date + 14 days`, computed
  in the library time zone.
- A loan targets a **copy** (`book_copies`); the copy's status changes with it.
- A new loan is **blocked** when the user:
  - has an active sanction;
  - has an `OVERDUE` loan;
  - already has the maximum number of active loans;
  - or when the work has open requests from other users. **Exception**: the loan fulfils the user's
    own `READY` request for that same copy.
- Returning a loan after its `due_date` marks it `RETURNED_LATE` and applies the sanction below.

## Renewals

- A renewal adds 14 days to the **current** `due_date`, not to today.
- Only before the due date: an `OVERDUE` loan cannot be renewed.
- Blocked when the user has an active sanction, when the loan has reached the maximum number of
  renewals, or when the work has open requests from other users.
- A `USER` renews only their own loans; a librarian can renew any loan.
- Every renewal is recorded in `loan_renewals`.

## Loan requests

- Created by the user, or by a librarian on their behalf, **for the work** (`books`), not a copy.
- Blocked when the user has an active sanction, already has the maximum number of open requests, or
  currently has that work on loan. Open requests survive a later sanction.
- With no free copy, the request waits in a **FIFO queue** ordered by `requested_at`.
- When a copy is free, a librarian sets it aside: the request becomes `READY` and the 48-hour window starts.
- Handing the copy over creates the loan and marks the request `FULFILLED`.
- If the window passes, the scheduled job marks the request `EXPIRED` and sets the copy aside for the
  next `PENDING` request of the queue (`READY`, new 48-hour window, `REQUEST_READY` notice). With no
  one in the queue, the copy becomes `AVAILABLE`.
- A `USER` cancels their own `PENDING` or `READY` requests; cancelling a `READY` one frees its copy
  the same way as an expiry.
- Only staff (`LIBRARIAN`, `ADMIN`) reject requests.
- Rejecting requires who and why; cancelling requires who and when (the database `CHECK`s enforce it).

## Sanctions

- A late return automatically sanctions the user for 21 days, starting on the return date in the
  library time zone. `end_date` is the **first day the user can borrow again**: the sanction is active
  while today < `end_date`.
- **A user has at most one active sanction.** A late return while one is active does not create a
  second one: it extends the current `end_date` by 21 days. The database enforces it with
  `UNIQUE (member_id, open_flag)` on `sanctions`.
- An active sanction blocks loans, renewals and new requests.
- `ADMIN` or `LIBRARIAN` can lift it manually, with a **mandatory reason**.
- The schema also allows the reasons `LOST_ITEM`, `DAMAGED_ITEM` and `MANUAL`, and the `LOST` status
  for loans and copies. Their rules are **not defined yet: ask before implementing them**.

## Concurrency

Lending, setting a copy aside and renewing all race. They are protected by optimistic locking
(`@Version`) plus the database `UNIQUE` constraints on `open_flag`. Never replace them with in-memory
checks, and let the global handler turn the conflict into a 409.

## Where the logic goes

Entities are anemic. Every rule above is enforced in the service implementation, one method per state
transition, with a test for the valid case, the boundary and the rejection.
