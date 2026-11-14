# Quiz game: live play (part 1 of 3)

## Summary

A self-contained browser quiz game in `quiz/`. A host opens a room from a bundled quiz and shows a room code and QR code on a big screen. Up to 50 players join from their phones with a code and nickname, answer timed questions, and see live reveals, scores and a podium. This item covers hosting, joining, playing, real-time updates, rejoin, scoring, tests and a one-command local run. The quiz editor with JSON import/export (part 2) and saved results with CSV download (part 3) are split out (see TECH.md, "Scope and split"). It must not depend on or change the to-do app in #10.

## Behavior

### Hosting
1. The host opens the host page and sees the list of available quizzes (title and question count). The quizzes are JSON files that ship with the app; at least one sample quiz with both multiple-choice and true/false questions is included.
2. Choosing a quiz opens a room. The big screen shows a 4-character room code (uppercase letters and digits, no easily confused characters such as 0/O or 1/I), a QR code that opens the join page with the code filled in, the join URL as text, and a live list of joined nicknames with a count.
3. Two open rooms never share a code. A room that is never started and has no host connected for 30 minutes is closed.
4. The host sees a Start button, enabled once at least one player has joined. Pressing it begins question 1.
5. If the host's page disconnects, the game pauses on the current state (the countdown stops, with remaining time kept) and resumes when the host reopens the host page for that room. The host page remembers its room in the browser so a reload returns to it.

### Joining
6. A player opens the join page on a phone, enters the room code (case-insensitive) and a nickname, and joins. There are no accounts.
7. Nicknames are trimmed, 1 to 20 characters, and unique in the room ignoring case. An empty, too-long or taken nickname is rejected with a clear message and the player can try again. An unknown or closed room code shows "room not found".
8. A new player can join only while the room is in the lobby. Joining after the game has started shows "game already started".
9. The 51st player is refused with a "room full" message. Rooms hold up to 50 players.
10. After joining, the phone shows a waiting screen with the player's nickname.

### Playing
11. For each question the big screen shows the question text, the optional image, the answer options, a countdown and how many players have answered. It does not show which option is correct or the answer spread yet.
12. Phones show only the answer choices (coloured and labelled, for multiple-choice up to 4 options; true/false shows two) and the countdown. Tapping one locks in that player's answer; it can't be changed.
13. Each question has its own time limit (default 20 seconds if the quiz doesn't give one). The countdown is the same for everyone and is decided by the server, not the phones' clocks. Answers arriving after time is up are ignored.
14. The question ends when the time runs out or when every connected player has answered, whichever is first.
15. After each question the big screen shows the correct answer, a bar chart of how many players chose each option, and the leaderboard (top 5 with scores and rank changes). Each phone shows whether that player was right, the points earned, their total and rank. A player who did not answer earns 0 and is told so.
16. The host presses Next to move on. After the last question, Next becomes "Finish".
17. At the end the big screen shows a podium with the top 3 (ties share a rank) and the full ranking is available to scroll. Phones show the player's final rank and score.

### Scoring
18. A wrong or missing answer scores 0. A correct answer scores 500 plus up to 500 more depending on speed: 500 × (time remaining ÷ time limit), rounded to a whole number. So an instant correct answer scores 1000, one at the buzzer scores 500. A faster correct answer never scores less than a slower one on the same question.
19. Scores accumulate across questions. Ranking is by total score, descending; ties share a rank and are listed in join order.

### Real time and rejoin
20. Joins, answer counts, reveals and scores reach all screens within about 1 second on a local network, with 50 players.
21. If a player's phone loses connection or the page is reloaded, the player can reopen the join page, enter the same code and the same nickname, and return to the game with their score and answered state intact. The phone remembers the player in the browser, so a plain reload rejoins automatically.
22. Rejoin by nickname is allowed only for a player who is currently disconnected, so a second device can't take over a connected player's identity. The rejoin attempt of a connected nickname is rejected as "nickname taken".
23. Rejoining is possible in any game state, including after the game has started. A disconnected player is shown on the host's lobby list as disconnected and doesn't hold up "everyone answered".
24. The big screen shows a "reconnecting" notice when its own connection drops and recovers on its own when the connection returns.

### Robustness
25. The server ignores malformed messages, out-of-order actions (for example a player trying to start the game, or answering twice) and actions for the wrong room, without crashing or changing state.
26. Player-supplied text (nicknames) and quiz text are always shown as plain text, never interpreted as HTML.
27. The host and player pages are usable on a phone-sized screen (no horizontal scrolling at 360 px wide) and answer buttons are large tap targets and reachable by keyboard.
28. One room's activity doesn't affect another's; several rooms can run at once.

### Running
29. One command from `quiz/` (`npm start`) installs nothing beyond `npm install` and starts the server, then prints the host URL and the address phones should use on the local network. The port can be set with the `PORT` environment variable.
30. The app lives entirely in `quiz/`, has its own `package.json`, and nothing outside `quiz/` is changed or required.

## Open questions

- **Open question (stack):** the spec assumes Node.js 20+ with the `ws` WebSocket library and plain browser HTML/JS (no build step). Is that acceptable, or is another stack preferred?
- **Open question (scoring):** is the 500 + speed bonus up to 500 formula acceptable?
- **Open question (split):** is the three-part split acceptable, with the editor/import-export and results/CSV deferred to follow-up issues?
- **Open question (late joiners):** the spec blocks new players once the game has started (only rejoin is allowed). Should late joining be allowed instead?
