---
name: outsystems-bem-css
description: Write BEM-compliant CSS for OutSystems Reactive Web and ODC components that consumes design tokens from :root, follows the user's stored prefixing conventions, and is upgrade-safe against OutSystems UI updates. Use this skill whenever the user needs to write CSS for an OutSystems component, customize an OutSystems UI pattern, add a CSS class to a Block or Theme, override styles via ExtendedClass, or asks for BEM in any OutSystems context.
---

# OutSystems BEM CSS Generator

Generate production-grade BEM CSS for OutSystems Reactive Web and ODC.

## Pre-flight

1. Check `memory_user_edits` for "OutSystems convention:" entries. If missing → invoke `outsystems-onboarding` first.
2. Use the stored prefix (e.g., `acme-`) throughout. Never ask "what prefix?" — it's in memory.

## When to use

- User shares a component spec or screenshot and asks for CSS
- User asks to customize an OutSystems UI pattern (Card, Button, Tabs, Accordion, etc.)
- User asks to write styles for a custom Block
- User mentions `ExtendedClass`, Theme CSS, Block CSS, or Screen CSS
- User shares Figma component details and wants the OutSystems implementation
- User asks "how do I override [pattern name]?"

## When NOT to use

- Component doesn't exist in OutSystems UI → use `outsystems-web-component` skill instead (L5)
- User wants just a token update → use `outsystems-token-extractor` skill

## Rule zero: restyle, don't shadow

**Before you write one BEM class, find out whether OutSystems UI already names this thing. If it
does, you restyle THAT name.** BEM is how you name what the framework has *no* name for — it is
not a parallel vocabulary to run beside the framework's.

Grep the framework's own SCSS (`vendor/outsystems-ui/src/scss/`) and the project's captured
rendered HTML (`outsystems-widgets-reference/`) for the widget **and every variant of it**. Then:

| What you found | Write this | Never this |
| --- | --- | --- |
| Framework has the widget class | `.btn { … }` — the **bare** class | `.acme-button.btn { … }` |
| Framework has the variant class | `.btn-primary`, `.btn-large`, `.btn-success` | `.acme-button--primary`, `--big` |
| Framework has the token | redefine `--color-primary`, `--space-m` in the theme | mint `--acme-color-primary` beside it |
| Framework has **no** equivalent | `.acme-<block>--<variant>` + `ExtendedClass` | — |
| Widget doesn't exist at all | → `outsystems-web-component` (L5) | — |

**Why the second column is not "safer".** `.acme-button.btn` reads like careful, scoped hygiene.
It isn't: the platform never emits `.acme-button`, so the rule only fires when a developer types
it into `ExtendedClass` on that one widget. Every button they forget renders framework-grey. A
theme built that way doesn't theme anything — it ships a per-widget opt-in and calls itself a
design system. Restyling the bare class is what makes the brand the default.

**Restyling a bare class disturbs its siblings.** Your rule lands at the same specificity as the
framework's and later in the cascade, so `.btn { height: 38px }` silently overrides
`.btn-small { height: 32px }`. Enumerate the framework's sibling modifiers; map each to a design
variant, or exclude it on purpose (`.btn:not(.btn-small)`). Never leave one to break by accident,
and never invent a size or colour the design doesn't specify just to fill a gap — say the gap
exists.

**Deliver the mapping, not just the CSS.** When you restyle framework classes, the developer's
instruction is "set the widget's **Style** property to Primary", not an `ExtendedClass` string.
Output the mapping table (framework class → design variant) alongside the code.

## Inputs needed

From context or quick inference:
1. **What component?** (name, screenshot if available)
2. **OutSystems UI pattern this extends?** (Card, Button, etc. — or "Custom Block")
3. **CSS location?** (Theme / Block / Screen — see references/css-locations.md)
4. **States needed?** (hover, focus, disabled, active, error, loading…)

If anything is unclear, make a reasonable assumption and state it. Don't pause for clarification on every detail.

## BEM rules

```
.{prefix}-{block}                            /* Block */
.{prefix}-{block}__{element}                 /* Element */
.{prefix}-{block}--{modifier}                /* Modifier */
.{prefix}-{block}__{element}--{state}        /* Element state */
```

### Forbidden (refuse to generate)

```css
.acme-card.is-open { }                      /* coupling */
.acme-card[data-state="open"] { }           /* data attribute styling */
.acme-card { color: #1A73E8; padding: 16px; }  /* hard-coded values */
#b1-CardWrapper { }                         /* OutSystems-generated IDs */
.acme-card { color: red !important; }       /* unjustified !important */
.acme-button.btn { }                        /* opt-in gate on a framework class — style .btn */
.acme-button--big { }                       /* duplicates .btn-large — style .btn-large */
```

### Required patterns

```css
.acme-card--is-open { }                     /* state as modifier */
.acme-card { color: var(--color-primary); padding: var(--space-m); }  /* tokens */
.acme-card:hover { }                        /* pseudo-classes OK */
```

## Output format

Two shapes, and rule zero decides which:

- **Restyling a framework widget** (the common case) — selectors are the framework's own bare
  class names. Head the file with the mapping table and deliver "set the widget's Style property
  to X", not an `ExtendedClass` string. See `references/common-patterns.md`.
- **Building something the framework has no name for** — full BEM under the project prefix,
  applied via `ExtendedClass`. That is the shape below.

```css
/* ============================================
   Component: [Name]
   Pattern: [OS UI Pattern this extends, or "Custom Block"]
   Location: [Theme CSS / Block CSS / Screen CSS]
   Escalation Level: L1 / L2 / L3 / L4
   Tokens consumed: [list]
   ============================================ */

/* Block */
.acme-card { /* ... */ }

/* Elements */
.acme-card__header { }
.acme-card__body { }
.acme-card__footer { }

/* Modifiers (variants) */
.acme-card--featured { }
.acme-card--compact { }

/* States */
.acme-card--is-loading { }
.acme-card__header--is-active { }

/* Pseudo-state interactions */
.acme-card:hover { }
.acme-card:focus-visible { }
.acme-card[aria-disabled="true"] { }

/* Responsive (mobile-first) */
@media (min-width: 768px) {
  .acme-card { /* tablet+ */ }
}
```

Then provide:
1. **How the developer applies it.** For a framework restyle, the widget property that emits the
   class (`Style = Primary` → `.btn-primary`) — plus the mapping table. Only for classes the
   framework has no name for, the **ExtendedClass string**:
   ```
   ExtendedClass = "acme-card acme-card--featured"
   ```
2. **Where to paste this CSS** (Theme / Block / Screen)
3. **Any Block input parameters** that should be added (if building a custom Block)
4. **Which framework siblings you touched**, and which you deliberately left alone.

## References

- `references/css-locations.md` — Where each type of CSS belongs
- `references/state-vocabulary.md` — Standard state modifier names
- `references/common-patterns.md` — BEM examples for common OS UI patterns
- `references/responsive-patterns.md` — Mobile-first patterns
