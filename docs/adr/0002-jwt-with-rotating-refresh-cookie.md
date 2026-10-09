# ADR-0002: Short-lived JWT plus a rotating refresh token in a cookie

- **Status:** Accepted
- **Date:** 2026-10-09

## Context

The API is stateless and serves an Angular single-page app. Sessions must survive a page reload,
tokens must not be readable by injected scripts, and a stolen token must stop working quickly. Logout
and account changes (password, lock, deletion) must take effect at once.

## Decision

- **Access token:** JWT signed with HS256, 15 minutes, sent in `Authorization: Bearer`. The frontend
  keeps it in memory only.
- **Refresh token:** opaque and random, 7 days, in an `HttpOnly`, `Secure`, `SameSite=Strict` cookie
  scoped to `/api/auth`. Stored in Redis as a hash only.
- **Rotation:** every refresh consumes the token and issues a new one; reusing a consumed token
  revokes the whole session.
- **Revocation:** logout puts the access token's `jti` on a Redis denylist for its remaining life;
  password change, lock, deactivation or deletion revoke every refresh token of the account.

## Alternatives considered

| Alternative | Why it was discarded |
|---|---|
| HTTP session with a cookie | Server state and CSRF protection, against the stateless design |
| Long-lived JWT in `localStorage` | Readable by any injected script, and cannot be revoked before it expires |
| OAuth2 / external identity provider | More infrastructure than a single application with its own users needs |

## Consequences

- **Gains:** XSS cannot read the refresh token, a leaked access token lives 15 minutes, logout is real.
- **Costs:** Redis becomes part of authentication; API and frontend must share the registrable domain
  because of `SameSite=Strict`.
- **Watch for:** a second client (mobile app) that cannot use cookies.
