---
name: outsystems-mentor-build
description: Drive a build inside a real ODC module through Mentor over the OutSystems MCP — screens, layouts, widget trees, theme CSS — and verify it actually landed. Use this skill whenever work is being applied to a live OutSystems app rather than emitted as files: creating or editing a screen, assembling a widget tree, pasting a theme, or sequencing and publishing Mentor turns. Also use it whenever a Mentor turn reports success and you are about to believe it.
---

# Driving an ODC build through Mentor

This skill is the **process and verification layer** for changing a live ODC module over the
MCP. It holds no platform facts: what a Layout is, which placeholders it exposes, which
widget is right for a piece of content — all of that is upstream's, cited below and never
restated here.

What it owns is the part that has repeatedly gone wrong in practice: **Mentor reporting
success for a change that did not land**, and agents inventing structure the platform
already defines.

## The one-line rule

> Build with what the platform already has, in the shape the module already uses — then
> **prove it in a browser**, because nothing else in the pipeline can see the failure.

## Before the first turn — read the module, don't guess it

Two lookups, always, before you write a prompt:

1. **The module's default Theme.** The same context lookup that returns the theme stylesheet
   also returns `layout` (name + key), `menu` and `grid`. That `layout` is the Layout block
   every screen in the module uses, and it is the one your screen must use. Do not infer it
   from the app name, and do not let Mentor pick.
2. **The existing screens.** They show you the convention already in force — which flow
   screens live in, how they are named, what the module actually looks like.

Guessing either is how a screen ends up detached from the app it lives in.

## Layouts and widget choice — upstream owns this, read it

| Need | Read |
| --- | --- |
| Which Layout, its placeholders, one-Layout-per-screen, the default-Layout deletion trap | `vendor/outsystems-frontend-skills/ui-frameworks/outsystems-ui/layouts.md` |
| Which widget for which content, and the semantic hierarchy | `vendor/outsystems-frontend-skills/common/atomic-design.md` |
| Headings, typography hierarchy, section structure | `vendor/outsystems-frontend-skills/ui-frameworks/outsystems-ui/polish-checklist.md` |
| Every block's arguments, placeholders, events | `vendor/outsystems-frontend-skills/ui-frameworks/outsystems-ui/blocks-index.md` |
| Where CSS belongs — Theme vs Screen vs Block | `vendor/outsystems-frontend-skills/common/css-customization.md` |
| Semantic structure and landmarks for a11y | `vendor/outsystems-frontend-skills/common/accessibility.md` |

Two upstream points are worth naming here only because ignoring them is the common failure,
not because this skill restates them:

- **A screen is not a blank page.** It wraps in exactly one Layout block, and content goes
  into that Layout's *named placeholders* — the page title in `Title`, the body in
  `MainContent`. Content authored at the screen root renders detached: no menu, no header,
  no chrome.
- **`Container` is the last resort in the widget hierarchy, not the default.** Reaching for
  a styled `Container` where an OS UI block or a semantic `AdvancedHtml` tag exists is a
  downgrade, and for headings it is an accessibility regression. See the polish checklist's
  typography table.

## ⚠ Never build UI as an HTML literal

Do not hand Mentor a block of markup to drop into a widget. Two distinct things are being
ruled out, and one thing is **not**:

| Approach | Verdict |
| --- | --- |
| A whole layout as an HTML string in an **Expression** widget | **Never.** |
| A whole layout as an HTML string in an **`AdvancedHtml`** widget | **Never** — same defect, different widget. |
| `AdvancedHtml` with a **single semantic tag** (`h1`…`h6`, `p`, `strong`) wrapping real content | **Correct, and upstream-mandated.** Not what this rule bans. |

The ban is on *pasting markup instead of building widgets*. It is not a ban on
`AdvancedHtml`, which is how you get a real `<h2>` instead of a `<div>` that merely looks
like one. An over-broad reading of this rule produces screens whose every heading is a
styled `Container` — which is its own defect.

Why the literal is wrong:

- **An Expression renders as a `<span>`.** Wrap a layout in one and nothing inside it is a
  widget the platform models. It cannot be inspected, restyled, reordered or reused in
  Studio or by a later Mentor turn, and the `ExtendedClass` / BEM-on-native-widgets approach
  has nothing to attach to.
- **`Escape Content = No` does not exist on the ODC reactive Expression.** It is a
  Traditional Web property. Mentor will accept the instruction, store an `EscapeContent`
  extended property, and report `change_applied: true` with zero validation errors — and the
  page will render `<div class="…">` as visible text. The model is not lying about what it
  stored; the property has no effect on that widget.

