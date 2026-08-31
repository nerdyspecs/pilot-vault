---
type: concept
module: M1
updated: 2026-08-14 (split across both levels — ADR-012)
---
# Blocker
The **overlay axis** — something pausing a job, or the whole visit. The job/intake keeps its
stage/status; the blocker sits on top.

> [!warning] Split across two levels 2026-08-14 by [[ADR-012 Intake-Job two-level aggregate]]
> Everything below was written when a blocker only ever sat on `Job`. Since ADR-012, `blocks`
> is the **single discriminator** for which of two tables a type's items land in, not just
> which stage it guards: `blocks: done` → `JobBlocker` (unchanged, this page); `blocks:
> delivered` → **`IntakeBlocker`**, a visit-level hold vetoing intake `deliver!` instead of
> `Job#transition!`. One shared catalog, no "level" column — `blocks` already implies both the
> table and the door. See §The `blocks` stage-guard and §Acknowledgement below for what carries
> over and what's new.

## Definition (strict)
A job is blocked only when **both**: (1) work can't progress, and (2) clearing it needs a
**specific other role** to act. Self-resolving waits (paint drying) are *not* blockers — that
keeps "blocked" from becoming a junk drawer.

## Two axes
Stage = workflow position; blocker = *why* it's stuck. **Waiting states are blockers, never
stages** — this is settled.

## Co-responsibility (attribution)
While blocked, responsibility shifts to each active blocker's resolver, but the job's own owner
stays co-responsible. A job can hold several blockers at once, each attributed to its own resolver.
The point is measuring *which role* cost the time.

## Three records: catalog + item + events *(built Sprint 3, 2026-07-24)*
A blocker is **three** records, not two — the tracker restructure (Session 17) split the item from
its events, mirroring `Job` + `JobStageTransition` but with an extra layer because a job has *many
independent* blockers where it has only one stage line.

- **`Blocker`** (catalog) — `belongs_to :workshop`; `label, raised_by_role, cleared_by_role,
  blocks`. The workshop's own list of blocker types, **and** who may raise/clear each — permission
  lives on the catalog, not hardcoded ([[M1-F1 Status flow and transitions]]).
  `workshop_manager`/`owner` always override. Never a stateful row.
- **`JobBlocker`** (item) — `workshop_id, job_id, blocker_id`. **One independent to-do** carrying a
  single incident ("wiring check @ Ah Seng"). Written once, never deleted (its events FK to it
  directly). **Item identity = the row id** — two subcons on one car = **two items**, each with its
  own cycle; recurrence is a *new item*, never a re-raise. This replaced any "no raising the same
  blocker twice" rule.
- **`JobBlockerTransition`** (events) — `workshop_id, job_id, job_blocker_id, action
  (raised | resolved | noted), note, created_by, acknowledged_at, acknowledged_by`. Reaches the
  catalog via `job_blocker.blocker` (no direct `blocker_id`). `created_by`/`acknowledged_by` point
  at **`WorkshopStaff`** with a composite `(actor_id, workshop_id)` FK (ADR-010); the ack pair is
  dormant until Sprint 4.

