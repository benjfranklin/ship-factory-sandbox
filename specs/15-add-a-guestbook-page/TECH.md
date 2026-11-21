# Guestbook: technical plan

## Context

The repository tracks only `README.md` (empty), per `/sf/in/repo-files.txt`. There is no existing code, build, test runner, package manifest, CI or convention, and no owner guidance. The to-do app in #10 doesn't exist yet and must stay untouched. User-visible behaviour is in `PRODUCT.md`. The owner has decided all five open questions (moderation, data kept, spam, storage, readers), and this plan builds on those answers. Hosting is out of scope: the service only needs to run locally.

## Proposed changes

Everything goes under `guestbook/`. Nothing outside it is added or edited (invariant 16). Specs stay in `specs/`.

Stack, chosen because the repository has none: Node.js, server-rendered HTML, SQLite. Server-rendered pages work without JavaScript, reload cleanly, and keep the page number in the URL (invariant 3). Keep dependencies minimal and pin them in `guestbook/package.json`. Prefer Node's built-in `node:sqlite` if the pinned Node version supports it; otherwise one pinned SQLite package. State the required Node version in `guestbook/package.json` (`engines`).

Layout:
- `guestbook/src/server.js`: HTTP routes.
  - `GET /` renders the approved list and form (`?page=N`).
  - `POST /` validates, applies spam checks, stores a pending message, then redirects with a 303 to `/?sent=1` (post/redirect/get, invariant 9). The confirmation text is the pending-approval one.
  - `GET /admin` (token required) lists pending messages (and approved ones, for the proposed delete in invariant 10). `POST /admin/approve` and `POST /admin/delete` act on a message id.
- Owner token (invariant 10): read from the `ADMIN_TOKEN` environment variable. If unset or empty, all `/admin` routes return 404. Compare in constant time. Accept the token via an `Authorization: Bearer` header, or a login form that sets an `HttpOnly`, `SameSite=Strict` cookie. Don't accept it in a URL query string, so it stays out of logs. Admin POSTs also need a CSRF defence (`SameSite=Strict` plus an Origin check). Wrong or missing token: 401 or 404 with no message data.
- `guestbook/src/validate.js`: pure function that trims, counts user-perceived characters (`Intl.Segmenter`), and checks limits of 50 and 500 (invariants 5, 6, 7). Names can't contain line breaks (invariant 8). Returns field errors.
- `guestbook/src/store.js`: SQLite access. Table `messages(id INTEGER PRIMARY KEY, name TEXT NOT NULL, body TEXT NOT NULL, created_at INTEGER NOT NULL, status TEXT NOT NULL DEFAULT 'pending')`. Public list query: `WHERE status='approved' ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET ?` (invariants 2, 3, 9). The count for paging also counts approved only. Parameterised statements only. Only name, message, time and status are stored (invariant 11). Function `purgeOlderThan(cutoff)` deletes any row with `created_at` older than 12 months, whatever its status.
- `guestbook/src/retention.js`: runs the purge at startup and then every 24 hours (timer `unref`'d), using an injectable clock so tests don't wait. 12 months is computed as calendar months, and the choice is documented in the README.
- `guestbook/src/spam.js`: honeypot check and an in-memory rate limit of 3 per 10 minutes per client address, with an injectable clock (invariant 12). The address is held only in memory for the window and is never written to the database or logs. Rate-limited and honeypot submissions don't write rows. A honeypot hit returns the same redirect as a success. Count only attempts that pass validation, so a rejected form doesn't use the allowance. (Decision for the implementer: keep it as stated and note it in the README.)
- `guestbook/src/render.js`: HTML templating with every dynamic value escaped; message line breaks become `<br>` after escaping (invariant 8). Labels, `aria-live` regions for errors and the confirmation, and a CSS-hidden honeypot with `tabindex="-1"`, `autocomplete="off"` and `aria-hidden` (invariants 1, 7, 15). No `noindex` (invariant 13).
- `guestbook/public/style.css`: responsive layout.
- `guestbook/README.md`: how to run locally, environment variables (`ADMIN_TOKEN`, port, database path, trusted proxy setting), retention, backup of the database file, and a record of the owner's five answers.

Edge cases: page values that are absent, non-numeric, zero or too big, which are clamped (invariant 3); database errors, which render the unavailable message without detail (invariant 14); approving a message that was already deleted or approved, which is a harmless no-op with a clear response.

## Testing and validation

The acceptance criteria are in `criteria.json`. Use the Node built-in test runner (`node --test`) from `guestbook/`. Run tests against a temporary SQLite database and an injected clock.

- AC1 (validation limits, invariants 5, 6, 7): unit tests of `validate.js` at 0, 1, 50, 51, 500 and 501 characters, whitespace-only input, and multi-code-point emoji; a route test that a direct POST over the limits is rejected.
- AC2 (order and paging, invariants 2, 3, 4): store and route tests with 45 approved messages, equal timestamps, an out-of-range page, and the empty list.
- AC3 (plain text, invariant 8): submit `<script>` and Markdown, approve them, and assert that the response HTML contains them escaped.
- AC4 (submission flow, invariants 7, 9): a valid POST gives a 303 redirect and the confirmation text. The message isn't in `GET /` until approved. An invalid POST re-renders with typed values and errors, and stores nothing.
- AC5 (owner page, invariant 10): without a token, or with a wrong one, `/admin` and the actions reveal and change nothing; with `ADMIN_TOKEN` unset, they return 404. Approve makes a message public, ordered by its original time. Delete removes it.
- AC6 (retention, invariant 11): with an injected clock, a message just under 12 months old survives the purge and one just over is deleted, whether pending or approved. The purge runs at startup. The schema holds no column beyond name, body, time and status.
- AC7 (spam, invariant 12): a filled honeypot isn't stored but looks the same as success. The fourth submission in 10 minutes is refused with typed text kept; after the clock advances 10 minutes it is accepted.
- AC8 (persistence and failure, invariants 14, 17): reopen the store and see the data; concurrent inserts all land; a broken database path gives the unavailable message without a stack trace.
- AC9 (isolation, invariant 16): review that the diff touches only `guestbook/` (and `specs/`).
- AC10 (accessibility, invariant 15): by hand, keyboard-only submission and a narrow-screen check on the public and owner pages; review that labels and live regions exist.

## Risks and mitigations

- Public text input invites abuse. Escaping, limits, pre-approval and the rate limit are the controls.
- The admin token is a single shared secret. Keep it out of URLs and logs, and compare in constant time. Document how to rotate it.
- Per-address rate limiting behind a proxy needs the trusted forwarded address. Document the setting, and fail safe to a global limit.
- The in-memory rate limit resets on restart. This is acceptable for the sandbox.
- Hosting is not chosen. When one is, it needs a persistent disk for the SQLite file. This is a follow-up, not part of this change.

## Follow-ups

- Choose and set up a host, once the owner has one.
- Once the to-do app (#10) exists, check that the two don't share tooling in a way that couples them.
