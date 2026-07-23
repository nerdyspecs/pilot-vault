---
type: concept
module: M1
updated: 2026-07-24 (Sprint 3 built — three records, blocks stage-guard, Design B)
---
# Blocker
The **overlay axis** — something pausing a job. The job keeps its stage; the blocker sits on top.

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

## The `blocks` stage-guard — the resolved HFP collision
Each catalog type names, in `blocks`, the **one forward stage it vetoes**. The 2026-07-16 ruling
"open blockers stop `→ done`" is refined to **"stop entry into the guarded stage,"** of which
`→ done` is one case:

- Work blockers (Subcon, Parts, Technical) guard **`done`** — an unfinished job can't be called done.
- **Hold for payment guards `delivered`**, not `done` — the car *is* done, you just can't hand it
  over until paid. This is the release hold, and it's what resolved the old collision (HFP as a
  `done`-guard would have blocked `done` forever).

`blocks` is constrained to the three forward stages (`in_progress` / `done` / `delivered`) by a DB
`CHECK` + model validation, so the veto — which lives **once**, in the door's `transition!` — can
never bite `send_back!` / `cancel!` / `assign` / `remove`. A blocked job therefore stays cancellable.
"Currently blocked by" = items with no `resolved` event — a **query, not a column** ([[Design laws]]
#3); `noted` events are ignored. Rows are never deleted; the rows ARE the history.

## Design B — the long-lived thread *(2026-07-17, built Sprint 3)*
One item carries a **whole incident**: `raised` once, `resolved` at most once, and any number of
**`noted`** events in between. A subcon that fails then a second subcon that works is **one item**,
the A→B bounce recorded as notes — you never resolve-then-reopen (that would falsely claim it was
solved). **New challenge = new item; same challenge bouncing = notes.** `noted` is state-neutral (no
ack, invisible to `active_blockers`) and **ships in v1** as a first-class action — verb + mobile UI
(this is **B1**; see the B1/B2 split under Acknowledgement).

## Permission (built)
Checked at the door against the catalog, and **crew-aware**: a `technician`-side check requires the
actor be on *this job's* crew (mirrors `start_work!`/`mark_done!`), not merely hold the technician
role somewhere; any other role means holding that role; `manager`/`owner` always override. Raising
also **refuses when the job has already reached the guarded stage** — no raising a `done`-guard
blocker on an already-done car (the veto only stops *entering* a stage, never pulls a job back).

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

## Acknowledgement — dormant until Sprint 4
Raising a blocker is a **handoff**. Receivers are derived, not stored: **raise → the catalog's
`cleared_by_role`**; **resolve → the item's raiser** (`created_by` on its `raised` event — the echo
back, doubling as a verification receipt in the subcon flow). Same-role blockers (Hold for payment,
SA-raised SA-cleared) generate **zero ack traffic** by design. The `acknowledged_at` /
`acknowledged_by` pair already exists on every event row (incl. notes) but stays NULL in v1 — lit in
Sprint 4 ([[ADR-005 Acknowledged handoffs in V1]]).

**B1 / B2 split.** B1 (built) is the long-lived item + the neutral note chain. **B2 (deferred to
Sprint 4)** is routing an *individual* note to someone's inbox (a parts advisor's "arrived, please
verify" landing in the tech's "waiting on me") — the note's receiver can flip mid-thread. No S3
rework: if directed notes need a receiver it's one nullable `directed_to_role` added additively, old
notes reading as neutral.

## Related
- [[Job]] · [[Stage model]] · [[Event log]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-010 WorkshopStaff supersedes the edge split]]
