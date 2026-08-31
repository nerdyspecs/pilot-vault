---
type: agent-guide
updated: 2026-08-31 (Session 39 — reading lists reorganized into three lanes; §Naming, §Comments and the convention rules moved to [[Conventions]] verbatim; prior: Session 38 — screen-decision notes in the build list; hard rule: propose before writing into model/service/migration)
---
# Agent Guide
Instructions for Claude. The builder's own entry point is [[Home]]; this note is yours.

At the start of every session, **identify which lane(s) the session's work falls in — UI,
solution/backend, business — state the classification, and ask only if the request doesn't make
it clear.** Lanes combine: a Sprint 5A task is UI *and* backend, so read both lists. When in
doubt, or for anything touching the Rails app, use the solution list. The vault is linked so a
task needs its *neighborhood*, not the whole graph.

## Reading list — solution / backend (default)
Read in this order:
1. [[Builder identity]]
2. [[Product overview]]
3. [[User stories]]
4. [[Tech stack]]
5. [[Decisions]]
6. [[Architecture laws]] — the invariants every decision must obey
7. [[Conventions]] — **the card**: the builder's coding rules (naming, comments, method shape,
   DRY-vs-flatten criteria, change discipline). Binding in every code session.
8. [[Overview]] — current work (Module 1: Job Monitoring)
9. [[Roadmap]] — what's designed vs left to build
10. [[Sprint plan]] — task-level build order and progress ticks
11. [[worklog]] — the latest entry: recent discussions, decisions, and what's next

Making a decision that outlives the session? Follow [[Workflows]] §Solution.

## Reading list — UI
Everything in the solution list, plus:
1. [[UI rules]] — **the card**: the seven rules with their values, held open while building
2. [[Design system]] — **the book, authoritative on any conflict**: tokens, type scale, geometry,
   elevation, components, UI laws, and the UI build order *(renamed from `Visual theme`
   2026-08-25; it also absorbed the design plan that lived in [[Screen map]])*
3. [[UI rollout]] — names the files, components and style rules each Sprint 5A task builds.
   Status lives in [[Sprint plan]], spec lives there; never both.
4. **Working on one specific screen?** Its note in [[Screen decisions]] — one per screen, holding
   *why the screen is the way it is*: what was settled, what is open, and what was rejected and
   why. Read the *Rejected* section before proposing a change to that screen; it exists so the
   same option is not re-argued. Reasoning only — never a tick, a file list, or a rule.

The working loop is [[Workflows]] §UI.

## Reading list — business / brand
1. [[Positioning]] — the anchor: name story (Knot), audience, message hierarchy, pricing posture
2. [[Voice and tone]] — the voice laws every line of copy obeys
3. [[Design system]] — brand color, typography, the "industrial & confident" mood
4. [[Product overview]] — what the product is and is NOT
5. [[User stories]] — the audience and their language

(When `07 Brand` grows past a few notes, reinstate a [[Brand overview]] hub as item 1.)

Skip the ADRs, data model, tech stack, and sprint plan — they don't bind brand work.
If a brand decision would touch the app's UI, [[Design system]] is the source of truth and
[[Architecture laws]] still apply.

## How to work with me
- Explain the thinking first, then show the code.
- If my approach has a problem, say so directly before offering an alternative.
- Flag when a suggestion adds a dependency — let me decide if it's worth it.
- Match my existing patterns and naming if I share code.

## The rules themselves live on the cards
The builder's coding rules — naming, comments, method shape, DRY-vs-flatten, dependency policy,
and the change-discipline hard rules (depth boundary, no dev server, no unprompted rewrites) —
moved to **[[Conventions]]** on 2026-08-31 (Session 39), verbatim with their provenance. The UI
equivalents are on **[[UI rules]]**. Read the card for the lane you're in; a new rule produced
mid-session is captured per [[Workflows]] §Rule capture — never appended here.

## Default working mode
- Reason before building: for non-trivial work, propose the approach before writing code.
- Small, single-purpose changes over big sweeping ones.
- At a real fork, pick the best option and explain — don't offload the choice onto me.
- Close every session per [[Workflows]] §Session close.
