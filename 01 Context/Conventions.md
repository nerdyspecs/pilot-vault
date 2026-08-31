---
type: context
created: 2026-08-31
updated: 2026-08-31 (Session 39 — created; §Naming, §Comments and the convention rules moved here from [[Agent guide]], verbatim; DRY criteria stated from [[Architecture laws]] law 9 + ADR-013)
---
# Conventions — the card

> **The builder's coding rules, held open *during* Ruby work.** One home for every rule that used
> to live scattered across [[Agent guide]] (a note addressed to Claude), the repo's `CLAUDE.md`,
> and law footnotes. New rules land **here**, dated — see [[Workflows]] §Rule capture. The
> architecture invariants themselves stay in [[Architecture laws]]; this card is about *how code
> is written*, not what the system must never do.

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

## Method shape
Methods always use explicit `def`/`end` — never endless methods (`def foo = expr`), even for
one-liners. The builder reads code by block shape. *(From the repo's CLAUDE.md.)*

## When to DRY up, when to stay flat
A separate action class is earned, never defaulted to. The criteria *(2026-07-13, reaffirmed;
narrowed 2026-08-14 by [[ADR-013 The door decomposed]])*:
- A door/action PORO is justified only by **cross-model orchestration plus a mandatory audit
  log** co-occurring (e.g. `JobActions`). "Shared permission rules" no longer qualifies — that
  moved to `Permissions`, checked at the controller boundary.
- **Single-aggregate commands stay on the model as bang methods.** A `WorkshopActions` layer was
  considered and rejected (worklog Session 13).
- Calculations live in the model layer, never controllers or views — [[Architecture laws]] law 9
  holds the full text and footnote trail; this card only states the working rule.

## Dependencies
- **Never a gem without explaining why the Rails built-in isn't enough.** Currently: devise,
  slim-rails, bootstrap, dartsass-rails. Anything that adds a dependency is flagged — the
  builder decides.

## Change discipline
- **Never rewrite working code into a "better pattern" unprompted.** No three-version answers —
  pick the best, explain why. Small single-purpose changes.
- **Reason before writing into a deeper layer, and wait for a yes.** A view or a controller is
  cheap to redo and reviewable by looking at the screen. A **model, a door, a concern or a
  migration is shared API** — a change there changes every caller, including the ones not in
  front of us. Before adding to or altering anything under `app/models`, `app/services` or
  `db/migrate`: say **what** changes, **why the shallower layer can't carry it**, and **who
  already calls it** — then stop and wait. Reporting it afterwards is not the same thing.
  *(Session 38: asked to fix `workshops#show`, the agent also added two `Job` class methods and
  changed `Intake#ready?` in the same pass — both sound, both disclosed after, but `ready?`
  guards `IntakeActions`' refusal to deliver a car with unfinished repairs and is read by three
  views. A depth boundary crossed quietly is the one the builder cannot catch by reading the
  page.)*
- **Never start the dev server or drive the in-app browser unless the builder asks.** Verify by
  reading the code and running the suite; if seeing it rendered is the only way to be sure, say
  so and wait. *(Session 37: browser-driving put two stray `WorkshopStaffRole` rows on a seeded
  persona and twice left an orphaned server holding the port. Running the app is the builder's
  call.)*
