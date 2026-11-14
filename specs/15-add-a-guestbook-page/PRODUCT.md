# Guestbook page

## Summary

A page in the sandbox where visitors leave a name and a short plain-text message, and anyone allowed to read it can see the messages left so far, newest first. It is standalone: it lives in `guestbook/` and doesn't depend on or change the to-do app (#10).

Five decisions belong to the owner and are open (see the **Open question** entries). The behaviour below holds whichever way they go. Where an answer changes behaviour, the invariant says so and gives the recommended default as an assumption.

## Behavior

1. The guestbook page shows a list of messages and a form with a name field and a message field, plus a submit button. Each field has a visible label.

2. Each listed message shows the name, the message text and the time it was left. Messages are ordered newest first. Messages with the same timestamp keep a stable order between page loads.

3. The list shows 20 messages per page. When there are more than 20, the page offers "Older" and "Newer" navigation. The first page is the default, and the current page is part of the URL so it can be linked and reloaded. A page number beyond the last page shows the last page. A non-numeric or zero page number shows the first page.

4. When there are no messages, the page shows a friendly empty-state line and the form, not an error.

5. Name limits: required, plain text, at most 50 characters. Message limits: required, plain text, at most 500 characters. Leading and trailing whitespace is trimmed before checking, and a value that is empty after trimming is rejected. Exactly 50 and exactly 500 characters are accepted; 51 and 501 are rejected. Characters are counted as visible characters (a user-perceived character, such as an emoji, counts as one).

6. The form shows a live character count for each field. The browser stops typing beyond the limits, and the limits are enforced again on submission, so a request that skips the form is rejected too.

7. A rejected submission keeps what the visitor typed, shows an error next to the field that failed, and moves keyboard focus to the first field with an error. Nothing is stored.

8. Messages are plain text. Anything that looks like HTML or Markdown is shown literally and never interpreted or executed. Line breaks in a message are shown as line breaks. Names can't contain line breaks.

9. After a successful submission the visitor sees a clear confirmation. The form is cleared. Reloading the page doesn't submit the message a second time. Under the recommended default for moderation, the new message is visible at once.
   - **Open question (moderation):** do messages appear at once, or only after you approve them? If you approve them, how? *Recommendation:* appear at once, and you can hide or delete any message afterwards through a simple owner-only action protected by a secret. Reason: this keeps the page feeling alive, and approval-first needs an owner inbox and a notification route that the sandbox doesn't yet have. If you choose approval-first, the visitor sees "Thanks, your message will appear once it's approved" instead, and the message isn't listed until approved.

10. A visitor can't edit or delete their own message once it is sent. *Assumption, tied to the open question below.*
    - **Open question (what is kept about visitors):** only name and message, or an email address too, and for how long? Can visitors delete their own message? *Recommendation:* keep only the name, the message and the time. No email, no account, and nothing else about the visitor is shown or kept beyond what spam control (the next question) needs. Keep messages until you remove them. Don't let visitors delete: with no accounts there is no safe way to prove who wrote what. A visitor who wants a message removed can ask you, and you remove it by the owner action in invariant 9.

11. Spam is limited so that a script can't flood the page.
    - **Open question (spam):** hidden honeypot field plus a rate limit, a CAPTCHA, or your approval alone? *Recommendation:* a hidden honeypot field plus a rate limit (for example 3 messages per 10 minutes per visitor), with no CAPTCHA. Reason: it costs nothing, adds no third-party script and no friction for people. Revisit if spam appears.
    - Under that recommendation: a submission with the honeypot filled looks successful to the sender but isn't stored. A visitor over the rate limit is told to try again later, keeps what they typed and nothing is stored. The honeypot is invisible and skipped by keyboard and screen-reader users.

12. Who can read the guestbook is decided by the owner.
    - **Open question (who can read):** everyone, or only people who have the link? *Recommendation:* everyone. "Anyone can read the messages left so far" is the issue's purpose, and a link-only page is hard to keep private anyway. If link-only, the page is unlisted and carries a no-index signal, and the link must not be shown in any public index.

13. Storage persists across restarts and deployments: a message left now is still listed after the service restarts. If storage is unavailable, reading shows a plain "The guestbook is unavailable right now" message and submission shows a retry-later error that keeps the typed text. Neither exposes technical details.
    - **Open question (storage):** a small backend or a hosted service, at no cost. *Recommendation:* a single small backend with a file-based database (SQLite) on a free-tier host that offers persistent disk, or failing that a free hosted database tier. The owner should choose from what they already have an account for. Reason: one component, no vendor sign-up for visitors, and a file is easy to back up.

14. The page is usable by keyboard alone and by screen readers: the form controls are labelled, errors are announced, the confirmation is announced, and pagination links are real links. It works on a narrow phone screen without sideways scrolling.

15. The guestbook is self-contained: all its code and assets live in `guestbook/`, and nothing outside that folder (including anything for the to-do app) is added to, changed or required by it.

16. Two visitors submitting at nearly the same moment both have their messages stored, and neither overwrites the other.
