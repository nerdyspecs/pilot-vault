---
type: concept
module: M1
updated: 2026-07-16
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
Vehicle  ─*─ Job     ── triple-stamped: workshop_id + vehicle_id + customer_id
Job      ─*─ JobSheetFieldValue
Workshop ─1─ JobSheet ─*─ JobSheetField ─*─ JobSheetFieldValue

Job      ─*─ JobStageTransition                     (stage events + ack)
Job      ─*─ JobMechanic ─*─ JobMechanicTransition  (crew engagement + joined/left events)
              └ belongs_to Employment   (the stint, not the login — 2026-07-16, see Resolved)
Job      ─*─ JobBlocker ─*─ JobBlockerTransition    (blocker items + events — Sprint 3)
              └ belongs_to Blocker
Workshop ─*─ Blocker                                (workshop's blocker catalog)
```

- **User** — global, thin Devise login. No role, no org. Roles live on **Employment** ([[Design laws]] #1).
- **Workshop** — the tenant. Every tenant row carries `workshop_id`, queried via `Current.workshop` ([[Design laws]] #2).
- **Employment** — User↔Workshop edge + role + `ended_at`. See [[ADR-004 Multi-tenant foundation]].
- **Customer** — tenant-scoped owner *record* (not a login). `kind: person | company`; holds `name, phone, email` directly.
  **Phone is required** *(builder ruling 2026-07-15: no customer without a phone — it's the
  person-lookup key ([[Intake flow]]) and the notification channel; NOT NULL + validation,
  `0e4204d`)*; contact keys canonical at storage, blank → nil never `""`.
- **Vehicle** — tenant-scoped; `belongs_to :customer` (required). **`registration_number`**
  (renamed from "plate" 2026-07-15 — verbose JPJ term), canonicalized (ALL whitespace
  collapsed + upcased); `unique(workshop_id, registration_number)`. VIN = optional identity.
- **Job** — tenant-scoped, `belongs_to :vehicle` **and `belongs_to :customer`** (stamped at
  registration — see Resolved below). The tracked unit ([[Job]]). Per-visit.
- **JobSheet / JobSheetField / JobSheetFieldValue** — configurable inspection form (below).
- **Trackers — entity + event log** *(restructured 2026-07-16)*: `JobMechanic` (engagement)
  + `JobMechanicTransition` (joined/left); `JobBlocker` (item) + `JobBlockerTransition`
  (Sprint 3); stage events ride `Job` itself via `JobStageTransition`. Ack columns live on
  **event rows only**, never entities ([[Event log]], [[ADR-005 Acknowledged handoffs in V1]]).
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
- **Job stamps its customer** *(2026-07-14, builder — raise again at S2 Phase 1 kickoff,
  alongside R5)*. Vehicles change owners: if a job's customer is only reachable through
  `job.vehicle.customer`, selling the car retroactively rewrites who every past job was for —
  history derived through a **mutable pointer** violates the same append-only principle behind
  [[Design laws]] #8 (frozen answers) and ADR-009's leaning. So `jobs.customer_id` is written
  **once at registration** (fact at the moment it was true): `job.customer` = who brought the
  car in and pays (immutable history); `vehicle.customer` = who owns it *now* (mutable
  present). They legitimately diverge after a sale — that divergence is the correct answer.
  One creation-time validation: the stamped customer must equal the vehicle's customer at
  registration (divergence may only arise later, by reassignment — never at birth).
  *(2026-07-15: the match **validation** is deferred — too rigid without a recovery path;
  legitimate payer≠file cases exist. The stamp + default-copy stand; see [[Deferred design]].)* Also pins
  the job's notification target ("External" side) against mid-job reassignment. A
  `VehicleOwnership` period-edge table was considered and rejected for v1 (over-engineering).
- **Who tracker rows point at — the actor/holder split** *(2026-07-16, builder ruling,
  Session 19; app commit `6799438`)*. Two different questions, two different FK targets:
  - **Engagements point at `Employment`** (`job_mechanics.employment_id`, not `user_id`).
    A crew engagement is a *responsibility held by a stint* — "Ah Boy-as-technician-here",
    not "login #42". No owner gap: every legal assignee holds an active technician
    employment by the door's own guard. Payoff is direct stint attribution —
    `employment.jobs` answers "what did he do while employed here"; a re-employed mechanic's
    two stints each own their jobs; an engagement pointing at an ended employment reads
    truthfully as history.
  - **Actor/audit columns stay `User`** (`created_by`/`acknowledged_by` on all event
    tables). Actions are taken by *people*, and owners act through the door constantly but
    hold no Employment (ADR-006) — an employment FK there is unrecordable for them. The
    considered fixes were all rejected: polymorphic actor (loses FK integrity), auto-created
    owner employments (fictional records — would reintroduce the `owner` role as data and
    quietly rewrite the Employment-OR-Ownership access rule). Role-at-action-time stays
    derivable: `(user_id, workshop_id, created_at)` finds the exact employment — **because
    employments are append-only**, the invariant this ruling makes load-bearing (below).
- **Employments are append-only** *(pinned 2026-07-16, now load-bearing)*: terminate =
  `end!` (sets `ended_at`), role change = end + new row, **delete = never** once referenced
  — engagements FK to employments, so deletion would orphan job history. Enforced twice:
  `dependent: :restrict_with_error` + the DB FK. If in-place role edits ever appear, the
  audit story above collapses — don't.

## v2 — additive, do not build
- **Company + CompanyEmployment** (roles: owner / fleet_manager / driver); Company claims company-kind Customers via `customers.company_id`.
  *(2026-07-13, builder: Company also gets its own **governance edge** (`CompanyOwnership`-style),
  mirroring Workshop's Ownership — both are instances of one reusable pattern: organisation ─
  membership edges ─ people, governance on top. Same **pattern**, never the same **table**:
  Workshop is the tenant and the RLS boundary; Company is a *customer* whose vehicles cross
  workshop boundaries. Merging them into one "organisation" entity would tangle tenancy at the
  root. ⚠ Open for the v2 design pass: how RLS serves Company reads **across** tenanted
  workshops — see worklog Session 14.)*
- **Owner logins:** `customers.user_id` (nullable = unclaimed); check constraint — `user_id` and `company_id` never both set.
- **Fleet** (grouping inside Company) — replaces the old v1 "Group" idea.
- **Household / shared claims** *(2026-07-15, from the intake discussion)*: let two Users
  see each other's claimed customer records ("Lim and his wife see each other's vehicles").
  Same claim machinery as Company — a small membership edge over customer records — so it
  rides the identical v2 rails (person → claim → frozen stamp), zero v1 cost. Motivating
  case: a household's history legitimately split across two file cards; perfect intake can't
  (and shouldn't) always prevent it — filing under "who deals with us" is correct routing.
- **Global vehicle identity** (plate/VIN) → cross-shop history, owner-side read only.

## Related
- [[Job]] · [[Job visibility]] · [[Intake flow]] · [[Overview]] · [[ADR-004 Multi-tenant foundation]] · [[Design laws]] · [[ADR-003 Digitized jobsheet in V1]]
