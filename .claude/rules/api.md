---
paths:
  - "src/main/java/**/controller/**"
  - "src/main/java/**/dto/**"
  - "src/main/java/**/shared/**"
  - "src/main/resources/messages*.properties"
---

# REST API rules

Full contract, examples and the error-code catalog: `docs/api-contract.md`. Read it before adding or
changing an endpoint, a DTO or an error.

## Endpoints

- Prefix `/api`. Plural, kebab-case resources: `/api/books`, `/api/loan-requests`.
- Administration endpoints under `/api/admin/**`.
- Business actions are `POST` sub-resources: `/api/loans/{id}/renewals`, `/api/sanctions/{id}/lift`.
- No version prefix. Do not break the contract (remove or rename a field, change a type or a code) without asking.
- Controllers only translate HTTP: validate input, call one service method, map the status code.
- Input is validated with Bean Validation (`@Valid`) on request records.
- Constraints on path and query parameters (`@Positive Long id`) are validated too; the global handler
  maps that failure to 400 `VALIDATION_ERROR`, never to a 500.
- Never accept the acting user's id in a request: take it from the authenticated user. A `USER` only
  reaches their own resources.
- Every endpoint carries OpenAPI annotations (`@Operation`, `@ApiResponse`, `@Tag`).

## Payloads

- JSON in `camelCase`. Enums as upper-case strings.
- Instants in ISO-8601 UTC with `Z` (`2026-10-05T09:30:00Z`); business dates as `2026-10-19`.
- Lists are always paginated: `page` (from 0), `size` (default 20, max 100), `sort=field,asc|desc`.
  Return `PageResponse<T>` from `shared`, never a raw Spring `Page<T>`.
- Sorting only by a closed list of fields per endpoint; any other field is a 400.
- Filters are optional query parameters combined with AND: `q` for text, a repeated parameter for
  several values, `from` and `to` for date ranges.

## Success codes

Declare the code explicitly on each endpoint and in its `@ApiResponse`.

| Code | When | Body |
|---|---|---|
| 200 | `GET`; `PUT` returning the updated resource (`PATCH` is not used); actions that return a result without creating a resource (login, token refresh, lifting a sanction) | Response DTO |
| 201 | `POST` that creates a resource | Created DTO + `Location` header |
| 202 | Accepted work that finishes in the background (RAG ingestion and re-indexing) | Process status |
| 204 | `DELETE`, logout, requesting a verification or recovery code, actions with nothing to return | Empty |

- An empty list is **200 with an empty page**, never 404 or 204.
- 404 only when the resource identified in the URL does not exist.
- Exports answer 200 with the format's `Content-Type` and `Content-Disposition: attachment`.
- Never return 200 with an error field in the body.

## Errors

One `@RestControllerAdvice` in `shared` answers with `ProblemDetail` (RFC 9457).

| Code | When |
|---|---|
| 400 | Invalid input, malformed parameters |
| 401 | Missing, invalid, expired or denylisted token; wrong credentials |
| 403 | Authenticated without permission; unverified email; locked account |
| 404 | Resource does not exist, or belongs to another user |
| 409 | State conflict: duplicate, optimistic lock, copy already on loan |
| 422 | Business rule violated: loan limit, active sanction, no renewals left |
| 429 | Rate limit exceeded, with a `Retry-After` header |
| 500 | Unhandled error: generic message to the client, details only in the log |
| 503 | The assistant's AI provider is down or timed out |

- Every error includes `code` (an `ErrorCode` enum value), `timestamp` and `requestId`. Validation errors
  add an `errors` list of `field` and `message`.
- `code` is the contract with the frontend: a published code is never renamed or reused.
- Never return stack traces, SQL or class names.

## User-facing messages

- Every text an end user can read lives in `src/main/resources/messages.properties` (UTF-8, **Spanish**).
  No Spanish literal in Java code.
- Keys in English, lower case, dot-separated:
  - `validation.<resource>.<field>.<rule>` → `validation.book.isbn.size`
  - `error.<code-in-kebab-case>.title` and `.detail` → `error.loan-limit-exceeded.detail`
  - `email.<type>.subject` and `.body` → `email.request-ready.subject`
- Bean Validation references the key in braces: `@NotBlank(message = "{validation.book.title.required}")`.
  The validator resolves keys against the same `MessageSource`; there is no `ValidationMessages.properties`.
- Business exceptions carry an `ErrorCode` and its arguments, never the text. The global handler resolves it.
- Variable values with `{0}`, `{1}` placeholders; never string concatenation.
- Wording: a full sentence with a final period, neutral tone, no jargon, saying what the user can do.
- Single language (`es-ES`): the `Accept-Language` header does not change the response.
- Add every new key in the same change. A test checks that each `ErrorCode` has both of its keys.
