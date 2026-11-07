# Research: User Accounts

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Date**: 2026-11-06

Each entry records a decision, why it was made, and what was rejected. Technology choices
fixed by the constitution (FastAPI, SQLModel, PostgreSQL, Next.js, etc.) are not re-debated here.

## R1. How a signed-in browser is identified

- **Decision**: Server-side sessions. On sign-in the backend generates a random 32-byte token
  (`secrets.token_urlsafe`), stores only its SHA-256 hash in a `session` table, and returns the
  token in an `HttpOnly`, `Secure` (except local development), `SameSite=Lax` cookie named
  `lendloop_session`, with `Path=/`.
- **Rationale**: FR-009, FR-011, FR-017 and SC-005 require that sign-out, password change and
  deactivation take effect on the very next request. Deleting a row does that directly. Hashing
  the token means a database leak does not leak usable sessions. Building it by hand is the point
  of constitution principle V.
- **Alternatives considered**:
  - JWT access tokens: cannot be revoked before expiry without a denylist, which is a session table
    with extra steps.
  - JWT plus refresh tokens: more moving parts, same revocation problem for the access token.
  - Token in `localStorage` with a bearer header: readable by any injected script; the cookie is not.

## R2. Session expiry (FR-009)

- **Decision**: A session is valid while `now - last_seen_at < 7 days`. Each authenticated request
  updates `last_seen_at` (only if it is more than 60 seconds old, to avoid a write on every
  request) and re-sends the cookie with `Max-Age=7 days`. Expired rows are deleted when found.
- **Rationale**: Gives "7 days without activity" exactly, with a cheap write pattern.
- **Alternatives considered**: Fixed 7-day lifetime from sign-in (does not match the spec);
  a background cleanup job (not needed at this scale; would also need visible-failure handling
  under principle VI).

## R3. Password hashing (FR-004, FR-005)

- **Decision**: Argon2id through `argon2-cffi` (`PasswordHasher` with library defaults). Use
  `check_needs_rehash` after successful sign-in to upgrade old hashes.
- **Rationale**: Argon2id is the current OWASP recommendation. `argon2-cffi` is small, has no
  opinions about the rest of the app, and exposes the concepts directly (hash, verify, rehash).
- **Alternatives considered**: `passlib` (unmaintained); `pwdlib` (thin wrapper over the same
  library, hides what is being learned); bcrypt (72-byte input limit conflicts with the
  128-character maximum).

## R4. Identical failure responses (FR-007, SC-004)

- **Decision**: Sign-in returns `401 {"detail": "Invalid email or password"}` for unknown email,
  wrong password, and deactivated account. When the email is unknown, the backend still verifies
  the submitted password against a fixed dummy Argon2 hash so response time does not reveal
  whether the account exists.
- **Rationale**: Without the dummy verify, unknown emails answer measurably faster.
- **Alternatives considered**: Only equal messages (leaks via timing).

## R5. Registration conflict vs. account enumeration

- **Decision**: Registration with an existing email returns `409 {"detail": "Email already
  registered"}`.
- **Rationale**: The spec (Story 1, scenario 2) requires refusing duplicate registration and the
  user must be told why. Without email confirmation (deferred), there is no way to avoid this leak
  while still telling the person what went wrong. SC-004 covers only sign-in responses.
- **Alternatives considered**: Pretend success and email the owner (needs email, deferred).

## R6. Duplicate and concurrent registrations (FR-002, constitution V)

- **Decision**: Store the normalized email (`strip().lower()`) in `account.email` with a unique
  constraint. The insert is attempted directly; an `IntegrityError` on that constraint becomes the
  409 response. No "check then insert".
- **Rationale**: Only the database can guarantee uniqueness under concurrent requests; principle V
  requires constraint-based idempotency.
- **Alternatives considered**: `citext` column (extra extension, hides the normalization step);
  preserving the original casing in a second column (no requirement needs it).

## R7. Sign-in throttling keyed by device or network (FR-008)

- **Decision**: A `sign_in_failure` table stores `(email, client_ip, attempted_at)` for every failed
  attempt, including unknown emails. Before checking the password, the backend loads the 5 most
  recent failures for that `(email, client_ip)`. If there are 5 and the newest is less than
  15 minutes old and all 5 are within 15 minutes of each other, the attempt is refused with
  `429` and a `Retry-After` header equal to the remaining block time. Refused attempts are not
  recorded. A successful sign-in deletes the failures for that pair. Rows older than 15 minutes
  are deleted when new failures are written.
- **Client IP**: Requests reach FastAPI through the Next.js server (see R9). Uvicorn runs with
  `--proxy-headers --forwarded-allow-ips=<frontend container>` so `request.client.host` is the
  browser's IP taken from `X-Forwarded-For`, and that header is trusted only from the frontend.
- **Rationale**: "Device or network" maps to client IP, which needs no device fingerprinting. A
  table instead of in-memory counters survives restarts and works with several backend workers.
  Throttling unknown emails the same way keeps the response from revealing which emails exist.
- **Alternatives considered**: Account-wide lockout (rejected in clarification Q2); in-memory
  counters (lost on restart, wrong with multiple workers); Redis (an extra service with no other
  use yet); device cookies (cleared easily by attackers).

## R8. Cross-site request forgery

- **Decision**: Cookies are `SameSite=Lax`, and every state-changing request (`POST`, `PATCH`,
  `PUT`, `DELETE`) must carry an `Origin` header equal to the configured frontend origin
  (`FRONTEND_ORIGIN`), or it is refused with `403`.
- **Rationale**: Lax blocks cross-site POSTs in modern browsers; the Origin check covers the rest
  without token plumbing.
