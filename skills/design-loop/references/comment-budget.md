# Comment budget — the artifact is not the review surface

Applies to every file the loop writes: `tokens/*.css`, `src/blocks/*.css`,
`src/components/*.js`, `style-guide/`. Read by the `maker` (what to write), the `checker`
(what to fail), `outsystems-bem-css` and `outsystems-token-extractor`.

**The rule: the code says WHAT. The PR says WHY.**

## Why this exists

The loop's reasoning used to be written into the artifact. One block CSS reached `main` at 941
lines, of which the first 120 were a numbered argument about which framework dropdown pattern the
design meant — a decision log, a deferral rationale, a ref-vs-drawing discrepancy list and an
unfiled contrast observation, in a file whose entire job is to be pasted into an ODC Block.

Every reader was the wrong one. The **reviewer** reads the PR, where the same decision log already
sits verbatim under §Decision log. The **developer** pastes the file into ODC and scrolls past it.
The **customer** gets `build:theme:ship`, which strips ordinary comments — so prose that only ever
appears in the dev copy was never documentation; it was a PR body written into the wrong file.
And a comment 400 lines into a stylesheet is unfindable and unreviewable: nobody diffs it, nobody
is notified by it, and it goes stale the first time the file is edited.

The reasoning is not being thrown away. Every row below has a destination that a person can
actually find.

## Where each kind of prose goes

| Prose | Destination | Not |
|---|---|---|
| Approach chosen, alternatives ruled out, assumptions | PR body §Decision log, verbatim | the file header |
| What this item deliberately does **not** build, and why | PR §What a human still has to check + the item's issue | a banner in the CSS |
| Ref-vs-drawing disagreements, re-ref requests | `loop/refs/<id>/spec.md` under `## Ref discrepancies` | a comment block |
| "What the framework actually emits" selector inventories | the maker's DECISION-LOG → PR §Plan | the file header |
| Design variant → framework class mapping | `handover/<artifact>.md` — the developer sets the widget's Style property from it | the file header |
| Computed contrast ratios that pass | PR §Gates | beside every token |
| A design conflict built faithfully | a filed finding (bug) | an `OBSERVATION:` note in code |
| A conflict you noticed but chose not to file | `findings/findings-register.md` | a code comment |

## The budget

Per file:

- **Header: 4 lines maximum** — what the file is, its spec ref path, the framework baseline it was
  written against. Plus the `/* @section Group / Name */` line where the theme build needs one.
- **Section markers: one line, no ASCII art** — `/* Sizes */`. No box-drawing characters, no rules
  of `═` or `─`, no centred banners.
- **Inline: one line each, and only where the code cannot say it itself.** The whole allowed list:
  a non-obvious framework selector, a justified `!important`, a deliberate `:not()` exclusion of a
  framework sibling, a `:host` fallback literal, and a value whose conflict is filed —
  `/* #12 filed: 2.4:1, built as drawn */`.

Rough ceiling: **comments under 10% of the file's lines, and no comment over 3 lines.** If you are
writing paragraph four, you are writing the PR body in the wrong file. If a note does not fit the
budget, it is not too important to cut — it is too important to bury, so move it to its
destination above.

## Not a comment problem — leave these alone

- The TOC and section banners in `dist/theme.css`. The build generates them and the project
  requires them; they survive `build:theme:ship` on purpose.
- A Web Component's header API contract — attributes, properties, events, slots, one line each.
  That is the element's public interface and it is read in ODC, not in review. Keep it; it is not
  a place for rationale.
- Licence and attribution headers in vendored files.
