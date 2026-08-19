# Surfaces — where a run happens, and what it can reach

**This is the only copy.** `design-loop`, `board-ship`, `board-sync` and `project-setup` all
read it. Do not restate the capability model in a SKILL.md — it was
restated in five places once, and two of them contradicted each other.

## One axis, not two

There is a single question — **where does the process run?** — and it has two answers:

| Surface | What it is | Lifetime | Board (`gh project`, GraphQL) |
|---|---|---|---|
| **local** | A Claude session on the user's machine: at the keyboard, or on a schedule inside that session | only while the session is open | **reachable** |
| **cloud** | A routine in Claude Code on the web: a trigger firing into a fresh remote container | persistent — fires whether or not the laptop is on | **unreachable** |

**"Scheduled" is not the axis.** Both surfaces schedule. A run being unattended says nothing
about what it can reach; only the surface does. This is the distinction the docs used to miss,
which is why `board-ship` was simultaneously described as "local, human-invoked only" and "the
one stage safe to run in a cloud routine". It is neither: it is **local, and schedulable there**.

## Why the board is local-only

Projects v2 is GraphQL-only. In the cloud container there is no `gh` on PATH, no `project_*`
tool, no GraphQL passthrough, and raw GraphQL to `api.github.com` is refused by the egress
proxy. Measured, not assumed: an hourly board routine fired nine times in a weekday and every
run was a guaranteed no-op.

Locally, `gh` is on PATH with the `project` scope (`board-api.md` §0) and all of it works.

The consequence is a routing rule, not a ban:

| Work | Needs the board | Surface |
|---|---|---|
| `design-loop` — build items, open PRs | no (the queue is `loop/goal.md`) | **either** |
| `revalidate` — re-judge a built artifact | no | **either** |
| Token-drift reconcile, findings digest | no | **either** |
| `board-sync` — reconcile, reclaim, snapshot | yes | **local** |
| `board-ship` — `Approved` → merge → handover | yes | **local** |

`design-loop` stays surface-agnostic because its queue is a **file**, and files are readable
from anywhere. That is the whole reason the board is never the queue.

## What a local schedule actually is

Be honest with the user about this when offering one, because it is not a cron daemon:

- It lives **inside a Claude session**. Close the session and the schedule is gone — nothing is
  written to disk.
- It fires only while that session is **idle**, so a fire during a long run lands after it.
- A recurring session job **auto-expires after 7 days**, firing one last time.

So a local routine runs "while my laptop is open" in the literal sense: while a session is up.
That is fine for board work — the board is a human view, and it does not need reconciling at
3am — but it means **every local routine must be catch-up safe**. Design for missed fires:

1. **Reconcile from current state, never from a diff since the last run.** Both board skills
   already do: `board-sync` compares board/git/state as they are now, `board-ship` selects on
   the `Approved` lane as it is now.
2. **Be idempotent.** A second fire over the same cards must open no second PR, no second
   handover issue, and no duplicate comment. Both skills key on existing artifacts
   (`gh pr list --head`, the `[node:<id>]` dedup marker) rather than on a cursor.
3. **A run with nothing to do reports one quiet line and exits 0.** A no-op is the expected
   outcome of most fires, and it must not read as a failure.

Nothing about being unattended relaxes a rule. `status-map.md`'s signature lanes hold on every
surface: **no agent moves a card into `Approved` or `Done`**, and no schedule makes that safe.

## Setting them up

Local: create the schedule with the session's cron tooling, or run the skill on an interval with
`/loop`. Cloud: create the routine trigger in Claude Code on the web, with the Figma connector
attached at creation time.

The exact prompts for every routine, on both surfaces, are in
`../../project-setup/references/routines.md`.
