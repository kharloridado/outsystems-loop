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
| **Every block's arguments, placeholders, events — read this BEFORE reaching for a `Container`** | `vendor/outsystems-frontend-skills/ui-frameworks/outsystems-ui/blocks-index.md` |
| Picking the right block for a piece of content | `vendor/outsystems-frontend-skills/.claude/skills/references/component-selection.md` |
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
  typography table, and the ladder below — this is the mistake that survives fixing the
  obvious one.

## ⚠ Build from the widget ladder — not from markup, and not from bare Containers

There are **two** mistakes here, and fixing only the first one still leaves a bad screen.

### Mistake 1 — pasting markup

Do not hand Mentor a block of HTML to drop into a widget:

| Approach | Verdict |
| --- | --- |
| A whole layout as an HTML string in an **Expression** widget | **Never.** |
| A whole layout as an HTML string in an **`AdvancedHtml`** widget | **Never** — same defect, different widget. |
| `AdvancedHtml` with a **single semantic tag** (`h1`…`h6`, `p`, `strong`) wrapping real content | **Correct, and upstream-mandated.** Not what this rule bans. |

The ban is on *pasting markup instead of building widgets*. It is **not** a ban on
`AdvancedHtml`, which is how you get a real `<h2>` instead of a `<div>` that merely looks
like one.

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

### Mistake 2 — a tree of generic Containers

Escaping the literal by building everything from `Container` + hand-rolled classes is the
*second* failure, and it is the easy one to feel good about. Upstream's hierarchy is explicit
(`.claude/commands/design-to-app.md`, citing `ui-frameworks/outsystems-ui/SKILL.md`):

> OS UI block → `AdvancedHtml` with semantic HTML5 tag → platform interactive widget →
> `Container` (last resort only). Never style a `Container` to LOOK like a card/button/banner
> — use the matching block.

So walk the ladder for **every** part of the tree, and only fall to `Container` when nothing
above it fits:

| The thing you are building | Reach for | Not |
| --- | --- | --- |
| A titled group of content | `Section` (`UsePadding`; `Title` / `Actions` / `Content`) | a `Container` + your own heading + border CSS |
| A responsive grid of items | `Gallery` (`RowItemsDesktop` / `RowItemsTablet` / `RowItemsPhone`, `ItemsGap`) | a `Container` with your own flex/`nowrap` rules |
| A heading | `AdvancedHtml Tag="h2"` | a `Container` styled to look like one |
| A card, tabs, an accordion, a list | the matching OS UI block — check `blocks-index.md` | a parallel implementation |

Two things you get for free by doing this, which a `Container` tree makes you re-invent
badly: **the responsive behaviour** (a `Gallery` reflows by breakpoint; a hand-rolled
`flex-wrap: nowrap` row does not) and **the semantics** (a `Section` title is a real heading;
a styled `div` is not).

"Nothing in OutSystems UI does this" is a claim about a catalog this skill does not carry.
Check `blocks-index.md` before making it.

### What a good prompt looks like

Name each widget, its block type, its Style Class and its literal text — a tree, not markup:

```
Section [uswds-specimen__section]  UsePadding=False
  Title:   "Theme palette"
  Content:
    Gallery [uswds-specimen__ramp]  RowItemsDesktop=7 RowItemsTablet=4 RowItemsPhone=2 ItemsGap="xs"
      Content:
        Container [uswds-specimen__item]
          Container [uswds-specimen__chip uswds-swatch--base-lightest]   (empty)
          Container [uswds-specimen__name] "--color-base-lightest"
          Container [uswds-specimen__hex]  "#F0F0F0"
```

`Container` survives at the leaves — a colour chip really is just a painted box, and that is
what last-resort means. It does not survive as the layout.

### Measured

On one real 52-swatch screen, the same content three ways:

| Built as | `internal_retry_count` | Outcome |
| --- | --- | --- |
| One HTML literal in an Expression | **9** | Shipped rendering its own tags as visible text |
| A tree of 233 generic Containers | **1**, then **0** | Rendered correctly; responsive behaviour hand-rolled, headings not headings |
| `Section` + `Gallery` + leaf Containers | — | Grid and titled groups come from the framework |

The widget tree is both more reliable for the builder *and* the architecturally correct
answer. But note what the middle row cost: it passed a browser check and still had to be
redone, because "not a literal" is a lower bar than "the right widget".

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
6. **The right widgets, not merely real ones.** A screen built entirely from `Container` will
   pass every check above and still be wrong. Ask: is each titled group a `Section`, each
   grid a `Gallery`, each heading an `AdvancedHtml` tag? Resize the window — if the layout
   does not reflow, the responsive behaviour was hand-rolled instead of inherited.
7. **The content itself** — counts, values, order, against the spec of record.

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
