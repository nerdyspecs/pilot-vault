---
type: log
updated: 2026-08-14 (Session 30 — Sprint 4.5 built, decomposed further into ADR-013, and reconciled with the vault)
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

---

## 2026-08-14 · Session 30 — Sprint 4.5 built, split further into ADR-013, vault caught up

**Summary.** Picked up Session 29's built-but-uncommitted Intake/Job split and took it the rest of
the way: a codebase sweep, then layered commits (schema → models → services/controllers →
remainder), a comment-noise pass, and a naming/comment-discipline standard added to the [[Agent
guide]]. Then, reviewing the service layer, the builder pushed back hard on `JobActions` doing too
much — three separate concerns (state moves, authorization, creation) crammed into one class — which
produced a second decision this session, **[[ADR-013 The door decomposed]]**: creation left for
`CreateIntake`/`CreateJob`, authorization left for a new **`Permissions`** class checked at the
controller boundary, and the door split into `JobActions` (repair) + `IntakeActions` (visit).
Migrated the 29 tests still calling the removed `register_job!`, extracted the role-gating
assertions that no longer belonged in the door's own tests into a new `permissions_test.rb`, deleted
the stale system-test suite (UI is parked pending a rebuild), and — the bulk of this session — swept
the whole vault for what ADR-012 and ADR-013 broke and wrote it back to the truth. 129 runs, 0
failures, 0 errors, committed as two commits (`33f426d`, `636d47b`) on `s4.5-intake-ui`.

**Why the door split further, past what ADR-012 planned.** ADR-012 §Vocabulary had ruled the single
service object stays, intake verbs just get added to it. Building it exposed the reason that was
wrong: an Intake has no "start work" or "assign technician" verb at all — its state is *derived*
from its jobs — so a single class trying to hold both levels either lied by omission or by naming
(`JobActions.deliver!(intake)` names the whole after a part; `IntakeActions.start_work!(job)` would
be the reverse mistake). The builder's own framing carried the rest: *"job actions are doing way too
much… should this be literal actions taken to it, rather than a check on whether the action should
fire?"* — which is exactly the authorize-at-the-boundary pattern. Landed on: doors execute moves and
trust the caller; `Permissions` is where "may you?" gets answered, once, at the controller.

**One thing target-validation clarified along the way.** Not every guard on a verb is an
authorization check. `assign_technician!`'s check that the *technician being assigned* actually
holds an active technician role is validating the **argument**, not the caller — the same kind of
check as a stage guard — so it stayed on the door instead of moving to `Permissions`. The dividing
line that fell out: **authorize the actor, validate the target** — different questions, different
homes.

**A CanCanCan detour, and why it didn't take.** Introducing `Permissions` prompted the obvious
question — reach for a gem? Argued through and declined for now: the surface is small and
idiosyncratic (a handful of verbs, not a CRUD matrix), crew-awareness is a join through
`job_technicians` a static `Ability` table expresses badly, catalog rules (`blocker.raised_by_role`)
are workshop *data* not code, and — the decisive point — every guard here must **return the
`WorkshopStaff` to stamp `created_by`**, not just a boolean, so the gem would sit beside a
hand-rolled resolver rather than replace it. Revisit only if the surface grows into a large,
standard CRUD-by-role matrix, and reach for Pundit over CanCanCan then — `Permissions` is already
shaped like a policy object.

**The safe-params + naming/comment threads, folded in along the way.** Naked `params[:x]` reads
became named, presence-checked accessors (`require` for structural ids the form always sends,
`permit` for user-picked ids handled with a friendly flash) — no new params-object abstraction, just
using Rails' own strong-params deliberately. And reading the diff together surfaced the builder's own
naming signature as a real, describable pattern (verbs as actions not CRUD; sigils mean things;
parallel things get parallel names; spend characters to kill guesswork; the tell that a name is wrong
is "what does this refer to?") — written into the [[Agent guide]] alongside the comment-discipline
rule from the prior pass.

