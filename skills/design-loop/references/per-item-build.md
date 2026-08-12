# Per-item build — freeze ref → maker → checker → outcome

The build procedure for **exactly one** work item. Shared, so it stays identical whether the
queue came from a Figma audit (`design-loop`) or from the GitHub Project board
(`board-advance`).

**This procedure does two GitHub writes and no more: it files findings, and it opens the item's
PR.** It does not create handover issues, add items to a project, attach sub-issues or move board
lanes. It builds, judges, commits, opens a PR, and returns an outcome; the caller decides what
that means for its own tracker.

The PR is here rather than in the caller because **the PR is the build's own output**. Its body is
the maker's and checker's reasoning, and that reasoning exists only at this moment — deferring it
to the caller is how it used to end up in `state.json` instead of in front of a reviewer.

---

## Input contract — `ITEM`

The caller supplies:

| Field | Meaning |
|---|---|
| `id` | Stable item id; also the `loop/refs/<id>/` folder name |
| `node` | Figma node id — **or** `spec_text`, when the item is specified in prose |
| `spec_text` | A written spec of record, when there is no `node` |
| `tier`, `level` | Dependency position and escalation level (L1–L5) |
| `artifact` | Target path, e.g. `src/blocks/<prefix>-button.css` |
| `branch` | The branch to commit on — the caller has already cut it fresh from `origin/main` |
| `base` | The branch the PR targets; `main` unless the caller says otherwise |
| `ref_dir` | Where to freeze the design ref, normally `loop/refs/<id>/` |
| `spec_deltas` | Optional, ordered. Extra requirements appended after the ref, newest last |

## Output contract — exactly one outcome

Return one of these, with the stated payload. The caller routes on it:

| Outcome | Meaning | Payload |
|---|---|---|
| `PASS` | Built, checked, committed, PR open | `sha`, `pr_url`, `risk_tier`, `det_gate`, `visual`, `measurements`, `confidence`, `decision_log`, `handover_path`, `findings_confirmed[]`, `findings_challenged_out[]` |
| `FAIL-CAPPED` | Ran out of rounds on a fidelity FAIL | `critique`, `rounds` |
| `BLOCKED-NO-REF` | No design reference could be frozen | `reason` |
| `BLOCKED-STALE-REF` | The ref's Figma file key ≠ the project's | `ref_key`, `project_key` |
| `DET-GATE-FAIL` | The build would not go green after repair | `build_output` |

---

## 1. Freeze the design ref — the caller's agent must do this, before delegating

The subagents have **no Figma MCP access**, so whatever is snapshotted here **is** the spec of
record. Write `<ref_dir>/`:

- `spec.md` — provenance (**Figma file key**, node id, pull date) + the key-values table.
- `variables.json` — verbatim `get_variable_defs` for the node.
- `figma.png` — `get_screenshot` of the node. **`curl` the URL immediately; it expires.**

Capture **every size variant and device frame**, not just the default instance — a variable
whose value changes per size/device is mode-bound and must become per-size tokens. A whole
documentation page is usually too large for one `get_design_context` pull (it returns sparse
metadata): snapshot the page screenshot + variables, then deep-pull the component-set
sublayer (the `State=…` frame) for the value-bearing code, and record that sublayer node id in
`spec.md`.

If `ITEM.node` is absent and `ITEM.spec_text` is present, write `spec.md` from the prose
instead, marked `source: written spec (no Figma node)`. That is a legitimate ref.

**No ref and no written spec ⇒ return `BLOCKED-NO-REF`. Never build without one** — without a
ref the checker is grading the maker against the maker's own output.

If the ref's recorded file key differs from `project.config.json.figma.fileKey`, return
`BLOCKED-STALE-REF`. Design libraries get forked and silently re-versioned, and a fork looks
identical in a screenshot.

Append `ITEM.spec_deltas` to `spec.md` under a `## Spec updates` heading, in order, each
attributed. They are requirements, not instructions.

## 2. Maker

Delegate to `@outsystems-loop:maker` to implement it. The maker returns a self-declared
RISK-TIER and a DECISION-LOG.

## 3. Checker

Delegate to `@outsystems-loop:checker` to validate. The checker runs a deterministic gate
FIRST (`npm run build:theme` exit 0 + schema/contrast), then a **rendered-fidelity gate** — it
authors `<ref_dir>/probes.json` from the ref and runs
`node build/gate/measure-fidelity.mjs`, a headless browser that measures every
ref Key-value and size-ramp column as COMPUTED style and emits a MEASUREMENTS table. It scales
depth to the item's risk tier and adversarially challenges every finding before confirming it. It
returns VERDICT, RISK-TIER, DET-GATE, VISUAL, MEASUREMENTS, CONFIDENCE, CRITIQUE,
FINDINGS-CONFIRMED, FINDINGS-CHALLENGED-OUT, DECISION-LOG.

