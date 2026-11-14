# To-do app with accounts: phase 1, the backend API

## Summary

A small multi-user to-do service. People sign up and sign in, and each person can only see and change their own lists. Lists hold items that can be ticked off, reordered and given due dates, and the API marks overdue items. This first phase delivers the backend API, its database, automated tests and a local run command. The browser front end is a later phase (see the split proposal).

"User" here means the caller of the HTTP API (the future front end, or a developer with `curl`).

## Behavior

### Accounts and sign-in

1. A caller can sign up with an email address and a password. On success the account exists and the caller is signed in.
2. Sign-up is rejected with a clear error (4xx, JSON body with a message) when the email is malformed, the password is shorter than 8 characters, or the email is already registered. Email comparison ignores case. The error for a duplicate email does not change the existing account.
3. A caller can sign in with a registered email and password. A wrong password and an unknown email give the same error, so callers can't tell which accounts exist.
4. A caller can sign out. After that, the previous session no longer works.
5. Every list and item operation requires a signed-in caller. Without a valid session the API answers 401 and changes nothing.
6. Passwords are never returned by the API or stored in readable form.

### Lists

7. A signed-in caller can create a list with a name (1 to 100 characters after trimming), rename it, delete it, and fetch all of their lists. A new account has no lists.
8. Fetching lists returns only the caller's own lists, in creation order.
9. Deleting a list deletes its items.
10. A caller who asks for, changes or deletes another person's list, or any item in it, gets 404, the same answer as for something that does not exist. Nothing is changed or revealed.

### Items

11. A caller can add an item to one of their lists with a title (1 to 200 characters after trimming) and an optional due date (a calendar date, no time of day). New items are unticked and go to the end of the list.
12. A caller can edit an item's title and due date, and can clear the due date.
13. A caller can tick an item as done and untick it. Ticking does not change its position.
14. A caller can delete an item. The remaining items keep their relative order.
15. Fetching a list returns its items in their current order. Each item has a title, done state, due date (or none) and an `overdue` flag.
16. An item is overdue when it has a due date earlier than today and is not done. An item due today is not overdue. Ticking an overdue item clears its overdue flag; unticking it sets it again if the date is still past. "Today" is the server's calendar date in UTC.
    - **Open question:** should "today" follow each person's own time zone instead of UTC?

### Ordering

17. A caller can move an item to a given position within its list. The other items keep their relative order and positions stay gap-free, so the order the caller sees is the order saved. A position beyond the end places the item last; a negative or non-numeric position is rejected with 4xx.
18. Order changes persist across sign-out, sign-in and server restart.
19. Items can't be moved between lists in this phase.

### Errors and robustness

20. Invalid input (missing title, over-long text, impossible date such as 2026-02-30, unknown fields of the wrong type) returns 4xx with a JSON message and changes nothing.
21. Two quick changes to the same list (for example two moves) leave it consistent: no duplicate or missing positions.

### Running it

22. A developer can start the service locally with a single documented command, with no external services to install, and run the automated tests with another single command.

## Open questions

- **Stack:** this spec assumes Node.js with a small HTTP framework and SQLite, so it runs locally with no setup. Does the owner want a different stack?
- **Sign-in mechanism:** this spec assumes email and password with a server-side session in an HTTP-only cookie. Is that acceptable, or should it be tokens or an external identity provider?
- **Deployment target:** left out of this phase. Where should it run?
