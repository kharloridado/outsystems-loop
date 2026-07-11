---
description: Run the autonomous Figma -> OutSystems design loop until the goal in loop/goal.md is met. Supports single-screen and full-library mode.
---
Run the design loop, governed by this project's CLAUDE.md and the outsystems-* skills.

Read `project.config.json` (the project's values — `classPrefix`, `jsNamespace`, `conventions` with their `confirmed|assumed|TBD` status, `knownFalsePositiveClasses`), `loop/goal.md` (goal, Figma URL + file key, mode, scope, signed-off inventory, checkpoints, caps) and `loop/state.json` (queue + progress).

## Pre-flight (every run — cheap, and it has caught real drift)
1. **Reconcile state against git.** `state.json` can go stale while commits have already shipped later tiers. Compare the queue's `status` values against what is actually committed on the loop branch before advancing; fix state first, and say so in the REPORT.
2. **Confirm the signed-off inventory.** No item enters the queue without a row in the canonical, client-confirmed component inventory named in `goal.md`. Building speculatively is how fully-finished components get thrown away.
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
1b. **Freeze the Figma ref — YOU must do this, before delegating.** The subagents have **no Figma MCP access**, so whatever you snapshot here IS the spec of record. Write `loop/refs/<item-id>/`:
   - `spec.md` — provenance (**Figma file key**, node id, pull date) + the key-values table.
   - `variables.json` — verbatim `get_variable_defs` for the node.
   - `figma.png` — `get_screenshot` of the node. **`curl` the URL immediately; it expires.**
   Capture **every size variant and device frame**, not just the default instance — a variable whose value changes per size/device is mode-bound and must become per-size tokens. A whole documentation page is usually too large for one `get_design_context` pull (it returns sparse metadata): snapshot the page screenshot + variables, then deep-pull the component-set sublayer (the `State=…` frame) for the value-bearing code, and record that sublayer node id in `spec.md`.
   **No ref ⇒ the item goes `needs-human`. Never build an item without one** — without a ref the checker is grading the maker against the maker's own output.
2. Delegate to @outsystems-loop:maker to implement it. The maker returns a self-declared RISK-TIER and a DECISION-LOG.
3. Delegate to @outsystems-loop:checker to validate. The checker runs a deterministic gate FIRST (`npm run build:theme` exit 0 + schema/contrast), scales depth to the item's risk tier, and adversarially challenges every finding before confirming it. It returns VERDICT, RISK-TIER, DET-GATE, CONFIDENCE, CRITIQUE, FINDINGS-CONFIRMED, FINDINGS-CHALLENGED-OUT, DECISION-LOG.
   - DET-GATE: fail -> this is a build break, not a design miss. Feed the breakage to @outsystems-loop:maker to fix; log it as `det_gate: fail` but do NOT count it against `max_rounds_per_item` the same way a fidelity FAIL does (a broken build is mechanical, fix and re-run).
   - PASS -> record `risk_tier`, `det_gate: pass`, `confidence`, and `decision_log` (maker + checker) on the item in state.json; status "built"; commit on the loop branch (outsystems-git-helpers message); ensure the component's GitHub issue exists (dedup: search for `[node:<id>]`; create only if absent) as type Component/handover; include the DECISION-LOG in the handover doc inside a collapsed `<details>` ("Why / alternatives ruled out"); add it to the GitHub Project (`gh project item-add <num> --owner <owner> --url <issue-url>`); attach to the tier's handover epic as a sub-issue (`gh issue edit <child> --parent <epic>`); update the Style Guide page.
   - BLOCKED -> the checker could not judge fidelity (missing or stale frozen ref). Do NOT count a round. Re-snapshot the ref (step 1b) and re-run; if the ref cannot be obtained, status "needs-human" with the reason.
   - FAIL (subjective) -> feed the CRITIQUE to @outsystems-loop:maker, increment the item's round counter; if over caps.max_rounds_per_item, status "needs-human" with the critique, move on.
   - After a PASS, do a **visual check**: render the component in the local preview (`npm run preview`) and compare against the ref's `figma.png`. The build gate proves it compiles, not that it looks right.
4. Findings — file ONLY what survived the checker's challenge:
   - For every FINDINGS-CONFIRMED: dedup by `[node:<id>]`, then file a GitHub Bug (type Bug + labels finding,bug,<type>,sev:*), add it to the Project, set `disposition: filed`. NEVER resolve a brand/a11y conflict.
   - For every FINDINGS-CHALLENGED-OUT: record it in findings/findings-register.md as `disposition: not-reproduced` with the checker's usage evidence and `challenged_by: round <n>`. Do NOT open a bug. This is the false-positive filter — a refuted finding never reaches a human's triage queue.
5. Persist state.json after every item; increment iteration. Keep the per-item `rounds`, `risk_tier`, `det_gate`, `confidence`, `decision_log`, and per-finding `disposition` so the run's Review metrics can be computed.

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
Do NOT touch OutSystems — integration is manual via the handover sub-issues.

$ARGUMENTS
