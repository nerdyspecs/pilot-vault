---
type: concept
module: M1
updated: 2026-07-04
---
# Data model
Customers, vehicles, jobs, and who's allowed to touch them.

**The shape: a routing problem, not a customer database.** Every entity answers, per job,
*"who's responsible and who do we talk to?"* — see [[Product overview]]. Tenancy + user rules
live in [[ADR-004 Multi-tenant foundation]] and [[Design laws]]; this note is the entity map.

## Entities (v1)
```
User (thin login) ─*─ Employment ─*─ Workshop (tenant)
      role: technician | service_advisor | parts_advisor | workshop_manager | owner

Workshop ─*─ Customer (person | company; name, phone, email)
Workshop ─*─ Vehicle ── belongs_to Customer (required)
Vehicle  ─*─ Job     ── double-stamped: workshop_id + vehicle_id
Job      ─*─ JobSheetFieldValue
Workshop ─1─ JobSheet ─*─ JobSheetField ─*─ JobSheetFieldValue

Job      ─*─ JobStageTransition       (stage history + ack)
Job      ─*─ JobBlockerTransition ── belongs_to Blocker   (blocker events + ack)
Job      ─*─ JobMechanic ── belongs_to User               (crew, plural + ack)
Workshop ─*─ Blocker                  (workshop's blocker catalog)
```

- **User** — global, thin Devise login. No role, no org. Roles live on **Employment** ([[Design laws]] #1).
- **Workshop** — the tenant. Every tenant row carries `workshop_id`, queried via `Current.workshop` ([[Design laws]] #2).
- **Employment** — User↔Workshop edge + role + `ended_at`. See [[ADR-004 Multi-tenant foundation]].
- **Customer** — tenant-scoped owner *record* (not a login). `kind: person | company`; holds `name, phone, email` directly.
- **Vehicle** — tenant-scoped; `belongs_to :customer` (required). Plate normalized (strip/upcase); `unique(workshop_id, plate)`. VIN = optional identity.
- **Job** — tenant-scoped, `belongs_to :vehicle`. The tracked unit ([[Job]]). Per-visit.
- **JobSheet / JobSheetField / JobSheetFieldValue** — configurable inspection form (below).
- **JobStageTransition / JobBlockerTransition / JobMechanic** — the three job event trackers, each with ack columns ([[Event log]], [[ADR-005 Acknowledged handoffs in V1]]).
- **Blocker** — the workshop's catalog of blocker types, each with `raised_by_role`/`cleared_by_role` (seed "Hold For Payment"); all state lives in `JobBlockerTransition` ([[Blocker]]).

## The jobsheet — a configurable inspection form
The paper jobsheet, digitized — the adoption wedge ([[ADR-003 Digitized jobsheet in V1]]). Fill
the sheet, status tracking is a byproduct. Fields are **rows, not columns** → the owner adds them
at runtime, no migration.

- **JobSheet** — the **blank form**. **One per workshop** (`belongs_to :workshop`), owner-configured.
- **JobSheetField** — a field: `label` + `kind` (checkbox | text).
- **JobSheetFieldValue** — one car's **answer** (`belongs_to :job, :job_sheet_field`).

**JobSheet = the workshop's blank form; a car's filled sheet = its JobSheetFieldValues.**
"Car ABC's jobsheet" = `job.job_sheet_field_values` read against the fields — a *view*. Complaints
& mileage are fields; vehicle info is referenced from Vehicle, never copied. Flat fields only.

The form is a **record, not something churned per job** — v1 supports **adding** fields but has
**no destructive delete** (removing a field would orphan past answers). And once a job is **Done,
its answers are frozen** — corrections open a new job ([[Design laws]] #8). Per-answer snapshots
are deferred ([[Deferred design]]). Lighter build alt: two `jsonb` columns.

## A Job's two responsibility sides
- **Internal** — which staff/role acts. Resolved by Employment role; all changes via ONE DOOR ([[M1-F1 Status flow and transitions]]).
- **External** — who we notify. v1: the Customer's own `phone`/`email` + the job's token link.

## Resolved
- **Vehicle key:** registration = lookup key; VIN = optional identity. `make/model/year/origin` loose → seed v2 `VehicleModel`.

## v2 — additive, do not build
- **Company + CompanyEmployment** (roles: owner / fleet_manager / driver); Company claims company-kind Customers via `customers.company_id`.
- **Owner logins:** `customers.user_id` (nullable = unclaimed); check constraint — `user_id` and `company_id` never both set.
- **Fleet** (grouping inside Company) — replaces the old v1 "Group" idea.
- **Global vehicle identity** (plate/VIN) → cross-shop history, owner-side read only.

## Related
- [[Job]] · [[Overview]] · [[ADR-004 Multi-tenant foundation]] · [[Design laws]] · [[ADR-003 Digitized jobsheet in V1]]