Build the tree instead: name each widget, its Style Class, and its literal text. A widget
tree in a prompt is longer to write than a blob of HTML and dramatically more reliable to
apply — measured on one real screen, a 52-item grid took `internal_retry_count` **1** and
**0** as two Container-tree turns, against **9** for the single HTML literal.

## Classing a widget — Style Classes first, Extended Properties as the fallback

A widget gets its classes from the **Style Classes** property. Use it whenever it can do the
job. Reach for an **Extended Properties** entry named `class` only when Style Classes cannot
— and when you do, carry the platform's classes yourself.

| Mechanism | What it does |
| --- | --- |
| **Style Classes** property | **Prefer this.** Appends to what the platform already put on the element. |
| Extended Property `class` | **Fallback only.** Sets the raw HTML attribute, *replacing* the platform's own classes wholesale. |

The order matters because of that difference: Style Classes adds, an Extended Property
`class` overwrites. Falling back is fine when it is the only way to express what you need —
but the moment you do, every class the platform would have supplied becomes yours to
restate, `btn` and any grid or layout classes included.

The failure mode is silent and total: the widget keeps rendering, so nothing in the pipeline
objects, but it has quietly lost every class the platform gave it.

**Buttons always keep the platform `btn` base.** A Button's classes must read `btn` first,
then any variant:

    btn                         <- correct: a bare Button is already the low-emphasis variant
    btn btn-primary             <- correct: framework variant
    btn uswds-btn--secondary    <- correct: design-system variant, base still present
    uswds-btn--secondary        <- WRONG: no base

The platform does **not** prepend `btn` for you — whatever the class string says is what
ships. Put the base and the modifier in one field together, and do not split them across
Style Classes and a second mechanism. If a Button has to be classed through an Extended
Property `class`, `btn` must appear in that value too; nothing adds it back.

**Why the base matters more than it looks.** A design-system button stylesheet typically
puts every visual property (background, border, padding, radius, type) on the `.btn` base
and lets the variant classes assign nothing but custom properties. That collapses dozens of
states into a flat, readable cascade — at the cost of zero graceful degradation. A button
carrying only `uswds-btn--secondary` sets nine custom properties that **nothing reads**, and
falls back to the browser's own `2px outset` / `padding: 1px 6px` chrome. Not degraded:
unstyled.

This shipped. On one `ButtonSpecimen` screen, 14 of 25 buttons rendered as raw browser
buttons behind a green build gate and a clean Mentor validation.

**Telling the two mechanisms apart from the DOM — a weak signal, so do not lean on it.** The
platform adds classes of its own (e.g. `ThemeGrid_MarginGutter`), and their absence alongside
a missing base *suggests* the attribute was replaced rather than appended. But it is a
correlate, not the thing:

- **False negative.** A Button with its margin property set to `None` emits no
  `ThemeGrid_MarginGutter` at all. Absence is not evidence of a clobber.
- **It misses the failure you are more likely to have.** A page can pass this check on every
  button while many carry the *wrong variant* — a well-formed class list with the wrong
  contents. Measured on one real screen: 25/25 passed the gutter check, 11 were wrong.

Measure `btn` presence directly — it is one line and it is the property you care about — and
assert the expected *variant* per row on top of it. Ask the model to read its own tree when
you need to know the mechanism.

**Check it in the browser, not in the model** — and assert the variants, not just the base:

```js
const b = [...document.querySelectorAll('button')];
({ total: b.length,
   missingBase: b.filter(x => !x.classList.contains('btn')).length,
   variants: b.reduce((m, x) => (m[x.className] = (m[x.className] || 0) + 1, m), {}) })
```

`missingBase` must be 0, and the variant tally must match what the design says each row holds.
A base-only check passes a page whose variants are scrambled.

**Defensive CSS is the belt, not the braces.** Writing the base rule as
`:is(.btn, [class*="ds-btn--"])` makes a modifier self-sufficient if someone does drop the
base, and keeps specificity at 0,1,0. Worth doing — but it repairs the stylesheet, not the
widget, and the widget is still wrong. Fix both.

## Sequencing turns

- **One coherent section per turn.** A turn carrying an entire large screen is the least
  reliable thing you can ask for.
- **Publish between turns.** Everything a turn changes lives only in the server-side session
  until published, and a session that GCs re-downloads the OML from the tenant — unpublished
  work is simply gone. Publishing between turns means a failure costs one turn, not the
  build.
- **Publish the data model before the screens that bind to it.**
- **Resume the same session** with `mentor_session_id` + the newest `mentor_session_token`.
  A failed or cancelled turn never advances committed state — resume it, don't start a fresh
  `app_key` session, which burns a tenant slot and drops unpublished edits.
- **On a timeout, raise `max_turn_time` and retry the same app.** Never create a new app; that
  discards everything the session already applied.
