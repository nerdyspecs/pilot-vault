---
type: concept
module: M1
updated: 2026-08-14 (re-split for the Intake/Job aggregate — ADR-012)
---
# Stage model
The **progress axis** — how far a *repair* (Job) has moved through its work. Exactly one
stage at a time. Tenant-scoped (`workshop_id`) — see [[ADR-004 Multi-tenant foundation]].

> [!warning] Re-split 2026-08-14 by [[ADR-012 Intake-Job two-level aggregate]]
> The old single **Registered** stage split across two levels: **Intake** now stores its own
> `open → delivered` (or `→ cancelled`), and a Job is born straight into **`unassigned`** under
> an already-open intake. **`Delivered` left Job's enum entirely** — it's the one stored Intake
> fact, never a Job stage. A Job's terminals are now `done` and `cancelled` only; see [[Intake]]
> for the visit's own two-state story.

## Stages
```
Unassigned → Assigned → In-Progress → Done
     └──────────┴──────────┴─→ Cancelled (terminal — from active stages only, never Done)
```
*(One level up: `Intake` — `open → delivered`, or `open → cancelled` once every job is
terminal with none done. See [[Intake]].)*

| Stage | What happens | Owner (role) |
|---|---|---|
| Unassigned | Repair born under an already-open intake, awaiting a technician | service_advisor |
| Assigned | Technician allocated — set by the assignment action, in one motion | service_advisor |
| In-Progress | Diagnose + repair — includes the technician's own testing | technician |
| Done | Work complete — **job frozen / immutable** | technician |
| Cancelled | Repair died — declined / withdrawn — *terminal* | — |

## Rules
- Every job has exactly one stage.
- Stage changes are timestamped and logged via a **door**, per level — **no silent changes**
  (see [[Event log]], [[ADR-013 The door decomposed]]).
- Transitions are an **allow-list**, enforced in the service. Forward by default; backward/exit
  moves only where explicitly allowed (e.g. In-Progress → Assigned when new faults appear).
- **Cancelled** is reachable from any active stage (service_advisor / workshop_manager / owner).
- **Open blockers stop `→ done`** — the job-level veto (see [[Blocker]]). Hold for payment
  moved to the **intake** level and vetoes `→ delivered` instead — no collision anymore, each
  veto guards its own door's boundary.
- Stage changes are **acknowledged** by the service advisor — see
  [[ADR-005 Acknowledged handoffs in V1]]. **Done freezes the job** — corrections open a new job
  ([[Architecture laws]] #8).

## V1 note
Quality Check was considered and **dropped** — verification folds into In-Progress. Add it back
as its own stage if it ever earns one (cheap — just a new enum value).

## In Rails
`Job.stage` enum `{ unassigned: 0, assigned: 1, in_progress: 2, done: 3, cancelled: 4 }`.
Current stage is a real column (fast for the live list); history lives in [[Event log]].

## Related
- [[Job]] · [[Intake]] · [[Blocker]] · [[Event log]] · [[Architecture laws]] ·
  [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-012 Intake-Job two-level aggregate]] ·
  [[ADR-013 The door decomposed]]
