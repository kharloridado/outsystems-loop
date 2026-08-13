# Customization Decision Tree (L1–L5 Escalation)

Walk top to bottom. Stop at the first level that satisfies the design.

```
┌─────────────────────────────────────────────────────┐
│  Q: Is the difference just a token value?           │
│     If YES → L1: Update :root token only            │
└─────────────────────────────────────────────────────┘
                       │ NO
                       ▼
┌─────────────────────────────────────────────────────┐
│  Q: Does an OS UI utility class do this?            │
│     If YES → L2: Attach utility via Style Classes   │
└─────────────────────────────────────────────────────┘
                       │ NO
                       ▼
┌─────────────────────────────────────────────────────┐
│  Q: Does OS UI already NAME this thing — the        │
│     widget class, or a variant of it?               │
│     If YES → L3a: restyle THAT class, BARE          │
│              (.btn, .btn-primary, .card, .tag)      │
└─────────────────────────────────────────────────────┘
                       │ NO — framework has no name for it
                       ▼
┌─────────────────────────────────────────────────────┐
│  Q: Same pattern, an EXTRA variant the framework    │
│     ships no class for?                             │
│     If YES → L3b: ExtendedClass + BEM modifier      │
└─────────────────────────────────────────────────────┘
                       │ NO
                       ▼
┌─────────────────────────────────────────────────────┐
│  Q: Custom API needed but OS UI pattern exists?     │
│     If YES → L4: Wrap pattern in custom Block       │
└─────────────────────────────────────────────────────┘
                       │ NO
                       ▼
┌─────────────────────────────────────────────────────┐
│  L5: Build vanilla JS Web Component + Block wrapper │
│  No OS UI pattern fits.                             │
└─────────────────────────────────────────────────────┘
```

## Per-level summary

### L1 — Token change
```css
:root { --color-primary: #7C3AED; }
```
**1 minute. Global effect. No CSS written.**

### L2 — Utility class
```
Style Classes: "text-neutral-8 pt-2"
```
**30 seconds. No CSS written.**

### L3a — Restyle the framework's own class (the default, and the common case)
```css
/* Every button in the app. No ExtendedClass anywhere, nothing to remember. */
.btn         { border-radius: var(--border-radius-soft); font-weight: var(--font-weight-bold); }
.btn-primary { background-color: var(--color-primary); color: var(--color-base-white); }
```
**Nothing for the developer to apply — they set the widget's own Style property.**

L3a is where most of a design system lands, and skipping it is the single most common mistake
in this whole tree. `.acme-button.btn` is **not** L3a: it is a gate on a class the platform
never emits, so the widget renders unbranded unless someone types it in per instance. Restyling
the bare class is what makes the brand the default instead of an opt-in.

Restyling a bare class disturbs its siblings — `.btn { height }` overrides `.btn-small`'s own
height at equal specificity, later in the cascade. Map every sibling modifier or exclude it on
purpose (`.btn:not(.btn-small)`).

### L3b — ExtendedClass + BEM modifier (only where the framework has NO class)
```
ExtendedClass = "acme-card acme-card--featured"

.acme-card--featured {
  border: var(--border-size-m) solid var(--color-primary);
}
```
**5–15 minutes per variant.** Verify the framework really lacks it first — grep the widget SCSS
**and** `src/scripts/OSFramework/OSUI/Pattern/*/Enum.ts`, where much of the class vocabulary
actually lives.

### L4 — Wrap pattern in custom Block
```
Block: ProductCard (Patterns Library)
  Inputs: ProductName, Price, ImageURL, OnAddToCart
  Internal: OS UI Card + Image + Button widgets
  Block CSS: .acme-product-card BEM rules
```
**30–60 minutes. Reusable everywhere.**

### L5 — Web Component + Block wrapper
```
1. acme-pricing-toggle.js  (vanilla JS Web Component)
2. PricingToggle Block (wraps the Web Component)
3. Test HTML page for browser QA
```
**1–4 hours. Best for components OutSystems UI doesn't have.**

### L6 — Fork OutSystems UI
**Almost never. You'd lose all upstream updates.**

## Healthy distribution
- L1 / L2: 60–70%
- L3a / L3b: 20–25% — and **most of that should be L3a.** A component audit that routes a
  widget OutSystems UI already ships to L3b is describing a parallel design system, not a
  theme. If L3b outnumbers L3a, re-read the framework's class vocabulary before writing CSS.
- L4: 5–10%
- L5: 1–5%
- L6: 0%

If your codebase has too many L4–L5, the design system is fighting OutSystems UI. Talk to design.

## Why L5 = Web Component (not raw Block)

Previously, L5 meant "build a Block from scratch with HTML widgets." The user has chosen a better approach: **vanilla JS Web Components wrapped in Blocks.**

Benefits:
- Shadow DOM encapsulation (no CSS leaks)
- Framework-agnostic (works in O11 and ODC)
- Easy to import (single .js file as Script resource)
- Other devs use the Block, not the Web Component directly
- Future-proof (web standard, not platform feature)

Always route L5 to the `outsystems-web-component` skill.
