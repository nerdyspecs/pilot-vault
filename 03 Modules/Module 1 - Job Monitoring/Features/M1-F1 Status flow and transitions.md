---
id: M1-F1
type: feature
module: M1
status: settled
updated: 2026-07-12
---
# M1-F1 — Status flow & role-gated transitions

## Goal
Move a job through its [[Stage model]] stages, each transition permitted only for the right
Employment role, **all through ONE DOOR** (`JobActions` — [[Design laws]] #7). Every
ownership handoff is **acknowledged** by the receiver ([[ADR-005 Acknowledged handoffs in V1]]).

## Settled 2026-07-12 (pre-Sprint-2 design session, builder decisions)
- **Job creation goes through the door too** — `JobActions` owns it. Creation is already a
  multi-record motion (Job + the `nil → registered` transition) and will grow (jobsheet
  values in Sprint 6; a possible parts list later). No `Job.create!` from controllers.
- **`in_progress → assigned` (the backward/reassignment move): service_advisor** (+
  workshop_manager/owner as always). It is a handoff, so it acknowledges like any other
  once Sprint 4 lands.
- **Assignment eligibility: active `technician` employment only; primary-only in v1.**
  The `primary` boolean column exists from day one but helpers stay dormant.
- **No cancel from `done`.** Done is frozen, not active; its only exit is `delivered`.
  The two terminals are `delivered` (success) and `cancelled` (died before done).
- **Concurrency guards deferred** (stage-change race + `lock_version`) — see
  [[Deferred design]]; failure mode is loud (double transitions visible in the log),
  fix is one line in the one door.

## Permission matrix (v1)
| Role | Stage transitions (via ONE DOOR) | Blockers | Crew |
|---|---|---|---|
| technician | Assigned → In-Progress → Done | raises/clears per its `Blocker.raised_by_role`/`cleared_by_role` | — |
| service_advisor | create Registered; Registered → Assigned; Done → Delivered; Cancel | raises/clears per role fields | assigns mechanic(s) |
| parts_advisor | — | raises/clears per role fields | — |
| workshop_manager | all | all (overrides any blocker's role fields) | all |
| owner | all | all (overrides any blocker's role fields) | all |

- **Stages:** Registered → Assigned → In-Progress → Done → Delivered, + **Cancelled** (see [[Stage model]]).
- **Assignment** = one motion: create a `JobMechanic` (primary) **and** move Registered → Assigned.
- **Blockers** — who can raise/clear which blocker type is data on the [[Blocker]] catalog
  (`raised_by_role` / `cleared_by_role`), not hardcoded here. `workshop_manager`/`owner` always override.
- **Cancelled** reachable from any active stage (service_advisor / workshop_manager / owner).
- **Vehicle owner** is **read-only** — no ONE DOOR access at all ([[Design laws]] #8).

## Acknowledgement (the handshake)
The change takes effect immediately, but **ownership isn't transferred until acknowledged** — an
unacknowledged handoff stays visible as limbo. Three triggers:

| Trigger | Passed by | Acknowledged by |
|---|---|---|
| Mechanic added | service_advisor | that mechanic (accepts the job) |
| Stage change | technician | service_advisor |
| Blocker raised | raiser (per catalog) | resolver (per catalog) |

Inbox ("waiting on me") = a query across the trackers where `acknowledged_at IS NULL` ([[Event log]]).

## State axis — before vs after Done
- **Before Done:** workshop roles edit per the matrix above.
- **At/after Done:** the job is **immutable** for everyone — answers and history frozen.
  Corrections open a **new** job ([[Design laws]] #8).

## Transition rules
- Allow-list, enforced in the service ([[Design laws]] #7). Forward by default; backward only where
  explicit (In-Progress → Assigned for reassignment / new fault) — **never** once Done.
- Every change writes a [[Event log|JobStageTransition]] — no silent changes.

## Acceptance (draft)
- [ ] A job moves through the stages; each transition permitted only for the right role.
- [ ] Assignment creates a primary `JobMechanic` + moves stage in one motion; the mechanic acknowledges.
- [ ] A raised blocker checks the catalog's `raised_by_role`; resolving checks `cleared_by_role`.
- [ ] Illegal transitions rejected; a Done job rejects all edits.
- [ ] Every transition writes a `JobStageTransition`.

## Related
- [[Stage model]] · [[Blocker]] · [[Event log]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-004 Multi-tenant foundation]]
