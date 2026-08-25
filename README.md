# OutSystems Design Loop — Claude Code plugin

The **behaviour** half of the Figma → OutSystems design-system workflow: 16 skills plus the
`maker` / `checker` agent pair that drive the autonomous build loop.

It holds **no OutSystems platform knowledge**. Block names, arguments, CSS variables, utility
classes, widget conventions and the WCAG-to-widget mapping all live in OutSystems' own
frontend-skills pack, vendored as a pinned submodule and read at runtime. This repo holds the
process, the policy and the judgement. See [ARCHITECTURE.md](ARCHITECTURE.md) — that separation is
load-bearing, not tidiness.

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
| `/outsystems-loop:project-setup` | **Start here on a fresh clone.** One pass: interview → config → deps + submodule → labels → deliverables into the signed inventory and the queue → routines → verify it all builds. |
| `/outsystems-loop:design-loop` | The orchestrator: freeze ref → maker → checker → commit → **one PR per deliverable** → findings → report. Queue is the signed inventory. |
| `/outsystems-loop:revalidate` | Re-run the checker against an **already-built** artifact, item or PR — no maker, no rebuild. For review questions, hand-edits, or a verdict you distrust. |
| `/outsystems-loop:board-ship` | Board *view*, local surface: `Approved` → PR → squash-merge to `main` → handover Task → `Handover`. |
| `/outsystems-loop:board-sync` | Board *view*, local surface: reconcile board/git/state, reclaim stale claims, regenerate `deliverables.md`. |
| `figma-to-outsystems` | Master workflow orchestrator. |
| `outsystems-component-audit` | Triage a design: exists as-is / customize / build custom (L1–L5). |
| `outsystems-token-extractor` | Figma variables → `:root` custom properties. |
| `outsystems-figma-integration` | Read Figma directly via MCP. |
| `outsystems-bem-css` | BEM CSS that overrides native widgets, token-only. |
| `outsystems-web-component` | Vanilla-JS Web Components + Block wrapper (L5 only). |
| `outsystems-mentor-build` | Driving a build into a live ODC module through Mentor/MCP, and verifying it landed. |
| `outsystems-design-findings` | The flag-don't-fix pipeline: classify, refute, route. |
| `outsystems-style-guide-doc` | Live Style Guide pages. |
| `outsystems-git-helpers` | Conventional commits, branches, PRs, changelog. |
| `outsystems-onboarding` / `outsystems-project-context` | One-time convention + project capture. |

## Two contracts, same shape

The plugin sits between two things it does not own, and reads both at runtime.

**Upstream — platform knowledge.** `vendor/outsystems-frontend-skills/`, a pinned submodule of
OutSystems' own pack. Never edited, never copied from. A skill that needs to know whether a block
exists reads `blocks-index.md`; it does not carry a list of its own. When the submodule is missing,
skills say so and stop rather than answering from memory — a guessed catalog inflates L4/L5 and
produces a costed plan for work that did not need doing.

Bump it with a submodule checkout, then re-check any mapping table that names upstream identifiers.
Full rules in [ARCHITECTURE.md](ARCHITECTURE.md).

**Downstream — project values.** The agents hold **no project values**. They read them at runtime from `project.config.json` in the
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

## The output: one PR per deliverable

The loop's unit of work is one item, and its unit of *review* is one PR. Each item is cut fresh
from `origin/main`, built, checked, and landed as an **open PR** whose body carries the plan, the
spec of record, the gate results including the MEASUREMENTS table, both decision logs verbatim, and
the findings — filed and challenged-out. The template is
`skills/design-loop/references/pr-body.md`.

This is where the loop's reasoning goes. It used to live in `state.json` and a run report, which
meant a reviewer opened a diff and re-derived intent from the code — the expensive half of review,
repeated per component. The build already produces all of it; the PR is simply where a human is
actually looking.

The loop **never merges and never approves**. It stops at an open PR, and the handover Task is
opened on the *next run* after that PR has merged — self-healing, so a crashed run loses nothing.

## Two surfaces, and the queue is a file

Everything in this plugin runs on one of two surfaces, and there is exactly one question to ask:
**can it reach the board?**

| Surface | What it is | Board |
|---|---|---|
| **local** | A Claude session on the laptop — at the keyboard, or on a schedule inside that session | **yes** |
| **cloud** | A routine in Claude Code on the web, firing into a fresh container | **no** |

**Scheduled is not the axis.** Both surfaces schedule; only the surface decides reach. A local
schedule lives inside an open Claude session, so it runs "while the laptop is open" in the literal
sense, and every local routine is written to be catch-up safe about missed fires. The single
statement of all this is `skills/design-loop/references/surfaces.md`; the prompts for each routine,
on both surfaces, are in `skills/project-setup/references/routines.md`.