**The stashed remainder revealed a broken commit before it shipped.** Mid-session the builder asked
to squash the four-commit chain into one; before squashing, a stash-pop of parked work turned up
that the branch tip's `db/seeds.rb` still called the just-removed `register_job!` and
`JobActions.deliver!` — squashing at that moment would have baked a non-running commit into history.
Fixed first, then squashed (`git write-tree` before/after confirmed byte-identical trees — the squash
changed only history shape, not code), then a second squash-and-reopen when the builder asked to
split the migration/model/service layers back out for review.

**Two things chipped rather than resolved.** *Role-addressed blocker pins* — the builder noticed
`raise`/`note` blockers have no receiver because the target is a role, not a person, and worked out
a real rule mid-session (*address a request by capability, a reply by identity*) plus a possible
free lunch (`cleared_by_role` may already carry it, as a board query rather than a schema change).
Folded into the existing **B2** entry in [[Deferred design]] rather than left loose or duplicated —
still parked pending a real workshop, per B2's original trigger. *The Intake timeline* — the
original ask that opened this whole reconciliation thread (trace one intake's history, or the deep
trace including all its jobs) — got designed (`Intake#timeline` / `Intake#timeline_with_jobs`, the
missing `intake_blocker_transitions` association, a `handoff_state` presentation helper for the
mixed acknowledgeable/non-acknowledgeable merged feed) but not built. Recorded as **S4.5.10** in
[[Sprint plan]] — the actual next code task.

