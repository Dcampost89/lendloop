# API Contract: User Accounts

**Feature**: [../spec.md](../spec.md) | **Data model**: [../data-model.md](../data-model.md)

This is the contract written before implementation (constitution III). Once built, the backend's
generated `/openapi.json` is the contract the frontend uses, and it must match this document.

## Conventions

- All paths are under `/api`. The browser reaches them through the Next.js origin (research R9).
- Request and response bodies are JSON. Field names are `snake_case`.
- **Authentication**: the `lendloop_session` cookie (research R1). "Member" means any signed-in
  active account; "Admin" means a signed-in active account with role `admin`.
- **State-changing requests** (`POST`, `PATCH`) without an `Origin` header equal to the frontend
  origin get `403 {"detail": "Invalid origin"}` before anything else runs (research R8).
- **Errors** use FastAPI's shape: `{"detail": "<message>"}`. Validation errors are `422` with
  FastAPI's standard list of field errors.
- **Common auth errors**:
  - `401 {"detail": "Not authenticated"}`: no cookie, unknown or expired session, or the account
    was deactivated. The response also clears the cookie.
  - `403 {"detail": "Admin access required"}`: signed in but not an administrator.
- Timestamps are ISO 8601 in UTC, for example `2026-11-06T18:30:00Z`.

## Schemas

```text
MeResponse
  id                 uuid
  email              string
  display_name       string
  neighborhood_area  string | null
  role               "member" | "admin"
  created_at         datetime

MemberPublic
  id                 uuid
  display_name       string
  neighborhood_area  string | null
  is_deactivated     boolean

AdminAccountView
  id                 uuid
  email              string
  display_name       string
  role               "member" | "admin"
  status             "active" | "deactivated"
  created_at         datetime

AccountRef
  id                 uuid
  display_name       string

StatusChangeView
  id                 integer
  action             "deactivated" | "reactivated"
  created_at         datetime
  account            AccountRef
  admin              AccountRef

Page[T]
  items              T[]
  total              integer
  limit              integer
  offset             integer
```

## Endpoints

### Authentication

#### `POST /api/auth/register` (public)

Creates an active `member` account and signs it in (FR-001–FR-005).

Request:
```json
{ "email": "ana@example.com", "password": "correct horse battery", "display_name": "Ana",
  "neighborhood_area": "Elm Street" }
```
`neighborhood_area` is optional.

| Status | Body | Notes |
|--------|------|-------|
| `201` | `MeResponse` | Sets the `lendloop_session` cookie |
| `409` | `{"detail": "Email already registered"}` | Case- and whitespace-insensitive match (research R5, R6) |
| `422` | validation errors | Bad email, password outside 12–128, display name outside 1–50, area over 100 |

If the caller is already signed in, the existing session is left untouched and a new one is
created for the new account.

#### `POST /api/auth/sign-in` (public)

Request: `{ "email": "ana@example.com", "password": "..." }`

| Status | Body | Notes |
|--------|------|-------|
| `200` | `MeResponse` | Sets the `lendloop_session` cookie |
| `401` | `{"detail": "Invalid email or password"}` | Same body and comparable timing for unknown email, wrong password, deactivated account (FR-007, research R4) |
| `429` | `{"detail": "Too many failed attempts. Try again later."}` | `Retry-After` header in seconds. Same for known and unknown emails (FR-008, research R7) |
| `422` | validation errors | Missing fields only; password length is not checked on sign-in |

#### `POST /api/auth/sign-out` (public, idempotent)

Ends the current session if there is one and clears the cookie (FR-006, FR-009).

| Status | Body |
|--------|------|
| `204` | none, also when there was no valid session |

### Current member

#### `GET /api/me` (Member)

| Status | Body |
|--------|------|
| `200` | `MeResponse` |
| `401` | not authenticated |

#### `PATCH /api/me` (Member)

Updates the caller's own profile only; there is no account ID in the path (FR-010, FR-013).

Request (all fields optional; `neighborhood_area: null` or `""` clears it):
```json
{ "display_name": "Ana R.", "neighborhood_area": "Elm & 5th" }
```

