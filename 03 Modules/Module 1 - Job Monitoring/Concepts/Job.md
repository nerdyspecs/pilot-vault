---
type: concept
module: M1
updated: 2026-07-04
---
# Job
A single repair/service instance for one vehicle. The centre of Module 1.
Tenant-scoped, **double-stamped**: `workshop_id` (whose board) + `vehicle_id` (whose history).

## A Job always knows
- **Stage** — where it is in the work (exactly one). See [[Stage model]].
- **Active blockers** — what's pausing it, if anything (**can be several**). See [[Blocker]].
- **Crew** — which mechanic(s) are on it (one primary + optional helpers). See [[Event log]].
- **Responsible owner** — who is accountable *right now*. Shifts to the resolver(s) of any active blocker while blocked.
- **History** — the immutable trail of stage changes, blockers, and crew. See [[Event log]].

## All changes go through ONE DOOR
Stage transitions, blocker raise/resolve, and assignment are never scattered updates — they all
flow through a single service object ([[Design laws]] #7). This is what makes the [[Event log]]
trustworthy: nothing can change state without also logging it.

## Two axes, not one
A job's situation answers two *independent* questions:
- *How far along?* → **stage** (progress axis)
- *What's holding it up, and whose court?* → **blocker** (overlay axis)

Keeping these separate is why a role like parts_advisor can act on a job (resolve a parts blocker)
without ever moving its stage.

## In Rails
- `belongs_to :workshop, :vehicle` — the double-stamp
- `has_secure_token :token` — the owner status-page link
- `Job.stage` — enum column (current phase)
- `Job has_many :job_stage_transitions` — stage history + ack (see [[Event log]])
- `Job has_many :job_blocker_transitions` — blocker events + ack; **active blockers** = raises with
  no matching resolve (a query, can be several)
- `Job has_many :job_mechanics` — crew; current = `removed_at IS NULL`, primary flagged
- `Job has_many :job_sheet_field_values` — this car's jobsheet answers (see [[Data model]])
- **Immutable once Done** — answers/history frozen; corrections open a new job ([[Design laws]] #8)
- **No price fields** in V1 — see [[ADR-002 V1 scope]]
- `lock_version` (optimistic locking) — considered, **backlogged**: see [[Deferred design]]

## Related
- [[Overview]] · [[Stage model]] · [[Blocker]] · [[Event log]] · [[Data model]] · [[Design laws]]
