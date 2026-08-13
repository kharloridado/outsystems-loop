---
name: outsystems-token-extractor
description: Translate Figma design tokens into OutSystems UI CSS custom properties declared in :root. Use this skill whenever the user shares Figma token specs, design system values, brand colors, or asks how to set up :root CSS variables. Pull tokens directly from Figma via get_variable_defs MCP tool when a URL is provided.
---

# OutSystems Token Extractor (Figma-aware)

## Pre-flight
1. Check memory for `OutSystems convention:` entries. If missing → invoke `outsystems-onboarding`.
2. Use stored spacing base, token style, and prefix silently.

## When to use
- User shares Figma token specs / Variables / brand guidelines
- User asks "generate :root", "update brand colors", "set up theme"
- User says "translate these tokens for me"

## Figma MCP integration
If user provides a Figma URL:
1. Call `get_variable_defs(url)` → pull all Figma Variables
2. Map each to OutSystems UI naming convention
3. Generate :root block
4. List which tokens are new vs. modifications

## The names below collide with OutSystems UI on purpose

Every variable in the right-hand column is one **OutSystems UI already declares**. Redefining it
is the point: the theme loads after the framework, so its `:root` wins, and every OSUI widget in
the app — including the ones this project will never open — renders in the customer's brand.

So: **never namespace a token away from a framework name to dodge a collision**
(`--acme-space-m` beside `--space-m`), and **never file a collision as a finding.** Both have
been tried on a real project; both are backwards. A theme that shares no names with the
framework themes nothing.

What a collision *does* deserve is a recorded decision, because you are re-pointing a value the
framework's own widgets consume. `--space-m` is read by ~146 rules in compiled OutSystems UI —
if the design's `m` step is 16px where the framework's was 24px, every one of those rules moves.
That is a real question for a designer — *which step of our scale is `--space-m`?* — and it
belongs in the register with the consumer count, addressed as a mapping question. It is not a
defect in the theme, and the answer is never "add a prefix".

State in the handover which framework properties the theme re-points, and what moves as a result.

A **component-scoped** token (`--acme-button-height`) is a different thing and stays prefixed:
the framework has no name for it, so there is nothing to override.

## Token mapping (see references/full-token-list.md for complete inventory)

| Figma Category | OutSystems UI Variable |
|---|---|
| Color / Brand / Primary | `--color-primary` |
| Color / Brand / Primary Hover | `--color-primary-hover` |
| Color / Brand / Secondary | `--color-secondary` |
| Color / Neutral / 0-10 | `--color-neutral-0` … `--color-neutral-10` |
| Color / Semantic | `--color-success/error/warning/info` |
| Typography / Family / Body | `--font-family-body` |
| Typography / Family / Heading | `--font-family-heading` |
| Typography / Size / Display | `--font-size-display` |
| Typography / Size / H1-H6 | `--font-size-h1` … `--font-size-h6` |
| Typography / Size / Base | `--font-size-base` |
| Typography / Size / xs-2xl | `--font-size-xs` … `--font-size-2xl` |
| Typography / Weight | `--font-weight-regular/medium/semi-bold/bold` |
| Spacing / Base | `--space-base` |
| Spacing / 0-96px | `--space-none/xs/s/m/l/xl/2xl/3xl` |
| Radius | `--border-radius-none/soft/rounded/circle/pill` |
| Shadow | `--shadow-xs/s/m/l/xl` |
| Breakpoint | `--breakpoint-small/medium/large` |

## Output format

```css
/* ============================================
   Theme: [Customer / Project from memory]
   Generated: [date]
   Source: [Figma URL or "manual input"]
   Accessibility: WCAG 2.2 AA (color contrast verified)
   ============================================ */

:root {
  /* ----- Colors: Brand ----- */
  --color-primary: #1A73E8;        /* WCAG AA on white: 5.3:1 ✓ */
  --color-primary-hover: #1557B0;  /* WCAG AA on white: 8.4:1 ✓ */

  /* ----- Colors: Neutral ----- */
  --color-neutral-0:  #FFFFFF;
  /* ... etc ... */

  /* ----- Colors: Semantic ----- */
  --color-success: #16A34A;
  --color-error:   #DC2626;        /* WCAG AA on white: 5.9:1 ✓ */
  --color-warning: #F59E0B;
  --color-info:    #0EA5E9;

  /* ----- Typography ----- */
  --font-family-body: 'Inter', system-ui, sans-serif;

  /* ----- Spacing ----- */
  --space-base: 8px;
  /* ... */
}
```

After block, provide:
1. **Where to paste:** Theme module (O11) / Theme Library (ODC)
2. **Token delta summary**
3. **A11y verification:** All text/bg combinations meet WCAG 2.2 AA contrast (4.5:1 normal, 3:1 large/UI)
4. **Style Guide TODO:** Update swatches in Foundations/Colors page

## A11y verification (always do this)

For every color paired with a text or border context, calculate contrast and annotate:
- Pass AA: `/* 5.3:1 ✓ */`
- Pass AA Large only: `/* 3.2:1 ✓ (large text only) */`
- Fail: `/* ⚠️ 2.1:1 — FAILS WCAG AA, suggest darkening */`

If a token fails contrast for its expected use case, recommend a darker/lighter alternative.

## Reference
- `references/full-token-list.md` — Complete inventory
