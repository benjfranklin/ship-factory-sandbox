# Tech spec: to-do backend API (phase 1)

## Context

The repository tracks only `README.md`, which is empty (`/sf/in/repo-files.txt`). There is no application code, test setup or CI, so everything is new. Behaviour is in `PRODUCT.md`; the acceptance criteria are in `criteria.json`. Choices below are assumptions pending the owner's answers to the open questions in `PRODUCT.md` (stack, sign-in, deployment).

## Proposed changes

Stack (assumed): Node.js 20+, plain JavaScript (ES modules), Fastify (or Express) for HTTP, `better-sqlite3` for storage, `node:test` for tests (no test framework dependency), and Node's built-in `crypto.scrypt` for password hashing (no native hashing dependency).

New files:
- `package.json`: scripts `start` and `test`; dependencies above.
- `src/server.js`: starts the app on `PORT` (default 3000), database path from `DB_PATH` (default `./data/todo.db`).
- `src/app.js`: `buildApp({ dbPath })` builds the HTTP app, so tests use an in-memory database.
- `src/db.js`: opens SQLite, runs the schema on start (`CREATE TABLE IF NOT EXISTS`), enables foreign keys.
- `src/auth.js`: signup, login, logout routes and a pre-handler that resolves the session cookie to a user, else 401 (PRODUCT 1-6).
- `src/lists.js`, `src/items.js`: routes (PRODUCT 7-21).
- `README.md`: how to install, start and test (PRODUCT 22).
- `.gitignore`: `node_modules`, `data/`.

Data model:
- `users(id, email UNIQUE COLLATE NOCASE, password_hash, created_at)`
- `sessions(id random 32 bytes hex, user_id, created_at)`; cookie is `HttpOnly`, `SameSite=Lax`.
- `lists(id, user_id → users ON DELETE CASCADE, name, created_at)`
- `items(id, list_id → lists ON DELETE CASCADE, title, done, due_date TEXT 'YYYY-MM-DD' NULL, position INTEGER)`

API (JSON): `POST /api/signup`, `POST /api/login`, `POST /api/logout`; `GET/POST /api/lists`, `PATCH/DELETE /api/lists/:id`; `GET /api/lists/:id` (list with ordered items); `POST /api/lists/:id/items`; `PATCH/DELETE /api/items/:id`; `POST /api/items/:id/move` with `{position}`.

Key decisions:
- Ownership: every list/item query joins on `user_id = session user`; a miss gives 404 (PRODUCT 10). Never look up by id alone.
- Ordering: `position` is 0-based and gap-free. Move and delete renumber the affected list inside one SQLite transaction (better-sqlite3 is synchronous, so this also covers PRODUCT 21).
- Overdue: computed on read as `due_date < today(UTC) AND done = 0`, not stored, so it never goes stale. Take "today" from an injectable clock function so tests can fix it.
- Dates validated strictly with a round-trip parse (rejects 2026-02-30).
- Login error is identical for unknown email and wrong password; compare hashes with `timingSafeEqual`.

## Testing and validation

The acceptance criteria are in `criteria.json`. Run with `npm test` (`node --test`), using `buildApp` with an in-memory database and Fastify's `inject` (no network).

- AC1 (signup/login/logout, errors): `test/auth.test.js`, PRODUCT 1-6.
- AC2 (401 without session): `test/auth.test.js`, PRODUCT 5, over every route.
- AC3 (isolation between two users, 404): `test/isolation.test.js`, PRODUCT 8, 10.
- AC4 (list and item CRUD, cascade): `test/lists.test.js`, `test/items.test.js`, PRODUCT 7, 9, 11-15.
- AC5 (overdue rules with fixed clock): `test/overdue.test.js`, PRODUCT 16.
- AC6 (moving, gap-free, persists after reopening the db file): `test/order.test.js`, PRODUCT 17-19, 21.
- AC7 (validation): `test/validation.test.js`, PRODUCT 20.
- AC8 (local run): review of `README.md` and `package.json`; manually run `npm start` and `curl` signup.

## Risks and mitigations

- Cross-user data leaks are the main risk; mitigated by the isolation tests covering every item and list route.
- Stack choices are unconfirmed; if the owner changes them, only the "Proposed changes" section changes, not the behaviour.
- Native module `better-sqlite3` needs a prebuilt binary; if that proves a problem, use Node's built-in `node:sqlite`.

## Follow-ups

Front end, CI (issue 11) and deployment come in later phases. Rate limiting and CSRF tokens for the cookie session are deferred to the front-end phase.
