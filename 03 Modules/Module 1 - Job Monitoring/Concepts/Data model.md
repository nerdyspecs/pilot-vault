---
type: concept
module: M1
updated: 2026-08-14 (entity map re-drawn for the Intake/Job split — ADR-012)
---
# Data model
Customers, vehicles, jobs, and who's allowed to touch them.

**The shape: a routing problem, not a customer database.** Every entity answers, per job,
*"who's responsible and who do we talk to?"* — see [[Product overview]]. Tenancy + user rules
live in [[ADR-004 Multi-tenant foundation]] and [[Design laws]]; this note is the entity map.

> [!warning] The entity map below predates [[ADR-010 WorkshopStaff supersedes the edge split]] (2026-07-21)
> The `WorkshopEmployment` / `WorkshopOwnership` edges shown below **no longer exist** — they
> collapsed into one **`WorkshopStaff`** record (`owner` boolean) + append-only
> **`WorkshopStaffRole`** rows, and every actor/holder column points at `WorkshopStaff`. The old
> shape is kept for its reasoning; the reconciliation is the `[!note]` under §Resolved and ADR-010.

> [!warning] The entity map also predates [[ADR-012 Intake-Job two-level aggregate]] (2026-08-03)
> **`Job` no longer sits directly under `Vehicle`.** An **`Intake`** (one car's visit) now sits
> between them and carries the triple-stamp + jobsheet + token this note used to put on `Job`;
> a `Job` (one repair) `belongs_to :intake` and reaches vehicle/customer through it. The diagram
> below is redrawn to the current shape; §Resolved's "Job stamps its customer" reasoning is kept
> — it now applies to **Intake**, not Job (a visit is stamped once at intake; a job never was
> the thing that should have carried it). See [[Intake]] for the full note.

## Entities (v1)
```
User (thin login) ─*─ WorkshopEmployment ─*─ Workshop (tenant)
      role: technician | service_advisor | parts_advisor | workshop_manager
User (thin login) ─*─ WorkshopOwnership ─*─ Workshop   (governance — ADR-006)

Workshop ─*─ Customer (person | company; name, phone, email)
Workshop ─*─ Vehicle ── belongs_to Customer (required)
Vehicle  ─*─ Intake  ── triple-stamped: workshop_id + vehicle_id + customer_id; has_secure_token
Intake   ─*─ Job     ── belongs_to :intake only; reaches vehicle/customer through it
Intake   ─*─ JobSheetFieldValue                 (the car's intake form — keys on intake_id)
Workshop ─1─ JobSheet ─*─ JobSheetField ─*─ JobSheetFieldValue

Intake   ─*─ IntakeStatusTransition             (open→delivered / open→cancelled; NOT ack'able)
Intake   ─*─ IntakeBlocker ─*─ IntakeBlockerTransition (visit-level hold, e.g. Hold for payment;
              └ belongs_to Blocker                     NOT ack'able — no direction to pin)

Job      ─*─ JobStageTransition                 (stage events + ack)
Job      ─*─ JobTechnician                      (crew membership NOW — deleted on remove)
              └ belongs_to WorkshopEmployment   (the stint, not the login — see Resolved)
Job      ─*─ JobTechnicianTransition            (joined/left history — self-contained:
              carries job_id + workshop_employment_id directly; no FK to membership)
Job      ─*─ JobBlocker ─*─ JobBlockerTransition (blocker items + events — Sprint 3)
              └ belongs_to Blocker
Workshop ─*─ Blocker                            (workshop's blocker catalog — shared by both levels)
```
*(2026-07-17, Session 21: edges renamed `Employment`→`WorkshopEmployment`,
`Ownership`→`WorkshopOwnership` — organisation-prefixed, mirroring v2's
`CompanyEmployment`/`CompanyOwnership`; crew tracker restructured to "Design B" —
membership + self-contained events, technician vocabulary; a stale `owner` in the role
list above was also dropped, removed by ADR-006 long before.)*

- **User** — global, thin Devise login. No role, no org. Roles live on **Employment** ([[Design laws]] #1).
- **Workshop** — the tenant. Every tenant row carries `workshop_id`, queried via `Current.workshop` ([[Design laws]] #2).
- **WorkshopEmployment** — User↔Workshop edge + role + `ended_at` (renamed from `Employment`
  2026-07-17; `WorkshopOwnership` likewise from `Ownership`). See [[ADR-004 Multi-tenant foundation]].
- **Customer** — tenant-scoped owner *record* (not a login). `kind: person | company`; holds `name, phone, email` directly.
  **Phone is required** *(builder ruling 2026-07-15: no customer without a phone — it's the
  person-lookup key ([[Intake flow]]) and the notification channel; NOT NULL + validation,
  `0e4204d`)*; contact keys canonical at storage, blank → nil never `""`.
- **Vehicle** — tenant-scoped; `belongs_to :customer` (required). **`registration_number`**
  (renamed from "plate" 2026-07-15 — verbose JPJ term), canonicalized (ALL whitespace
  collapsed + upcased); `unique(workshop_id, registration_number)`. VIN = optional identity.
- **Intake** — tenant-scoped, `belongs_to :vehicle` **and `belongs_to :customer`** (stamped at
  intake — see Resolved below). One car's visit; owns one or more Jobs ([[Intake]]).
- **Job** — tenant-scoped, `belongs_to :intake` only; reaches vehicle/customer through it. One
  repair on a visit ([[Job]]). Per-repair, not per-visit — a visit can own several.
- **JobSheet / JobSheetField / JobSheetFieldValue** — configurable inspection form (below).
- **Trackers** *(restructured 2026-07-16; crew re-restructured to Design B 2026-07-17)*:
  `JobTechnician` (present-tense membership, deleted on remove) + `JobTechnicianTransition`
  (self-contained joined/left history); `JobBlocker` (item, written once) +
  `JobBlockerTransition` (Sprint 3); stage events ride `Job` itself via `JobStageTransition`.
  Ack columns live on **event rows only**, never membership/items ([[Event log]],
  [[ADR-005 Acknowledged handoffs in V1]]).
- **Blocker** — the workshop's catalog of blocker types, each with `raised_by_role`/`cleared_by_role` (seed "Hold For Payment"); all state lives in `JobBlockerTransition` ([[Blocker]]).

## The jobsheet — a configurable inspection form
The paper jobsheet, digitized — the adoption wedge ([[ADR-003 Digitized jobsheet in V1]]). Fill
the sheet, status tracking is a byproduct. Fields are **rows, not columns** → the owner adds them
at runtime, no migration.

- **JobSheet** — the **blank form**. **One per workshop** (`belongs_to :workshop`), owner-configured.
- **JobSheetField** — a field: `label` + `kind` (checkbox | text).
- **JobSheetFieldValue** — one visit's **answer** (`belongs_to :intake, :job_sheet_field`) —
  keyed on `intake_id`, not `job_id`: it's the car's intake form (complaints, vehicle
  condition), filled once per visit, not once per repair
  ([[ADR-012 Intake-Job two-level aggregate]] §Consequences). *(2026-08-14: was `job_id` when
  this note predated the split — per-repair jobsheet fields would be a v2 additive if ever
  needed.)*

**JobSheet = the workshop's blank form; a visit's filled sheet = its JobSheetFieldValues.**
"This car's intake jobsheet" = `intake.job_sheet_field_values` read against the fields — a
*view*. Complaints & mileage are fields; vehicle info is referenced from Vehicle, never copied.
Flat fields only.

The form is a **record, not something churned per job** — v1 supports **adding** fields but has
**no destructive delete** (removing a field would orphan past answers). And once a job is **Done,
its answers are frozen** — corrections open a new job ([[Design laws]] #8). Per-answer snapshots
are deferred ([[Deferred design]]). Lighter build alt: two `jsonb` columns.

> [!question] Freeze condition needs re-deciding, not yet done (2026-08-14)
> "Once a job is Done" doesn't parse now that the answers live on **Intake**, not Job — a visit
> with several repairs has no single Done moment. Candidates: freeze when the *intake* reaches a
> terminal (`delivered`/`cancelled`), or when its `ready?`. Not decided; this note predates the
> split and Sprint 5 (where JobSheet is actually built) hasn't reached it yet.

## A Job's two responsibility sides
- **Internal** — which staff/role acts. Resolved by WorkshopEmployment role; all changes via a
  door, per level ([[M1-F1 Status flow and transitions]], [[ADR-013 The door decomposed]]).
- **External** — who we notify. v1: the Customer's own `phone`/`email` + the **intake's** token
  link (moved from Job — [[ADR-012 Intake-Job two-level aggregate]]).

## Resolved
- **Vehicle key:** registration = lookup key; VIN = optional identity. `make/model/year/origin` loose → seed v2 `VehicleModel`.
- **Job stamps its customer** *(2026-07-14, builder — raise again at S2 Phase 1 kickoff,
  alongside R5)*. **⚠ 2026-08-14, [[ADR-012 Intake-Job two-level aggregate]]: the stamp moved
  up a level — `intakes.customer_id`, not `jobs.customer_id`. The reasoning below is unchanged,
  read "the intake" wherever it says "the job" — a visit is stamped once at intake; a job
  reaches its customer through its intake, never directly.** Vehicles change owners: if a job's customer is only reachable through
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
  Session 19; app commit `6799438`)*. **Pinned as the general law (2026-07-17): holdings
  point at the stint, actions point at the person.** Two different questions, two different
  FK targets:
  - **Crew rows point at `WorkshopEmployment`** (`workshop_employment_id`, not `user_id`) —
    both the membership and its history events, post-Design-B. A crew membership is a
    *responsibility held by a stint* — "Ah Boy-as-technician-here",
    not "login #42". No owner gap: every legal assignee holds an active technician
    employment by the door's own guard. Payoff is direct stint attribution —
    `workshop_employment.jobs` answers "what did he do while employed here" (current work
    via membership; full per-stint history via the events); a re-employed technician's
    two stints each own their work; an event pointing at an ended employment reads
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
  — crew rows FK to employments *(post-Design-B: membership AND every history event —
  the history FK makes this pin harder, not softer)*, so deletion would orphan job history.
  Enforced twice: `dependent: :restrict_with_error` + the DB FKs. If in-place role edits
  ever appear, the audit story above collapses — don't.

> [!note] Superseded 2026-07-21 by [[ADR-010 WorkshopStaff supersedes the edge split]]
> The actor/holder split's **"holdings → stint" half stands**; its **"actions → person
> (`User`)" half is reversed** — actions point at the tenant-local person too. Both halves
> now target one record: **`WorkshopStaff`** (`WorkshopEmployment` + `WorkshopOwnership`
> collapsed into it; roles moved to append-only `WorkshopStaffRole` rows). `created_by` /
> `acknowledged_by` and the crew holding (`job_technicians.workshop_staff_id`) are all
> `WorkshopStaff`, each with a composite FK `(actor_id, workshop_id) → workshop_staff`.
> **Why the 2026-07-16 rejection above no longer holds:** "actions stay `User`" existed *only*
> because owners held no `Employment`, so an employment FK was unrecordable for them — that was
> the polymorphic-actor trap. Collapsing the two edges removes it: an owner is a `WorkshopStaff`
> like anyone (an `owner` boolean, no role), so the actor is a single non-polymorphic FK that
> the DB can tenant-check. Role-at-action-time no longer needs the `(user_id, workshop_id,
> created_at)` derivation — the stamped `WorkshopStaffRole` is immutable per row and carries it
> directly. The append-only invariant migrates from employments to `WorkshopStaffRole`. See
> ADR-010 for the full shape and accepted trade-offs (no ownership history in v1).

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
- [[Job]] · [[Intake]] · [[Job visibility]] · [[Intake flow]] · [[Overview]] ·
  [[ADR-004 Multi-tenant foundation]] · [[Design laws]] · [[ADR-003 Digitized jobsheet in V1]] ·
  [[ADR-012 Intake-Job two-level aggregate]] · [[ADR-013 The door decomposed]]
