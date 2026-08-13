---
id: ADR-013
type: decision
status: accepted
date: 2026-08-14
extends: ADR-012 (Intake/Job two-level aggregate)
supersedes:
superseded_by:
---
# ADR-013 — The door, decomposed

From building Sprint 4.5. ADR-012 §Vocabulary ruled: *"The single service object stays
`JobActions` (ONE DOOR, [[Design laws]] #7); intake verbs are added there rather than a
parallel `IntakeActions`."* Building it exposed three unrelated jobs crammed into one class,
and a name that lied both ways: `JobActions.deliver!(intake)` names the whole after a part;
`IntakeActions.start_work!(job)` would be equally wrong the other direction. An intake has no
"start work" verb at all — that state is *derived* from its jobs, never a move of its own.
This ADR reverses the single-door ruling.

## Decision

A door owns **state moves only**. Everything else that had accreted onto `JobActions` left:

1. **Creation leaves.** A birth is authoring, not a transition — `CreateIntake` opens a visit
   (never empty: it and its first repair, one transaction); `CreateJob` adds a repair under an
   intake it is **given**, never creating one. `register_job!` is deleted. Its opens-or-finds
   semantics went **nowhere in production** — the caller now decides which service to call
   (`intakes#create` opens a visit, `jobs#create` adds to a given one). The old behavior
   survives only as a test helper (`test/test_helper.rb#register_job!`), composed from the two
   real services, kept because several tests add a second job to the same open intake.

2. **Authorization leaves.** `ensure_counter_staff!`, `ensure_job_crew!`, `ensure_catalog_role!`
   and their `*_actor` resolvers move to a new **`Permissions`** class — the one home for "who
   may do what." Controllers gate with `require_*!` (raising `ActionRefused`, or returning the
   authorizing `WorkshopStaff`); views hide with matching `can_*?` predicates; leaf services
   resolve the acting staff with `Permissions.staff!` once past a gate. A door verb still
   resolves *who to stamp* as `created_by` — it just no longer judges whether they may.
   One exception stays on the door: **target-validation**. `ensure_active_technician!` checks
   the technician being *assigned* (the argument), not the caller — that's the move's own
   legality, the same kind of check as a stage guard, so it stays with `assign_technician!`.

3. **The door splits by level.** `JobActions` = a repair's stage machine. `IntakeActions` =
   the visit's own moves (`deliver!`, `cancel_intake!`) plus the intake blocker trio.
   `reconcile_intake!` stays in `JobActions`, not `IntakeActions`, so the job → intake lock
   order lives in one file a reader can see end to end (see Consequences).

4. **One shared refusal type.** `JobActions::Refused` → a standalone `ActionRefused`, raised
   by both doors and rescued identically by both controller families.

## Why

- **Intake has no verb of its own to "do work."** Job's floor verbs (`start_work!`,
  `mark_done!`, blockers) have no intake equivalent — the visit's state is a read over its
  jobs. A single class can't be named honestly when one level has a rich verb set and the
  other has almost none; two classes let each carry only what's true of it.
- **The controller boundary is where "who may act" already lives for the coarse case**
  (`require_counter!` on `create`), so pushing the fine-grained checks to the same boundary is
  one mental model, not two. It also makes the authorization table *visible*: a controller's
  `before_action` list now reads as who-may-do-what for that resource, instead of being
  buried inside a service method nobody opens unless something breaks.
- **Removing the actor check from cascaded calls is not a hole, because nothing changed about
  who reaches a door verb.** `IntakeActions.cancel_intake!` still authorizes once, then
  cascades `JobActions.cancel!` per open repair — exactly as before, just with the check now
  living at the top of the use case instead of re-run on every repair in the loop.

## The load-bearing consequence

**Authorization is now per *use case*, not per *call*.** A door verb no longer defends itself
— it trusts that whoever invokes it already passed a gate. That is only safe under one
standing invariant, and it has to hold everywhere, forever:

> **A door verb is reached only from a gated controller action, or from another door verb.**

If a future code path calls `JobActions.cancel!` (or any door verb) from somewhere that isn't
one of those two — a background job, a console script acting on user input, a new controller
action someone forgets to gate — it runs **unchecked**. The lock-ordering law (job → intake,
kept in one file per ADR-012) has a comment-level home to defend it; this invariant does not
yet have an equivalent enforced check. Recorded here so it's a known standing obligation, not
a silent assumption.

## Rejected alternatives

- **One class, sectioned + renamed.** Keep `JobActions`/intake verbs together, split into
  commented sections, and rename off `JobActions` to something aggregate-flavored. Rejected on
  the name alone — `AggregateActions` drew an immediate "aggregate *what*?" A section comment
  doesn't fix a class whose two halves have genuinely different verb shapes.
- **A shared `Door` module.** Extract `refuse!` + actor resolution into a module both classes
  `extend`. Dissolved once authorization left entirely — the only thing left to share was the
  `Refused` error, which doesn't need a module, just a standalone class.
- **CanCanCan / Pundit.** Considered for the new `Permissions` layer. Rejected for now: the
  surface is small and idiosyncratic (a handful of verbs, not a CRUD matrix), crew-awareness is
  a join through `job_technicians` a static `Ability` table doesn't express well, and catalog
  rules (`blocker.raised_by_role`) are workshop *data*, not code. The decisive point: every
  guard here must **return the `WorkshopStaff` to stamp `created_by`** — a plain boolean
  (`can?`) isn't enough, so the gem would sit beside a hand-rolled resolver rather than replace
  it. Revisit only if the surface grows into a large, standard CRUD-by-role matrix — and reach
  for Pundit over CanCanCan then, since `Permissions` is already shaped like a policy object.

## Consequences

- **Design law #7** amended: "ONE DOOR" now means one door **per level**, not one door total.
- **Design law #9**'s justification for a separate action class drops "shared permission
  rules" — that's `Permissions`'s job now. What still uniquely justifies a door is cross-model
  orchestration plus the mandatory audit log.
- **Role-gating tests relocate.** Assertions like *"an off-crew technician is refused"* no
  longer belong in `job_actions_test.rb` — the door doesn't check that anymore, so asserting a
  refusal there would test something no longer true. They moved to a new `permissions_test.rb`,
  built directly against `Permissions`. What stays in the door's own tests: stage/state
  legality, the allow-list, the blocker veto, transaction integrity.
- **Controllers gained visible authorization tables and safe params.** Two controller families
  that previously had **no controller-level gate at all** (`Jobs::BlockersController`,
  `Intakes::BlockersController`, `Jobs::TechniciansController` — authorization lived only
  inside the door) now carry `before_action :require_*!` per verb.
- **`ActionRefused`** replaces `JobActions::Refused` everywhere — controllers, views, tests.

## Related
- [[ADR-012 Intake-Job two-level aggregate]] (extended by this) · [[Design laws]] ·
  [[Job]] · [[Intake]] · [[Sprint plan]]
