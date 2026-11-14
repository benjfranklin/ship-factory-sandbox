# Tech spec: colour contrast checker

## Context

The repository tracks only an empty `README.md` (`/sf/in/repo-files.txt`). It has no build, no package manifest, no test setup and no conventions file. `contrast/` is therefore a new, self-contained folder and nothing else changes, including the to-do app planned in #10. Behaviour is in `PRODUCT.md`; this spec assumes the owner picks layout option 1 (the open question there).

## Proposed changes

New files only, all in `contrast/`:

- `index.html`: the markup. Two `<fieldset>` groups (text and background), each with a `<label>`ed `<input type="text">` for hex and a labelled `<input type="color">`; an error `<p>` per field referenced by `aria-describedby` and `aria-invalid`; a results section; the sample. Links `style.css` and `<script src="contrast.js">` and `app.js` by relative path (plain scripts, not modules, so it works from `file://`).
- `contrast.js`: pure logic with no DOM access: `parseHex(str)` returns `{r,g,b}` or `null` (invariant 4); `luminance(rgb)` with the WCAG 2.2 sRGB formula (linearise with the 0.04045 breakpoint); `ratio(a,b)` (exact); `format(ratio)` truncating to two decimals (7); `results(ratio)` returns the four pass or fail flags against 4.5, 3, 7 and 4.5 (8). It attaches to `globalThis.Contrast` and also sets `module.exports` when `module` exists, so Node can test it.
- `app.js`: DOM wiring. It keeps the last valid colour per field. On `input` events it syncs hex and picker (2), validates (5), recomputes, and updates the ratio, the four results, and the sample's inline colours (6, 9). The results container is `role="status"` (`aria-live="polite"`) so changes are announced (12). Each result reads, for example, "AA normal text: Pass (needs 4.5:1)".
- `style.css`: fixed colours for the page chrome (10); `:focus-visible` outlines (11); a single-column flex or grid layout that goes two-up above about 600px (13); the sample's large line uses 24px.
- `contrast.test.js`: Node built-in test runner tests for `contrast.js` only. This is a dev-only file with no dependencies.

Trade-off: keeping logic in a DOM-free file costs one more file, but makes the maths testable without a browser and lets a later grid reuse it. Plain script tags avoid a build step and any `file://` module-loading limits.

## Testing and validation

The acceptance criteria are in `criteria.json`.

- AC1 (ratio): `node --test contrast/` tests known pairs: black on white is 21.00; white on white is 1.00; `#777777` on white is 4.48 (truncated); `#767676` on white is 4.54; `#fff` and `FFFFFF` parse equally. Also tests the truncation edge: a ratio just under 4.5 must not display `4.50`.
- AC2 (pass/fail): unit tests check each threshold at, just below and just above its boundary (exact ratio, not the displayed one).
- AC3 (input): unit tests for `parseHex` valid and invalid forms (4). By hand, check that typed hex and the picker stay in sync and that an invalid value shows the message and leaves the last results (2, 5, 6).
- AC4 (no external resources): review. Grep `contrast/` for `http`, `//`, `@import` and `src=`/`href=` values to confirm every one is relative, and open the page from disk with the network off (1).
- AC5 (keyboard): by hand, tab through the whole page, use every control, and see the focus indicator (11).
- AC6 (screen reader): review of the markup for labels, `aria-describedby`, and the live region (12). Spot-check by hand with a screen reader if available.
- AC7 (layout): by hand at 320px width, no horizontal scroll, sample visible (13, 9, 10).

## Risks and mitigations

- `<input type="color">` only produces `#rrggbb`, and its appearance differs by browser. The hex field is the primary path; the picker only syncs the value.
- Announcing on every keystroke can be noisy. The region is polite and its text is replaced only when the ratio changes.

## Follow-ups

- If the owner wants the palette grid (option 2 or 3), it becomes a separate item built on `contrast.js`, using a `<table>` with row and column headers.
- `README.md` is empty; a link to `contrast/` could be added there, but this item doesn't touch it.
