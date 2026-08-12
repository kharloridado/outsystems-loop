---
description: Run the autonomous Figma -> OutSystems design loop until the goal in loop/goal.md is met. Supports single-screen and full-library mode.
---
Run the design loop, governed by this project's CLAUDE.md and the outsystems-* skills.

**The queue is the signed inventory in `loop/goal.md`, and item status in `loop/state.json`.** A
GitHub Project board, when the project points at one, is a **human view** — a place to see what is
in flight. It is never read as the queue. That is not a preference: Projects v2 is GraphQL-only and
unreachable from a scheduled run, so a board-queued loop fails while claiming a card, before it
ever reaches the maker, every time and silently. Files in the clone are readable from anywhere.
`board-ship` and `board-sync` remain available as local, human-invoked skills.

**Output: one branch and one PR per item.** Each item is cut fresh from `origin/main`, built,
checked, and landed as an **open PR** carrying the plan, the measurements, and both decision logs
(`references/pr-body.md`). The loop never merges and never approves — that is the human signature
(hard rule 10).

Read `project.config.json` (the project's values — `classPrefix`, `jsNamespace`, `conventions` with their `confirmed|assumed|TBD` status, `knownFalsePositiveClasses`), `loop/goal.md` (goal, Figma URL + file key, mode, scope, signed-off inventory, checkpoints, caps) and `loop/state.json` (queue + progress).

## Pre-flight (every run — cheap, and it has caught real drift)
1. **Reconcile state against git.** `state.json` can go stale while commits have already shipped later tiers. Compare the queue's `status` values against what is actually committed on the loop branch before advancing; fix state first, and say so in the REPORT.
2. **Confirm the inventory of record.** No item enters the queue without a row in the signed inventory table in `loop/goal.md`. A component that is in Figma but not in that table is `needs-human`, not `queued`; to add scope, a scope owner adds a row (and `git log loop/goal.md` records who and when). Building speculatively is how fully-finished components get thrown away.
3. **Check the library file key.** If `goal.md`'s Figma file key differs from the key recorded in an item's frozen ref, that ref is stale → mark the item `needs-re-ref` and re-snapshot before building on it. Design libraries get forked ("Main Library (2)") and silently re-version values.
4. **Open the handover Task for anything that merged since the last run.** A deliverable's handover is created *after* its PR merges, never before — a handover says "paste this into ODC", and that must never be said about an unmerged branch. Nothing watches the merge, so the loop collects them at the start of every run:

   ```bash
   gh pr list --state merged --label deliverable --limit 30 \
     --json number,title,headRefName,mergedAt,body
   ```

   For each merged deliverable PR with no handover Task yet (dedup by searching open+closed issues for the item id), run `node build/embed-handover-code.mjs` and open it:

   ```bash
   gh issue create --title "[handover] <component> — add in OutSystems" \
     --body-file handover/<artifact>.md --label "handover,task" --type "Task" \
     --assignee <developer> --repo <owner/repo>
   ```

   This is self-healing: a run that crashes before the bookkeeping leaves the Task to be picked up next time, rather than losing it. Say in the REPORT which handovers were opened this way.

## Phase 0 — Tokens (library mode, runs once)
If `state.json.items` is empty and mode is "library":
1. Pull the full Figma library via Figma MCP. Extract the ENTIRE token set (color, spacing, type, radius, shadow, motion).
2. Reconcile against tokens/*.css: classify each as new / changed / off-scale / removed. File token-drift findings (design-token bugs) for anything off-scale or ambiguous.
3. Run `npm run build:theme` to produce dist/theme.css.
4. Seed `state.json.items[]` from the audit, each tagged with its `tier` (foundations|primitives|composites|patterns), `level` (L1-L5), `node` (Figma node id), status "todo".
5. Set each seeded item's status to `queued` in inventory order. Then honor the `after_tokens` checkpoint: if "pause", write loop/REPORT.md (token summary + findings) and STOP for sign-off. Branches are cut per item, not here.

For single mode: seed the queue from a normal Phase 1 audit; skip tiers/checkpoints.

## Per-item loop (repeat until done-criteria or a cap/checkpoint)
1. Pick the next `queued` item in dependency order: never start a tier until the previous tier is fully "built"/"needs-human"; within a tier, build primitives a composite depends on first.
2. **Cut its branch fresh from `origin/main`** — `git fetch origin && git switch -c loop/item/<slug> origin/main`. One item, one branch: a shared loop branch makes every PR contain every earlier item's diff, so the reviewer approves all or none, and a single rejected item blocks the rest.
3. Build it by following **`references/per-item-build.md`** — freeze the ref, @maker, @checker, verdict routing, findings, commit, **open the PR**, persist. Pass the item as the `ITEM` contract with `branch` set to the branch you just cut and `base` to `main`.
4. Route the outcome it returns:
   - `PASS` -> the PR is open and the item is `ready_for_review`. **Stop there.** Do not merge, do not approve, and do not open the handover Task — a handover says "paste this into ODC" and must never say it about an unmerged branch. If the project points at a board, add the PR to it as a view-only courtesy when `gh project` is available locally; skip it silently otherwise.
   - `FAIL-CAPPED` -> status "needs-human" with the critique attached; move on.
   - `BLOCKED-NO-REF` / `BLOCKED-STALE-REF` -> status "needs-human" / "needs-re-ref" with the reason. Do NOT count a round — the checker never got to judge.
   - `DET-GATE-FAIL` -> status "needs-human"; a build that will not go green is a stop, not a round.

## Tier boundaries
When a tier completes: run a CONSISTENCY PASS (do all components in the tier use tokens uniformly? naming consistent? no divergent one-offs? record result in consistency_passes[]). Then honor that tier's checkpoint (pause -> REPORT + STOP for sign-off; continue -> next tier).

## Stop
Stop when all items are "built"/"needs-human" (set status "done"), or iteration >= caps.max_global_iterations, or a checkpoint says pause, or a blocking finding needs a human.

## Headless one-step mode
When asked to advance one step, do exactly ONE item (or ONE phase-0/tier-boundary action), persist state, and exit.

## Report
Always end a run by writing loop/REPORT.md: tier progress, items built, **the PR URL for each**, bugs filed (links), needs-human items, consistency-pass results. Also write a `## Review metrics` section so review coverage is visible run-over-run:
- items auto-passed vs needs-human (review coverage)
- findings filed vs challenged-out (false-positive rate)
- checker rounds distribution (round-1-pass rate)
- risk-tier coverage (how many `core` items got full-stack depth)
- deterministic-gate pass rate
- **rendered-fidelity coverage** — items with `visual: pass` vs `drift` vs `unverified`, and the number of properties actually measured. This is the only metric that reflects whether the build LOOKS like the design; the others all measure process throughput, and a pipeline that ships drifted components scores 100% on every one of them.
Do NOT touch OutSystems — integration is manual, and it starts from the handover Task the human
opens after merging the PR.

$ARGUMENTS