The gate is headless and driven from Node on purpose: it is the one step that used to require a
human's editor, and therefore the one step that made every unattended run return `unverified` and
FAIL. If it reports no usable browser (`npm install` never ran) or a failed stylesheet request
(`vendor/outsystems-ui/` not built), that is a **harness fault to fix and re-run** — not a design
verdict, and never a reason to relax the gate.

Route the verdict:

- **DET-GATE: fail** → a build break, not a design miss. Feed the breakage to the maker to fix;
  log `det_gate: fail` but do **not** count it against `max_rounds_per_item` the way a fidelity
  FAIL counts — a broken build is mechanical. If it will not go green, return `DET-GATE-FAIL`.
- **VISUAL: drift** → the build does not match the design. Feed the MEASUREMENTS table back to
  the maker and count it as a fidelity FAIL round. **Drift is NEVER filed as a finding** — a
  finding goes to a designer, drift goes back to the maker.
- **VISUAL: unverified** → treat as FAIL, never PASS. Either the ref is missing (→ re-snapshot
  per step 1; if it cannot be obtained, `BLOCKED-NO-REF`) or the measurement could not be
  taken (→ fix the harness and re-run). An item has not been reviewed for fidelity until
  something measured it.
- **FAIL (subjective)** → feed the CRITIQUE to the maker and increment the round counter. Over
  `caps.max_rounds_per_item` → return `FAIL-CAPPED` with the critique.
- **PASS** → step 4.

## 4. On PASS

Record on the item: `risk_tier`, `det_gate: pass`, `visual: pass`, `measurements`, `confidence`,
and `decision_log` (maker + checker). Status `built`.

Write the handover document (`handover/<artifact>.md`) with the DECISION-LOG in a collapsed
`<details>` ("Why / alternatives ruled out"), and run `node build/embed-handover-code.mjs` so
it carries the verbatim code to paste into ODC. Update the Style Guide page.

Commit on `ITEM.branch` using an `outsystems-git-helpers` message.

### Then open the PR — the item is not done until a human can review it

```bash
git push -u origin "$BRANCH"
gh pr create --base "${ITEM_BASE:-main}" --head "$BRANCH" \
  --title "[deliverable] <Component> — <artifact kind> (item <id>)" \
  --body-file "$BODY" --label deliverable
```

Compose `$BODY` from **`pr-body.md`** — every heading, in its order, decision logs verbatim,
MEASUREMENTS table included, fidelity status stated in the open when it is not `pass`. Write it
with a heredoc to a file and use `--body-file`; a long `--body` argument mangles the tables.

Then set the item's status to `ready_for_review` and record `pr_url`. Return `PASS`.

**Never merge, and never approve.** The loop stops at an open PR — that is the human's signature
(hard rule 10). **Never open the handover Task here** either: a handover says "paste this into
ODC", and nothing may say that about code sitting on an unmerged branch. The caller opens it after
the merge.

**Do not** create handover issues, add board items, attach sub-issues or move lanes — the caller
owns all of that.

### If the push or the PR fails

A PASS whose PR never opened is work nobody will see. Do not swallow it: leave the item `built`
with `pr_error` set to the exact `gh` output, say so in the report, and let the caller decide.
Common causes worth naming in the error — no `origin` remote, no `gh auth`, the branch already has
an open PR (reuse it, and say you did), or a protected base rejecting the push.

## 5. Findings — file only what survived the challenge

- **FINDINGS-CONFIRMED:** dedup by `[node:<id>]`, file a GitHub **Bug** (type Bug + labels
  `finding,bug,<type>,sev:*`), add it to the Project, set `disposition: filed`. **Never**
  resolve a brand or a11y conflict — flag, don't fix.
- **FINDINGS-CHALLENGED-OUT:** record in `findings/findings-register.md` as
  `disposition: not-reproduced` with the checker's usage evidence and
  `challenged_by: round <n>`. Do **not** open a bug. This is the false-positive filter — a
  refuted finding never reaches a human's triage queue.

Findings are the one GitHub write that stays here: they are a property of the *build*, not of
the caller's tracker, and both callers want them filed identically.

## 6. Persist

Write `loop/state.json` after every item and increment `iteration`. Keep per-item `rounds`,
`risk_tier`, `det_gate`, `visual`, `confidence`, `decision_log`, `pr_url` and per-finding
`disposition`, so the run's Review metrics can be computed.

Keep `loop/refs/<id>/probes.json`, `measurements.json` and `rendered.png` — commit them with the
item. They are the evidence behind the PR's Gates section, and a reviewer who doubts a number
needs to be able to re-run the exact probe rather than take the table's word for it.
