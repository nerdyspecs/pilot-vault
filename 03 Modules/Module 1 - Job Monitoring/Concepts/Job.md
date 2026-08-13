---
type: concept
module: M1
updated: 2026-08-14 (re-pointed at the Intake/Job split — ADR-012, ADR-013)
---
# Job
One repair on a visit. No longer the centre of Module 1 by itself — see [[Intake]], the visit
that owns it. Tenant-scoped: `workshop_id` + `belongs_to :intake`. The vehicle/customer
triple-stamp and the owner status-page token that this note used to describe on `Job` **moved
to Intake** ([[ADR-012 Intake-Job two-level aggregate]]) — a `Job` reaches them as delegates
through its intake, since several repairs on one visit share one car and one bill-to.

## A Job always knows
- **Stage** — where it is in the work (exactly one). See [[Stage model]].
- **Active blockers** — what's pausing it, if anything (**can be several**). See [[Blocker]].
- **Crew** — which technician is **responsible** for it (single technician in v1; helpers +
  `lead` flag deferred). A membership asserts responsibility, not real-time presence.
  See [[Event log]].
- **Responsible owner** — who is accountable *right now*. Shifts to the resolver(s) of any active blocker while blocked.
- **History** — the immutable trail of stage changes, blockers, and crew. See [[Event log]].

## All changes go through ONE DOOR — per level
Stage transitions, blocker raise/resolve, and assignment are never scattered updates — they all
flow through `JobActions`, this level's door ([[Design laws]] #7). This is what makes the
[[Event log]] trustworthy: nothing can change state without also logging it. `JobActions` no
longer checks *who* may act, though — that moved to `Permissions`, checked at the controller
boundary before a door verb is ever called ([[ADR-013 The door decomposed]]). The door still
resolves the acting `WorkshopStaff` to stamp `created_by` on every event row.

## Two axes, not one
A job's situation answers two *independent* questions:
- *How far along?* → **stage** (progress axis)
- *What's holding it up, and whose court?* → **blocker** (overlay axis)

Keeping these separate is why a role like parts_advisor can act on a job (resolve a parts blocker)
without ever moving its stage.

## In Rails
- `belongs_to :workshop, :intake` — `vehicle`/`customer` are **delegates through the intake**,
  not owned here ([[Intake]])
- `Job.stage` — enum column (current phase); **`delivered` is not a Job stage** — it's the one
  stored Intake fact, so a Job's own terminals are `done` and `cancelled` only
- `Job has_many :job_stage_transitions` — stage events + ack (see [[Event log]])
- `Job has_many :job_blockers` (+ their `job_blocker_transitions`) — blocker items + events
  (Sprint 3); **active blockers** = items with no resolve (a query, can be several)
- `Job has_many :job_technicians` — the crew **right now** (membership rows, deleted on
  remove — Design B 2026-07-17); no `lead` flag in v1 ([[Deferred design]])
- `Job has_many :job_technician_transitions` — joined/left **history**, a direct
  association (events are self-contained; no longer reached `:through` membership)
- `Job#timeline` — the event tables merged by timestamp (see [[Event log]])
- **Immutable once Done** — answers/history frozen; corrections open a new job ([[Design laws]] #8)
- **No price fields** in V1 — see [[ADR-002 V1 scope]]
- `lock_version` (optimistic locking) — considered, **backlogged**: see [[Deferred design]]

## Related
- [[Overview]] · [[Intake]] · [[Stage model]] · [[Blocker]] · [[Event log]] · [[Data model]] ·
  [[Design laws]] · [[ADR-012 Intake-Job two-level aggregate]] · [[ADR-013 The door decomposed]]