**The vault reconciliation itself.** Confirmed by grep that nothing anywhere documented
`IntakeActions`, `CreateIntake`, `CreateJob`, or `Permissions`, and that three places still asserted
the single-door shape ADR-013 reversed. Wrote a new [[Intake]] concept note (mirroring [[Job]]);
corrected the "In Rails" fact-lists in [[Job]], [[Data model]], [[Stage model]], [[Event log]],
[[Blocker]] (zero intake awareness before this pass despite ADR-012 §6 splitting blockers across
both levels), [[Job visibility]] (re-anchored from `jobs.token`/`customer_id` to `intakes` — this is
Sprint 7's design input, corrected before that sprint starts rather than after), [[M1-F1]], plus
[[Overview]] and [[Open questions]] — stale, but missing from S4.5.8's own reconciliation list.
Ticked S4.5.2–S4.5.7, rewrote S4.5.5 (its `register_job!`-opens-or-finds framing was overtaken by
ADR-013), added S4.5.9 (retro, the Permissions split) and S4.5.10 (the timeline goal), re-pointed
S5/S6/S7's stale task text, and added a warning near the top of [[Sprint plan]] recording — as fact,
no tasks scheduled — that the app currently has no working UI path (`workshops#show` and
`customers#show` both 500 on route helpers the backend restructure removed).

**Next:** S4.5.10, the Intake timeline — the task this whole session's reconciliation thread traces
back to.

---

## 2026-08-03 · Session 29 — the Intake/Job aggregate morph: design pass + Sprint 4.5

**Summary.** Took the routing/screen-map session's chip — split today's overloaded `Job` (car +
visit + one repair + one stage + one technician) into a two-level aggregate — through a full design
pass with the builder, wrote **[[ADR-012 Intake-Job two-level aggregate]]**, and sequenced it as
**Sprint 4.5** *before* the S5 board. Then built it: 4 new tables, the door grown to two levels,
130 green (122 unit/service + 8 system). **Not committed yet** — codebase sweep pending.

**Why it goes before Sprint 5, not after.** Two reasons that compound: the S5 board groups by the
**car**, so it has to sit on `Intake` or be rebuilt; and there is **no production data**, so this is
the cheapest the migration will ever be — a schema squash + reseed, the ADR-010 play, never a data
migration. Building S5 first would have meant building it twice.

**The lighter alternatives were ruled out first, and the premise recorded.** The Sprint-6 jobsheet
already covers multi-item-per-visit — but a checklist has no per-item stage, crew, blocker, or
acknowledgement, so a car with a blocked brake job and a done aircon job reads as one lie. Bare
many-jobs-per-vehicle (no `Intake`) gives independent repairs but loses the **visit**: delivery
smears across N jobs, HFP has no honest home, and the board has nothing to group by. The split's
load-bearing premise is written into the ADR rather than assumed: *it earns its keep only because
repairs need independent technicians **and** independent blockers.*

**The four sub-decisions, settled with the builder.** (1) An all-cancelled car that leaves derives
**`cancelled`**, never `delivered` — delivery means "a car with real work done was collected".
(2) Cancel-the-car cancels the remaining **open** jobs only; done work survives (law #8), and the
terminal falls out rather than being chosen — 0 done → cancelled, ≥1 done → still collectible.
(3) **`mark_done!` keeps pinning the registering SA** — the builder pushed back on dropping it, and
was right: the gap wasn't the pin, it was that `deliver!` moving to the intake left the pin with no
closing action. Fix is one line — **intake `deliver!` sweeps `acknowledge_pending!` across all its
jobs**, clearing only what's addressed to the actor delivering (a technician's open pin stays
honestly open — stamping it would be the append-only lie ADR-011 forbids). (4) **HFP moves to the
Intake** as an intake blocker: payment is a per-visit hold, and hanging it on an arbitrary one of
the car's repairs would be a domain lie.

**Intake blockers are three records but NOT acknowledgeable — the builder's catch.** The first pass
gave them the full mirror "for symmetry". The builder pushed: *"i dn think intake blockers are
acknowledgeable"* — correct, and [[Event log]] already says why: **the ack pair belongs to a
direction**. HFP is the counter holding its own car until paid (SA raises, SA clears); nobody is
ever waiting on anyone. So they keep catalog + item + events (the **note chain** is real — payment
back-and-forth) but carry **no `receiver_id`/`acknowledged_at` at all**. Identical-but-dead columns
would have been cosmetic symmetry that misleads the next reader. Recorded trigger: a cross-role
`blocks: delivered` type (e.g. manager sign-off) would be directional — additive when it lands.

**A refinement the schema forced: intake `status` is stored, not fully derived.** ADR-012 §2 first
stored only `delivered_at` and derived `cancelled`. That broke against **R5**: a Postgres partial
unique index can only key on a stored column of its own table, so a derived `cancelled` left the
busy-vehicle guard unable to tell a live-open visit from a dead-cancelled one — a fully-cancelled
car would block its vehicle's next visit. Ruled with the builder: store the terminal **position**
(`status` enum, exactly like `Job.stage`), keep **`ready?`** derived — that's the genuine
about-the-children question. The door is the only writer (`reconcile_intake!` after every
job-terminal move), so `status` is a door-owned memoized derivation, not a second source of truth.
Law #3 intact; the guard stays DB-enforced. **R5 is now moved *and inverted*** ([[Risk ledger]]): a
second **job** per vehicle is legal and expected, a second **open intake** is the violation.

**Also settled:** the **jobsheet attaches to the Intake** (builder ruling — it's the car's intake
form, one per visit), and ETA follows it there; vocabulary locked before any code (**intake** = the
visit, **job** = a repair) to avoid a repeat of the `JobService`→`JobActions` confusion.

**One scare worth logging.** Mid-verification, Bash/`git status` stopped seeing this session's
edits — `git status` read "working tree clean" while the files were plainly there via the editor,
minutes after `bin/rails test` had run green against them. Stopped and flagged rather than running
any git recovery; it resolved itself on its own and all 40 files were intact. **Lesson: a
`git status` that contradicts work you just verified is a reason to pause, not to "fix" the tree.**

**Next:** codebase sweep, then commit. S4.5.8 (concept-note reconciliation) still open — and now
also covers [[Data model]] + [[Job visibility]], since the token moved to `Intake` and Sprint 7's
RLS design reads those.

---

## Sessions 1–28 — archived
Vault/Rails setup through Sprint 2.5, the job engine and blockers groundwork, the tenant-spine
collapse (ADR-010), Sprint 3 (blockers), and Sprint 4 (acknowledgement as stored visibility) —
Sessions 1–28. Moved to [[Worklog (Sessions 1-28)]] to keep this file focused on the Sprint 4.5
arc (Sessions 29–30).