- **When moving existing widgets, say MOVE.** "Move the tree into the Content placeholder,
  do not rebuild it" preserves classes, text and order. Left ambiguous, Mentor may
  regenerate — and a regenerated tree is a new chance to drift from the spec.

## Verification — the part that is not optional

**`change_applied: true` is not evidence.** Neither is `validation.error_count: 0`, nor a
confident summary. All three were present on a screen that shipped rendering its own markup
as visible text.

Nothing else in the pipeline can catch this class of defect: the deterministic build gate
checks the repo, not the tenant; Mentor's validation checks the model, not the rendering;
Studio Preview is explicitly not trusted. **Load the published URL in a real browser.**

After publishing, check in this order:

1. **The publish actually deployed.** `no_changes_detected: true` means the deployment live
   in that environment is *not* this publication — nothing landed. Never report it as shipped.
2. **The screen renders at all**, and its title is right.
3. **The app's chrome is present** — menu, header. Absence means the Layout was skipped.
   Check this deliberately: *a screenshot cropped to the content area looks identical either
   way*, which is exactly how a layout-less screen gets signed off.
4. **No HTML tag text is visible.** Reading `<div class="…">` or `&#45;` as words on the page
   means the UI was built as a literal.
5. **The widget tree is what you asked for** — the elements are real widgets, headings are
   real headings, not one `Expression`.
6. **The content itself** — counts, values, order, against the spec of record.
7. **The rendered class strings**, for anything you styled. A widget that lost its
   platform base class still renders, so only the DOM shows it — see "Classing a widget"
   above for the one-liner.

Read the model back through the context lookups as well as looking at the page, but treat
the browser as the authority. Note that the context lookups can lag a publish by some
seconds; a screen missing from `context_screens` immediately after a successful publish is
usually index lag, and the browser settles it.

### Per-widget property edits can silently no-op — check the revision number

Bulk work (replacing a theme stylesheet, a screen stylesheet) applies reliably. **Per-widget
property edits are a different story**, and the failure is not an error — it is a confident,
itemised, entirely fabricated report.

Observed across eight attempts on one screen: three applications that moved values onto the
wrong widgets, one that damaged widgets which had been correct, and one complete no-op whose
terminal result carried a verbatim per-widget "read-back" table and six passing self-checks
for changes that were never written.

**The mechanism: positional addressing is unsound, because traversal order is not render
order — and is not stable between sessions.**

This is the part worth carrying. On the screen in question, the widgets in *flat sibling*
rows were enumerated in render order every single time. The widgets in rows nested inside a
wrapper Container were not. One session's enumeration matched the rendered DOM one-for-one,
so keying the edit on "your own tree order" looked like the fix; a **fresh session over the
same OML produced a different order** and wrote ten individually-correct values onto ten
wrong widgets.

So every positional scheme fails here, and they fail *plausibly* — the values are right, the
count is right, only the assignment is wrong:

| Addressing | How it failed |
| --- | --- |
| Row label ("the row labelled accent cool") | Mapping slipped one row; correct rows damaged. |
| Ordinal ("the 15th Button") | Values applied in a shuffled order. |
| The model's own enumeration | Correct in the session that produced it, wrong in the next. |

**A self-check on the unambiguous rows does not protect you.** Asking it to verify the first
N widgets before editing passes trivially, because the flat rows are exactly the ones that
always map correctly. The check confirms nothing about the nested ones you actually care
about.

Prefer an addressing scheme the tree itself carries — a widget Name where one exists, or a
property already unique to the target. Where the widgets are anonymous and only position
distinguishes them, per-widget editing over this transport is not reliable; say so and hand
it over.

Two cheap tells, neither of which requires trusting the summary:

1. **The publish creates no new revision.** A publish that returns `succeeded` with the SAME
   revision number as the previous one deployed nothing, whatever `no_changes_detected` says.
   Record the revision before you start.
2. **A read-back that disagrees with itself.** If you ask the model to re-read the values it
   just wrote and some *unrelated* property (Enabled, caption, order) has drifted from an
   earlier enumeration of the same tree, the read-back is describing a tree that does not
   exist. Treat the whole report as void.

When per-widget edits fail twice, stop and hand the human a checklist. More turns cost far
more than the couple of minutes the edit takes in Studio, each one risks damaging widgets
that were already right, and if the targets are distinguished only by position the next
attempt is not more likely to work than the last.

## Reporting

Say which revision landed, and what you verified versus what you inferred. When a turn
reported success and the browser disagreed, say so plainly — that gap is the single most
useful thing to carry back, and burying it is how the next build repeats it.

If a Mentor turn's terminal result carries `internal_retry_count >= 3`, submit an
`agent_observation` / `builder_retry_friction` through `submit_feedback`. It is the signal
that the prompt shape is fighting the builder.
