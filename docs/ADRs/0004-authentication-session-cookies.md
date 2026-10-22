# ADR-0004: Authentication via server-managed session cookies

- **Status:** Accepted
- **Date:** 2026-10-22
- **Decision owners:** Core backend (Team 2)
- **Related areas:** `API_CONTRACTS.md` section 2, `ARCHITECTURE.md` section 7, `RULES.md` section 4

## Context

`API_CONTRACTS.md` section 2 prefers a server-managed session using a Secure, HttpOnly
cookie in production, and requires an ADR if bearer tokens are chosen instead. The team
confirmed the cookie-session model. Authentication must be enforced server-side on every
protected route and WebSocket room action.

## Decision drivers

- Security: keep long-lived credentials out of reach of JavaScript.
- Simplicity for four teams; avoid hand-rolled token rotation.
- CSRF protection configured deliberately.

## Options considered

### Option A — Server-managed session cookie (chosen)
Benefits: session state is revocable server-side; HttpOnly/Secure cookies are not
readable by scripts; simple mental model. Drawbacks: requires a session store and explicit
CSRF handling; needs correct SameSite/CORS configuration.

### Option B — Bearer JWT in memory
Benefits: stateless verification. Drawbacks: revocation/refresh complexity, risk of
insecure storage, and more surface for a student team to get wrong now.

## Evidence

No benchmark. This is a security/architecture decision; no measured trade-off applies.

## Decision

Use a server-managed session with a `Secure`, `HttpOnly` cookie in production, plus
CSRF protection, correct `SameSite` configuration, and HTTPS. Sessions are persisted in a
**PostgreSQL `sessions` table**, and CSRF uses a **double-submit cookie** (a non-`HttpOnly`
cookie carrying a random token that the client echoes in a request header, validated
server-side). Bearer tokens are **not** used for the first release; a future switch requires
a superseding ADR documenting token storage and rotation.

Not chosen: bearer JWT in `localStorage` (explicitly prohibited by `API_CONTRACTS.md`);
Redis-backed sessions (Redis stays unadopted until an ADR names a purpose).

## Consequences

- Session persistence: a **PostgreSQL `sessions` table** — server-side revocable and covered
  by the existing database backup/restore (ADR-0002, ADR-0007). Redis is **not** used for
  sessions; it remains unadopted until an ADR names a purpose.
- CSRF: **double-submit cookie** — a non-`HttpOnly`, `SameSite`-scoped cookie holds a random
  token that the client echoes in a request header; the server rejects the request when the
  header and cookie disagree. Token rotation and per-session binding are defined in M1.
- `SECRET_KEY`, `SESSION_COOKIE_SECURE`, and `CORS_ORIGINS` are required configuration
  (already listed as placeholders in `.env.example`).
- WebSocket connections authenticate with the session cookie and enforce origin checks.
- Registration must never allow self-assigning professor/admin roles.

## Revisit conditions

- Mobile or third-party clients require a token model.
- Cross-site constraints make cookie sessions impractical for the chosen deployment.
