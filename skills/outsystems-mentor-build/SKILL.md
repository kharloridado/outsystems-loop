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
- **`Container` is the last resort in the widget hierarchy, not the default.** Upstream's
  order is: **OS UI block → `AdvancedHtml` with a semantic HTML5 tag → platform interactive
  widget → `Container`**. Reaching for a styled `Container` where any of the first three
  exists is a downgrade, and for headings it is an accessibility regression. See the polish
  checklist's typography table.

  Read the hierarchy as a *search order you must actually walk*, not a preference. The two
  steps skipped most often, because a `Container` looks right on screen either way:

  | Content | Correct | The downgrade |
  | --- | --- | --- |
  | Screen title | `AdvancedHtml Tag="h1"` in the Layout's `Title` placeholder — **one per screen** | plain text in `Title`, which emits no heading at all |
  | Titled group of content | the **`Section`** block (`Title` + `Content` placeholders), or **`SectionGroup`** to compose several with a sticky index | `Container` + a styled title `Container` |
  | Section heading | `AdvancedHtml Tag="h2"`–`"h6"`, in document order, no skipped levels | styled `Container` |
  | Body copy | `AdvancedHtml Tag="p"` | `Container` |

  **Confirm the block before you name it in a prompt.** Search the tenant for it and read
  its real placeholders — they vary by OS UI version. `Section` in `OutSystemsUI_2_16`
  exposes `Title` and `Content` only, while the pattern reference also documents an
  `Actions` placeholder; a prompt that fills `Actions` against that version fails. And if
  Mentor cannot find a block, it must **say so, not silently substitute a `Container`** —
  tell it that explicitly in the prompt.

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

   **Assert the document outline, don't eyeball it.** This is the one check that catches a
   page built entirely from styled `Container`s, and nothing else in the pipeline can: such
   a page renders *pixel-perfect*, passes validation with zero errors, and looks correct in
   every screenshot. In the browser:

   ```js
   [...document.querySelectorAll('h1,h2,h3,h4,h5,h6')].map(h => h.tagName + ': ' + h.textContent.trim())
   ```

   - **Zero headings on a content screen is a FAIL**, always. It means the `Title`
     placeholder got plain text and every section heading is a `div`.
   - Expect **exactly one `h1`** (the screen title, from the Layout's `Title` placeholder)
     and one `h2` per section, in document order.
   - A useful companion signal: count `div`s inside your content root. A specimen or detail
     screen in the low hundreds of `div`s with no headings is the signature of this defect.

   Measured on a real build: two reference screens shipped with **0** headings and 108 and
   246 `div`s. Both had passed a Mentor turn reporting `change_applied: true`, zero
   validation errors, and a visual screenshot review.
6. **The content itself** — counts, values, order, against the spec of record.

Read the model back through the context lookups as well as looking at the page, but treat
the browser as the authority. Note that the context lookups can lag a publish by some
seconds; a screen missing from `context_screens` immediately after a successful publish is
usually index lag, and the browser settles it.

## Reporting

Say which revision landed, and what you verified versus what you inferred. When a turn
reported success and the browser disagreed, say so plainly — that gap is the single most
useful thing to carry back, and burying it is how the next build repeats it.

If a Mentor turn's terminal result carries `internal_retry_count >= 3`, submit an
`agent_observation` / `builder_retry_friction` through `submit_feedback`. It is the signal
that the prompt shape is fighting the builder.
