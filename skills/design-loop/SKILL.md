---
description: Run the autonomous Figma -> OutSystems design loop until the goal in loop/goal.md is met. Supports single-screen and full-library mode.
---
Run the design loop, governed by this project's CLAUDE.md and the outsystems-* skills.

**Board-driven mode — decided by `board.drivesLoop`, NOT by the board pointer.** Read
`project.config.json` → `board` and check the two halves separately:

| | Meaning | Effect on this skill |
|---|---|---|
| `board.owner` + `.number` + `.url` (the **pointer**) | *where* the board is | none — a pointer alone never diverts the loop |
| `board.drivesLoop` (the **mode**) | whether the board is the work **queue** | `true` → use `board-advance` instead of this skill |

**Only `drivesLoop: true` diverts you.** In that case the queue is the board's `Ready` lane:
use `board-advance` (Ready → build → Ready for Review), `board-ship` (Approved → merged to
main → handover Task) and `board-sync` (reconcile) instead of this skill. The build itself is
identical — all of them follow `references/per-item-build.md`. See `references/status-map.md`
for the lanes.

**If `drivesLoop` is `false` or absent, this skill is the right one and you must not read the
board at all** — not to pick work, not to check a lane, not to "confirm" a status. The queue
is `loop/goal.md`'s signed inventory plus `loop/state.json` `items[].status`, and an item is
eligible when its status is `queued`. A populated pointer with `drivesLoop: false` is the
supported configuration for a board that is a **human view only**.

Why the split exists, stated plainly: Projects v2 is GraphQL-only and unreachable from a
Claude Code cloud routine — no `gh` on PATH, no `project_*` tool in the cloud GitHub MCP, no
GraphQL passthrough, and raw GraphQL to api.github.com refused by the egress proxy. Keying
board-mode off the *pointer* therefore made every project that merely owned a board
un-runnable in the cloud: the loop diverted to `board-advance` and died before reaching the
maker, silently, on every run. Pointer says where the board is; `drivesLoop` says whether the
loop reads it.

Read `project.config.json` (the project's values — `classPrefix`, `jsNamespace`, `conventions` with their `confirmed|assumed|TBD` status, `knownFalsePositiveClasses`), `loop/goal.md` (goal, Figma URL + file key, mode, scope, signed-off inventory, checkpoints, caps) and `loop/state.json` (queue + progress).

## Pre-flight (every run — cheap, and it has caught real drift)
1. **Reconcile state against git.** `state.json` can go stale while commits have already shipped later tiers. Compare the queue's `status` values against what is actually committed on the loop branch before advancing; fix state first, and say so in the REPORT.
2. **Confirm the inventory of record.** No item enters the queue without a row in it. Which artifact that is comes from `goal.md`'s **Inventory source**: `artifact` means the canonical, client-confirmed component inventory named there; `board` means the GitHub Project card itself, signed by a scope owner having moved it to `Ready` — in which case `board.drivesLoop` must be `true`, and you stop and use `board-advance`, which is the skill that reads the board. (`Inventory source: board` with `drivesLoop: false` is a misconfiguration: say so and stop, rather than guessing which one the project meant.) Either way, building speculatively is how fully-finished components get thrown away.
3. **Check the library file key.** If `goal.md`'s Figma file key differs from the key recorded in an item's frozen ref, that ref is stale → mark the item `needs-re-ref` and re-snapshot before building on it. Design libraries get forked ("Main Library (2)") and silently re-version values.

## Phase 0 — Tokens (library mode, runs once)
If `state.json.items` is empty and mode is "library":
1. Pull the full Figma library via Figma MCP. Extract the ENTIRE token set (color, spacing, type, radius, shadow, motion).
2. Reconcile against tokens/*.css: classify each as new / changed / off-scale / removed. File token-drift findings (design-token bugs) for anything off-scale or ambiguous.
3. Run `npm run build:theme` to produce dist/theme.css.
4. Seed `state.json.items[]` from the audit, each tagged with its `tier` (foundations|primitives|composites|patterns), `level` (L1-L5), `node` (Figma node id), status "todo".
5. Create/checkout the branch from goal.md. Then honor the `after_tokens` checkpoint: if "pause", write loop/REPORT.md (token summary + findings) and STOP for sign-off.

For single mode: seed the queue from a normal Phase 1 audit; skip tiers/checkpoints.

## Per-item loop (repeat until done-criteria or a cap/checkpoint)
1. Pick the next "todo" item in dependency order: never start a tier until the previous tier is fully "built"/"needs-human"; within a tier, build primitives a composite depends on first.
2. Build it by following **`references/per-item-build.md`** — freeze the ref, @maker, @checker, verdict routing, findings, persist. Pass the item as the `ITEM` contract with `branch` set to the loop branch from goal.md. That file is the single copy of the build procedure; `board-advance` follows the same one, so the build is identical whether the queue came from a Figma audit or from the board.
3. Route the outcome it returns:
   - `PASS` -> ensure the component's GitHub issue exists (dedup: search for `[node:<id>]`; create only if absent) as type Component/handover; add it to the GitHub Project (`gh project item-add <num> --owner <owner> --url <issue-url>`); attach to the tier's handover epic as a sub-issue (`gh issue edit <child> --parent <epic>`).
   - `FAIL-CAPPED` -> status "needs-human" with the critique attached; move on.
   - `BLOCKED-NO-REF` / `BLOCKED-STALE-REF` -> status "needs-human" / "needs-re-ref" with the reason. Do NOT count a round — the checker never got to judge.
   - `DET-GATE-FAIL` -> status "needs-human"; a build that will not go green is a stop, not a round.

## Tier boundaries
When a tier completes: run a CONSISTENCY PASS (do all components in the tier use tokens uniformly? naming consistent? no divergent one-offs? record result in consistency_passes[]). Then honor that tier's checkpoint (pause -> REPORT + STOP for sign-off; continue -> next tier).

## Stop
Stop when all items are "built"/"needs-human" (set status "done"), or iteration >= caps.max_global_iterations, or a checkpoint says pause, or a blocking finding needs a human.

## Headless one-step mode
When invoked to advance one step (from loop/run.sh), do exactly ONE item (or ONE phase-0/tier-boundary action), persist state, and exit.

## Report
Always end a run by writing loop/REPORT.md: tier progress, items built, Project URL, handover epics + sub-issues, bugs filed (links), needs-human items, consistency-pass results. Also write a `## Review metrics` section so review coverage is visible run-over-run:
- items auto-passed vs needs-human (review coverage)
- findings filed vs challenged-out (false-positive rate)
- checker rounds distribution (round-1-pass rate)
- risk-tier coverage (how many `core` items got full-stack depth)
- deterministic-gate pass rate
- **rendered-fidelity coverage** — items with `visual: pass` vs `drift` vs `unverified`, and the number of properties actually measured. This is the only metric that reflects whether the build LOOKS like the design; the others all measure process throughput, and a pipeline that ships drifted components scores 100% on every one of them.
Do NOT touch OutSystems — integration is manual via the handover sub-issues.

$ARGUMENTS
