---
type: roadmap
module: M1
updated: 2026-07-28 (slice 5 reshaped by ADR-011 — receiver stored at write time, no inbox; board only)
---
# Module 1 — Build roadmap & design backlog

V1 = Module 1. Each slice is a thin vertical, built in order. **Designed** = spec/concept
exists; **needs design** = real thinking still required before/while building.

> **Execution:** this is the slice map. Task-level breakdown (junior-sized) lives in [[Sprint plan]].

| #   | Slice                                                        | Status                   |
| --- | ------------------------------------------------------------- | ------------------------ |
| 0   | Setup (repo, Rails, Postgres, Devise, deploy skeleton)        | mechanical               |
| 1   | Engine — Job + stages + transitions + JobStageTransition      | ✅ **built** (Sprint 2 closed 2026-07-17) |
| 2   | Tenancy + WorkshopEmployment/WorkshopOwnership edges + role-gated transitions (ONE DOOR) | ✅ **built** (Sprint 2 closed 2026-07-17) |
| 3   | Live job list & filtering                                     | mostly mechanical        |
| 4   | Blockers                                                      | ✅ **built** (Sprint 3 closed 2026-07-24) |
| 5   | Acknowledged handoffs, surfaced on the board                  | ✅ **built** (Sprint 4 closed 2026-07-28, `982f7e9`+`8fad8c9`; colour deferred to S5.7) |
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

### 5 — Acknowledged handoffs, surfaced on the board *(no inbox)*
- **Acknowledgement is in v1** — see [[ADR-005 Acknowledged handoffs in V1]] and
  [[ADR-011 Acknowledgement as stored visibility]]. Every handoff (stage change, blocker resolved,
  technician added) stamps a **stored `receiver_id`** at write time; the "waiting on whom" read is a
  plain query over the event tables (`receiver_id IS NOT NULL AND acknowledged_at IS NULL`), grouped
  by job — not a new table, and **not an inbox**. There is no routing and no confirm button: the
  board carries a muted *"Waiting on &lt;name&gt;"* line and the manager walks over.
  *(**Reshaped 2026-07-28 by ADR-011**, settling [[Product gaps]] #5: the receiver is *stored*, not
  derived — which restores ADR-005's original `to_user`, deletes the `.pending_ack` predicate that
  only existed because it was removed, and fixes an append-only bug (editing a blocker type's
  `cleared_by_role` re-points open handoffs). Acknowledgement is implicit — acting on a job clears
  what you owe. The 2026-07-24 holder model was studied and dropped; see ADR-011's Rejected
  alternatives. Ageing colour is deferred to S5.7.)*
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
