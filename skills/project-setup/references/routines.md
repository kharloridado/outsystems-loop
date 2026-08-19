# Routines — the scheduled runs, created for the user

Exact prompts for the scheduled runs `project-setup` offers to create. **Ask first** — a
scheduled run is persistent configuration that runs unattended and spends the user's budget, so
it is theirs to authorise.

Read `../../design-loop/references/surfaces.md` first. It decides **which surface each routine
goes on**, and that is the only thing about scheduling that is not obvious:

- **cloud** — a routine in Claude Code on the web. Fires whether or not the laptop is on.
  Cannot touch a GitHub Project board.
- **local** — a schedule inside a Claude session on the laptop, created with the session's cron
  tooling or run on an interval with `/loop`. Reaches the board. Lives only while that session
  is open, fires only while it is idle, and a recurring session job expires after 7 days.

| # | Routine | Surface | Cadence |
|---|---|---|---|
| 1 | Loop advance | either — cloud by default | nightly |
| 2 | Board sync | **local** | daily, while working |
| 3 | Board ship | **local** | on demand, or hourly during a review day |
| 4 | Token-drift reconcile | either — cloud by default | weekly |
| 5 | Findings digest | either | daily or weekly |

Say plainly which surface each one is on and why, at the moment they authorise it. "This one
only runs while you have a Claude session open" is the difference between a routine they trust
and one they think is broken.

## What any routine can and cannot do

**Can:** read the repo, read Figma (with the connector attached), run the build, run the
**rendered-fidelity gate** — a headless browser driven from Node, so it runs in a routine exactly
as it does at a keyboard — commit, push, open issues, open PRs. All REST, all fine.

**Cannot, on cloud:** touch a GitHub Project board. See `surfaces.md` for the measurement.

**Must not, on any surface:** merge outside `board-ship`, approve, move a card to `Approved` or
`Done`, open a handover Task for unmerged work, or push past a `pause` checkpoint. Those are the
human's signature, and being unattended does not loosen one of them.

**Needs the Figma connector attached at creation time** (routines 1 and 4). Without it no ref can
be frozen, and an item with no ref goes `needs-human` rather than being built from a guess — a
whole night producing nothing. Tell the user this while they are creating the routine, not after.

---

## 1. Loop advance — the main event

**Surface:** either; cloud by default, because its queue is a file and it should run whether or
not the laptop is on. **Cadence:** nightly (a weekday-evening slot works well: the PRs are
waiting in the morning).

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

## 2. Board sync — keep the human view honest

**Surface: local only.** **Cadence:** daily, on a working day — mid-morning is a good slot,
since the session is open and the previous night's PRs have landed on the board.

The board drifts against git the moment a cloud loop run opens PRs it cannot see. This is the
routine that closes that gap, and it is the reason a board is still worth having alongside a
cloud loop.

```
Follow the /outsystems-loop:board-sync skill procedure for this repo, with --reclaim-stale.

Reconcile the board against git and loop/state.json, correct loop/state.json where it
is wrong — never the board — reclaim any card stranded In Progress past
board.staleAfterMinutes, and regenerate deliverables.md.

Never move a card to Approved or Done. Reclaim only out of In Progress, and only where
no live runner holds the lock. Post one comment per reclaim saying which case it was.

If nothing needed doing, say exactly that in one line and stop — a quiet run is the
expected outcome. Otherwise report the drift corrected, the cards reclaimed, anything
flagged for a human, and the deliverables.md diff.
```

Run it once with `--dry-run` in front of the user before scheduling it. Reclaim moves cards, and
they should watch it get one right before it does so unattended.

## 3. Board ship — merge what the human already signed

**Surface: local only.** **Cadence:** on demand is the honest default. Schedule it hourly only
on a day the user is actively reviewing, and let it lapse with the session.

Moving a card into `Approved` **is** the approval, so this routine is not deciding anything — it
is executing a signature already given. That is what makes it safe to schedule at all.

```
Follow the /outsystems-loop:board-ship skill procedure for this repo.

For each card in the Approved lane, oldest first: find its build, open or reuse a PR,
squash-merge it into board.shipBase, RE-READ the PR state, and only on MERGED create
the handover Task and move the card to Handover.

Never use --admin. Never ship a card with no loop:built record — that goes to Blocked.
Never move a card to Approved or Done. If a merge does not complete, leave the card in
Approved, comment with the PR link and the mergeStateStatus, and move on.

Report cards shipped with PR and handover links, cards left in Approved and why, and
cards blocked and why — the incomplete merges first. If the Approved lane was empty,
say so in one line.
```

The card must already carry a `loop:built` record, so this can only ever ship work the loop
built and a human then approved. Both halves are required, and neither is this routine's to
supply.

## 4. Token-drift reconcile — recurring forever

**Surface:** either; cloud by default. **Cadence:** weekly. Optionally also a webhook on the
Figma library publishing.

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

## 5. Findings digest — optional, read-only

**Surface:** either. **Cadence:** daily, or weekly on a slower engagement.

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

For the two local routines, the failure mode to watch for is the opposite one: **silence**. A
board routine that stops reporting has almost certainly lost its session rather than found
nothing to do. Check with the session's cron listing before concluding the board is clean.
