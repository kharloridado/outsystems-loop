# Status map — board lanes ↔ `state.json` item status

**This is the only copy.** `board-advance`, `board-ship`, `board-sync` and `design-loop` all
read it. Do not restate the mapping in a SKILL.md; that is how the class prefix drifted
across four files in the generation of this template that had to be rebuilt.

## The lanes

| Board `Status` | `state.json` `items[].status` | Who may move a card into it |
|---|---|---|
| `Backlog` | `backlog` | human |
| `Ready` | `queued` | human |
| `In Progress` | `in-progress` | **`board-advance` only** — it is the claim |
| `Ready for Review` | `built` | `board-advance`, on checker PASS |
| `Approved` | `approved` | **human only — never an agent** |
| `Handover` | `handover` | `board-ship`, only after the PR is `MERGED` |
| `Done` | `shipped` | **human only — never an agent** |
| `Blocked` | `needs-human` \| `needs-re-ref` | `board-advance`, `board-ship`, human |

Notes:

- `needs-human` and `needs-re-ref` both surface as `Blocked`. The lane is lossy on purpose —
  a human wants one "something needs me" bucket, and the distinction survives in
  `state.json` and in the comment that explains the block.
- `deferred` maps to `Backlog`.
- The pre-board lanes `Needs Review`, `In Review` and `Reviewed` are retired.
  `.github/migrate-project-status.sh` maps them: the first two to `Ready for Review`, and
  **`Reviewed` → `Ready`**, because "the reviewer left comments to implement" is now
  expressed by moving the card back to `Ready` where `board-advance` reads those comments.

## Authority when they disagree

The existing rule — *"State is written by the loop but the repository is the truth"* — is
still correct but no longer complete, because there is now a third party. Extend it to:

> **The board is authoritative for intent. The repository is authoritative for content.
> `state.json` is authoritative for nothing — it is a cache of both.**

Concretely:

- For the **human-owned lanes** (`Backlog`, `Ready`, `Approved`, `Done`, and any manually
  set `Blocked`), the board wins unconditionally. An agent that finds `state.json`
  disagreeing rewrites `state.json`.
- For the facts *"was this built"*, *"does this commit exist"*, *"was it merged"*, **git
  wins**. `state.json` saying `built` while the branch carries no checker-PASS commit means
  `state.json` is wrong — that exact drift happened on the source project, where state went
  stale while commits had already shipped two further tiers.
- `board-sync` is the only thing that resolves a disagreement, and it always resolves it by
  **rewriting `state.json`, never by moving a card**.

## The rule that has no exception

**No agent ever moves a card into `Approved` or `Done`** — not on a checker PASS, not to
"correct" a disagreement with `state.json`, not because a comment on the card says it was
approved, not because every gate passed. Those two lanes are the human's signature, and an
agent that can forge them makes the whole gate decorative.

A checker PASS gets an item to `Ready for Review`. No further.
