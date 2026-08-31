---
id: ADR-012
type: decision
status: accepted
date: 2026-08-03
extends: ADR-011 (receivers stay stored at write time — now across two levels)
supersedes:
superseded_by:
---
# ADR-012 — Intake/Job two-level aggregate

> [!note] Sprint numbering — this ADR predates the Sprint 5 ↔ 6 swap *(2026-08-17)*
> Where the body says **Sprint 5 / S5** (the live board) read **Sprint 6**; where it says **Sprint 6
> / S6.1–S6.3** (the intake + jobsheet vertical) read **Sprint 5**. The two were renumbered so the
> intake/jobsheet vertical runs first. The body keeps its original numbers — ADRs aren't edited.
> See [[Sprint plan]].

From the 2026-08-03 routing/screen-map session. Today's `Job` row carries five things at once — a
car, a visit, one repair, one stage, one technician. A real car arrives for **several repairs**,
worked by **several technicians in parallel**, each with **its own blockers** — which one-job-per-visit
cannot represent. This ADR splits `Job` into a **two-level aggregate**: an **Intake** (the car's
visit) owns many **Jobs** (the repairs). It is done as a design pass + schema squash **before Sprint
5**, because the live board (Sprint 5) is built directly on this aggregate and there is **no
production data** — the cheapest this migration will ever be, and the S5 screens would otherwise be
built on the wrong unit and rebuilt.

