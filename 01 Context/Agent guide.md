---
type: agent-guide
updated: 2026-07-06
---
# Agent Guide
Instructions for Claude. At the start of every session, pick the reading list that matches the
session's work — the vault is linked so a task needs its *neighborhood*, not the whole graph.
When in doubt, or for anything touching the Rails app, use the build list.

## Reading list — building (default)
Read in this order:
1. [[Builder identity]]
2. [[Product overview]]
3. [[User stories]]
4. [[Tech stack]]
5. [[Decisions]]
6. [[Design laws]] — the invariants every decision must obey
7. [[Visual theme]] — palette, typography, and the UI laws for every screen
8. [[Overview]] — current work (Module 1: Job Monitoring)
9. [[Roadmap]] — what's designed vs left to build
10. [[Sprint plan]] — task-level build order and progress ticks
11. [[worklog]] — the latest entry: recent discussions, decisions, and what's next

## Reading list — branding / marketing
1. [[Positioning]] — the anchor: name story (Knot), audience, message hierarchy, pricing posture
2. [[Voice and tone]] — the voice laws every line of copy obeys
3. [[Visual theme]] — brand color, typography, the "industrial & confident" mood
4. [[Product overview]] — what the product is and is NOT
5. [[User stories]] — the audience and their language

(When `07 Brand` grows past a few notes, reinstate a [[Brand overview]] hub as item 1.)

Skip the ADRs, data model, tech stack, and sprint plan — they don't bind brand work.
If a brand decision would touch the app's UI, [[Visual theme]] is the source of truth and
[[Design laws]] still apply.

## How to work with me
- Explain the thinking first, then show the code.
- If my approach has a problem, say so directly before offering an alternative.
- Flag when a suggestion adds a dependency — let me decide if it's worth it.
- Match my existing patterns and naming if I share code.
- Keep comments minimal — the code should explain itself.

## Hard rules
- Never suggest a gem without explaining why the Rails built-in isn't enough.
- Never rewrite working code into a "better pattern" unless I ask.
- Never give me three versions — pick the best one and explain why.

## Default working mode
- Reason before building: for non-trivial work, propose the approach before writing code.
- Small, single-purpose changes over big sweeping ones.
- At a real fork, pick the best option and explain — don't offload the choice onto me.
