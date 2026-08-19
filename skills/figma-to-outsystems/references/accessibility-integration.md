# Where accessibility comes from, phase by phase

There is no `outsystems-accessibility` skill in this plugin. Accessibility is not one skill that
runs alongside the others — it is three sources with a clean split, and this file says which one
applies where.

| Source | Owns | Lives in |
| --- | --- | --- |
| **Upstream** | WCAG 2.1 AA, and how each criterion maps onto an OutSystems widget — `Label.TargetWidget`, `Mandatory`, `Fieldset`, `AdvancedHtml` headings, alt text | `vendor/outsystems-frontend-skills/common/accessibility.md` |
| **`outsystems-design-findings`** | The 2.2-over-2.1 delta, contrast maths, and the fix-vs-flag decision | `references/wcag-2.2-delta.md`, `references/contrast-calculator.md`, and the table in its `SKILL.md` |
| **`outsystems-web-component`** | The inside of a hand-built custom element — keyboard, ARIA, focus management | `references/keyboard-patterns.md`, `references/aria-recipes.md` |

The split follows the plugin's boundary rule: upstream owns platform facts, the loop owns policy and
anything upstream has no reason to cover. Nothing here restates a rule from any of the three — go
read the source.

## Per phase

### Phase 1 — `outsystems-component-audit`
Flow-level criteria surface here and nowhere else, because they are only visible across a set of
frames: consistent help (3.2.6), redundant entry (3.3.7), accessible authentication (3.3.8). Also
flag designed targets under 24px, drag-only interactions with no alternative, and states the design
never draws a focus treatment for. All of it goes into the audit's findings register.

### Phase 2 — `outsystems-token-extractor`
Compute contrast for every pair the palette implies, annotate the measured ratio in a comment beside
each token, and **emit the token as designed**. A failing pair is an `accessibility/contrast`
finding; the suggested shade goes in the finding, never into `:root`.

### Phase 3, L1–L4.5 — `outsystems-bem-css`
Auto-applied because none of it changes a decision the designer made: `:focus-visible` treatments in
the design's own colours, `prefers-reduced-motion` guards on anything that transitions,
`[aria-disabled="true"]` styling beside `:disabled`, and target sizing **where the layout has room
to grow**. Where it does not have room, that is a finding.

### Phase 3, L5 — `outsystems-web-component`
A hand-built element ships with none of the accessibility a framework pattern provides. Semantic
HTML inside the Shadow DOM, keyboard handlers per the pattern, ARIA state, focus management on
open/close, and a single-pointer alternative for any drag (2.5.7). See that skill's references.

### Phase 4 — `outsystems-style-guide-doc`
The Accessibility Report section is mandatory: what was verified, what needs manual testing, what is
not applicable, which SCs were addressed, and every open finding against the component.

## There is no "skip accessibility" override

Earlier versions of this plugin accepted `"skip accessibility for this"` and emitted a
`// WCAG-OVERRIDE` comment. That path is gone, and its absence is deliberate.

It was built for a workflow where accessibility fixes *overwrote* the design — so an escape hatch
was needed to protect fidelity. Fidelity-first inverted that: the implementation already matches the
design, and conflicts already leave as findings rather than edits. There is nothing left for an
override to protect, so what it actually bought was silence.

The one override that exists runs the other way — a brand owner approving a specific deviation, per
finding, recorded against its ID. See **Brand-owner override** in `outsystems-design-findings`.

## Before any component is marked Approved

- Every criterion in upstream's `common/accessibility.md` addressed, plus the seven in
  `wcag-2.2-delta.md`
- Accessibility Report section of the Style Guide page complete
- Keyboard-tested: Tab, Enter, Space, arrows, Esc
- Tested at 200% zoom
- Tested with `prefers-reduced-motion: reduce`
- Contrast verified by measurement, not by reading the source
- Screen-reader announcement checked (VoiceOver / NVDA / JAWS)

The last two are the ones that get skipped, and they are the two a source-only review cannot fake.
