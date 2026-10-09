# API contract

Reference for `.claude/rules/api.md`. This is the agreement with the Angular frontend: read it before
creating or changing an endpoint, a DTO or an error.

The list of endpoints does not live here: OpenAPI is the source of truth (`/v3/api-docs`, Swagger UI).

## 1. General format

- JSON in UTF-8. Properties in `camelCase`.
- Numeric identifiers (`Long`). Enums travel as upper-case strings, exactly as in the database
  (`"ACTIVE"`, `"ON_LOAN"`).
- A nested resource is returned as a summary (`BookSummaryResponse`) when the screen needs it, not in
  full and not as a bare id; otherwise only its id.
- The API has no version prefix. Avoid breaking changes; if one is unavoidable, ask first.
- Compatible changes: adding an optional field to a response, an optional parameter or an endpoint.
  Breaking changes: removing or renaming a field, changing a type, a status code or an error `code`.
- Every endpoint requires an access token, except the authentication endpoints listed in
  `.claude/rules/security.md`.
- Updates use `PUT` with the full DTO. `PATCH` is not used.

## 2. Dates and times

| Kind | Format | Example | Java type |
|---|---|---|---|
| Instant | ISO-8601 in UTC with `Z` | `2026-10-05T09:30:00Z` | `Instant` |
| Business date | ISO-8601, date only | `2026-10-19` | `LocalDate` |

- Never numeric timestamps, never a date-time without a zone.
- The frontend converts instants to local time for display. Business dates (`dueDate`, sanction start
  and end) are already in the library time zone and are shown as they come.

## 3. Pagination

Every list is paginated. Query parameters:

| Parameter | Default | Limits | Description |
|---|---|---|---|
| `page` | `0` | ≥ 0 | Page number, **starting at 0** |
| `size` | `20` | 1–100 | Items per page. Above 100 it is capped at 100 |
| `sort` | per endpoint | repeatable | `field,asc` or `field,desc` |

Response — `PageResponse<T>`, a record in `shared`:

```json
{
  "content": [
    { "id": 12, "title": "La Regenta", "isbn": "9788437600956" }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 134,
  "totalPages": 7,
  "first": true,
  "last": false
}
```

- Never return Spring Data's `Page<T>` directly: its JSON is not a stable contract.
- An empty list answers 200 with `content: []` and `totalElements: 0`.
- Each endpoint declares its default order, always deterministic (it ends by breaking ties on `id`).

## 4. Sorting

- `sort=title,asc&sort=publicationYear,desc`. The default direction is `asc`.
- Each endpoint has a **closed list** of sortable fields, documented in OpenAPI. A field outside the
  list answers 400; a client value never reaches the query unvalidated.
- Names are the DTO's (`camelCase`), not the column's.

## 5. Filters

- Query parameters in `camelCase`, all optional, combined with **AND**.
- Collected in one record per endpoint (`BookSearchCriteria`), validated with Bean Validation.

| Case | Convention | Example |
|---|---|---|
| Free text | `q` | `/api/books?q=quijote` |
| Exact match | the field name | `/api/books?categoryId=4&language=es` |
| Several values (OR) | repeated parameter | `/api/loans?status=ACTIVE&status=OVERDUE` |
| Date range | `from` and `to`, both inclusive | `/api/loans?from=2026-09-01&to=2026-09-30` |
| Boolean | `true` or `false` | `/api/books?available=true` |

- An invalid value (unknown enum, malformed date, `from` after `to`) answers 400.
- `q` is between 2 and 100 characters long.
- A `USER` never filters by someone else: the server imposes their own id, whatever the request says.

## 6. Success codes

Every endpoint declares its success code explicitly, in the code and in its `@ApiResponse`.

| Code | When | Body |
|---|---|---|
| 200 OK | `GET`; `PUT` returning the updated resource; actions that return a result without creating a resource | Response DTO |
| 201 Created | `POST` that creates a resource | Created DTO + `Location` header with its URI |
| 202 Accepted | Work accepted that finishes in the background | `{ "id", "status" }` + `Location` header where the status can be checked |
| 204 No Content | `DELETE` and actions with nothing to return | Empty |

