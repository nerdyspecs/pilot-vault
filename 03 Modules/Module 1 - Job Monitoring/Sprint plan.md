---
type: plan
module: M1
created: 2026-07-04
updated: 2026-07-21 (closed sprints archived)
---
# Module 1 — Sprint plan (execution)
Small, assignable tasks per sprint — sized for a junior dev to pick up one at a time. The
[[Roadmap]] is the higher-level slice map; this is the build order. Each sprint is a thin vertical
that ends in something demoable.

## Conventions (read once)
- **Vanilla Rails MVC.** Models (AR: associations, validations, scopes) · controllers (thin, RESTful)
  · views (ERB + Hotwire) · routes (RESTful). No extra layers.
- **The one exception:** all job state changes go through a single **service object** — `JobActions`
  (`app/services/job_actions.rb`), a plain PORO. This is **ONE DOOR** ([[Design laws]] #7).
  *(Renamed from `JobService` 2026-07-12, before any code existed: in workshop language a
  "service" is the repair itself — the old name confused the builder, the domain expert, and
  collided with a likely future `Service` catalog entity. One class, granular verb methods —
  builder rejected a per-action class split to keep the shared guards and the door auditable
  in one file.)*
- **No gem without justifying why Rails' built-in isn't enough** — currently Devise only ([[Agent guide]]).
- **Tenancy:** every workshop-owned table has `workshop_id`; never query it bare — always scope by
  `Current.workshop` ([[Design laws]] #2).
- **Tests (set 2026-07-08):** two layers, hollow middle. (1) **Minitest model unit tests** are
  the *priority* — calculations live in models ([[Design laws]] #9), so they're fast, isolated,
  and consistent everywhere reused. (2) **Capybara system tests** (`test/system/`, already
  gemmed) for end-to-end flows — written once the pages exist. Controller/request tests
  **deferred** (add when pages are present; low priority — pages are isolated, shared logic isn't).
- **Per-sprint test-review ritual:** at each sprint's exit, list which models were created or
  edited that sprint → decide whether unit tests need writing or revisiting. Tests need not be
  written the moment code is; they're evaluated as a batch at sprint close.
- **Commits** reference the sprint/task id (e.g. `S2.7`).
- **Dropdown-source ivars are `@<subject>_options`** *(2026-07-21)* — a controller ivar that
  populates a `<select>` is named for **what the user picks**, not the AR model behind it
  (`@technician_options`, not `@technician_roles` or `@workshop_staff_role_options`;
  `@blocker_options` in S3). Reserve the suffix for picker collections — a plain display list
  doesn't get it, or the suffix stops meaning anything.
- Each task links the design doc that explains the *why*.

---

## Sprints 0–2.5 — closed
Setup, tenancy spine, the job engine, and cold-start intake are built, tested, and shipped.
Full task detail (ticks, dated annotations, commit hashes) moved to
[[Sprint plan (Sprints 0-2.5, closed)]] to keep this file focused on current work.

## Tenant-spine collapse — done before Sprint 3 *(2026-07-21)*
A prerequisite, not a numbered sprint: `WorkshopEmployment` + `WorkshopOwnership` collapsed into
`WorkshopStaff` + `WorkshopStaffRole` ([[ADR-010 WorkshopStaff supersedes the edge split]]) so
every tenant-scoped sprint from here builds on a DB-tenant-checkable actor. Schema squash +
reseed (no prod data), 72/72 green incl. new composite-FK isolation tests. Sprint 3's blocker
event tables adopt `created_by`/`acknowledged_by → workshop_staff` from birth — no rework.

---

## Sprint 3 · Blockers
**Goal:** the overlay axis — see [[Blocker]]. **Exit:** raise/clear blockers per role; a job shows its (possibly several) active blockers.

- [ ] **S3.1** Migration + model **Blocker** (catalog): `workshop_id, label, raised_by_role,
      cleared_by_role`. **Seed "Hold For Payment"** (`raised_by/cleared_by: service_advisor`).
- [ ] **S3.2** Migration + model **JobBlockerTransition**: `workshop_id, job_id, blocker_id,
      action:integer(raised/resolved), note, created_by_id, acknowledged_at, acknowledged_by_id`.
      *(⚠ 2026-07-16: respecify at Sprint 3 kickoff — the tracker restructure makes blockers
      **three records** (catalog + `JobBlocker` items + events, with a `noted` action);
      also carry the ruled `→ done` guard + the unresolved Hold-For-Payment collision into
      S3.3 — see [[Blocker]] and [[Event log]].)*
- [ ] **S3.3** Extend `JobActions`: `raise_blocker` / `resolve_blocker` — check the actor's role against
      the catalog's `raised_by_role` / `cleared_by_role` (manager/owner always override).
- [ ] **S3.4** `Job#active_blockers` scope: raises with no later matching resolve (a query, [[Design laws]] #3).
- [ ] **S3.5** Controller + views: raise a blocker (pick from catalog + note), resolve a blocker, show
      active blockers on the job. Tech raise flow mobile-friendly.
- [ ] **S3.6** *(deferrable)* Blocker-catalog admin — owner adds blocker types. Can slip to later.
- [ ] **S3.7** Tests: raise/resolve · role checks via catalog · multiple active · manager override.

---

## Sprint 4 · Acknowledgement + inbox
**Goal:** the differentiator — see [[ADR-005 Acknowledged handoffs in V1]]. ⚠️ **Decide Product-gap #5
(partial-adoption) before building** ([[Product gaps]]). **Exit:** handoffs land in the receiver's inbox; ack clears them; stale ones flag.

- [ ] **S4.1** `JobActions#acknowledge(record, by:)` — sets `acknowledged_at` + `acknowledged_by` on an
      **event row**: `JobStageTransition` / `JobTechnicianTransition` / `JobBlockerTransition`.
      *(2026-07-17 correction: pre-restructure wording said `JobMechanic` — engagements carry
      **no ack columns** after the 2026-07-16 tracker split; acks live only on events — [[Event log]].
      Names updated to Design B, Session 21.)*
- [ ] **S4.2** Receiver logic (a query, not stored): stage change → service advisor · technician added →
      that technician · blocker raised → the `cleared_by_role` holder(s).
      *(⚠ design-pass item, noted 2026-07-17: a technician **removed before acking their
      `joined` event** — the debt's receiver is no longer on the crew; decide whether the
      dropped-handoff debt dies with the removal or stays owed. Exists in both crew designs;
      the events survive removal (Design B self-containment), so either answer is buildable.)*
- [ ] **S4.3** "Waiting on me" query across the event tables — via **the handoff predicate**
      (`.pending_ack`, design-pass item named 2026-07-16): unacked AND not-self-caused AND
      role-resolved to me, in ONE shared scope used by inbox, manager board, and limbo flag
      alike (a bare `acknowledged_at IS NULL` overcounts — NULL also means "never was a
      handoff"; [[Event log]]). Orphaned debts (role resolves to nobody) render as "pending,
      unheld role" — never vanish. Manager's chase board = the union view grouped by debtor,
      distinct from each person's own inbox.
- [ ] **S4.4** Inbox controller + view: my pending acks + an "acknowledge" button each. **Mobile-friendly
      — techs live here.**
- [ ] **S4.5** Limbo surfacing: flag handoffs unacked beyond a threshold.
- [ ] **S4.6** *(Product-gap #5)* graceful degradation per the decision — e.g. ack-on-behalf (logged as
      "by X for Y") and/or a limbo threshold/snooze.
- [ ] **S4.7** Tests: inbox shows the right pending items · ack clears them · limbo flagged.

---

## Sprint 5 · Live job list
**Goal:** answer "where is every job right now?" **Exit:** a filterable board of active jobs.

- [ ] **S5.1** Jobs index: active jobs for `Current.workshop` (exclude `delivered/cancelled`), showing
      stage, crew, active blockers.
- [ ] **S5.2** Filters: by stage, by technician (crew), by blocker.
- [ ] **S5.3** *(Product-gap #2)* aging highlight: flag jobs sitting in a stage beyond N days.
- [ ] **S5.4** *(optional)* Turbo Streams for near-live updates — a plain refresh is fine first.
- [ ] **S5.5** PC layout primary; a clean "my jobs" view readable on a technician's phone.
- [ ] **S5.6** Tests: tenant scoping of the index · filters · aging flag.

---

## Sprint 6 · Intake + digitized jobsheet
**Goal:** register a car end-to-end — see [[ADR-003 Digitized jobsheet in V1]], [[Data model]].
**Exit:** full intake creates Customer/Vehicle/Job + jobsheet answers in one flow.

- [ ] **S6.1** Migration + model **JobSheet** (`workshop_id`, one per workshop).
- [ ] **S6.2** Migration + model **JobSheetField** (`job_sheet_id, label, kind:integer(checkbox/text),
      position`).
- [ ] **S6.3** Migration + model **JobSheetFieldValue** (`job_id, job_sheet_field_id, value`).
- [ ] **S6.4** Field-admin: owner **adds** fields (reorder ok; **no destructive delete** — [[Data model]]).
- [ ] **S6.5** Intake flow: pick/create Customer → pick/create Vehicle (lookup by
      registration number) → create Job → fill jobsheet field values + complaints. One
      screen/flow. **Spec: [[Intake flow]]** (full SA decision tree, both lookup keys,
      two-branch mismatch confirm — designed 2026-07-15).
- [ ] **S6.6** *(Product-gap #1)* ETA: add `promised_ready_at` to Job; SA sets at intake; show on the job.
- [ ] **S6.7** *(Product-gap #9)* vehicle history: on the vehicle/intake screen, list that vehicle's
      prior jobs.
- [ ] **S6.8** Plate normalization + existing-vehicle lookup.
- [ ] **S6.9** Tests: intake creates the full graph · jobsheet values saved · ETA persists · history shows.

---

## Sprint 7 · Owner status page ⚠️ *needs design pass*
**Goal:** the no-login owner view. **Do a short design pass first** (like we did for the model).
**Exit:** owner opens the token link on mobile, sees status, cannot write anything.

- [ ] **S7.1** Design pass: what the owner sees (stage, ETA, are blockers hidden?), token lifecycle
      (expire? revoke?).
- [ ] **S7.2** Public route `GET /jobs/:token` (no auth); find job by `token`.
- [ ] **S7.3** Mobile-first read-only view (stage + ETA, no internal notes unless decided).
- [ ] **S7.4** SA "copy status message" button — generates a message containing the link.
- [ ] **S7.5** Tests: valid token shows the page · bad token 404 · no write paths exist.

---

## Sprint 8 · Reporting & attribution ⚠️ *needs design pass*
**Goal:** workshop health & bottleneck attribution — see [[Event log]]. **Design pass first.**
**Exit:** a manager dashboard + an owner mobile health summary.

- [ ] **S8.1** Design pass: which reports (time-in-stage, time-blocked-by-role, time-to-acknowledge,
      workshop health). *(Candidate from 2026-07-12 timeline talk: per-job **parallel-lanes view** —
      stage/blocker/crew lanes on a proportional time axis, "where did the time go?" Same tracker
      rows as `Job#timeline`, purely a second rendering — no data work needed.)*
- [ ] **S8.2** Scopes/queries per metric — **queries, not tables** ([[Design laws]] #3).
- [ ] **S8.3** Manager dashboard (PC); owner mobile health summary ([[Tech stack]] device posture).
- [ ] **S8.4** Tests: each metric query is correct against seed data.

## Related
- [[Roadmap]] · [[M1-F1 Status flow and transitions]] · [[Data model]] · [[Design laws]] · [[Product gaps]]
