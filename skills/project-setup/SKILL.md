---
name: project-setup
description: Stand up a freshly cloned OutSystems design-system project in one pass — interview for the project's values, fill project.config.json, install dependencies and the OutSystems UI submodule, create the GitHub labels, turn a list of deliverables + Figma links into the signed inventory and the loop queue, set up the scheduled routines, and verify the whole thing builds. Use this when a project is newly cloned from the template, when project.config.json still contains <<PLACEHOLDER>> values, or whenever the user asks to set up / initialise / bootstrap / configure a new project or engagement.
---

# Project setup — one sitting, one pass

The user has just cloned the template. Everything between that clone and a loop that produces
reviewable PRs happens here, in one conversation, without them running a runbook.

**What this replaces.** Standing up a project used to be roughly fifteen manual steps spread
across two documents: `npm install`, an interactive `npm run init`, a submodule init and pin, a
`gh auth refresh`, two or three shell scripts, hand-editing three design docs, hand-writing the
inventory table in `loop/goal.md`, hand-seeding `loop/state.json`, and hand-copying routine
prompts into the routines UI. Every one of them was a step someone could skip, and the ones people
actually skipped — the submodule, the inventory — are the ones that make the loop silently produce
garbage. Ask once, do all of it, verify it.

> **Do not guess a value to keep the flow moving.** The two things this skill must never invent are
> **conventions** (spacing base, grid, breakpoints, default size) and **scope** (what is being
> built). Both have burned a real project: an unverified `4pt` spacing base manufactured a queue of
> false-positive bugs, and an unsigned scope meant a fully finished component set was thrown away.
> A blank is cheap. A credible-looking wrong default is not.

---

## 0. Read the ground truth first

Never interview for something you can read.

```bash
cat project.config.json 2>/dev/null | head -40
git remote -v; git branch --show-current
git submodule status
gh auth status 2>&1 | head -5
node -v; ls node_modules >/dev/null 2>&1 && echo "deps installed" || echo "deps missing"
```

Then decide what this is:

- **`<<PLACEHOLDER>>` values present** → a fresh clone. Full pass, below.
- **Values already real** → an existing project. Do **not** re-interview. Say what is already
  configured, offer the specific things that look unfinished (a `TBD` convention with a known
  answer, an empty inventory, missing routines), and act only on what they pick.
- **Values real but the user says it is a new engagement** → they cloned a *project*, not the
  template. Confirm before overwriting anything: `npm run init` is re-runnable and will rewrite
  their config.

## 1. Interview — one batch, no drip-feed

Ask for everything at once. Use `AskUserQuestion` for the constrained choices and plain prose for
the free-text ones; do not ask nine questions in nine turns.

| Value | Flag | Notes |
|---|---|---|
| Customer | `--customer` | e.g. "Acme Corp" — appears in comments and issue titles |
| Project short name | `--project` | e.g. "ACME" |
| Design system name | `--design-system` | defaults to the project name |
| CSS class prefix | `--prefix` | lowercase, trailing hyphen. **`acme-` is rejected** — it is the template's own example |
| JS namespace | `--js-namespace` | `window.<Ns>Toast` for Web Component helpers |
| ODC theme module | `--theme-module` | used in self-hosted font paths |
| GitHub repo | `--repo` | `owner/repo` |
| Figma library URL | `--figma-url` | the file key is parsed out of it automatically |
| Project board URL | `--board-url` | **optional, and a human view only** — leave blank unless they want a kanban to look at |

Two you must also raise, because they cost the most when missed:

- **The deliverables** — see §4. Ask for them in this same batch: *"list the components you want
  built, each with its Figma link"*.
- **Conventions** — ask whether the designer has actually confirmed a spacing base, grid,
  breakpoints or default component size. If the answer is anything short of "yes, they told us",
  it stays `TBD`. Say plainly that `TBD` means the checker will not enforce it, and that this is
  the safe direction.

**On the board.** If they ask for one, set it up as a *view*: `board.drivesLoop` stays `false`.
The board is never the loop's queue — Projects v2 is GraphQL-only and unreachable from a scheduled
run, so a board-queued loop dies while claiming a card, before the maker, every time. The queue is
the signed inventory in `loop/goal.md`, which is a file and readable from anywhere.

