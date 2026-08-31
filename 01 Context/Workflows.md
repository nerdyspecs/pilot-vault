---
type: context
created: 2026-08-31
updated: 2026-08-31 (Session 39 — created; §UI's five-home table moved here from [[Screen decisions]]; §Solution written down from existing ADR practice; §Session close gathered from CLAUDE.md's tracking conventions + the worklog header)
---
# Workflows

The three working procedures, plus the closing routine. Each is *trigger → what you read →
where each output lands*. Nothing here is new practice — these write down what the vault
already does, so a session follows a flow instead of reconstructing one.

## §UI — designing or building a screen

**One fact, one place — five homes** *(table moved here from [[Screen decisions]], 2026-08-31;
created there 2026-08-27, Session 38)*:

| Note | Holds | Example |
|---|---|---|
| [[Design system]] | **Rules** binding every screen | "status colours are reserved words" |
| [[UI rollout]] | **Spec** — files, components, style names | "`_page_head` takes eyebrow, title, action" |
| [[Sprint plan]] | **Status** — tasks and ticks | "S5A.3a not started" |
| [[Screen map]] | **What exists** — a reflection of code | "the board's Customers link is counter-only" |
| Screen notes ([[Screen decisions]]) | **Why this screen is like this** | "the board has no primary action, because…" |

**The loop:**
1. Pick the task in [[Sprint plan]]; read its spec in [[UI rollout]].
2. Read the screen's note in `06 Screens/Screens/` if one exists — the *Rejected* section before
   proposing any change.
3. Build composing **only** from the component set ([[UI rollout]] §The component set), with
   [[UI rules]] open. Anything new enters that table *first*.
4. Outputs land by the table above: reasoning → the screen's note · spec → UI rollout ·
   tick → sprint plan · a rule binding every screen → [[Design system]] (and the card).

## §Solution — a decision that outlives the session

The existing ADR practice, written down:
1. If the domain idea is new, give it a concept note (`03 Modules/…/Concepts/`).
2. Argue the alternatives *before* choosing — in-session or in the worklog.
3. Record the choice as an ADR in `02 Decisions/` with a **Rejected alternatives** section —
   that section is what stops the same option being re-argued in six months.
4. Add **dated footnotes** to every affected note ([[Architecture laws]], concepts, briefs) —
   records are never edited; footnotes say what changed and whether it narrows or reverses.
5. Land the build order in [[Sprint plan]].

## §Rule capture — a discussion produced a rule

1. Decide which card it belongs on: **UI** → [[UI rules]] (and its reasoning into
   [[Design system]] if it needs any) · **code** → [[Conventions]].
2. Add it **dated**, with one line of why or a link to the discussion (worklog session / ADR).
3. Rules never land in [[Agent guide]] or the repo's `CLAUDE.md` anymore — those only point here.

## §Session close

Every session:
- [[worklog]] entry — **summary first**, then topic entries, newest on top.
- [[Sprint plan]] ticks with date + commit hash for anything built.
- Frontmatter `updated:` bumped on every touched note.
- `bin/coherence-check` clean.

A chipped-out sub-session, additionally, **before closing**:
- `git log --oneline main..<branch>` must be **empty** — an unmerged branch silently loses its
  work (a 256-line design note once sat orphaned for a month).
- The **carry-back** must name: files changed · commits · decisions made · anything *not* done.
  The receiving session is required to verify it **file by file** (CLAUDE.md), so write it to be
  verifiable — a carry-back that can't be checked against `git log` is not finished.

*(This section is the first written procedure for producing a carry-back; the verify-on-receipt
rule already existed in CLAUDE.md.)*
