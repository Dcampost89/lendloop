# Implementation Plan: User Accounts

**Branch**: `001-user-accounts` | **Date**: 2026-11-06 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-user-accounts/spec.md`

## Summary

Members register with email, password and display name, are active immediately, and sign in and
out. They edit their own profile and change their password. Administrators list accounts,
deactivate and reactivate them, and see a permanent record of those actions. Only signed-in
members can see public profiles.

Technical approach: a FastAPI backend with server-side sessions (random token in an `HttpOnly`
cookie, SHA-256 hash stored in PostgreSQL), Argon2id password hashes, throttling of failed
sign-ins per email and client IP, and every rule enforced in API dependencies and queries. A
Next.js App Router frontend is the only origin the browser talks to; it forwards `/api/*` to the
backend and uses a client generated from the backend's OpenAPI schema. This is the first feature,
so it also sets up the repository skeleton: Docker Compose, Alembic, test databases and tooling.

## Technical Context

**Language/Version**: Python 3.13 (backend), TypeScript 5 in strict mode on Node.js 22 LTS
(frontend)

**Primary Dependencies**: FastAPI, SQLModel (synchronous sessions), Alembic, psycopg 3,
argon2-cffi, email-validator; Next.js 16 (App Router) with React 19, Tailwind CSS, shadcn/ui,
openapi-typescript, openapi-fetch

**Storage**: PostgreSQL 17, schema changes through Alembic only

**Testing**: pytest with FastAPI `TestClient` against a real PostgreSQL test database (backend);
Vitest with React Testing Library and jsdom (frontend)

**Target Platform**: Linux containers via Docker Compose for local development; modern evergreen
browsers

**Project Type**: Web application (separate backend API and frontend)

**Performance Goals**: Sign-in and registration respond in under 1 second locally, dominated by
Argon2 hashing; other endpoints under 200 ms at p95 locally. Enough to meet SC-001 and SC-002.

**Constraints**: In-house authentication only (constitution V); revocation effective on the next
request (SC-005); no account enumeration through sign-in (SC-004); no secrets in the repository

**Scale/Scope**: One neighborhood, up to about 1,000 accounts; 4 tables, 12 endpoints, 8 pages

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | How this plan complies | Pre-design | Post-design |
|-----------|------------------------|------------|-------------|
| I. Spec Before Code | Spec and clarifications are done and committed; this plan holds every technical decision; the spec names no technologies | ✅ | ✅ |
| II. Server-Side Enforcement | `current_account` and `require_admin` dependencies guard every protected endpoint. Profile writes use `/api/me`, so the query filters by the caller's ID and no other account can be targeted. Status transitions use conditional `UPDATE ... WHERE status = ...` (R13). Frontend redirects are only a convenience (R10) | ✅ | ✅ |
| III. Explicit API Contracts | Dedicated response schemas (`MeResponse`, `MemberPublic`, `AdminAccountView`, `StatusChangeView`); `password_hash` is never in any schema. `contracts/api.md` is written before code; the frontend uses a client generated from `/openapi.json` (R11) | ✅ | ✅ |
| IV. Test-First Development | Tests written first against real PostgreSQL (R15). Negative tests: member A cannot touch member B; members get 403 on all admin endpoints; visitors get 401 on profiles. Component tests for every Client Component form (R16). Server Components stay thin | ✅ | ✅ |
| V. Secure by Default | Secrets (`DATABASE_URL`, etc.) only from environment; `.env` is gitignored and `.env.example` has no real values. Registration and status changes are idempotent through a unique constraint and conditional updates (R6, R13). In-house sessions and hashing (R1, R3). No calls between systems in this feature, so request signing is not yet needed | ✅ | ✅ |
| VI. Failures Are Visible | No background work or outbound notifications in this feature. Unhandled errors are logged by the backend with a request ID and return 500; no `except: pass` | ✅ | ✅ |
| VII. Small, Learnable Increments | `/speckit-tasks` splits the work into tasks of about an hour, each naming the FastAPI or Next.js concept it teaches. Every added library is justified in research.md (R3, R11, R17) | ✅ | ✅ |
| VIII. Human-Written Core, AI-Assisted Edges | `/speckit-tasks` will label scaffolding, configuration, generated client, test cases and shadcn/ui primitives `[AI]`, and endpoints, dependencies, security, models, queries, pages and feature components `[HUMAN]` | ✅ | ✅ |
| Technology Constraints | Stack exactly as listed. n8n is not added to Docker Compose yet because no feature uses it | ✅ | ✅ |

No violations; Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-user-accounts/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R1–R18
├── data-model.md        # Phase 1: tables, constraints, state transitions
├── quickstart.md        # Phase 1: how to run and validate
├── contracts/
│   └── api.md           # Phase 1: endpoints, schemas, frontend routes
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 (/speckit-tasks; not created here)
```

### Source Code (repository root)

```text
docker-compose.yml             # db (PostgreSQL 17), backend, frontend
.env.example                   # every variable, with no real secrets

backend/
├── pyproject.toml             # uv project; ruff and pyright settings
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/              # one migration for this feature's 4 tables
├── app/
│   ├── main.py                # FastAPI app, router registration, origin check
│   ├── config.py              # settings read from environment variables
│   ├── db.py                  # engine and get_session dependency
│   ├── models.py              # SQLModel tables: Account, Session, SignInFailure, AccountStatusChange
│   ├── schemas.py             # request and response schemas (contracts/api.md)
│   ├── security.py            # password hashing, token generation and hashing
│   ├── deps.py                # current_account, require_admin
│   ├── throttle.py            # sign-in failure counting (R7)
│   ├── cli.py                 # create-admin command (R14)
│   └── routers/
│       ├── auth.py            # register, sign-in, sign-out
│       ├── me.py              # profile and password
│       ├── members.py         # public profiles
│       ├── admin.py           # account list, deactivate, reactivate, status changes
│       └── health.py
└── tests/
    ├── conftest.py            # test database, transaction rollback, client fixtures
    ├── unit/                  # security.py, throttle rule
    └── api/                   # one file per router, including negative authorization tests

frontend/
├── package.json               # scripts: dev, build, test, lint, gen:api
├── next.config.ts             # /api/* rewrite to BACKEND_URL (R9)
├── vitest.config.ts
├── src/
│   ├── proxy.ts               # redirect visitors away from members-only routes (R10)
│   ├── app/
│   │   ├── page.tsx           # landing
│   │   ├── register/page.tsx
│   │   ├── sign-in/page.tsx
│   │   ├── (members)/
│   │   │   ├── layout.tsx     # loads /api/me on the server; redirects on 401
│   │   │   ├── home/page.tsx
│   │   │   ├── profile/page.tsx
│   │   │   └── members/[id]/page.tsx
│   │   └── admin/
│   │       ├── layout.tsx     # role check for display only
│   │       ├── accounts/page.tsx
│   │       └── status-changes/page.tsx
│   ├── components/
│   │   ├── ui/                # shadcn/ui primitives
│   │   ├── register-form.tsx
│   │   ├── sign-in-form.tsx
│   │   ├── profile-form.tsx
│   │   ├── change-password-form.tsx
│   │   ├── sign-out-button.tsx
│   │   └── account-status-button.tsx
│   └── lib/
│       └── api/
│           ├── schema.d.ts    # generated by openapi-typescript
│           ├── client.ts      # openapi-fetch client for Client Components
│           └── server.ts      # server-side fetch that forwards the session cookie
└── tests/
    └── components/            # one test file per Client Component
```

**Structure Decision**: Web application with `backend/` and `frontend/` at the repository root,
matching the constitution's split between FastAPI and Next.js. The backend uses a flat `app/`
package with one router per area. That is small enough to read in one sitting and can be split
into packages when a later feature grows it.

## Complexity Tracking

No constitution violations to justify.
