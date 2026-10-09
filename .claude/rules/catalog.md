---
paths:
  - "src/main/java/**/catalog/**"
---

# Catalog rules

- **Work ≠ copy**: `books` is the title; `book_copies` is the physical item that gets lent.
- The catalog requires a signed-in user, like the rest of the API. Writes need `LIBRARIAN` or `ADMIN`.
- Text search uses the `FULLTEXT` index `ft_books_search`.
- **ISBN**: input may contain hyphens; it is stored as the 13 digits only, and its check digit is
  validated. A duplicate ISBN answers 409 `DUPLICATE_RESOURCE`.

## Works, authors and categories

- **Covers** are a URL in `books.cover_url`; the API stores no images. The field is optional: when it
  is empty and the work has an ISBN, the service fills it from `libryx.catalog.cover-url-template`
  (Open Library covers by ISBN). Only `https` URLs are accepted. The frontend shows a generic cover
  when the image does not load.

- A work is **never deleted**: `books.active = false` deactivates it.
- A work cannot be deactivated while it has open loans or open requests: 409 `INVALID_STATE`.
- A `USER` never sees inactive works, neither in searches nor by id (404). Staff see them with
  `?active=false`.
- An author, publisher or category linked to works cannot be deleted (the database `RESTRICT`s it):
  409 `INVALID_STATE`.

## Copies

- Only staff change a copy's status.
- `MAINTENANCE` and `WITHDRAWN` apply only to an `AVAILABLE` copy, never to one on loan or set aside.
- `ON_LOAN` and `ON_HOLD` are set only by the loan and request flows, never by hand.

## Cache

Spring Cache (`@Cacheable`, `@CacheEvict`) on Redis. TTLs under `libryx.cache.*`.

| Cache | Content | TTL |
|---|---|---|
| `book-detail` | One work by id | 10 min |
| `catalog-reference` | Categories, authors and publishers | 1 hour |
| `catalog-search` | Result pages of searches and listings | 2 min |

- Cache **response DTOs**, never JPA entities. JSON serialization, not native Java serialization.
- **Availability is never cached**: free copies, a copy's status and the request queue are always
  read from the database. If a detail view shows availability, compute it apart from the cached DTO.
- Only the catalog is cached. Nothing about users, loans, requests or sanctions, and nothing that
  depends on who is asking. **Never personal data.**
- Only the default view is cached, the same for everyone. Searches with the `available` filter or
  with `active=false` are never cached.
- Every catalog write carries its `@CacheEvict`. Eviction happens **after commit**, with a
  transaction-aware cache manager:
  - a work: its `book-detail` entry and all of `catalog-search` (`allEntries = true`);
  - an author, publisher or category: all of `catalog-reference` and `catalog-search`;
  - a copy: nothing, because availability is never cached.
- `@Cacheable` goes on public methods of the service implementation, called from another bean.
- Cache names are constants and keys are explicit. Prefix `libryx:cache:` keeps them apart from
  token and rate-limit keys.
- If Redis fails, the read falls back to the database and logs a `WARN`: a cache failure never breaks
  the catalog.
- Each cache has a test for a hit and for eviction after a write.