Examples:

| Request | Code | Why |
|---|---|---|
| `GET /api/books?q=quijote` | 200 | A page of results, even when it is empty |
| `GET /api/books/12` | 200 | One resource |
| `POST /api/books` | 201 | Creates a work; `Location: /api/books/12` |
| `PUT /api/books/12` | 200 | Returns the updated work |
| `GET /api/users/me` | 200 | The signed-in user's own profile |
| `PUT /api/users/me` | 200 | Returns the updated profile |
| `POST /api/admin/users` | 201 | Creates an account; the person sets the password with an `ACCOUNT_SETUP` code |
| `DELETE /api/authors/7` | 204 | Nothing to return |
| `POST /api/loans` | 201 | Creates a loan |
| `POST /api/loans/{id}/renewals` | 201 | Creates a renewal |
| `POST /api/loan-requests/{id}/cancel` | 200 | Returns the cancelled request |
| `POST /api/notifications/{id}/read` | 204 | Marks the user's own notice as read |
| `POST /api/assistant/chat` | 200 | Returns the answer and its sources |
| `POST /api/sanctions/{id}/lift` | 200 | Returns the lifted sanction; nothing new is created |
| `POST /api/auth/register` | 201 | Creates the user account |
| `POST /api/auth/verify-email` | 204 | The account is now verified; nothing to return |
| `POST /api/auth/verify-email/resend` | 204 | Same answer whether or not the email exists |
| `POST /api/auth/login` | 200 | Returns the access token; the refresh token goes in a cookie |
| `POST /api/auth/refresh` | 200 | Returns a new access token and rotates the cookie |
| `POST /api/auth/logout` | 204 | Nothing to return |
| `POST /api/auth/password-reset` | 204 | Same answer whether or not the email exists |
| `POST /api/auth/password-reset/confirm` | 204 | The password is changed; nothing to return |
| `GET /api/exports/loans?format=PDF` | 200 | The file, with `Content-Disposition: attachment` |
| `POST /api/admin/assistant/reindex` | 202 | Re-indexing runs in the background |

Login and refresh body:

```json
{ "accessToken": "eyJhbGciOiJIUzI1NiJ9…", "tokenType": "Bearer", "expiresIn": 900 }
```

`expiresIn` is in seconds, so the frontend can refresh before the token expires.

- An empty list is **200 with an empty page**, never 404 or 204.
- Never 200 with an error field in the body: a failure always carries its 4xx or 5xx code.

## 7. Errors

Every error answers with `application/problem+json` (`ProblemDetail`, RFC 9457) and these properties:

| Property | Description |
|---|---|
| `type` | Stable URN of the problem type: `urn:libryx:problem:<code-in-kebab-case>` |
| `title` | Short summary in Spanish, the same for every error of that type |
| `status` | HTTP status code |
| `detail` | Explanation in Spanish of this particular case, fit to show to the user |
| `instance` | Request path |
| `code` | **Stable identifier** of the error. The frontend decides on it, never on the text |
| `timestamp` | Instant of the error |
| `requestId` | Correlation id (see `observability.md`) |
| `errors` | Validation errors only: a list of `{ "field", "message" }` |

Validation error (400):

```json
{
  "type": "urn:libryx:problem:validation-error",
  "title": "Datos no válidos",
  "status": 400,
  "detail": "La petición contiene 2 campos no válidos.",
  "instance": "/api/books",
  "code": "VALIDATION_ERROR",
  "timestamp": "2026-10-05T09:30:00Z",
  "requestId": "3f1c9a52-7d0e-4b61-9c2a-5e8f0d1b7a44",
  "errors": [
    { "field": "isbn", "message": "El ISBN debe tener 13 dígitos." },
    { "field": "title", "message": "El título es obligatorio." }
  ]
}
```

Business rule violated (422):