## 2. Write the config — through `init-project.mjs`, never by hand

It already accepts every value as a flag, so drive it non-interactively:

```bash
node build/init-project.mjs \
  --customer="Acme Corp" --project="ACME" --design-system="Nimbus" \
  --prefix="nimbus-" --js-namespace="Nimbus" --theme-module="NimbusTheme" \
  --repo="acme/design-system" \
  --figma-url="https://www.figma.com/design/KEY/Library" \
  --board-url=""
```

It writes `project.config.json` **and** substitutes every `<<PLACEHOLDER>>` across the scaffold.
Do not hand-edit `project.config.json` for these values and do not write a prefix into any other
file — one source, many readers, and a previous generation of this template shipped a project
whose two config files disagreed about its own class prefix for its entire life.

Then confirm: `npm run check:config` must print ok.

## 3. Dependencies and the submodule — the step everyone skipped

```bash
npm install
git submodule update --init
npm run build:osui
```

**`vendor/outsystems-ui/` is a submodule and clones empty** (`git submodule status` shows a leading
`-`). It is layer 1 of the preview — the real framework CSS the checker measures against. Skip it
and the preview loads a 404 where the framework should be, the cascade silently falls back, and
every measurement describes a page nobody will ever see. The fidelity gate now detects and refuses
this, so the failure is loud, but it is still a wasted run.

**Then pin it to the version the target ODC environment actually runs** — not `main`, not the
newest tag. Ask which version; if they do not know, say so in the report and leave it, but record
that it is unpinned. Designing overrides against markup the environment never produces is a whole
class of defect that only appears after handover.

```bash
git -C vendor/outsystems-ui checkout <vX.Y.Z>
git add vendor/outsystems-ui
```

## 4. Deliverables → the signed inventory → the queue

This is the step that turns "here are the components I want" into work the loop can do. Take the
user's list of **component + Figma link** pairs and write it to two places.

**Parse the node id from each Figma URL** — `?node-id=1868-83` (or `1868%3A83`) → `1868-83`.
A row with no node id and no written spec is not buildable: the ref cannot be frozen, and without
a ref the checker would be grading the maker against the maker's own output. Flag those rows and
ask for either a link or a written spec.

**(a) `loop/goal.md` — the signed inventory of record.** Fill the goal, the Figma library URL +
**file key**, the mode (`library` for a full design system, `single` for a handful), and the
inventory table:

| # | Component | Tier | Figma node | Notes |
|---|---|---|---|---|

Order it by dependency — foundations (tokens, type) before primitives (button, input) before
composites (card) before patterns. The loop builds in this order and gates at tier boundaries.

Record **who signed it off and when**. This table is a scope contract, and `git log loop/goal.md`
is what later answers "who agreed to this". State plainly to the user that they are signing it.

**(b) `loop/state.json` — the queue.** Seed `items[]` from the same table, one entry per row, with
`id`, `tier`, `node`, and `status: "queued"`. Set `figma_url`, `figma_file_key`, `repo`, `mode`,
and leave `level` unset — level is the outcome of the maker's audit, not a backlog-time guess.
Set the tier checkpoints; `after_tokens` and `after_primitives` default to `pause` so a run stops
for review instead of rolling the whole library in one night.

**Never seed an item that has no row in the table.** Scope creep here is invisible until review.

## 5. GitHub — labels, and only what is needed

```bash
gh auth status || gh auth login
```

Create the taxonomy the findings and handovers depend on. Call `gh` directly rather than the
`.github/*.sh` scripts — those are bash, and a Windows user running through PowerShell cannot
execute them:

```bash
R=<owner/repo>
gh label create finding     -R "$R" -c "#B60205" -d "Design-conformance finding" --force
gh label create bug         -R "$R" -c "#D73A4A" -d "Finding filed as a bug"     --force
gh label create handover    -R "$R" -c "#0E8A16" -d "Code handover for OutSystems" --force
gh label create task        -R "$R" -c "#0052CC" --force
gh label create deliverable -R "$R" -c "#C5DEF5" -d "A queued deliverable"       --force
for t in a11y brand token consistency; do gh label create "$t" -R "$R" -c "#1D76DB" --force; done
gh label create sev:blocker -R "$R" -c "#B60205" --force
gh label create sev:high    -R "$R" -c "#D93F0B" --force
gh label create sev:medium  -R "$R" -c "#FBCA04" --force
gh label create sev:low     -R "$R" -c "#0E8A16" --force
for s in acknowledged accepted-as-designed fixed-in-design waived; do
  gh label create "status:$s" -R "$R" -c "#5319E7" --force
done
```

