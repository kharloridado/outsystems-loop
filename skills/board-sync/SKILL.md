---
name: board-sync
description: Reconcile the GitHub Project board against git and loop/state.json, reclaim cards stranded In Progress by a crashed run, and regenerate the deliverables.md snapshot. Use this when the board and the repo look out of step, when a run died mid-item, when a card has been stuck In Progress, or on a daily cadence. Read-mostly and needs no Figma. Runs at the keyboard or on a local schedule; the board is unreachable from a cloud routine.
---

# board-sync — reconcile, reclaim, snapshot

The janitor of the board loop. It owns the three jobs that belong to neither `board-advance`
nor `board-ship`, and it exists mostly so that **stale-lock reclaim lives somewhere safe**: if
`board-advance` reclaimed stale claims, two concurrent runs would reclaim each other's live
work.

Read first: `../design-loop/references/status-map.md`,
`../design-loop/references/board-api.md`, and `../design-loop/references/surfaces.md` —
the last one decides **where this may run**: local only, because Projects v2 is unreachable
from a cloud routine. Local does not mean by hand; see *Running on a schedule* below.

## Modes

| Invocation | Does |
|---|---|
| (default) | Reconcile + regenerate `deliverables.md`. Reports drift; **moves no cards.** |
| `--reclaim-stale` | Additionally rescues cards stranded in `In Progress`. The only mode that moves a card. |
| `--dry-run` | Reads and reports only. No `item-edit`, `issue comment`, `git push`, graphql mutation or `state.json` write. Print suppressed commands under `## Would execute`. |

## Rules

1. **Reconciliation rewrites `state.json`, never the board.** The board owns intent, git owns
   content, `state.json` is a cache of both (`status-map.md`).
2. **Never move a card to `Approved` or `Done`** — including "to correct" a disagreement.
3. **Only `--reclaim-stale` moves a card at all**, and only out of `In Progress`.
4. A card is stale only if its claim is older than `board.staleAfterMinutes` (default 120)
   **and** no live runner holds the lock. Never reclaim on age alone.

## 1. Reconcile

For every board item, compare three sources and correct `state.json` where it is wrong:

| Question | Authority |
|---|---|
| What lane is it in? | the board |
| Does the commit / branch / merge exist? | git |
| Everything else | `state.json`, which is only ever a cache |

Report the drift you corrected, and anything you could not explain. The failure this catches
is real: on the source project `state.json` went stale while commits had already shipped two
further tiers, and the loop nearly rebuilt work that already existed.

Flag, but do not act on:
- a card in `Ready for Review` with no `loop:built` comment;
- a card in `Handover` whose branch is not an ancestor of `main` — that is the board claiming
  work shipped when it did not, and it needs a human;
- a `Blocked` card whose blocking reason has since been satisfied (a dependency merged, a node
  added). Say so; let the human move it.

## 2. `--reclaim-stale`

For each card in `In Progress` whose `loop:claim` is older than `staleAfterMinutes` and whose
runner is not live, resolve by what is actually on the claimed branch:

| Branch state | Action |
|---|---|
| No commits beyond `main` | → `Ready`. Comment: reclaimed after a crashed run, nothing was built. Delete the branch. |
| Commits, but no `loop:built` comment | → `Blocked`. Comment with the branch and its last commit. **Leave the branch** for a human. |
| A `loop:built` comment exists | → `Ready for Review`. The run crashed after the checker passed but before the lane move — the work is real and complete. |

That third case is neither rare nor theoretical: the lane move is the last thing
`board-advance` does. Handle it, or good work gets silently rebuilt.

Post one comment per reclaim saying which case it was. A card that changes lane with no
explanation is worse than one that stayed stuck.

## 3. Regenerate `deliverables.md`

`WORKFLOW.md` has always claimed a root `deliverables.md` mirroring the board. It was never
written by hand and never will be — generate it.

One row per card: component, lane, tier, level, issue, PR, Figma node, date entered `Ready`,
date merged. Sort by tier then component. Give it a generated-file header (`docs/LESSONS.md`
§4.1 — generated files must look generated) naming this skill and the board URL, and commit it.

It earns its place three times over: it is the human-readable map `WORKFLOW.md` promises, a
diffable in-repo record of what was accepted into scope and when, and — for a project whose
`Inventory source` is `board` — the client-showable artifact that stands in for a signed
inventory table. Someone with authority over scope can counter-sign it weekly.

## Running on a schedule

This is the routine most worth having, and `surfaces.md` says where: **a local schedule, inside
an open Claude session** — the session's cron tooling, or `/loop`. It cannot go on a cloud
routine, because every board read there is a no-op.

Daily, mid-morning, is the slot that earns its keep. A cloud loop run opens PRs overnight that
the board cannot see; this is what closes that gap before the user looks at the board. Schedule
it with `--reclaim-stale` — reconcile-only fires are nearly free, and the reclaim is the half
that rescues a crashed night.

Three things follow from a schedule that only fires while a session happens to be open, and this
skill already satisfies all three — keep it that way:

1. **Reconcile from current state, never from a diff since the last run.** Missed fires must cost
   nothing. Compare board, git and `state.json` as they are now.
2. **Be idempotent.** A second fire over the same cards posts no duplicate comment and reclaims
   nothing twice.
3. **A quiet run is the expected outcome.** Report it in one line and exit; do not dress a no-op
   up as a failure.

Being unattended relaxes nothing. Rules 2 and 3 above hold on every fire: no card reaches
`Approved` or `Done`, and only `--reclaim-stale` moves a card at all.

Run it once with `--dry-run` in front of the user before scheduling it. Reclaim moves cards, and
they should watch it get one right first.

## 4. Report

Drift corrected, cards reclaimed (with which case each was), anything flagged for a human, and
the `deliverables.md` diff. If nothing needed doing, say that plainly — a quiet run is the
expected outcome and should read as one.

$ARGUMENTS
