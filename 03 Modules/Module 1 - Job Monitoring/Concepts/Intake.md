---
type: concept
module: M1
updated: 2026-08-14
---
# Intake
One car's visit — split out of the old overloaded `Job` by [[ADR-012 Intake-Job two-level
aggregate]]. Tenant-scoped, carries the facts a `Job` used to carry alone: `workshop_id`,
`vehicle_id`, `customer_id` (frozen at intake — who this visit bills to, not necessarily the
vehicle's current owner), and the owner status-page `token`. One `Intake` `has_many :jobs` —
one or more repairs, each with its own stage, crew, and blockers. See [[Job]].

## An Intake always knows
- **Status** — `open`, `delivered`, or `cancelled`. Stored, not derived — see below.
- **Its jobs** — one or more repairs, each independently progressing.
- **Active blockers** — a hold on the *whole visit*, not one repair (today: Hold for payment).
  See [[Blocker]].
- **Who registered it** — the SA who opened the visit (`registered_by`); `mark_done!`'s
  receiver reaches back to this person.
- **History** — its own status changes and blocker events. See [[Event log]].

## Stored status, derived `ready?`
`status` is a **stored** enum (`open`/`delivered`/`cancelled`), written only by the doors
([[Design laws]] #7). This looks like it should be derived — [[Design laws]] #3 says
dashboards are queries, not tables — and the first design pass agreed, storing only
`delivered_at` and deriving `cancelled` from the jobs. That broke on the **busy-vehicle
guard**: it's enforced by a Postgres *partial unique index*
(`index_intakes_one_open_per_vehicle WHERE status = 0`), and a partial index can only key on a
stored column of its own table. With `cancelled` derived, a dead visit would still read as
open to the index and wrongly block the vehicle's next intake. So the terminal *position* is
stored (a door-owned memoized derivation — nothing outside a door can move it, so it can't
drift from the jobs it summarizes); what stays genuinely derived is:

**`ready?`** — is every job terminal, with at least one actually `done` (not just
cancelled)? A live query over the children, never cached onto a column — the *"Done —
awaiting delivery"* board group's condition.

## Two verbs, and why there isn't a third
`IntakeActions` (see [[ADR-013 The door decomposed]]) holds exactly two moves:
- **`deliver!`** — the car goes home. Refuses unless `ready?`; sweeps `acknowledge_pending!`
  across every one of the intake's jobs, closing every `mark_done!` pin addressed to the
  delivering SA (a technician's still-open pin, addressed to someone else, is left alone).
- **`cancel_intake!`** — the whole visit is called off. Cascades `JobActions.cancel!` over
  the *open* jobs; done work survives. The terminal isn't chosen here — it derives from the
  cascade (zero done among them → the intake ends `cancelled`; ≥1 done → it stays `open` and
  reads `ready?`, still collectible via an explicit `deliver!`).

There is deliberately **no "start work" or "assign technician" verb on Intake** — those are a
*repair's* own moves, on `JobActions`. The visit's progress is never a state of its own; it's
always read off its jobs.

## In Rails
- `belongs_to :workshop, :vehicle, :customer` — the triple-stamp, moved here from `Job`
- `has_secure_token :token` — the owner status-page link, moved here from `Job`
- `enum :status, { open: 0, delivered: 1, cancelled: 2 }` — stored, door-written only
- `has_many :jobs`
- `has_many :intake_status_transitions` — append-only, `created_by` — see [[Event log]]
- `has_many :intake_blockers` — items only; the transitions association is missing (see
  §Timeline below). Neither table is `Acknowledgeable` — an intake blocker has no direction
  (SA raises, SA clears — nobody is ever waiting on someone else), so there's no receiver to
  pin. See [[Blocker]].
- `#ready?` — the one derived query (above)
- `#active_blockers` — items with no resolved event, a query not a column
- `#registered_by` — reads the birth transition (`from_status: nil`)
- `has_many :job_sheet_field_values` *(Sprint 5, not yet built)* — the jobsheet is the car's
  intake form (customer complaints + vehicle condition), one per visit, so it keys on
  `intake_id`, not `job_id` — [[ADR-012 Intake-Job two-level aggregate]] §Consequences

## Timeline — the goal, not yet built
The same question `Job#timeline` answers for one repair — trace this record's own event
history — has no Intake equivalent yet. Designed but unbuilt:

- **`Intake#timeline`** — the visit's own story (status changes + blocker events), merged by
  timestamp, mirroring `Job#timeline`'s shape.
- **`Intake#timeline_with_jobs`** — the deep trace: the visit's own timeline plus every one of
  its jobs' events, one merged chronological list. Batched the same way
  `Job.pending_acknowledgements_by_job` avoids N+1 — five queries flat, however many jobs are
  on the intake, not three per job.
- **Missing association**: `Intake` has no `has_many :intake_blocker_transitions` yet, even
  though the table carries a direct, indexed `intake_id` — today those events are only
  reachable by walking each `IntakeBlocker` item.
- **A presentation seam worth noting**: the merged deep timeline mixes acknowledgeable job
  events with non-acknowledgeable intake events. Not a defect — the plan is a
  `handoff_state(event)` helper (history / waiting / picked-up) that reads `Acknowledgeable`
  where present and falls back to `:history` for intake rows, giving the merged feed a free
  visual hierarchy: intake rows are the spine of the story, job rows carry the handoff drama.

Tracked as **S4.5.10** in [[Sprint plan]].

## Related
- [[Job]] · [[Event log]] · [[Blocker]] · [[Stage model]] · [[Data model]] · [[Design laws]] ·
  [[ADR-012 Intake-Job two-level aggregate]] · [[ADR-013 The door decomposed]]