`--force` makes this re-runnable. Verify with `gh label list -R "$R"`. If `gh` is not
authenticated, do not silently skip it — findings and handovers will fail much later with a
confusing error. Report it as outstanding and carry on with the rest of setup.

Repo not created yet? Offer `gh repo create <owner>/<repo> --private --source=. --push`, and
**ask before running it** — creating a repository is not something to do on an assumption.

## 6. Routines — set them up rather than describing them

The user should not copy prompts into a routines UI. Create the schedule for them, using the
scheduling capability available in this session (the `schedule` skill or the cron tooling).

**Ask before creating any of them.** A scheduled agent is persistent configuration that runs
unattended and spends the user's budget; it is theirs to authorise. Present the set, let them pick.

See **`references/routines.md`** for the exact prompts. The default set:

| Routine | Cadence | Why |
|---|---|---|
| **Loop advance** | nightly | Builds `queued` items → one PR each. The main event. |
| **Token drift reconcile** | weekly | Re-pulls the Figma library and PRs any token drift. Recurring forever — a design system's tokens move. |
| **Findings digest** | daily, optional | Read-only summary of open findings by severity. |

Two constraints to state honestly rather than discover later:

- **A routine needs the Figma connector attached**, or it cannot freeze a ref and every item goes
  `needs-human`. Tell them to attach it when they create the routine.
- **A routine stops at every checkpoint** in `loop/goal.md`, and never approves, merges, or opens a
  handover. It advances work and opens PRs; the human signs.

## 7. Verify — do not report success you have not seen

Run the whole gate, in order, and show the real output:

```bash
npm run check:config     # no placeholders, no leaked example prefix, conventions well-formed
npm run build:theme      # check:config → assemble dist/theme.css → validate
```

Then prove the **fidelity gate can actually run**, because it is the thing that silently degrades
to `unverified` and fails every item:

```bash
cat > /tmp/smoke.json <<'JSON'
{ "probes": [ { "name": "body", "selector": "body", "props": ["font-family", "color"] } ] }
JSON
node "$CLAUDE_PLUGIN_ROOT/scripts/measure-fidelity.mjs" --probes /tmp/smoke.json
```

Exit `0` means a browser was found, the preview served, and the cascade was complete. Exit `4`
means no browser (`npm install` did not take). Exit `3` with a failed-request line usually means
the submodule step in §3 did not happen. **Fix it now** — this is exactly the failure that makes
an unattended run produce nothing at 2am.

## 8. Commit, then hand back

```bash
git add -A && git commit -m "chore: initialise <project> from the template"
```

Report in this shape — short, and honest about what is not done:

- **Configured** — prefix, namespace, repo, Figma file key, board (or "none — inventory drives the loop").
- **Queued** — N deliverables, in build order, with the tier gates that will pause the run.
- **Routines** — what was created and when it first fires.
- **Verified** — the three gates, with their actual results.
- **Outstanding** — every `TBD` convention and why it is blank, an unpinned submodule, missing
  `gh` auth, any deliverable with no Figma node. Name the human who has to resolve each.
- **What happens next** — the first run builds item 1 and opens a PR; they review and merge; the
  handover Task follows the merge.

Then tell them the two things they can say from here: *"build the next deliverable"* to advance
the loop by hand, or nothing at all — the routine does it tonight.

---

## Rules

1. **Never invent a convention or a scope row.** `TBD` and "ask" are correct answers.
2. **Never write a project value anywhere but `project.config.json`**, and always through
   `init-project.mjs`.
3. **Never set `board.drivesLoop: true`.** A board is a view; the inventory is the queue.
4. **Never report a gate as passing without running it.**
5. **Ask before anything outward-facing** — creating a repository, creating a scheduled routine,
   pushing to a remote.
6. **Do not skip the submodule.** It is the most-skipped step and it invalidates the fidelity gate.
