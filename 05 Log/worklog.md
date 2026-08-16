---
type: log
updated: 2026-08-16 (Session 31 — UI surface mapped screen-first, feature model named, routes made consistent; front door is next)
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

---

## 2026-08-16 · Session 31 — the UI surface mapped, the feature model named, routes made consistent

**Summary.** A mapping session, not a building one. Wrote [[Screen map]] (every screen, its
components/actions, role-gates verified against `Permissions` and the controllers' `before_action`
tables, each carrying a build status) — which immediately paid for itself by exposing **two dead
buttons that were 500ing their pages**: the S4.5 aggregate split deleted `new_job_path` and the
nested `customer_jobs` routes without updating their callers, so `/workshop` raised for counter
staff and `/customers/:id` raised for everyone who could reach it. Cleaned those up and folded them
into the S4.5 commit before merging. Then named the **feature model** ([[Features overview]], F1–F8)
and the **flow model** ([[Screen flow]], 11 flows across Setup → Records → Daily loop), explored the
whole thing in FigJam, concluded the vault should stay the source of truth, and finished with a
routes consistency pass: job create now nests under its intake (`ed2595c`), plus a reusable
route-orphan check. Merged S4.5 to `main` and pushed (`6e0434c`); deleted 11 stale branches and 3
leftover worktrees.

**The mapping caught what the tests couldn't.** The two 500s shipped silently because Session 30
deleted the whole `test/system` suite in the same commit that removed the routes — and the model/
service suite never renders a view, so it stayed green through a broken UI. Worth stating plainly:
**this suite's green bar says nothing about whether a page renders.** The system tests that would
have caught it were themselves dropped as "stale UI"; only `cold_start_intake_test.rb` was genuinely
tied to a reworking surface, while `job_lifecycle`/`blocker_lifecycle`/`crew_management`/
`blocker_catalog_admin` covered screens that are staying. A thin request-level smoke layer (does
each surviving page render for a counter user?) would fence off exactly this regression class — not
built, flagged here deliberately.

**Per-record time analytics are not reporting.** The builder corrected the first draft of the
feature model: time-in-stage and blocked-by-department are *this car's story*, so they belong on
**Intake show / Job show / the intake timeline** (F2), not in F8. F8 shrank to **aggregate numbers
across visits** — sums and averages — and *which* reports earn their place is still undefined.
Both read the same source: the [[Event log]] renders per-record as a timeline, aggregated as
reports. That also promoted the timeline from "infrastructure" to a named surface.

**The V1 fence, restated.** Asked what [[ADR-002 V1 scope]] actually means, and correcting a wrong
claim made earlier in the session: there is **no line running through F1–F8**. ADR-002 is a fence
around *all of Module 1* — **F1–F7 are all V1, the owner status page included** (ADR-002 lists it
explicitly). Only **F8 aggregate reporting** is the fuzzy edge. What sits *outside* the fence isn't
a slice of these features but whole other domains: parts/warehouse → V2, technician skills → V3,
money (pricing/quotes/invoicing) → deferred, with "awaiting customer approval" handled as a blocker
type.

**Vocabulary: "visit" is retired, it's an intake.** The models, controllers, and routes all say
`Intake`; "visit" was a translation layer sitting on top of the code and the source of the same
job/visit/intake tangle that ADR-012 already had to untie. Docs now say **intake**. ("Deliver"
stays — you deliver the *car*.)

**FigJam explored, then deliberately demoted.** Mapped features → screens → flows on a board, plus
a per-screen card grid in a `Screen name / (C) components / (A) actions` format with role-gates on
every action. Useful for *exploring*; wrong as a home. It's disconnected from the code (goes stale
the moment routes change, which is the exact drift [[Screen map]] exists to prevent), and the Figma
MCP bridge hit its Starter-plan cap mid-session, which settled the argument. **The vault is
canonical; any diagram is a disposable projection** regenerated on demand. Figma stays for pixel
mockups later, where it's genuinely the right tool.

**Routes: one real inconsistency fixed, one rename declined.** Job create was the odd child —
flat `POST /jobs` with `intake_id` smuggled through the request body while every sibling
(intake blockers, job blockers, job technician) takes its parent from the path. Now
`POST /intakes/:intake_id/jobs`; member verbs stay flat at `/jobs/:id/...`, since once a job exists
its own id addresses it, deeper nesting would push blocker routes to four levels, and the URL could
otherwise carry an `intake_id` that disagrees with the job's real parent. **The security boundary
here is the workshop scope, not the intake** — so nesting adds path noise, not safety.
*Declined:* renaming the blocker-type catalog from `/blockers` to `/blocker_types`. Tried, then
reverted on the builder's challenge — the nested controllers already disambiguate catalog from
applied, this codebase already accepts label≠route (the crew page is "Crew" at `/staff`), and
renaming the route without the `Blocker` model trades one asymmetry for another. Do the full model
rename or nothing. *(Also declined: Rails' `shallow: true`, which would express the same URLs more
concisely — rejected because it isn't obvious to read, and routes should be legible at a glance.)*

**A route-first check, and what it proved.** Every inventory we keep is screen-first, so none can
answer "does anything actually *trigger* this endpoint?" Added **`bin/route-orphans`**, and it
found exactly two orphans in the whole app: **`POST /intakes`** and **`POST /intakes/:intake_id/jobs`**.
Both creates are implemented, tested, green — and **nothing in the view layer can reach either**.
Every other endpoint has a caller. That's the create-path hole proven from the route side rather
than asserted, and it's now a standing per-sprint check (exits non-zero when orphans exist).

**Housekeeping.** Merged `s4.5-intake-job-aggregate` → `main` fast-forward and pushed
(`8fad8c9..6e0434c`). Deleted 11 local branches (8 fully merged; 3 pre-squash duplicates whose only
unique files were the orphan templates and system tests we'd deliberately removed) and 3 stale
worktrees under `.claude/worktrees/`. Routes audited clean in both directions — no controller action
without a route, no route without an action, no broken path helper in any view. 129 runs, 0 failures.

**Next session — the front door (S6).** The whole daily loop is built from *assign technician*
onward; what's missing is the way in. Three screens, in dependency order:
1. **New intake** (`GET /intakes/new`) — the plate-first entry: registration number → find-or-create
   vehicle → find-or-create customer → open the intake. The one create gateway, reached from both
   the Intake board and a Customer. This unblocks everything else.
2. **Add repair** — a form on Intake show posting to `intake_jobs_path`; today an open intake can
   never gain a second repair through the UI.
3. **Add vehicle** — currently a vehicle only comes into being as a side effect of typing an unknown
   plate during intake; it has no screen of its own.
Design questions still open going in: what the plate-miss branch does (customer-first vs
vehicle-first), whether the jobsheet ([[ADR-003 Digitized jobsheet in V1]]) is part of the intake
form or a later step, and whether Add vehicle is its own page or an inline affordance on Customer
show. Orientation for that session: [[Screen flow]] (flows 6–8), [[Screen map]] (the not-built
table), [[Features overview]] (F2, F6).

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

