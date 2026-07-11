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
| `@outsystems-loop:checker` | Judges it. Deterministic build gate → risk-tiered depth → five domains → adversarial finding refutation → structured verdict. Never modifies files. |

**Skills**
| Skill | Role |
| --- | --- |
| `/outsystems-loop:design-loop` | The orchestrator: freeze ref → maker → checker → commit → handover → findings → report. |
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

## Local development

```bash
claude plugin validate . --strict
/plugin marketplace add ./          # test from a working copy before publishing
```

Bump `version` in **both** `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` to
ship an update — pushing commits without a version bump does not update installs.
