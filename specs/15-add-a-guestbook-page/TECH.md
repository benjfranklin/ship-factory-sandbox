# Guestbook: technical plan

## Context

The repository tracks only `README.md` (empty), per `/sf/in/repo-files.txt`. There is no existing code, build, test runner, package manifest, CI or convention to follow, and no owner guidance. The to-do app in #10 doesn't exist yet and must stay untouched. User-visible behaviour is in `PRODUCT.md`.

**This plan is provisional.** The storage, moderation, spam and visibility decisions are open in `PRODUCT.md` (invariants 9 to 13). The plan below assumes the recommendations. If the owner chooses differently, the parts named under "If the answers differ" change. Implementation shouldn't start until the owner has answered them.

## Proposed changes

Everything goes under `guestbook/`. Nothing outside it is added or edited (invariant 15). Specs stay in `specs/`.

Assumed stack, chosen because the repository has none: Node.js with no framework dependency beyond what's needed, server-rendered HTML, SQLite for storage. Reason: server-rendered pages work without JavaScript, reload cleanly, and keep the page URL as the source of the page number (invariant 3). Keep dependencies minimal and pin them in `guestbook/package.json`.

Layout:
- `guestbook/src/server.js`: HTTP routes. `GET /` renders the list and form (query `?page=N`). `POST /` validates, stores, then redirects with a 303 to `/?sent=1` (post/redirect/get, invariant 9).
- `guestbook/src/validate.js`: pure function that trims, counts user-perceived characters (use `Intl.Segmenter`), and checks limits of 50 and 500 (invariants 5, 6, 7). Returns field errors.
- `guestbook/src/store.js`: SQLite access. Table `messages(id INTEGER PRIMARY KEY, name TEXT, body TEXT, created_at INTEGER)`. List query orders by `created_at DESC, id DESC` with limit 20 and an offset (invariants 2, 3). Inserts use parameterised statements, and SQLite serialises writes (invariant 16). Only the name, message and time are stored (invariant 10).
- `guestbook/src/spam.js`: honeypot check and an in-memory rate limit, 3 per 10 minutes per client address. The address is kept only in memory and only for the window, and is never written to the database (invariant 11).
- `guestbook/src/render.js`: HTML templating with every dynamic value escaped. Message line breaks are rendered as `<br>` after escaping (invariant 8). Labels, `aria-live` regions for errors and the confirmation, and a CSS honeypot hidden with `tabindex="-1"` and `aria-hidden` (invariants 1, 7, 14).
- `guestbook/public/style.css`: responsive layout.
- Owner action (invariant 9): `POST /admin/hide` with an `ADMIN_TOKEN` environment variable, adding a `hidden` column that the list query filters on. Without the variable set, the route is disabled.
- `guestbook/README.md`: how to run, configure and deploy it, and how to back up the database file.

Edge cases to handle: page values that are absent, non-numeric, zero or too big, which are clamped (invariant 3); database errors, which render the unavailable message (invariant 13).

**If the answers differ:**
- Approval-first moderation: add a `status` column (`pending`/`approved`) and an approve route, and change the confirmation text.
- CAPTCHA: add a verification call in `spam.js`, which brings in a third-party script and a secret.
- Link-only reading: add a no-index header and meta tag, and possibly an unguessable path or access token.
- Hosted service instead of SQLite: replace `store.js` only, keeping its interface.
- Email or deletion by visitors: new fields, a retention rule and a delete flow. This is a larger change and may be split.

## Testing and validation

The acceptance criteria are in `criteria.json`. Use the Node built-in test runner (`node --test`) from `guestbook/`, as the repository has no runner of its own yet. Run tests against a temporary SQLite database.

- AC1 (validation limits, invariants 5, 6, 7): unit tests of `validate.js` at 0, 1, 50, 51, 500 and 501 characters, whitespace-only input, and multi-code-point emoji.
- AC2 (order and paging, invariants 2, 3, 4): store and route tests with 45 messages (three pages), equal timestamps, an out-of-range page, and the empty list.
- AC3 (plain text, invariant 8): submit `<script>` and Markdown, and assert that the response HTML contains them escaped.
- AC4 (submission flow, invariants 7, 9): a valid POST gives a 303 redirect and the message is listed. An invalid POST re-renders with typed values and errors, and stores nothing.
- AC5 (spam, invariant 11): a filled honeypot isn't stored but looks successful. The fourth submission in the window is refused.
- AC6 (persistence and failure, invariants 13, 16): reopen the store and see the data; concurrent inserts all land; a broken database path gives the unavailable message without a stack trace.
- AC7 (isolation, invariant 15): review that the diff touches only `guestbook/` (and `specs/`).
- AC8 (accessibility, invariants 1, 14): by hand, keyboard-only submission and a narrow-screen check; review that labels and live regions exist.
- Open decisions: review check that the owner's answers are recorded in `guestbook/README.md` before merge.

## Risks and mitigations

- Persistent storage on a free host may not exist or may be wiped. Mitigate by confirming the host's disk persistence before choosing it, and documenting backup.
- Per-address rate limiting behind a proxy needs the trusted forwarded address. Mitigate by documenting the setting, and failing safe to a global limit.
- Public text input invites abuse. Escaping, limits and the owner's hide action are the minimum controls.
- The in-memory rate limit resets on restart. This is acceptable for the sandbox.

## Follow-ups

- Once the to-do app (#10) exists, check that the two don't share tooling in a way that couples them.