**The intake level mirrors this exactly in shape** — `IntakeBlocker` (item) +
`IntakeBlockerTransition` (events), same catalog, same three-record idea — with one structural
difference: `IntakeBlockerTransition` carries **no `acknowledged_at` / `receiver_id` at all**
(not `Acknowledgeable`, columns don't exist). See §Acknowledgement below for why.

## The `blocks` stage-guard — the resolved HFP collision
Each catalog type names, in `blocks`, the **one forward stage it vetoes**. The 2026-07-16 ruling
"open blockers stop `→ done`" is refined to **"stop entry into the guarded stage,"** of which
`→ done` is one case:

- Work blockers (Subcon, Parts, Technical) guard **`done`** — an unfinished job can't be called done.
- **Hold for payment guards `delivered`**, not `done` — the car *is* done, you just can't hand it
  over until paid. This is the release hold, and it's what resolved the old collision (HFP as a
  `done`-guard would have blocked `done` forever). *(2026-08-14, ADR-012: this is no longer just a
  differently-guarded `JobBlocker` — `delivered` isn't a Job stage anymore, so a `blocks: delivered`
  type is a wholly different table, `IntakeBlocker`, vetoed in intake `deliver!`. Per-car, not
  per-repair — HFP hanging off one of a car's several repairs would be a lie.)*

`blocks` is constrained to the three forward stages (`in_progress` / `done` / `delivered`) by a DB
`CHECK` + model validation, so the veto — which lives **once per level**, in `Job#transition!` for
work blockers and intake `deliver!` for HFP — can never bite `send_back!` / `cancel!` / `assign` /
`remove`. A blocked job therefore stays cancellable. "Currently blocked by" = items with no
`resolved` event — a **query, not a column** ([[Architecture laws]] #3); `noted` events are ignored. Rows
are never deleted; the rows ARE the history.

## Design B — the long-lived thread *(2026-07-17, built Sprint 3)*
One item carries a **whole incident**: `raised` once, `resolved` at most once, and any number of
**`noted`** events in between. A subcon that fails then a second subcon that works is **one item**,
the A→B bounce recorded as notes — you never resolve-then-reopen (that would falsely claim it was
solved). **New challenge = new item; same challenge bouncing = notes.** `noted` is state-neutral (no
ack, invisible to `active_blockers`) and **ships in v1** as a first-class action — verb + mobile UI
(this is **B1**; see the B1/B2 split under Acknowledgement).

## Permission (built)
Checked against the catalog by `Permissions`, at the controller boundary
([[ADR-013 The door decomposed]]), and **crew-aware for job blockers**: a `technician`-side check
requires the actor be on *this job's* crew (mirrors `start_work!`/`mark_done!`), not merely hold the
technician role somewhere; any other role means holding that role; `manager`/`owner` always
override. Raising also **refuses when the job has already reached the guarded stage** — no raising a
`done`-guard blocker on an already-done car (the veto only stops *entering* a stage, never pulls a
job back). **Intake blockers are role-only** — there's no crew concept at the visit level, so the
`technician`-crew special case doesn't apply; today's only intake type (HFP) is same-role anyway.

## The seed catalog (four defaults)
Planted at `Workshop.create_with_owner!` (a product default, not demo data), editable via the S3.6
catalog admin:

| label | raised_by | cleared_by | blocks |
|---|---|---|---|
| Subcon work | technician | service_advisor | done |
| Waiting for parts | technician | parts_advisor | done |
| Technical challenge (escalate) | technician | workshop_manager | done |
| Hold for payment | service_advisor | service_advisor | delivered |

A subcon blocker is `raised_by: technician, cleared_by: service_advisor` on purpose —
tech-cleared would make the raise self-caused and the SA would never hear of it.

## Acknowledgement — stored receiver *(built, Sprint 4)*
**Resolving** a blocker is a handoff; **raising** one is not. Receivers are **stored** at write time
(ADR-011), not derived: **`raised` → nobody** (a raised blocker is already visible, and
`cleared_by_role` routes who may clear it — pinning a person would only manufacture a to-do);
**`resolved` → the item's raiser** (`created_by` on its `raised` event, via `JobBlocker#raised_by`).
Same-role blockers (Hold for payment, SA-raised SA-cleared) generate **zero ack traffic** — the
resolver already holds the raise side, so the sweep clears it in the same breath. The `receiver_id` /
`acknowledged_at` pair rides every event row; a `noted` event pins nobody and stays NULL
([[ADR-005 Acknowledged handoffs in V1]], [[ADR-011 Acknowledgement as stored visibility]]).

**B1 / B2 split.** B1 (built) is the long-lived item + the neutral note chain. **B2** is stamping a
`raised` or `noted` event with a **stored receiver pinned to a person** (a parts advisor's "arrived,
please verify" showing as *"waiting on"* that technician on the board) — the receiver can flip
mid-thread, which is why Sprint 4 pins **nobody** on `raised`. No rework when it lands: the
`receiver_id` column already exists; the verbs just start stamping it, old rows reading as neutral.
*(**B2 deferred again, past Sprint 4** — ADR-011. Still additive, so nothing is lost by waiting; the
base acknowledgement loop should meet a real workshop before a second traffic generator is added.
Now parked in [[Deferred decisions]] rather than scheduled.)*

**The resolve echo sweeps like everything else** *(reshaped 2026-07-28, [[ADR-011 Acknowledgement as stored visibility]])*.
A `resolved` event pins the **raiser** as receiver; the raiser acting on the job acknowledges it. The
2026-07-24 draft carved this out ("tapped, never inferred", because a resolve doubles as verification
— "I checked the part arrived") — but the reshape has **no confirm button**, so exempting it would
leave it permanently unacknowledgeable. It sweeps like every other handoff; the verification content
lives in the `note` column, which carries "I checked the part arrived" better than a tap ever did.

**Intake blockers carry no acknowledgement at all** *(2026-08-14, ADR-012)* — not "raised → nobody"
like a job blocker (which still *could* pin someone, and chooses not to); `IntakeBlockerTransition`
has no `receiver_id`/`acknowledged_at` columns, full stop. HFP is the counter holding its own car
until paid — SA raises, SA clears, nobody is ever waiting on someone else. Carrying inert ack columns
that are NULL on every row forever would be cosmetic symmetry, not a real mirror; considered and
rejected in ADR-012's Rejected alternatives. **Directed intake holds are a stated future limit**: a
workshop *could* configure a cross-role `blocks: delivered` type (e.g. "manager sign-off before
release") via the S3.6 catalog admin, which would be a genuine directional handoff this schema can't
surface as "waiting on" anyone yet — purely additive if it's ever needed (two columns + `include
Acknowledgeable`, no prod data cost).

## Related
- [[Job]] · [[Intake]] · [[Stage model]] · [[Event log]] · [[Architecture laws]] ·
  [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-010 WorkshopStaff supersedes the edge split]] ·
  [[ADR-011 Acknowledgement as stored visibility]] · [[ADR-012 Intake-Job two-level aggregate]] ·
  [[ADR-013 The door decomposed]]
