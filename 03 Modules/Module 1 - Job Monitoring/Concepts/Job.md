---
type: concept
module: M1
updated: 2026-07-17 (Design B crew names)
---
# Job
A single repair/service instance for one vehicle. The centre of Module 1.
Tenant-scoped, **triple-stamped**: `workshop_id` (whose board) + `vehicle_id` (whose history)
+ `customer_id` (whose visit — frozen at registration, [[Data model]] §Resolved; found stale
as "double-stamped" 2026-07-16 and corrected — the customer stamp landed 2026-07-14/15).

## A Job always knows
- **Stage** — where it is in the work (exactly one). See [[Stage model]].
- **Active blockers** — what's pausing it, if anything (**can be several**). See [[Blocker]].
- **Crew** — which technician is **responsible** for it (single technician in v1; helpers +
  `lead` flag deferred). A membership asserts responsibility, not real-time presence.
  See [[Event log]].
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
- `belongs_to :workshop, :vehicle, :customer` — the triple-stamp (customer frozen at registration)
- `has_secure_token :token` — the owner status-page link
- `Job.stage` — enum column (current phase)
- `Job has_many :job_stage_transitions` — stage events + ack (see [[Event log]])
- `Job has_many :job_blockers` (+ their `job_blocker_transitions`) — blocker items + events
  (Sprint 3); **active blockers** = items with no resolve (a query, can be several)
- `Job has_many :job_technicians` — the crew **right now** (membership rows, deleted on
  remove — Design B 2026-07-17); no `lead` flag in v1 ([[Deferred design]])
- `Job has_many :job_technician_transitions` — joined/left **history**, a direct
  association (events are self-contained; no longer reached `:through` membership)
- `Job#timeline` — the event tables merged by timestamp (see [[Event log]])
- `Job has_many :job_sheet_field_values` — this car's jobsheet answers (see [[Data model]])
- **Immutable once Done** — answers/history frozen; corrections open a new job ([[Design laws]] #8)
- **No price fields** in V1 — see [[ADR-002 V1 scope]]
- `lock_version` (optimistic locking) — considered, **backlogged**: see [[Deferred design]]

## Related
- [[Overview]] · [[Stage model]] · [[Blocker]] · [[Event log]] · [[Data model]] · [[Design laws]]