| Status | Body |
|--------|------|
| `200` | `MeResponse` |
| `401` | not authenticated |
| `422` | validation errors |

Fields other than `display_name` and `neighborhood_area` (for example `role`, `email`, `status`)
are rejected with `422`; the request schema forbids extra fields.

#### `POST /api/me/password` (Member)

Request: `{ "current_password": "...", "new_password": "..." }`

| Status | Body | Notes |
|--------|------|-------|
| `204` | none | Deletes all of the caller's other sessions; the current one stays (FR-011) |
| `400` | `{"detail": "Current password is incorrect"}` | |
| `401` | not authenticated | |
| `422` | validation errors | New password outside 12–128 |

### Members

#### `GET /api/members/{account_id}` (Member)

Public profile of any account, including deactivated ones (FR-012, FR-017).

| Status | Body | Notes |
|--------|------|-------|
| `200` | `MemberPublic` | |
| `401` | not authenticated | Visitors never see member data (clarification Q1) |
| `404` | `{"detail": "Member not found"}` | Unknown ID, or ID not a valid UUID |

### Administration

All endpoints in this section require Admin. Members get `403`; visitors get `401` (FR-018).

#### `GET /api/admin/accounts?limit=50&offset=0`

| Status | Body | Notes |
|--------|------|-------|
| `200` | `Page[AdminAccountView]` | Ordered by `created_at` descending. `limit` 1–100, default 50 |

#### `POST /api/admin/accounts/{account_id}/deactivate`

#### `POST /api/admin/accounts/{account_id}/reactivate`

| Status | Body | Notes |
|--------|------|-------|
| `200` | `AdminAccountView` | State changed and one status-change record written, or already in the target state and nothing written (idempotent, research R13) |
| `404` | `{"detail": "Account not found"}` | |
| `409` | `{"detail": "You cannot change the status of your own account"}` | FR-016 |

Deactivation also deletes every session of the account in the same transaction (FR-017).

#### `GET /api/admin/status-changes?limit=50&offset=0`

| Status | Body | Notes |
|--------|------|-------|
| `200` | `Page[StatusChangeView]` | Newest first (FR-020). `limit` 1–100, default 50 |

### Operations

#### `GET /api/health` (public)

`200 {"status": "ok"}` when the database is reachable; `503` otherwise. Used by Docker Compose.

## Requirement → endpoint map

| Requirement | Endpoint(s) |
|-------------|-------------|
| FR-001–FR-005 | `POST /api/auth/register` |
| FR-006, FR-007, FR-008 | `POST /api/auth/sign-in`, `POST /api/auth/sign-out` |
| FR-009 | every Member/Admin endpoint (session check) |
| FR-010 | `GET`, `PATCH /api/me` |
| FR-011 | `POST /api/me/password` |
| FR-012, FR-013 | `GET /api/members/{id}`; no endpoint accepts another account's ID for writes except admin ones |
| FR-014, FR-018, FR-019 | role check on all `/api/admin/*` |
| FR-015 | `GET /api/admin/accounts` |
| FR-016, FR-017 | `POST /api/admin/accounts/{id}/deactivate`, `/reactivate` |
| FR-020 | same, plus `GET /api/admin/status-changes` |

## Frontend routes (UI contract)

| Route | Access | Purpose |
|-------|--------|---------|
| `/` | public | Landing page with links to register and sign in; shows no member data |
| `/register` | public | Registration form |
| `/sign-in` | public | Sign-in form |
| `/home` | Member | Signed-in home showing the member's display name |
| `/profile` | Member | Edit own profile; change password |
| `/members/[id]` | Member | Public profile, with a "Deactivated" badge when applicable |
| `/admin/accounts` | Admin | Account list with deactivate and reactivate actions |
| `/admin/status-changes` | Admin | Record of administrator actions |

Members-only routes redirect visitors to `/sign-in`. Admin routes show "not found" to members.
These are conveniences only; the API enforces every rule (constitution II).
