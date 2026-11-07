# Quickstart: User Accounts

**Feature**: [spec.md](./spec.md) | **API**: [contracts/api.md](./contracts/api.md)

How to run the feature and prove it works. Implementation details live in `tasks.md`.

## Prerequisites

- Docker with Docker Compose v2
- For running tests and tools outside containers: Python 3.13 with `uv`, Node.js 22 with `pnpm`
- A `.env` file in the repository root, copied from `.env.example`

## Start everything

```bash
cp .env.example .env            # first time only; fill in the values
docker compose up --build
```

Expected: `db`, `backend` and `frontend` start; `curl http://localhost:3000/api/health` returns
`{"status":"ok"}` (through the Next.js rewrite to the backend).

Apply migrations and create the first administrator:

```bash
docker compose exec backend uv run alembic upgrade head
docker compose exec backend uv run python -m app.cli create-admin \
  --email admin@example.com --display-name "Admin"
# prompts for the password (12–128 characters)
```

## Automated checks (definition of done)

```bash
# backend (uses the lendloop_test database in the db container)
cd backend
uv run pytest
uv run ruff check . && uv run ruff format --check .
uv run pyright

# frontend
cd frontend
pnpm test
pnpm lint
pnpm tsc --noEmit
```

Expected: all pass with no errors. Every acceptance scenario in the spec maps to at least one
backend test, and every authorization rule has a negative test (constitution IV).

## Manual validation scenarios

Open http://localhost:3000.

### Story 1: register, sign in, sign out

1. Register as `ana@example.com` with a 12+ character password and display name "Ana".
   Expected: you land on `/home` showing "Ana".
2. Sign out, then register again as ` ANA@Example.com `. Expected: "Email already registered".
3. Sign in with a wrong password, then with `nobody@example.com`. Expected: the same
   "Invalid email or password" message both times.
4. While signed out, open `/home`. Expected: redirected to `/sign-in`.
5. Fail sign-in for Ana 5 times. Expected: the 6th attempt, even with the right password, shows
   "Too many failed attempts". From a different network (or after 15 minutes) Ana can sign in.

### Story 2: profile and password

1. As Ana, change the display name and neighborhood area on `/profile`. Expected: saved and shown
   on `/members/<ana-id>`.
2. Sign in as Ana in a second browser. Change the password in the first. Expected: the second
   browser is signed out on its next action; the new password works and the old one does not.
3. Change the password with a wrong current password. Expected: "Current password is incorrect".
4. Sign out and open `/members/<ana-id>`. Expected: redirected to `/sign-in`; the API returns 401.

### Story 3: administrators

1. Sign in as the administrator and open `/admin/accounts`. Expected: Ana is listed with email,
   role, status and registration date.
2. Deactivate Ana while she is signed in elsewhere. Expected: her next action signs her out, and
   she cannot sign in (generic error).
3. As another member, open `/members/<ana-id>`. Expected: profile shown with a "Deactivated" badge.
4. Reactivate Ana. Expected: she can sign in with her existing password.
5. Open `/admin/status-changes`. Expected: both actions listed newest first with the admin's name.
6. Try to deactivate your own admin account (button hidden; call the API directly):
   ```bash
   curl -i -X POST http://localhost:3000/api/admin/accounts/<admin-id>/deactivate \
     -H "Origin: http://localhost:3000" -b "lendloop_session=<admin token>"
   ```
   Expected: `409`.
7. As Ana (a member), call `GET /api/admin/accounts`. Expected: `403`.

## Troubleshooting

- `403 Invalid origin` from curl: add `-H "Origin: http://localhost:3000"` to state-changing calls.
- Backend tests cannot connect: make sure `docker compose up db` is running and `TEST_DATABASE_URL`
  in `.env` points to `lendloop_test`.
- Frontend type errors after an API change: run `pnpm gen:api` with the backend running.
