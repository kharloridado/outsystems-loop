# Routines — the scheduled runs, created for the user

Exact prompts for the scheduled agents `project-setup` offers to create. Create them with the
session's scheduling capability; **ask first** — a scheduled agent runs unattended and spends the
user's budget, so it is theirs to authorise.

## What a routine can and cannot do here

**Can:** read the repo, read Figma (with the connector attached), run the build, run the
**rendered-fidelity gate** — it is a headless browser driven from Node, so it runs in a routine
exactly as it does at a keyboard — commit, push, open issues, open PRs. All REST, all fine.

**Cannot:** touch a GitHub **Project board**. Projects v2 is GraphQL-only: no `gh` on PATH, no
`project_*` tool, no GraphQL passthrough, and raw GraphQL to `api.github.com` refused by the
egress proxy. This was measured, not assumed — an hourly board routine fired nine times a weekday
and every run was a guaranteed no-op. Hence: the inventory in `loop/goal.md` is the queue, and a
board is only ever a human view.

**Must not:** merge, approve, move a card to Approved or Done, open a handover Task for unmerged
work, or push past a `pause` checkpoint. Those are the human's signature.

**Needs the Figma connector attached at creation time.** Without it no ref can be frozen, and an
item with no ref goes `needs-human` rather than being built from a guess — a whole night producing
nothing. Tell the user this while they are creating the routine, not after.

## Four ways creating a routine goes wrong, all of them silent

Measured on 2026-08-14, when two routines shared one cron and only one of them worked. See
`project-setup` §7a; the short version, because every one of these fails without an error:

1. **It fires into the session that created it.** That is the DEFAULT. Set
   `create_new_session_on_fire: true`. A nightly once landed in an old plan-mode PR-review
   conversation, which cannot write, commit or push — so the run was literally silent.
2. **It carries no connectors.** The `connectors` parameter is unavailable on some organizations
   and simply errors. If it does, the routine cannot reach Figma at all and must be created from
   the claude.ai routines UI instead.
3. **It carries no repo.** There is no repo-source parameter on the MCP path. Either create it from
   the UI, or have the prompt's first step clone the project.
4. **It carries no model pin,** so fired sessions can run on a weaker tier than the session that
   designed the loop.

**Every routine's first instruction is `node build/preflight.mjs`,** which repairs the harness and
names what it cannot repair. **And verify a new routine by FIRING it** — with an override saying
pre-flight only, build nothing — rather than waiting for its first cron to find out.

---

## 1. Loop advance — the main event

**Cadence:** nightly (a weekday-evening slot works well: the PRs are waiting in the morning).

```
Follow the /outsystems-loop:design-loop skill procedure for this repo.

The queue is the signed inventory in loop/goal.md; build every item whose status in
loop/state.json is `queued`, in inventory order, up to <N> items this run.

One branch per item, cut fresh from origin/main. One PR per item, opened with the full
body from the skill's pr-body.md reference: the plan, the spec of record, the gate
results including the MEASUREMENTS table, and BOTH decision logs verbatim.

Never merge, never approve, never open a handover Task — a handover must never point at
an unmerged branch. RESPECT every checkpoint in loop/goal.md: on reaching one marked
`pause`, STOP immediately, write loop/REPORT.md, and do not proceed past it.

File findings as bugs only after they survive the adversarial challenge. Persist
loop/state.json after every item. Do NOT read or write any GitHub Project board. Do NOT
touch OutSystems.

End by writing loop/REPORT.md with the PR URL for each item and the Review metrics
block, then give a five-line summary naming anything left needs-human and why.
```

Pick `<N>` with the user. Small (3–5) is usually right: it is the number of PRs they are willing
to review in one morning, not the number the machine can produce.

## 2. Token drift reconcile — recurring forever

**Cadence:** weekly. Optionally also a webhook on the Figma library publishing.

A design system's tokens move after handover, and drift is invisible until something looks wrong
in production. This is the highest-value recurring routine for a long engagement.

```
Re-pull the Figma library named in project.config.json via the Figma MCP. Extract the
full token set and reconcile it against tokens/*.css: classify every token as new,
changed, off-scale, or removed.

For anything off-scale or ambiguous, file a design-token bug (type Bug, labels
finding,bug,token), deduplicated by a [token:<name>] marker in the body — check for an
existing issue before opening one.

Run `npm run build:theme`. If tokens changed, open a PR "chore(tokens): reconcile design
tokens" on a fresh branch from origin/main and summarise the drift in the PR body as a
table: token, old value, new value, classification. Do NOT merge it, and do NOT rebuild
any component.

Never enforce a convention whose status in project.config.json is not `confirmed`.
End with a five-line summary.
```

## 3. Findings digest — optional, read-only

**Cadence:** daily, or weekly on a slower engagement.

```
List open issues labeled "finding" in this repo, grouped by severity. Write a short
digest: counts per severity, then the blocker and high titles with links, and how long
each has been open. Post it to the team channel if one is configured; otherwise append
it to loop/REPORT.md. Make NO changes to any issue and open no PRs.
```

---

## Reviewing a routine's output

The PRs are the output. `loop/REPORT.md` is the run's own account of itself, and its
`## Review metrics` block is where a degrading pipeline shows up first:

- **rendered-fidelity coverage** — `visual: pass` vs `drift` vs `unverified`, and how many
  properties were actually measured. Watch this one. Every other metric measures throughput, and a
  pipeline shipping drifted components scores perfectly on all of them.
- findings filed vs challenged out — the false-positive rate.
- round-1 pass rate, deterministic-gate pass rate, risk-tier coverage.

A run of `unverified` items is a **harness fault**, not a design problem: no browser, or a failed
stylesheet request because `vendor/outsystems-ui/` was never built. Fix the harness, re-run; never
relax the gate to make the number go green.
