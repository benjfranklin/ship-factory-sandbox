# Guestbook page

## Summary

A page in the sandbox where visitors leave a name and a short plain-text message. Messages are shown to everyone only after the owner approves them, newest first. It is standalone: it lives in `guestbook/` and doesn't depend on or change the to-do app (#10).

The owner has settled the five decisions: moderation before display, what is kept (and for how long), spam controls, storage, and who can read. They are written into the behaviour below. Choosing and setting up a host is out of scope: the guestbook only has to run locally.

## Behavior

1. The guestbook page shows a list of approved messages and a form with a name field and a message field, plus a submit button. Each field has a visible label.

2. Each listed message shows the name, the message text and the time it was left. Messages are ordered newest first. Messages with the same timestamp keep a stable order between page loads.

3. The list shows 20 messages per page. When there are more than 20, the page offers "Older" and "Newer" navigation. The first page is the default, and the current page is part of the URL so it can be linked and reloaded. A page number beyond the last page shows the last page. A non-numeric or zero page number shows the first page.

4. When there are no approved messages, the page shows a friendly empty-state line and the form, not an error.

5. Name limits: required, plain text, at most 50 characters. Message limits: required, plain text, at most 500 characters. Leading and trailing whitespace is trimmed before checking, and a value that is empty after trimming is rejected. Exactly 50 and exactly 500 characters are accepted; 51 and 501 are rejected. Characters are counted as visible characters (a user-perceived character, such as an emoji, counts as one).

6. The form shows a live character count for each field. The browser stops typing beyond the limits, and the limits are enforced again on submission, so a request that skips the form is rejected too.

7. A rejected submission keeps what the visitor typed, shows an error next to the field that failed, and moves keyboard focus to the first field with an error. Nothing is stored.

8. Messages are plain text. Anything that looks like HTML or Markdown is shown literally and never interpreted or executed. Line breaks in a message are shown as line breaks. Names can't contain line breaks.

9. **Moderation.** A new message is not shown when it is sent. It waits as pending until the owner approves it. After a successful submission the visitor sees "Thanks, your message will appear once it's approved." and the form is cleared. Reloading the page doesn't submit the message a second time. A pending message is not listed on any page, in any count, or in any way to visitors.

10. **Owner page.** The owner has a page, separate from the public one, listing pending messages. For each the owner can approve it or delete it. Approving makes it appear on the public page, ordered by the time it was left, not the time of approval. Deleting removes it for good. The page is only available to someone who supplies the owner's secret token. Without a correct token it shows nothing about any message, and no change is possible. If no token has been configured, the owner page is disabled. **Proposed:** the owner page can also delete an already-approved message, so the owner can remove one later on request.

11. **What is kept.** Only the name, the message and the time it was left are kept for each message. No email, no account, and nothing else about the visitor is stored. Messages (pending or approved) are deleted automatically 12 months after they were left. Visitors can't edit or delete their own message once it is sent.

12. **Spam.** A hidden honeypot field plus a rate limit of 3 messages per 10 minutes per visitor. No CAPTCHA. A submission with the honeypot filled looks successful to the sender (same confirmation) but isn't stored. A visitor over the rate limit is told to try again later, keeps what they typed, and nothing is stored. The honeypot is invisible and skipped by keyboard and screen-reader users. What identifies a visitor for the rate limit is not kept after the 10-minute window, and is never stored with a message.

13. **Who can read.** Everyone: the public page needs no link secret, login or token, and may be indexed.

14. **Storage.** Messages are kept in SQLite behind one small Node backend, and persist across restarts: a message left now is still there after the service restarts. It runs locally with documented steps. If storage is unavailable, reading shows a plain "The guestbook is unavailable right now" message, and submission shows a retry-later error that keeps the typed text. Neither exposes technical details.

15. The page is usable by keyboard alone and by screen readers: the form controls are labelled, errors are announced, the confirmation is announced, and pagination links are real links. It works on a narrow phone screen without sideways scrolling. The owner page meets the same standard.

16. The guestbook is self-contained: all its code and assets live in `guestbook/`, and nothing outside that folder (including anything for the to-do app) is added to, changed or required by it.

17. Two visitors submitting at nearly the same moment both have their messages stored, and neither overwrites the other.
