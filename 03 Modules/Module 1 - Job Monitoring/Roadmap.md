---
type: roadmap
module: M1
updated: 2026-07-17
---
# Module 1 — Build roadmap & design backlog

V1 = Module 1. Each slice is a thin vertical, built in order. **Designed** = spec/concept
exists; **needs design** = real thinking still required before/while building.

> **Execution:** this is the slice map. Task-level breakdown (junior-sized) lives in [[Sprint plan]].

| #   | Slice                                                        | Status                   |
| --- | ------------------------------------------------------------- | ------------------------ |
| 0   | Setup (repo, Rails, Postgres, Devise, deploy skeleton)        | mechanical               |
| 1   | Engine — Job + stages + transitions + JobStageTransition      | ✅ designed               |
| 2   | Tenancy + Employment edges + role-gated transitions (ONE DOOR) | ✅ designed               |
| 3   | Live job list & filtering                                     | mostly mechanical        |
| 4   | Blockers                                                      | ⚠ needs respec at Sprint 3 kickoff |
| 5   | Acknowledged handoffs + in-app inbox                          | ✅ designed              |
| 6   | Job intake + digitized jobsheet                               | ✅ designed              |
| 7   | Owner status page (token link)                                | ⚠️ needs design          |
| 8   | Reporting & attribution                                       | ⚠️ needs design          |

## Slice notes (design status per slice)
Slices 5–6 are **designed** (notes kept for the reasoning); 7–8 still need their design pass —
don't build those blind.

### 6 — Job intake + digitized jobsheet
Designed — see [[Data model]] and [[ADR-003 Digitized jobsheet in V1]]:
- **Job grain:** ✅ **per-visit** (one job = one visit = one stage flow). Per-work-item is an additive `WorkItem` child if ever needed.
- **Customer / Vehicle model:** ✅ (routing shape, two user populations).
- **Vehicle key:** ✅ registration = lookup key, VIN = optional identity.
- **Digitized jobsheet:** ✅ configurable form — `JobSheet` → `JobSheetField` → `JobSheetFieldValue` (fields as rows, owner CRUDs).
- Quotation deferred (see [[ADR-002 V1 scope]]) — no entity in V1, only the approval blocker.

Left to build (not design): the actual intake/jobsheet forms + the field-admin screen.

### 5 — Acknowledged handoffs + in-app inbox
- **Acknowledgement is in v1** — see [[ADR-005 Acknowledged handoffs in V1]]. Every handoff
  (stage change, blocker raised, mechanic added) is acked by its receiver; the in-app "waiting
  on me" inbox is a query over the event tables, not a new table — via the **`.pending_ack`
  handoff predicate** (S4.3), *not* a bare `acknowledged_at IS NULL` check, which overcounts
  (NULL also means "never was a handoff" — [[Event log]]). *(Query wording corrected 2026-07-17.)*
- **Owner-facing delivery:** v1 = a copy-paste message the service advisor sends manually.
  WhatsApp / email automation is parked in [[Open questions]] — decide during the intake feature.

### 7 — Owner status page
- Token lifecycle: how it's generated, whether it expires, can it be revoked.
- What the owner actually sees (stage only? ETA? are blockers hidden?).
- Mobile-first, no login.

### 8 — Reporting & attribution
- The time math lives in [[Event log]] / [[Blocker]] — but *which reports* (time-in-stage,
  time-blocked-by-department, workshop health) still need defining.
- Per-blocker two-bucket attribution (owner time vs. raiser time).

## Mechanical (design as you build)
Setup (0), live job list (3), user fields (small) — straightforward CRUD/UI, no deep thinking.
