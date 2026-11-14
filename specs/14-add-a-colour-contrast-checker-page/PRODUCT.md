# Colour contrast checker page

## Summary

A small static page at `contrast/` that tells anyone choosing text and background colours whether the pair is readable, using the WCAG 2.2 contrast ratio. It is self-contained: no build step, no external scripts, fonts or stylesheets, and no dependence on the to-do app (#10).

## Layout options

| | 1. One pair at a time | 2. Palette grid (up to 8 colours) | 3. Hybrid |
|---|---|---|---|
| Shape | Two colour inputs side by side above a large live preview | Up to 8 colour inputs; an N×N grid of pairs, each cell showing ratio and passes | Grid; selecting a cell opens the single-pair view |
| Best for | Checking one combination | Checking a whole palette | Both |
| Phone use | Easy: stacks into one column | Hard: up to 56 cells need horizontal scrolling or a reflow | Hard, as for the grid |
| Accessibility | Simple: labelled inputs, one results region | Needs grid semantics (table with row and column headers), and each cell must be announced in full | Grid cost plus focus management between views |
| Build and test cost | Lowest | Medium to high | Highest |

**Recommendation: option 1.** It meets every requirement in the issue with the least risk, and is the best fit for the keyboard, screen reader and phone requirements. The pair logic is kept separate from the page, so a grid can be added later as its own item (see TECH.md, Follow-ups). The behaviour below describes option 1.

**Open question:** which layout do you choose: 1 (recommended), 2 or 3? If you choose 2 or 3, this spec's Behavior section needs revising before implementation.

## Behavior

1. The page opens at `contrast/index.html` and works when opened straight from disk or served as static files. It makes no network requests for scripts, fonts, stylesheets or anything else.
2. There are two colours: a text colour and a background colour. Each can be set by typing a hex value or by using the browser's colour picker. The two controls for a colour always show the same colour.
3. The page starts with a default pair that is already valid (black text on white), so results are visible on load.
4. Hex input accepts `#rgb` and `#rrggbb`, with or without the leading `#`, in either case. Anything else is invalid.
5. While a hex value is invalid, the page shows an error message beside that field, says the last valid colour is still in use, and doesn't change the results or the sample. Fixing the value clears the message.
6. Results update live on every change, with no submit button.
7. The contrast ratio is computed as defined by WCAG 2.2 (relative luminance of the lighter colour plus 0.05, divided by that of the darker plus 0.05) and shown as `N.NN:1`, from `1.00:1` to `21.00:1`. The displayed value is truncated, not rounded, to two decimals, so a failing pair never displays a threshold value such as 4.50.
8. Four results are shown, each with the word "Pass" or "Fail" (never colour alone) and its threshold:
   - AA normal text, 4.5:1;
   - AA large text, 3:1;
   - AAA normal text, 7:1;
   - AAA large text, 4.5:1.

   A result passes when the exact, unrounded ratio is greater than or equal to its threshold.
9. A sample shows text in the chosen text colour on the chosen background colour. It includes a normal-size line and a large-size line (at least 24px, or 18.66px bold, the WCAG large-text sizes).
10. The page's own controls and results don't use the chosen colours. They keep a fixed, readable style, so a bad pair can't make the checker unreadable. Only the sample uses the chosen colours.
11. Everything can be done with the keyboard alone, in a logical tab order, with a visible focus indicator on each control.
12. With a screen reader: every input has a programmatic label; error messages are tied to their field; the ratio and the four results are in a live region that announces the new result politely after a change; the sample is described as a sample, with the colours stated in text.
13. The page works down to a 320px-wide viewport without horizontal scrolling: the two colours stack into one column.

## Out of scope

Saving or sharing colours, suggesting fixes, alpha transparency, colour formats other than hex, and any change to the to-do app or to files outside `contrast/`.
