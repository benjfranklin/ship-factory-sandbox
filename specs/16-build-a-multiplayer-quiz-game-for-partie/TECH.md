# Tech spec: quiz game, live play (part 1)

## Context

The repository tracks only `README.md` (`/sf/in/repo-files.txt`); there is no existing code, no `specs/` precedent and no owner guidance. `quiz/` is built from scratch, and nothing outside it changes (PRODUCT 30). The to-do app of #10 is unrelated and untouched.

### Scope and split

The full issue is too much for one implement run. This spec covers: game rules, real-time server, host and player pages, bundled quizzes, tests, one-command run. Proposed separate issues: (2) quiz editor with image and time limit per question, JSON export/import; (3) persisted finished-game results and CSV download. To keep them easy, part 1 fixes the quiz JSON format and keeps the engine free of I/O, so part 3 can hook the "game finished" event.

## Proposed changes

Stack (see open question): Node.js 20+, ES modules, dependencies `ws` and `qrcode` only. Tests use the built-in `node:test`. No bundler; static files are served by the Node `http` server.

Layout:
- `quiz/package.json`: scripts `start` (`node server/index.js`) and `test` (`node --test`).
- `quiz/server/engine.js`: pure game logic. `createGame(quiz, {now})` returning a state machine with phases `lobby → question → reveal → (question…) → finished`. Functions: `addPlayer`, `rejoinPlayer`, `disconnectPlayer`, `submitAnswer`, `closeQuestion`, `advance`, `scoreboard`. Time is passed in (`now`), never read from a global clock, so scoring and timeouts are unit-testable. Scoring is `correct ? 500 + Math.round(500 * remainingMs / limitMs) : 0` (PRODUCT 18), clamped to 0..limit.
- `quiz/server/quiz.js`: load and validate quiz JSON from `quiz/quizzes/*.json`. Format: `{title, questions:[{type:"choice"|"truefalse", text, image?, options:[string], correct:int, timeLimit?}]}`; choice has 2–4 options, truefalse has fixed `["True","False"]`; timeLimit 5–120 s default 20. Invalid files are skipped with a console warning. Include `quiz/quizzes/sample.json`.
- `quiz/server/rooms.js`: room registry; code generator over alphabet `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`, collision-checked; timers (question deadline via `setTimeout` per room, cleared on pause/close); 30-minute idle cleanup (PRODUCT 3); host-disconnect pause holding remaining ms (PRODUCT 5).
- `quiz/server/index.js`: HTTP (static files from `quiz/public/`, `GET /api/quizzes`, `GET /api/qr?text=` returning SVG built by `qrcode`, restricted to the join URL form) plus `ws` upgrade on `/ws`. Messages are JSON `{type, ...}`; the server validates every message's shape and the sender's role, and ignores invalid ones (PRODUCT 25). Roles: host holds a `hostToken`, player holds a `playerToken`, both random (crypto) and kept in browser `localStorage`; rejoin matches token or, failing that, a nickname of a disconnected player (PRODUCT 21–23). The server sends the question deadline as a server timestamp plus a `serverNow` field so clients can compute offset (PRODUCT 13). The question message for players omits `correct`; reveal sends it. Heartbeat ping every 20 s to detect dead sockets.
- `quiz/public/`: `host.html`, `play.html`, `app.css`, `host.js`, `play.js`, small shared `net.js` (auto-reconnect with backoff, PRODUCT 24). All dynamic text is set with `textContent` (PRODUCT 26). Mobile-first CSS, buttons ≥ 64 px high, real `<button>` elements (PRODUCT 27). Join URL uses the host's address as seen by the request (`Host` header), and `npm start` prints the LAN IPv4 from `os.networkInterfaces()` (PRODUCT 29).

Trade-off: Socket.IO gives reconnection for free but is heavier; a plain `ws` plus ~30 lines of reconnect keeps the app small and the protocol explicit. Server holds the room state in memory; a restart ends games (acceptable for part 1).

## Testing and validation

The acceptance criteria are in `criteria.json`. Test command: `cd quiz && npm test`.

- `quiz/test/engine.test.js` (unit): scoring bounds and monotonicity, ties and ranking, answer lock, late answers ignored, early close when all connected players answered, disconnected players not blocking, rejoin rules. Covers PRODUCT 14, 18, 19, 22, 23 and AC1, AC2.
- `quiz/test/quiz.test.js`: validation of the sample quiz and rejection of malformed ones (AC6).
- `quiz/test/flow.test.js` (integration): starts the server on port 0, connects `ws` clients as a host and players, drives a full game and asserts the messages. Cases: lobby join validation and room full at 51 (PRODUCT 6–9, AC3); a full game to the podium (PRODUCT 11–17, AC4); drop and rejoin with score kept, and takeover refused (AC5); malformed and out-of-order messages leave the server running (PRODUCT 25, AC7); 50 simulated players all answering with every client receiving the reveal in under 1 s (PRODUCT 20, AC8). Use short question time limits injected through a test quiz.
- Manual UI check (`ui_flow`): `npm start`, open host page, scan the QR with a phone, play the sample quiz (PRODUCT 2, 12, 27, 29; AC9). Static review of `public/` for `innerHTML` use and of the changed-file list for PRODUCT 26 and 30 (AC10).

## Risks and mitigations

- Timer drift and races between "all answered" and the deadline: guard `closeQuestion` so it runs once per question index.
- Late-joiner policy and the scoring formula are product choices; both are isolated in `engine.js`, easy to change.
- Missing owner conventions: the layout above is a proposal for the owner to confirm.

## Follow-ups

- Part 2: editor, image handling (upload vs URL; decide then), JSON import/export, reusing `quiz.js` validation.
- Part 3: persistence format and CSV export hooked to the engine's finished event.
