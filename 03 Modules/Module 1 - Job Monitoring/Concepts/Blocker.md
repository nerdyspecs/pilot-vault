---
type: concept
module: M1
updated: 2026-07-04
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

## Two records: catalog + events
`Blocker` is the workshop's **catalog** of blocker types — not a stateful row. All *state* lives in
an event tracker, mirroring how [[Stage model]] uses `JobStageTransition`.

- **`Blocker`** — `belongs_to :workshop`; `label, raised_by_role, cleared_by_role`. The workshop's
  own list of blocker types, **and** who's allowed to raise/clear each one — so permission logic
  lives on the catalog, not hardcoded in the feature ([[M1-F1 Status flow and transitions]]).
  `workshop_manager`/`owner` always override, regardless of a blocker's roles.
  Seed **"Hold For Payment"** (`raised_by: service_advisor`, `cleared_by: service_advisor`); more
  are addable later (admin UI is deferrable — the structure is ready).
- **`JobBlockerTransition`** — the event tracker, parallel to `JobStageTransition`:
  `workshop_id, job_id, blocker_id, action (raised | resolved), note (free-text details),
  created_by, created_at, acknowledged_at, acknowledged_by`.

## Multiple blockers, derived from events
A job can carry **several blockers at once** (e.g. *Hold For Payment* + a parts wait).
"Currently blocked by" = raises with no matching resolve — a **query, not a column**
([[Design laws]] #3). Rows are never deleted; the rows ARE the history.

## Acknowledgement
Raising a blocker is a **handoff**: acknowledged by whoever resolves it; the resolve can notify the
raiser back. Tracked via `acknowledged_at` / `acknowledged_by` on the transition — see
[[ADR-005 Acknowledged handoffs in V1]].

## Related
- [[Job]] · [[Stage model]] · [[Event log]] · [[ADR-005 Acknowledged handoffs in V1]]
