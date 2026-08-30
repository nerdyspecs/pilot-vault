---
type: agent-guide
updated: 2026-08-27 (Session 38 — screen-decision notes in the build list; hard rule: propose before writing into model/service/migration)
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
7. [[Design system]] — **the single source of truth for design**: tokens, type scale, geometry,
   elevation, components, UI laws, and the UI build order *(renamed from `Visual theme` 2026-08-25;
   it also absorbed the design plan that lived in [[Screen map]])*
8. [[Overview]] — current work (Module 1: Job Monitoring)
9. [[Roadmap]] — what's designed vs left to build
10. [[Sprint plan]] — task-level build order and progress ticks
11. [[worklog]] — the latest entry: recent discussions, decisions, and what's next

**Building UI (a Sprint 5A task)?** Also read [[UI rollout]] — it names the files, components and
style rules each task builds. Status lives in [[Sprint plan]], spec lives there; never both.

**Working on one specific screen?** Also read its note in [[Screen decisions]] — one per screen,
holding *why the screen is the way it is*: what was settled, what is open, and what was rejected
and why. Read the *Rejected* section before proposing a change to that screen; it exists so the
same option is not re-argued. Reasoning only — never a tick, a file list, or a rule.

## Reading list — branding / marketing
1. [[Positioning]] — the anchor: name story (Knot), audience, message hierarchy, pricing posture
2. [[Voice and tone]] — the voice laws every line of copy obeys
3. [[Design system]] — brand color, typography, the "industrial & confident" mood
4. [[Product overview]] — what the product is and is NOT
5. [[User stories]] — the audience and their language

(When `07 Brand` grows past a few notes, reinstate a [[Brand overview]] hub as item 1.)

Skip the ADRs, data model, tech stack, and sprint plan — they don't bind brand work.
If a brand decision would touch the app's UI, [[Design system]] is the source of truth and
[[Design laws]] still apply.

## How to work with me
- Explain the thinking first, then show the code.
- If my approach has a problem, say so directly before offering an alternative.
- Flag when a suggestion adds a dependency — let me decide if it's worth it.
- Match my existing patterns and naming if I share code.
- Names land on the first read — see [[#Naming]]; comments carry what the code can't — see [[#Comments]].

## Naming
A name should land on the first read, with minimal guesswork. Prefer the concrete domain
word over the abstract or generic — if a reader has to stop and ask "what does this refer
to?", it's the wrong name (`AggregateActions` → what aggregate?; a `Door` module → jargon).
Rename toward the word someone at the workshop would actually say.
- **Verbs are actions, not CRUD** — `deliver!`, `send_back!`, not `update_status`.
- **Sigils mean things** — `!` changes state through the door, `?` is a derived query.
- **Parallel things get parallel names** — qualify (`raise_intake_blocker!`), don't reinvent.
- **Spend characters to kill guesswork** — a longer explicit name beats a clever short one.
- Good names are what let a comment stay rare — see [[#Comments]].

## Comments
Names and structure carry intent. If a competent reader would understand the line from the
code and its names, a comment is noise — delete it. The plain "why" that good naming already
conveys is **not** a comment.

Write one only when the code *can't* speak for itself — when something is non-local or
counterintuitive:
- **Surprise** — reads like a bug but is deliberate (`cancel_intake!` takes no lock: "looks
  unsafe, it isn't, because job verbs already lock job → intake").
- **Hidden coupling** — this line leans on, or is leaned on by, something off-screen (a lock
  order held in another method, a write another class depends on).
- **Off-screen constraint** — a DB quirk, an ordering requirement, an index that forces the
  shape (the partial-index reason `Intake#status` is stored, not derived).

The test before writing one: *would a competent reader be surprised or misled without it?*
If no, cut it.

Keep them short — one or two lines; a class header ~4 lines max (what it is + the one
invariant it carries). **No vault cross-refs in code** — ADR/sprint/doc names (`ADR-012`,
`Event log.md`, `M1-F1`) live here in the vault, not in comments. If it needs a paragraph to
explain, it belongs in an ADR or the module note, not inlined.

## Hard rules
- Never suggest a gem without explaining why the Rails built-in isn't enough.
- Never rewrite working code into a "better pattern" unless I ask.
- Never give me three versions — pick the best one and explain why.
- **Reason before writing into a deeper layer, and wait for a yes.** A view or a controller is
  cheap to redo and I can review it by looking at the screen. A **model, a door, a concern or a
  migration is shared API** — a change there changes every caller, including the ones not in front
  of us. Before adding to or altering anything under `app/models`, `app/services` or `db/migrate`,
  say **what** you want to add or change, **why the shallower layer can't carry it**, and **who
  already calls it**. Then stop and wait. Reporting it afterwards is not the same thing.
  *(Session 38: asked to fix `workshops#show`, the agent also added two `Job` class methods and
  changed `Intake#ready?` from `jobs.exists?` to `jobs.any?` in the same pass. Both were sound,
  the N+1 reasoning was real, and both were disclosed straight after — but `ready?` guards
  `IntakeActions`' refusal to deliver a car with unfinished repairs, and is read by three views.
  A depth boundary crossed quietly is the one I cannot catch by reading the page.)*
- **Never start the dev server or drive the in-app browser unless I ask.** Verify by reading
  the code and running the test suite; if seeing it rendered is the only way to be sure, say
  so and wait. *(Session 37: driving the browser to check a layout put two stray
  `WorkshopStaffRole` rows on a seeded persona — element refs went stale after a collapse and
  the clicks landed on the Crew screen — and twice left an orphaned server holding the port.
  Running the app is my call to make, not the agent's.)*

## Default working mode
- Reason before building: for non-trivial work, propose the approach before writing code.
- Small, single-purpose changes over big sweeping ones.
- At a real fork, pick the best option and explain — don't offload the choice onto me.