The board is cloud-unreachable because Projects v2 is GraphQL-only: a cloud run has no `gh` on
PATH, no `project_*` tool, no GraphQL passthrough, and raw GraphQL to `api.github.com` is refused
by the egress proxy. Measured on a real project — an hourly board routine fired nine times in a
weekday and every single run was a guaranteed no-op.

**That is why the queue is a file.** `loop/goal.md`'s signed inventory is the queue and
`loop/state.json` holds item status, so `design-loop` reads the same queue on both surfaces and a
nightly build needs nobody's laptop. A board-queued loop would instead fail while claiming a card,
before the maker, every time and silently. A GitHub Project board, where a project points at one,
is a **human view**.

`board-sync` and `board-ship` work against that view, on the local surface — which means at the
keyboard *or* on a local schedule, and `board-sync` daily is what keeps the board honest against
PRs a cloud loop opened overnight. They expect:

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

**It runs headless, from Node** — `build/gate/measure-fidelity.mjs`, driving Playwright. That is what
makes an unattended run able to return a real verdict. The gate used to go through an editor-only
Chrome extension, so every scheduled run reported `unverified`, `unverified` caps at FAIL, and the
nightly loop could never hand over finished work. The identical command now runs at a keyboard, in
a routine, and in CI.

The script measures; it does not judge. It never reads the ref, so it cannot call drift — it
reports numbers and an exit code (`0` measured · `3` something unmeasurable · `4` no browser or no
page), and the checker compares them to the ref. Keeping mechanism and judgment apart is what stops
the gate from grading itself.

### Judged once, verified forever

The checker's verdict is written into the PR at build time and never re-computed — so a fixup
commit, or a token changed on `main` after the branch was cut, would leave the PR carrying a
measurement that no longer describes what would merge.

`build/gate/compare-measurements.mjs` closes that. The `measurements.json` the checker commits with
each item becomes a **baseline**; re-running the same probes later and diffing against it answers a
strictly narrower question that needs no judgment at all:

| | The question | Who answers | Cadence |
|---|---|---|---|
| Build time | *Is this right?* — the render vs the frozen Figma ref | the checker (an agent) | once per item |
| Every push | *Did this change?* — vs the committed baseline | `compare-measurements.mjs` | free, forever |
| On demand | *Is this still right?* — re-judge, no rebuild | `revalidate` | when asked |

That makes a CI gate possible with **no API key and no tokens spent**, and it turns every committed
`probes.json` into a permanent visual regression test — the suite grows with each deliverable, so a
token change that breaks component #3 is caught while building component #9.

> ### Why both scripts live in the TEMPLATE, not in this plugin
>
> They were here first, and CI proved that wrong. This plugin repo is **private**; a consuming
> project may be **public**. A workflow in the project cannot clone a private plugin without a PAT,
> and putting a token for a private repo into a public repo's CI is the wrong trade — so the gate
> failed on its very first run with `could not read Username for 'https://github.com'`.
>
> The deeper point is that these two files are not *behaviour*. They are deterministic Node tooling
> that measures and diffs, exactly like `build-theme.mjs` and `validate-theme.mjs`, and they sit in
> `build/gate/` beside them. The plugin holds the **instructions** for using them — when to probe,
> what a drift means, what may never be relaxed — which is the part that actually goes stale when
> copied. One copy of the script, in the repo that runs it; one copy of the judgment, here.
>
> Consequence: `build/gate/` ships with the template, so an existing project picks up an
> improvement by pulling the scaffold, not by `/plugin update`. That is the same trade every other
> `build/` script already makes.

The comparison is **typed by stability**, and that detail is load-bearing: computed strings
(colour, `font-family`, `font-size`, radius, weight) are environment-stable and diff exactly, while
`rect`/`ink` geometry depends on font rasterisation and is compared with a tolerance and reported as
*informational*. A naive deep-equal would fail every pull request on sub-pixel noise, and a gate
that cries wolf every time teaches people to click through it — at which point it misses the real
regression too. The highest-value catch, a webfont silently falling back to a system stack, is a
`font-family` string diff, and that is perfectly stable.

Three consequences for the consuming project:

- It must expose `npm run preview` and `<link>` every block stylesheet in the harness, in layer
  order. CSS that isn't loaded means the preview proved nothing.
- It needs a browser: `playwright-core` plus an installed Chrome or Edge (no download), or the full
  `playwright` package with bundled Chromium.
- **A failed stylesheet or font request invalidates the whole viewport.** A missing CSS file does
  not throw — the cascade falls back and every computed value still reads as a plausible number
  describing a page nobody will ever see. The commonest cause is `vendor/outsystems-ui/` never
  being built, and it is why `git submodule update --init` is not optional.

## Local development

```bash
claude plugin validate . --strict
/plugin marketplace add ./          # test from a working copy before publishing
```

Bump `version` in **both** `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` to
ship an update — pushing commits without a version bump does not update installs.
