---
paths:
  - "src/main/java/**/catalog/**"
---

# Catalog rules

- **Work ≠ copy**: `books` is the title; `book_copies` is the physical item that gets lent.
- The catalog requires a signed-in user, like the rest of the API. Writes need `LIBRARIAN` or `ADMIN`.
- Text search uses the `FULLTEXT` index `ft_books_search`. ISBN is stored as 13 digits without hyphens.
- `books.active` hides a work from the catalog. Prefer deactivating a work over deleting it.

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
- Every catalog write (create, edit or deactivate a work, author, publisher or category) carries its
  `@CacheEvict`. Eviction happens **after commit**, with a transaction-aware cache manager.
- `@Cacheable` goes on public methods of the service implementation, called from another bean.
- Cache names are constants and keys are explicit. Prefix `libryx:cache:` keeps them apart from
  token and rate-limit keys.
- If Redis fails, the read falls back to the database and logs a `WARN`: a cache failure never breaks
  the catalog.
- Each cache has a test for a hit and for eviction after a write.