**Vocabulary, locked before any code** (to avoid a repeat of the `JobService`→`JobActions`
confusion): **"intake" = the car's visit** (one drop-off, one collection); **"job" = one repair** on
that visit. "Visit" and "service" are avoided in code — *service* is the repair itself in shop
language, *job* stays the repair. The single service object stays **`JobActions`** (ONE DOOR,
[[Architecture laws]] #7); intake verbs are added there rather than a parallel `IntakeActions`.

## The lighter alternatives, ruled out first
The split carries the weight of a new aggregate, so it must beat the cheaper options — recorded so
they are not re-proposed:

- **The Sprint-6 jobsheet checklist ([[ADR-003 Digitized jobsheet in V1]]).** A flat, owner-configured
  list of fields filled at intake. It can record *what* a visit needs, but has **no per-item stage,
  crew, blocker, or acknowledgement**. A car in for a brake job (waiting on parts) + an aircon regas
  (done) collapses into one stage / one blocker set / one technician, and the board reads the whole
  car as blocked or in-progress — the truth is lost. ✗
- **Many bare `Job`s per vehicle, no `Intake`.** Drop the busy-vehicle guard and let a car own several
  active jobs. This *does* give independent stage/crew/blockers per repair, but loses the **visit** as
  a first-class thing: **delivery** smears across N jobs (the customer drops off and collects *once*);
  **hold-for-payment** has no honest home (payment is per-car, not per-repair); and the **board's unit**
  (S5.2a: "the car, its stage, how long it's sat") has nothing to group by. ✗

**Load-bearing premise (recorded so it isn't re-derived):** the full split earns its keep over the
checklist *specifically because repairs need independent technicians **and** independent blockers* —
the builder's assertion about the target workshop. If a real shop turns out to run one-tech-plus-a-
checklist, the jobsheet was the cheaper answer. The premise is stated, not assumed.

## Decision

### 1. Customer → Vehicle → Intake → Job
A new model **`Intake`** sits between `Vehicle` and `Job`. One `Intake` is one car's visit; it
`has_many :jobs`. A `Job` is one repair and **keeps its own** stage machine, `job_technicians`
(crew), and work blockers. The triple-stamp migrates: `Intake belongs_to :workshop, :vehicle,
:customer` (the customer frozen at intake, per [[Data model]]); a `Job` reaches customer/vehicle
through its intake.

### 2. The intake's terminal position is stored; `ready` is derived ([[Architecture laws]] #3)
An Intake carries a **stored `status` enum `{ open, delivered, cancelled }`** — exactly parallel to
how a `Job` stores `stage`. **`ready` is the one derived reading**, a query over its jobs:

| Intake state | How it's known |
|---|---|
| **open** | stored `status` — work in progress |
| **ready** | **derived**: still `open`, every job terminal, ≥1 `done` — the car can be handed over |
| **delivered** | stored `status` — the collection happened (`deliver!`) |
| **cancelled** | stored `status` — reconciled from its jobs when all are terminal with 0 done |

`ready` is **never a column**: "is every repair finished?" is a genuine question about the children,
answered fresh at read time. The terminal *position*, by contrast, is stored — and the door
(`JobActions`) is its only writer, reconciling it after every job-terminal move
(`reconcile_intake!`). That makes `status` a **door-owned memoized derivation**, not a second source
of truth: nothing outside ONE DOOR can move it, so it cannot drift from the jobs it summarizes.

> [!note] Refined during implementation (2026-08-03, same session)
> This section originally stored **only `delivered_at`** and derived `cancelled` from the jobs. That
> broke on the **busy-vehicle guard**: the R5 invariant is enforced by a Postgres *partial unique
> index*, which can only key on a stored column of its own table. With `cancelled` derived, an
> `intakes` index could not tell a live-open visit from a dead-cancelled one, so a fully-cancelled
> car that went home would wrongly block its vehicle's next visit — or the guarantee would have to
> leave the database entirely, against the ethos ADR-007 (RLS) and ADR-010 (composite FKs) are built
> on. Storing the terminal position keeps the guard DB-enforced
> (`index_intakes_one_open_per_vehicle WHERE status = 0`) at no cost to law #3, since the genuinely
> derived question (`ready?`) stays derived. Recorded rather than silently rewritten — the original
> cut was a real position, and the index is why it moved.

### 3. Stage re-split; `deliver!` moves to the Intake
- **Job stages** become `{ unassigned, assigned, in_progress, done, cancelled }` — old `registered`
  → **`unassigned`** (a repair is born unassigned); **`delivered` leaves `Job` entirely**. A Job's
  terminals are `done` and `cancelled`.
- **Intake** stores `open → delivered`; `ready` / `cancelled` are derived (§2).
- **`deliver!` moves from a Job verb to an Intake verb.** `register_job!` becomes: **find-or-create the
  open Intake** for the vehicle, then create a Job under it. The busy-vehicle guard flips from "one
  active **job** per vehicle" to **"one open **intake** per vehicle."**
- **`registered_by`** (the intake SA) moves from the Job's birth row to the **Intake**.

### 4. Every backward move lives on Job; the Intake only steps forward
`send_back!`, `cancel!`, and `remove_technician!` are all **Job** moves. The **Intake only ever steps
`open → delivered`** — one forward step to a boundary, never backward. A returning delivered car is a
**new Intake** with new Jobs ([[Architecture laws]] #6/#8, restated below). **"Defer to next week" is not a
hanging job** — it is `cancel!` the unfinished job now + rebook it as a **new job on a new intake**
next week. This keeps *"delivered ⟺ every job terminal, ≥1 done"* a hard rule and never leaves a
zombie job open across visits.

### 5. Handoff receivers stay **stored at write time** — across two levels ([[ADR-011 Acknowledgement as stored visibility]])
The Intake's *status* being derived does **not** make any handoff's *receiver* derived. Status is a
read over jobs; a receiver is an **intent stamped on an event row the instant the verb fires** — the
whole append-only fix of ADR-011. Two consequences of the two-level split:

- **`mark_done!` still pins the registering SA**, now read from the intake: it stamps
  `intake.registered_by` **onto the job's `done` event at write time** (stored, never looked up at
  read time). This is per-repair done accountability — *which* repair finished, and the counter
  person who owns this car.
- **The closing action moved, so `deliver!` must sweep.** In the single-job app the loop closed
  because `deliver!` was a *job* verb whose first line acknowledged the job. With `deliver!` now an
  *intake* verb, no job-level verb ever runs on a `done` job again — so a `mark_done!` pin would have
  **no closing action** and hang for the whole *awaiting-delivery* window. The fix: **intake
  `deliver!` runs `acknowledge_pending!` across every one of its jobs** (`by:` the delivering SA),
  under the lock. Delivering the car *is* the SA saying "I've dealt with this finished work," so it
  acknowledges every done-notice on the car in one motion. It clears only the flags **waiting on the
  person delivering** — a technician's still-open flag stays honestly open (stamping it would record
  an acknowledgement that never happened — the append-only lie ADR-011 forbids).

Every other receiver is pure job-level and unchanged (`joined`/`left` → technician; `send_back!` →
current technician; blocker `resolved` → raiser). **No row of ADR-011's receiver table is superseded**
— `mark_done!` keeps `registered_by`; only *where it's read from* (the intake) and *what closes it*
(intake `deliver!`) changed.

### 6. Blockers split by what they block; intake blockers are **not acknowledgeable**
`Blocker.blocks` becomes the **single discriminator** for both storage and guard:

- **`blocks: done`** (work blockers — Subcon, Parts, Technical) → a **`JobBlocker`** item, vetoed in
  **`Job#transition!`** exactly as today.
- **`blocks: delivered`** (Hold for payment) → an **`IntakeBlocker`** item, vetoed in **Intake
  `deliver!`**. HFP is a per-*visit* hold (you pay for the whole car); hanging it on an arbitrary one
  of the car's repairs would be a lie.

One shared **`Blocker` catalog** — no "level" column, because `blocks` already implies the table and
the door. The intake blocker gets the **full three-record shape** — `Blocker` (catalog) +
`IntakeBlocker` (item) + `IntakeBlockerTransition` (events) — mirroring the job blocker so *"a blocker
is three records, everywhere"* stays true and the **note chain** (payment back-and-forth) has a home.
**Parallel mirror tables, not polymorphic:** a polymorphic `blockable` would unify the tables but
drops the parent to an app-only check, forfeiting the DB-enforced integrity ADR-010 was built to buy.

**But `IntakeBlockerTransition` is NOT acknowledgeable** — it carries **no `receiver_id` /
`acknowledged_at`** and does **not** include `Acknowledgeable`. Acknowledgement belongs to a
*direction* ([[Event log]]), and an intake blocker has none: HFP is the counter holding *its own* car
until paid — SA raises, SA clears, nobody is ever waiting on anyone else. Including inert ack columns
that are NULL on every intake row forever would be cosmetic symmetry that misleads a reader into
expecting a handoff that can't occur. So the honest mirror is: **both levels are catalog + item +
events with a note chain; only Job blockers are acknowledgeable, because only they have cross-party
direction.** (`IntakeBlockerTransition` keeps the composite `(created_by, workshop_id)` FK — that's
actor integrity, ADR-010, unrelated to acks.) The `acknowledge_pending!` sweep and the "waiting on
whom" board query stay **purely job-level** and never touch intake blockers.

## The four open sub-decisions, settled
1. **All-jobs-cancelled car that still leaves → `cancelled`.** No `deliver!` is called; the door's
   `reconcile_intake!` moves `status` to `cancelled` once every job is terminal with none done, and
   the intake drops off the active board. `status: delivered` is therefore reachable **only when ≥1
   job is done** — it means strictly "a car with real work done was collected." The physical
   hand-back of an unrepaired car needs no delivery act; the cancellations already say "nothing
   happened here."
2. **Cancel-the-car cascade = cancel the remaining *open* jobs; done work survives** ([[Architecture laws]]
   #8). The verb does **not** choose the terminal — it falls out of the jobs, via the same
   reconcile: 0 done → `cancelled`; ≥1 done → the intake stays `open` and reads **`ready`** (the
   customer still collects the completed repairs, via an explicit `deliver!`). The confirm UX must therefore
   **name consequences on both sides** — *"cancels 2 unfinished repairs (Brake, Aircon); 1 completed
   repair (Battery) stays and the car can still be handed over."* The dialog itself is a view-layer
   concern built with the board/intake UI (S5/S6), **not** in this migration sprint; only the content
   requirement is recorded here.
3. **`mark_done!` receiver = the registering SA** (`intake.registered_by`), stored at write time;
   **intake `deliver!` acknowledges across all the intake's jobs** so the pins actually close (§5).
4. **HFP lives on the Intake** as a non-acknowledgeable three-record blocker (§6).

## Why
- **The Intake is the visit made first-class.** It is the honest anchor for the three things bare
  many-jobs loses: **collection** (one delivery act), the **release-hold** (HFP), and the **board's
  grouping unit** (the car). The whiteboard's row was always the car; `Intake` is that row.
- **`ready?` obeys [[Architecture laws]] #3** without weakening acknowledgement: what's *derived* is a
  read (the car's readiness), what's *stored* is an intent (each event's receiver, and the terminal
  the door writes). Keeping those two separate is exactly what ADR-011 established; this ADR carries
  it across a second level.
- **`blocks` was already the discriminator within one model** (which forward stage a type vetoes);
  the split just lets it also pick the *table* and the *door*. One catalog, one rule, two homes.

## Consequences
- **New model `Intake`** (`workshop_id, vehicle_id, customer_id, status, token`) +
  **`IntakeStatusTransition`** for its forward steps (append-only, `created_by`, composite FK) —
  named *status*, not *stage*: an intake has a **status**, a job has a **stage**, per the vocabulary
  lock above. New **`IntakeBlocker`** + **`IntakeBlockerTransition`** (non-acknowledgeable). `Job`
  loses `delivered` from its enum and gains `belongs_to :intake`; `registered_by`/`token` move to
  `Intake`, and `vehicle`/`customer` become delegates through it.
- **`JobActions` grows intake verbs** and keeps ONE DOOR: `register_job!` opens-or-finds the intake;
  a new **`deliver!`** and **`cancel_intake!`** (the cascade) act on the intake; `deliver!` sweeps its
  jobs' acknowledgements. The blocker veto stays in `Job#transition!` for `done`-guards; a new
  delivered-guard check lives in intake `deliver!`.
- **`Job.active`** stays `unassigned/assigned/in_progress`; the board groups by **Intake** and shows
  a *Done — awaiting delivery* group for intakes deriving `ready` (the founding-pain surface, now
  car-level).
- **The R5 busy-vehicle guard moves table, and its meaning inverts** ([[Risk ledger]] R5).
  `index_jobs_one_active_per_vehicle` (`jobs(vehicle_id) WHERE stage IN (0,1,2)`) is **replaced** by
  `index_intakes_one_open_per_vehicle` (`intakes(vehicle_id) WHERE status = 0`). The ruling itself
  survives — a vehicle cannot be in two *visits* at once — but what it refuses changed: a second
  **job** on a vehicle is now legal and expected (that is the split's entire point, parallel
  repairs), while a second **open intake** is the violation. `register_job!` therefore
  *opens-or-finds* rather than refusing.
- **Migration = schema squash + reseed**, not a data migration — same play as
  [[ADR-010 WorkshopStaff supersedes the edge split]], justified by no production data. The
  `git diff structure.sql` drift check on unchanged tables is the safety net.
- **`Blocker.blocks` / `FORWARD_STAGES` / the DB `CHECK`** widen to keep `delivered` valid now that it
  is an intake boundary rather than a Job stage.
- **The jobsheet attaches to the Intake, not the Job** *(builder ruling 2026-08-03)*. It is the
  **car's intake form** — customer complaints + vehicle condition, one per visit — so
  `JobSheetFieldValue` keys on **`intake_id`, not `job_id`**. Per-repair notes live on the Job.
  Resolves a downstream question for [[ADR-003 Digitized jobsheet in V1]] / Sprint 6 (touches
  S6.1–S6.3); per-repair jobsheet fields would be a v2 additive if ever needed. [[Data model]] and
  [[Job]] (which today read "this car's jobsheet answers" on the Job) fold into the S4.5.8 concept-note
  reconciliation.

## Stated limits
*(Recorded so they are not rediscovered as surprises.)*
- **Premature done-pins.** While a car is still `open` (one repair done, another in progress), the
  done repair shows *"Waiting on \<SA\>"* before the car is deliverable. Truthful per-repair signal,
  the same "done thing the counter should notice" ADR-011 already embraces — not engineered away.
- **Intake blockers ship non-acknowledgeable.** The S3.6 catalog admin *could* let a workshop
  configure a **cross-role** `blocks: delivered` type (e.g. "manager sign-off before release",
  SA-raised/manager-cleared) — that *would* be a directional intake handoff. In v1 all intake blockers
  are same-role holds, so its resolve would not surface as "waiting on" anyone until the ack pair is
  added. **Purely additive** (two columns + include `Acknowledgeable`; no prod data). Decided, not
  discovered.
- **`registered_by` remains the weakest receiver** (ADR-011's own note): a done-notice for an
  off-shift SA sits on the board while the car reads ready. Unchanged by the split — fine at 1–4 SAs.

## Rejected alternatives
*(Do not re-propose without new information — see [[Rejected alternatives]].)*
- **The jobsheet checklist** and **bare many-jobs-per-vehicle** — ruled out above (no per-item work
  tracking; no first-class visit, respectively).
- **Polymorphic `blockable` (Job | Intake).** Would literally unify the blocker tables, but the
  parent association can no longer be a DB-level FK, dropping "this item's parent is in this workshop"
  from a foreign-key violation to an app check — against the DB-enforced-integrity ethos ADR-010 was
  built for. Parallel mirror tables keep that guarantee and still give one mental model.
- **Intake blockers carrying the ack pair "for symmetry."** Cosmetic symmetry over an inert core —
  columns NULL on every intake row forever, a sweep iterating a leg that never contributes, tests
  asserting "intake ack does nothing." Honest asymmetry (only directional blockers are acknowledgeable)
  beats identical-but-dead columns.
- **`mark_done!` receiver → nobody, signal via derived `ready` only.** Considered (it would retire
  ADR-011's weakest receiver). Dropped: per-repair done accountability is worth keeping, and the
  `deliver!`-sweeps-jobs mechanism closes the handoff cleanly, so the pin is not left dangling.

## Related
- [[ADR-011 Acknowledgement as stored visibility]] (extended by this) ·
  [[ADR-010 WorkshopStaff supersedes the edge split]] · [[ADR-002 V1 scope]] ·
  [[ADR-003 Digitized jobsheet in V1]] · [[Architecture laws]] · [[Stage model]] · [[Job]] ·
  [[Blocker]] · [[Event log]] · [[M1-F1 Status flow and transitions]] · [[Intake flow]] ·
  [[Sprint plan]] · [[Deferred decisions]]
