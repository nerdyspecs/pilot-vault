---
type: concept
module: M1
updated: 2026-07-17
---
# Stage model
The **progress axis** — how far a job has moved through its work. Exactly one stage at a time.
Tenant-scoped (`workshop_id`) — see [[ADR-004 Multi-tenant foundation]].

## Stages
```
Registered → Assigned → In-Progress → Done → Delivered
     └──────────┴──────────┴─→ Cancelled (terminal — from active stages only, never Done)
```

| Stage | What happens | Owner (role) |
|---|---|---|
| Registered | Intake done, awaiting assignment | service_advisor |
| Assigned | Mechanic allocated — set by the assignment action, in one motion | service_advisor |
| In-Progress | Diagnose + repair — includes the technician's own testing | technician |
| Done | Work complete — **job frozen / immutable** | technician |
| Delivered | Owner has taken the car — *terminal (success)* | — |
| Cancelled | Job died — declined / withdrawn — *terminal* | — |

## Rules
- Every job has exactly one stage.
- Stage changes are timestamped and logged via ONE DOOR — **no silent changes** (see [[Event log]]).
- Transitions are an **allow-list**, enforced in the service. Forward by default; backward/exit
  moves only where explicitly allowed (e.g. In-Progress → Assigned when new faults appear).
- **Cancelled** is reachable from any active stage (service_advisor / workshop_manager / owner).
- **Open blockers stop `→ done`** *(ruled 2026-07-16; ⚠ per-blocker guard nuance pending —
  the Hold-For-Payment collision, see [[Blocker]]; enforcement lands with Sprint 3's door verbs)*.
- Stage changes are **acknowledged** by the service advisor — see
  [[ADR-005 Acknowledged handoffs in V1]]. **Done freezes the job** — corrections open a new job
  ([[Design laws]] #8).

## V1 note
Quality Check was considered and **dropped** — verification folds into In-Progress. Add it back
as its own stage if it ever earns one (cheap — just a new enum value).

## In Rails
`Job.stage` enum. Current stage is a real column (fast for the live list); history lives in
[[Event log]].

## Related
- [[Job]] · [[Blocker]] · [[Event log]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]]
