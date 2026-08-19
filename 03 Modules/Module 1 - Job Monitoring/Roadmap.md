---
type: roadmap
module: M1
updated: 2026-08-19 (Session 35 — slice 6 jobsheet backend built (S5.1a/b), UI still pending; prior: Session 34 — slice 6 jobsheet storage decided, ADR-015, designed not yet built; prior: slice 6 jobsheet re-pointed at ADR-014, fixed/product-defined, storage chipped out; prior: 2026-08-14 slice 5.5 built; slice 7 re-pointed at Intake's token — ADR-012, ADR-013)
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
| 5   | Acknowledged handoffs, surfaced on the board                  | ✅ **built** (Sprint 4 closed 2026-07-28, `982f7e9`+`8fad8c9`; colour deferred to S6.7) |
| 5.5 | **Intake/Job aggregate morph** — the visit split out above the repair | ✅ **built** (Sprint 4.5 closed 2026-08-14, [[ADR-012 Intake-Job two-level aggregate]] + [[ADR-013 The door decomposed]]) |
| 6   | Job intake + fixed inspection jobsheet                        | 🔨 **backend built**, UI not yet built *(models + migration + catalog shipped, S5.1a/S5.1b — [[ADR-015 Jobsheet answers are rows against a frozen question set]]; intake/jobsheet UI = S5.5, see [[Sprint plan]])* |
| 7   | Owner status page (token link)                                | ⚠️ needs design *(token now on Intake, not Job — ADR-012, see §7 below)* |
| 8   | Reporting & attribution                                       | ⚠️ needs design          |

## Slice notes (design status per slice)
Slices 5–6 are **designed** (notes kept for the reasoning); 7–8 still need their design pass —
don't build those blind.

### 6 — Job intake + fixed inspection jobsheet
The intake half and the jobsheet's storage structure are **both designed** now — see
[[Data model]], [[ADR-014 Jobsheet is a fixed product-defined inspection]] (supersedes
[[ADR-003 Digitized jobsheet in V1]]'s owner-configurable core), and
[[ADR-015 Jobsheet answers are rows against a frozen question set]] (decides what
[[Inspection jobsheet — design brief]] chipped out). What's left is the build — [[Sprint plan]]
S5.1a–d.
- **Job grain:** ~~✅ **per-visit** (one job = one visit = one stage flow). Per-work-item is an additive `WorkItem` child if ever needed.~~
  **⚠ REVERSED 2026-08-03 by [[ADR-012 Intake-Job two-level aggregate]]** — the grain is now
  **per-repair**: one **Intake** (the visit) owns many **Jobs** (one repair each, with its own stage
  flow, crew, and blockers). The escape hatch this line predicted was taken, just *inverted* —
  rather than a `WorkItem` child under Job, the **visit was lifted out above it**, so the unit that
  already carried stage/crew/blockers/events didn't have to be rebuilt. Why it moved: a car
  realistically comes in for several services worked by several technicians in parallel, which
  one-job-per-visit cannot represent. Sequenced as **Sprint 4.5**, before the S6 board (which groups
  by the car) and while there is still no production data.
- **Customer / Vehicle model:** ✅ (routing shape, two user populations).
- **Vehicle key:** ✅ registration = lookup key, VIN = optional identity.
- **Inspection jobsheet:** ✅ **designed** — fixed, product-defined form (fields set by the
  product, versioned in code, no owner CRUD). *(2026-08-19, [[ADR-014 Jobsheet is a fixed product-defined inspection]]:
  reverses the owner-configurable `JobSheet`/`JobSheetField`/`JobSheetFieldValue` EAV model this
  line used to describe — that build was discarded. The per-visit anchor is unchanged: one
  inspection per **Intake**, keyed on `intake_id`. Storage structure — how the fixed fields and
  answers are actually stored — is decided by
  [[ADR-015 Jobsheet answers are rows against a frozen question set]]: code-defined templates +
  a thin `Jobsheet` header + field-level `JobsheetAnswer` rows. See
  [[Inspection jobsheet — design brief]] for the 39-item field list and the reasoning trail.)*
- Quotation deferred (see [[ADR-002 V1 scope]]) — no entity in V1, only the approval blocker.

Left to build: the model + migration + seed + tests (Sprint plan S5.1a–d), then the actual
intake/jobsheet forms. No field-admin screen — there's nothing left for an owner to administer.

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
  alternatives. Ageing colour is deferred to S6.7.)*
- **Owner-facing delivery:** v1 = a copy-paste message the service advisor sends manually.
  WhatsApp / email automation is parked in [[Open questions]] — decide during the intake feature.

### 7 — Owner status page
- **⚠ 2026-08-14, [[ADR-012 Intake-Job two-level aggregate]]: the token moved to Intake.** The
  owner's page is the car's **visit**, `GET /intakes/:token` — not a single job's token — since a
  visit can now own several repairs. Design pass still needed for what follows below; see
  [[Job visibility]] for the RLS policy shape this builds on (already re-anchored to Intake).
- Token lifecycle: how it's generated, whether it expires, can it be revoked.
- What the owner actually sees (stage only? ETA? are blockers hidden? — and now, whether they see
  one merged status or a breakdown per repair).
- Mobile-first, no login.

### 8 — Reporting & attribution
- The time math lives in [[Event log]] / [[Blocker]] — but *which reports* (time-in-stage,
  time-blocked-by-department, workshop health) still need defining.
- Per-blocker two-bucket attribution (owner time vs. raiser time).

## Mechanical (design as you build)
Setup (0), live job list (3), user fields (small) — straightforward CRUD/UI, no deep thinking.
