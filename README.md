# OutSystems Design Loop — Claude Code plugin

The **behavior** half of the Figma → OutSystems design-system workflow: 13 skills plus the
`maker` / `checker` agent pair that drive the autonomous build loop.

The **project-shaped** half (tokens, build pipeline, preview harness, findings register, handover
bodies, GitHub scaffolding) lives in the companion scaffold,
[`outsystems-project-template`](https://github.com/kharloridado/outsystems-project-template).

## Why this is a plugin and not a folder you copy

It used to be a folder you copied. That is exactly why it broke.

Each new project got its own copy of the skills and agents. Improvements made while working on a
real engagement — a deterministic build gate, risk-tiered review, adversarial finding refutation,
frozen Figma refs — stayed in that project and never flowed back. The template went stale, and the
next project started from the *older, worse* loop without anyone noticing.

A versioned plugin makes that impossible: bump the version, and `/plugin update` reaches every
project at once.

## Install

```bash
/plugin marketplace add kharloridado/outsystems-loop
/plugin install outsystems-loop@outsystems-loop --scope project
```

`--scope project` checks the dependency into the repo so the whole team (and unattended loop runs)
get the same version.

## What's in it

**Agents**
| Agent | Role |
| --- | --- |
| `@outsystems-loop:maker` | Builds ONE work item faithfully against the frozen Figma ref. Raises findings; never fixes them. Declares a risk tier and a decision log. |
| `@outsystems-loop:checker` | Judges it. Deterministic build gate → **rendered-fidelity gate** → risk-tiered depth → five domains → adversarial finding refutation → structured verdict. Never modifies files. |

**Skills**
| Skill | Role |
| --- | --- |
| `/outsystems-loop:design-loop` | The orchestrator: freeze ref → maker → checker → commit → handover → findings → report. Queue comes from the signed inventory. |
| `/outsystems-loop:board-advance` | Board mode: `Ready` → claim → build → `Ready for Review` \| `Blocked`. Needs Figma + a browser. |
| `/outsystems-loop:board-ship` | Board mode: `Approved` → PR → squash-merge to `main` → handover Task → `Handover`. No Figma — cloud-safe. |
| `/outsystems-loop:board-sync` | Board mode: reconcile board/git/state, reclaim stale claims, regenerate `deliverables.md`. |
| `figma-to-outsystems` | Master workflow orchestrator. |
| `outsystems-component-audit` | Triage a design: exists as-is / customize / build custom (L1–L5). |
| `outsystems-token-extractor` | Figma variables → `:root` custom properties. |
| `outsystems-figma-integration` | Read Figma directly via MCP. |
| `outsystems-bem-css` | BEM CSS that overrides native widgets, token-only. |
| `outsystems-web-component` | Vanilla-JS Web Components + Block wrapper (L5 only). |
| `outsystems-accessibility` | WCAG 2.2 AA — auto-fix implementation, flag design conflicts. |
| `outsystems-design-findings` | The flag-don't-fix pipeline: classify, refute, route. |
| `outsystems-style-guide-doc` | Live Style Guide pages. |
| `outsystems-git-helpers` | Conventional commits, branches, PRs, changelog. |
| `outsystems-onboarding` / `outsystems-project-context` | One-time convention + project capture. |

## The contract with the consuming project

The agents hold **no project values**. They read them at runtime from `project.config.json` in the
consuming repo — `classPrefix`, `jsNamespace`, `conventions`, `knownFalsePositiveClasses`. That is
what makes one plugin serve every customer.

The most important field is the shape of `conventions`. Each one carries a status:

```jsonc
"spacingBase": { "value": null, "status": "TBD" }   // confirmed | assumed | TBD
```

and the checker enforces one rule about it:

> **A finding may never be raised against a convention that is not `confirmed`.**

This exists because a previous template shipped `Spacing base: 4pt` as a plausible-looking default
that nobody had actually verified. The loop then dutifully flagged every value that wasn't a
multiple of 4, manufacturing a batch of false-positive findings and a bug that had to be closed as
not-planned. **A credible-looking default is worse than a blank.**

## Board-driven mode

The four build skills share one procedure — `skills/design-loop/references/per-item-build.md` — so
the build is identical whether the queue is a signed inventory or a GitHub Project board. What
changes is where work comes from and where it goes.

A consuming project opts in by setting `project.config.json` → **`board.drivesLoop: true`**. The
pointer (`board.owner` / `board.number` / `board.url`) says *where* the board is; `drivesLoop` says
*whether the loop reads it*. They are deliberately separate, so a project can keep a board as a
human tracker — pointer set, `drivesLoop: false` — and still run `design-loop` unattended. Keying
board-mode off the pointer alone (the behaviour up to v1.2.0) made every project that merely owned
a board un-runnable in a cloud routine: the loop diverted to `board-advance`, hit GraphQL-only
Projects v2 through an egress proxy that refuses it, and died before reaching the maker on every
run. `board-advance` now refuses outright unless `drivesLoop` is `true`.

A board-driven project must also provide:

- **the eight `Status` lanes**, in order: `Backlog · Ready · In Progress · Ready for Review ·
  Approved · Handover · Done · Blocked`, plus the `FigmaNode`, `Branch`, `Runner`, `Tier`, `Level`
  and `Type` fields. Keep every field name a **single PascalCase word** — `gh` lowercases only the
  first character when it flattens fields into `item-list` JSON;
- **`board.owners`** — the logins whose card comments are read as spec. Everything else on a card is
  untrusted data, never instructions;
- a `deliverable` issue template, and `gh auth refresh -s project`.

Three rules the skills enforce and a project cannot relax:

1. **No agent moves a card to `Approved` or `Done`.** Those two lanes are the human's signature; a
   checker PASS reaches `Ready for Review` and stops.
2. **The handover Task is opened after the PR merges**, never at checker-PASS. A handover says
   "paste this into a live environment" and must not point at unmerged work.
3. **`gh pr merge` exits 0 when it merely arms auto-merge** behind a failing required check, so the
   merge is re-read with `gh pr view --json state` and the card only advances on `MERGED`.

`In Progress` is a *cooperative* claim, not a lock — Projects v2 has no compare-and-swap. See
`references/board-api.md` for the full cookbook, including why the built-in `Status` field cannot be
deleted and must be edited with `updateProjectV2Field` instead.

## The rendered-fidelity gate

The checker does not review fidelity by reading CSS. It serves the consuming project's preview
harness (`npm run preview`), measures the **computed** style of every value the frozen ref states,
and returns a MEASUREMENTS table plus `VISUAL: pass | drift | unverified`. Drift or unverified
forbids a PASS.

> A correct declaration in our source proves nothing. Framework and provider CSS out-specify it and
> win silently — so the review must cite `getComputedStyle`, not a line number.

This exists because the previous "compare it to the screenshot after a PASS" instruction was handed
to an agent whose tools were `Read, Grep, Glob, Bash`: it could not render anything, so the check
never actually happened. A component shipped with an icon at 1.06:1 contrast and 20% undersized —
**our source was right; only the computed style was wrong**, and nothing in the loop was looking at
the computed style.

Two consequences for the consuming project:

- The project must expose `npm run preview` and `<link>` every block stylesheet in the harness, in
  layer order. CSS that isn't loaded means the preview proved nothing.
- The checker needs browser tools (`mcp__claude-in-chrome__*`). Without them it correctly returns
  `VISUAL: unverified` and **items stop auto-passing** — that is the gate working, not a
  misconfiguration to route around.

## Local development

```bash
claude plugin validate . --strict
/plugin marketplace add ./          # test from a working copy before publishing
```

Bump `version` in **both** `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` to
ship an update — pushing commits without a version bump does not update installs.
