---
paths:
  - "src/main/java/**/loan/**"
  - "src/main/java/**/sanction/**"
---

# Business rules: loans, loan requests and sanctions

Limits are **configurable** properties under `libryx.*` (see `docs/configuration.md`). Never write
them as literals. If a task forces you to change a rule in this file, **stop and ask**.

| Rule | Value |
|---|---|
| Loan duration | 14 days |
| Active loans per member | 3 at most |
| Renewals per loan | 2 at most, each adds 14 days |
| Open requests per member | 3 at most, and only one per work |
| Pick-up window for a ready request | 48 hours |
| Sanction for a late return | 21 days without borrowing, no fine |
| Due-soon notice | 2 days before the due date |

## Loans

- A librarian creates the loan when handing over the copy. `due_date = loan_date + 14 days`, computed
  in the library time zone.
- A loan or a renewal is **blocked** when the member has an active sanction, or when the work has
  pending requests from other members.
- Every renewal is recorded in `loan_renewals`.
- A loan targets a **copy** (`book_copies`); the copy's status changes with it.

## Loan requests

- Created by the member, or by a librarian on their behalf, **for the work** (`books`), not a copy.
- With no free copy, the request waits in a **FIFO queue** ordered by `requested_at`.
- When a copy is free, a librarian sets it aside: the request becomes `READY` and the 48-hour window starts.
- Handing the copy over creates the loan and marks the request `FULFILLED`. If the window passes, the
  request becomes `EXPIRED` and the copy goes to the next request in the queue.
- Rejecting requires who and why; cancelling requires who and when (the database `CHECK`s enforce it).

## Sanctions

- A late return automatically creates a 21-day sanction.
- `ADMIN` or `LIBRARIAN` can lift it manually, with a **mandatory reason**.
- The schema also allows the reasons `LOST_ITEM`, `DAMAGED_ITEM` and `MANUAL`. Their rules are
  **not defined yet: ask before implementing them**.

## Concurrency

Lending, setting a copy aside and renewing all race. They are protected by optimistic locking
(`@Version`) plus the database `UNIQUE` constraints on `open_flag`. Never replace them with in-memory
checks, and let the global handler turn the conflict into a 409.

## Where the logic goes

Entities are anemic. Every rule above is enforced in the service implementation, one method per state
transition, with a test for the valid case, the boundary and the rejection.
