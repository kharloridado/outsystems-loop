# The Escalation Ladder (L1–L5)

Always try cheapest first. Stop when satisfied.

## L1 — Token change in `:root`
**When:** Design differs only in token values.
```css
:root {
  --color-primary: #7C3AED;
}
```
**Effort:** 1 min. **Affects:** Global.

## L2 — Existing OS UI utility class
**When:** Adjustment that OS UI utilities cover.
```
Style Classes = "mt-3 px-2 text-neutral-8 d-flex"
```
**Effort:** 30 sec. **Affects:** Widget only.

**Utilities to know:**
- Spacing: `mt-0..6`, `mb-`, `mx-`, `my-`, `p-`, `pt-`, `px-` (with breakpoint suffixes)
- Display: `d-none`, `d-block`, `d-flex`, `d-grid`
- Text: `text-neutral-0..10`, `text-primary`, `text-success`, `text-center`, `text-bold`
- Background: `background-primary`, `background-neutral-1`
- Headings: `heading1-6`, `font-size-display`, `font-size-xs..2xl`
- Visibility: `HiddenInDesktop`, `HiddenInSmartphone`, `ShowOnlyInMobile`

## L3a — Restyle the framework's own class
**When:** OS UI already names this thing — the widget class or one of its variants. **This is
the default, and most of a design system lands here.**

```css
.btn         { border-radius: var(--border-radius-soft); font-weight: var(--font-weight-bold); }
.btn-primary { background-color: var(--color-primary); color: var(--color-base-white); }
```
**Effort:** minutes. **Affects:** every instance in the app, with nothing to apply — the
developer just sets the widget's Style property.

`.acme-button.btn` is **not** L3a. It gates the override behind a class the platform never
emits, so every widget nobody hand-edits renders unbranded. Restyle the bare class.

Restyling a bare class disturbs its siblings: `.btn { height }` beats `.btn-small { height }`
on source order at equal specificity. Map every sibling or exclude it (`.btn:not(.btn-small)`).

## L3b — ExtendedClass + BEM modifier
**When:** Visual variant of an existing OS UI pattern that the framework ships **no class for**.
Verify the gap in the widget SCSS *and* `src/scripts/OSFramework/OSUI/Pattern/*/Enum.ts`.

```
ExtendedClass = "acme-card acme-card--featured"

.acme-card--featured {
  border: var(--border-size-m) solid var(--color-primary);
}
```
**Effort:** 5–15 min. **Affects:** Pattern-wide where applied.

## L4 — Wrap pattern in custom Block
**When:** Need own API (input parameters) on top of OS UI pattern.

```
Block: ProductCard in Patterns Library
  Inputs: ProductName, Price, ImageURL, OnAddToCart
  Internal: OS UI Card pattern with ExtendedClass
  Block CSS: BEM rules
```
**Effort:** 30–60 min. **Affects:** All uses of the Block.

## L5 — Vanilla JS Web Component + Block wrapper
**When:** No OS UI pattern fits; truly custom UI.

Deliverables:
1. `acme-{component}.js` (Web Component with Shadow DOM)
2. OutSystems Block that wraps it (the API surface for other devs)
3. Standalone test HTML page

**Effort:** 1–4 hours. **Best for:** custom controls outside OS UI's 70 patterns.

## L6 — Fork OutSystems UI
**Almost never.** You'd lose all upstream updates.

## Healthy distribution
- L1/L2: 60–70%
- L3: 20–25%
- L4: 5–10%
- L5: 1–5%
- L6: 0%

Skewing higher than this means the design system is fighting OutSystems UI.

## Why Web Components for L5?

Better than building Blocks with raw HTML widgets because:
- **Encapsulation:** Shadow DOM prevents CSS leaks both ways
- **Portability:** Same .js file works in O11 and ODC
- **Versioning:** Single file with clear API contract
- **Easier to debug:** Source = production
- **Future-proof:** Web standard, not platform feature
- **Block wrapper still provides** the OutSystems-native dev experience

Always route L5 to the `outsystems-web-component` skill.