- **Alternatives considered**: Synchronizer CSRF tokens (more code for no added protection here).

## R9. How the browser reaches the backend

- **Decision**: The browser only ever talks to the Next.js origin. Next.js `rewrites` forward
  `/api/:path*` to the FastAPI service (`BACKEND_URL`, e.g. `http://backend:8000`). Client
  Components call `/api/...` directly and the browser attaches the session cookie. Server
  Components call `BACKEND_URL` directly from the server and forward the incoming
  `lendloop_session` cookie and `X-Forwarded-For`.
- **Rationale**: One origin means the cookie is first-party, no CORS configuration, and no token
  handling in JavaScript. It shows both Next.js data-fetching paths (server and client).
- **Alternatives considered**: Browser calls FastAPI on another port (CORS plus cross-site cookies);
  Next.js Route Handlers proxying every call by hand (boilerplate); Server Actions for all writes
  (they would have to copy `Set-Cookie` from the backend response; can be added later).

## R10. Frontend route protection

- **Decision**: `proxy.ts` (the Next.js 16 name for middleware) redirects to `/sign-in` when the
  `lendloop_session` cookie is missing on members-only routes. Pages then call `GET /api/me`; a
  401 redirects to `/sign-in`, and admin pages show "not found" when the role is not `admin`.
- **Rationale**: Principle II: the redirect is only a convenience. Every protected endpoint
  checks the session itself.
- **Alternatives considered**: Validating the session inside the proxy (an extra backend call on
  every navigation, and it still would not be the security boundary).

## R11. Typed API client (constitution III)

- **Decision**: Generate TypeScript types from the backend's `/openapi.json` with
  `openapi-typescript` and call the API with `openapi-fetch`. Generation runs via a `pnpm gen:api`
  script; the generated file is committed.
- **Rationale**: The constitution makes the generated OpenAPI schema the contract the frontend
  uses; these two tools turn it into compile-time checks with little runtime code. Generated clients
  are allowed AI output under principle VIII.
- **Alternatives considered**: Handwritten fetch wrappers (drift from the contract);
  `orval`/`openapi-generator` (heavier, generate code to read and maintain).

## R12. Account and record identifiers

- **Decision**: `account.id` and `session.id` are UUIDv4. `sign_in_failure.id` and
  `account_status_change.id` are auto-increment integers (never exposed in URLs).
- **Rationale**: Account IDs appear in profile URLs; UUIDs do not reveal member count or allow
  walking through IDs.
- **Alternatives considered**: Integers everywhere (enumerable).

## R13. Deactivate and reactivate safely (FR-016, FR-017, FR-020)

- **Decision**: One transaction does a conditional
  `UPDATE account SET status='deactivated' WHERE id=:id AND status='active' RETURNING id`. If a row
  changed, it inserts the `account_status_change` row and deletes the account's sessions in the
  same transaction. If no row changed (already deactivated), it returns the current account with
  `200` and writes nothing. Reactivation is the mirror image. Acting on one's own account returns
  `409`.
- **Rationale**: A retried request does not create a second record (principle V), and the record
  and the status change cannot exist without each other (FR-020).
- **Alternatives considered**: Returning `409` on "already deactivated" (makes retries fail);
  read-then-write in Python (race between two administrators).

## R14. Creating the first administrator

- **Decision**: A small command, `python -m app.cli create-admin --email ... --display-name ...`,
  prompts for the password (hidden input), creates an `admin` account, and fails if the email
  exists. Uses `argparse` and `getpass` from the standard library.
- **Rationale**: The spec says the first administrator is created outside the app.
- **Alternatives considered**: Seeding in a migration (puts a password into history); Typer
  (extra dependency for one command).

## R15. Backend tests against real PostgreSQL (constitution IV)

- **Decision**: Docker Compose runs a `db` service; tests use a separate `lendloop_test` database.
  A session-scoped fixture runs `alembic upgrade head` once. Each test runs inside a connection-level
  transaction that is rolled back afterwards, and the FastAPI `get_session` dependency is
  overridden to use that connection. Tests call the app through FastAPI's `TestClient` with the
  `Origin` header set.
- **Rationale**: Real constraint behavior (unique email, conditional update) is exactly what must
  be proven; rollback keeps tests isolated and fast.
- **Alternatives considered**: SQLite (forbidden by the constitution, and lacks some constraint
  behavior); recreating tables per test (slow).

## R16. Frontend component tests

- **Decision**: Vitest with `jsdom` and React Testing Library. Client Components receive their API
  calls through the `src/lib/api/client.ts` module, which tests replace with `vi.mock`.
- **Rationale**: Keeps tests about what the user sees, with no extra mocking library.
- **Alternatives considered**: MSW (another dependency; worth adding when request-level mocking is
  needed across many components).

## R17. Validation details

- **Email**: Pydantic `EmailStr` (needs `email-validator`, part of FastAPI's standard extras), max
  254 characters, normalized before storage.
- **Password**: 12–128 characters counted as Unicode code points; no other rules (FR-004).
- **Display name**: trimmed, 1–50 characters; also enforced by a database `CHECK`.
- **Neighborhood area**: trimmed, empty becomes `null`, max 100 characters.
- **Rationale**: Validate at the edge with Pydantic, back the important ones with constraints.

## R18. Versions

- **Decision**: Python 3.13 (meets 3.12+), PostgreSQL 17 (meets 16+), Node.js 22 LTS,
  Next.js 16 with React 19, pnpm as the package manager. psycopg 3 as the PostgreSQL driver.
- **Rationale**: Current stable releases that satisfy the constitution's minimums.
