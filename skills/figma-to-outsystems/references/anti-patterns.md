# OutSystems Frontend Anti-Patterns

Refuse, flag, or correct when encountered.

## CSS

### ❌ Hard-coded values
```css
.acme-card { color: #1A73E8; padding: 16px; }
/* USE: var(--color-primary), var(--space-m) */
```

### ❌ Class coupling
```css
.acme-card.is-open { background: blue; }
/* USE: .acme-card--is-open */
```

### ❌ Data attribute styling
```css
.acme-card[data-state="open"] { … }
/* USE: .acme-card--is-open */
```

### ❌ ID styling
```css
#b1-CardWrapper { padding: 24px; }
/* OS-generated IDs are unstable. USE: .acme-card */
```

### ❌ Unjustified `!important`
```css
.acme-card { color: red !important; }
/* Only for third-party widget overrides with comment */
```

### ❌ Inconsistent state names
```
.acme-modal--open
.acme-modal--opened
.acme-modal--isOpen
/* Pick one: --is-open */
```

### ❌ Generic class names you invented
```css
.button { }  .panel { }  .wrapper { }
/* Names nobody owns, colliding with everything. USE: .acme-button, .acme-panel */
```
**Not to be confused with restyling a framework class.** `.card` and `.btn` are *OutSystems
UI's* names, not generic ones — targeting them is correct and required (see below).

### ❌ Shadowing a class the framework already has
```css
.acme-button.btn  { }             /* opt-in gate — the platform never emits .acme-button */
.acme-button--big { }             /* .btn-large already exists */
:root { --acme-space-m: 16px; }   /* --space-m already exists — redefine THAT */
/* USE the framework's own name: .btn { }   .btn-large { }   --space-m: 16px; */
```
A prefixed class is right only where the framework has **no** name for the thing. Sharing a name
with OutSystems UI is how a theme reaches the widgets nobody hand-edits — it is the mechanism,
not a violation of "never edit OutSystems UI" (that rule is about editing its *files*).

Then handle the siblings: `.btn { height }` silently overrides `.btn-small { height }` at equal
specificity, later in the cascade. Map each sibling or exclude it (`.btn:not(.btn-small)`).

### ❌ Hooking an OS UI *internal* wrapper
```css
.osui-card__header-inner-wrapper { padding: 0; }
/* An undocumented structural div. Fragile, breaks on updates. */
```
The line between this and the entry above matters: a widget's **public** class or variant
(`.btn`, `.btn-primary`, `.card`, `.tag`, `.osui-tabs__header-item`) is the correct override
target; an internal wrapper the framework invented to hold a layout together is not. When
unsure, check `src/scripts/OSFramework/OSUI/Pattern/*/Enum.ts` — named there means public.

## Workflow

### ❌ Validating in Service Studio Preview
Preview injects `-servicestudio` attribute and pattern JS doesn't run. Always publish + real browser.

### ❌ Editing OutSystems UI module
Locked + breaks on upgrade. Override `:root` in your Theme.

### ❌ Skipping the Style Guide
No design contract, no QA baseline, no onboarding. Components get reinvented.

### ❌ Pixel-pushing in DevTools without copying back
DevTools Local Overrides are great for iteration — but paste final CSS back into Service Studio.

### ❌ Building before checking OS UI Cheat Sheet
Most "I need to build X" components already exist. Check first.

## Design System

### ❌ Token for every Figma value
Justify only if reused 2+ times. One-offs can stay hard-coded in one Block.

### ❌ Off-scale spacing values
18px when scale is 4/8/16/24/32 is an oversight. Snap to nearest token.

### ❌ Naming colors by value
```
--color-blue-500: #1A73E8;
/* USE: --color-primary (tomorrow it might be purple) */
```

### ❌ Variant explosion
`--featured-large-with-icon-no-image` = too many modifiers. Split into separate components.

## Web Components

### ❌ Forgetting `composed: true` on events
```javascript
new CustomEvent('change', { detail: value, bubbles: true })
/* USE: { ..., bubbles: true, composed: true } so it crosses Shadow DOM */
```

### ❌ No registration guard
```javascript
customElements.define('acme-toggle', Toggle);  // throws on second load
/* USE: if (!customElements.get('acme-toggle')) { ... } */
```

### ❌ Hard-coded styles in Web Component
```css
:host { color: #1A73E8; }
/* USE: :host { color: var(--color-primary, #1A73E8); } */
/* Fallback chain: OS Theme token → hard-coded fallback */
```

### ❌ Mode "closed" Shadow DOM
```javascript
this.attachShadow({ mode: 'closed' });  // ❌
/* USE: { mode: 'open' } so OutSystems can inspect and extend */
```

## ODC-Specific

### ❌ Theme CSS in an end-user App
In ODC, themes belong in Theme Libraries, not Apps.

### ❌ Breaking changes without Major version bump
Adding mandatory input parameter = Major version. Skipping breaks consumers silently.

### ❌ Skipping rebuild after library update
ODC apps consume frozen library versions until rebuilt.