```json
{
  "type": "urn:libryx:problem:loan-limit-exceeded",
  "title": "Límite de préstamos alcanzado",
  "status": 422,
  "detail": "El socio ya tiene 3 préstamos activos.",
  "instance": "/api/loans",
  "code": "LOAN_LIMIT_EXCEEDED",
  "timestamp": "2026-10-05T09:30:00Z",
  "requestId": "3f1c9a52-7d0e-4b61-9c2a-5e8f0d1b7a44"
}
```

### Code catalog

Codes live in the `ErrorCode` enum in `shared`. Adding one requires updating this table and
`messages.properties` in the same change. A published code is never renamed or reused.

| `code` | HTTP | When |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Invalid body fields or parameters, including a password that breaks the policy |
| `MALFORMED_REQUEST` | 400 | Unreadable JSON, wrong type, missing mandatory parameter |
| `INVALID_CREDENTIALS` | 401 | Wrong email or password |
| `UNAUTHENTICATED` | 401 | No token, or an invalid or revoked token |
| `TOKEN_EXPIRED` | 401 | Expired access token: the frontend must refresh |
| `EMAIL_NOT_VERIFIED` | 403 | Correct credentials, but the email is not verified yet |
| `ACCOUNT_LOCKED` | 403 | Locked or inactive account |
| `ACCESS_DENIED` | 403 | Role not allowed to perform the operation |
| `RESOURCE_NOT_FOUND` | 404 | The resource in the URL does not exist, or belongs to another user |
| `DUPLICATE_RESOURCE` | 409 | Already exists: email, ISBN, barcode |
| `CONCURRENT_MODIFICATION` | 409 | Another user changed the resource (optimistic lock) |
| `COPY_NOT_AVAILABLE` | 409 | The copy is no longer available |
| `INVALID_STATE` | 409 | The operation does not apply in the resource's current state |
| `LOAN_LIMIT_EXCEEDED` | 422 | Maximum active loans reached |
| `RENEWAL_LIMIT_EXCEEDED` | 422 | Maximum renewals reached |
| `REQUEST_LIMIT_EXCEEDED` | 422 | Maximum open requests reached |
| `ACTIVE_SANCTION` | 422 | The user has an active sanction |
| `OVERDUE_LOAN_EXISTS` | 422 | The user has an overdue loan and cannot borrow another |
| `LOAN_OVERDUE` | 422 | The loan is already overdue and cannot be renewed |
| `WORK_ALREADY_ON_LOAN` | 422 | The user already has this work on loan |
| `PENDING_REQUESTS_EXIST` | 422 | The work has queued requests from other users |
| `INVALID_VERIFICATION_CODE` | 422 | Wrong, expired or exhausted email verification code |
| `INVALID_RESET_CODE` | 422 | Wrong, expired or exhausted recovery or `ACCOUNT_SETUP` code |
| `EXPORT_TOO_LARGE` | 422 | The export exceeds the maximum number of rows; narrow the filters |
| `RATE_LIMIT_EXCEEDED` | 429 | Rate limit exceeded; includes a `Retry-After` header |
| `INTERNAL_ERROR` | 500 | Unhandled error. Generic `detail`, never the exception message |
| `ASSISTANT_UNAVAILABLE` | 503 | The assistant's AI provider is down or timed out |

### Rules

- Business exceptions carry an `ErrorCode` and its arguments, not a text. The global handler resolves
  `title` and `detail` against `messages.properties`.
- `INVALID_CREDENTIALS` does not distinguish "the email does not exist" from "the password is wrong".
- `EMAIL_NOT_VERIFIED` and `ACCOUNT_LOCKED` are returned only after the password has been checked.
- A 404 does not reveal that a resource exists but belongs to another user.
- Never stack traces, SQL, class names or table names in `detail`.

## 8. Headers

| Header | Direction | Use |
|---|---|---|
| `Authorization: Bearer <token>` | Request | Access token on every protected endpoint |
| `X-Request-Id` | Both | Correlation id; generated when missing |
| `Location` | Response | URI of the created resource, on every 201 |
| `Retry-After` | Response | Seconds to wait, on every 429 |
| `Content-Disposition: attachment; filename="…"` | Response | PDF and XLSX exports |
| `Set-Cookie` | Response | The refresh token, only from `/api/auth/login` and `/api/auth/refresh` |
