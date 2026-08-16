---
type: plan
module: M1
created: 2026-07-04
updated: 2026-08-14 (Sprint 4.5 built + ticked, S4.5.5 rewritten, S4.5.9/.10 added — ADR-013;
downstream sprints S5/S6/S7 re-pointed; prior: 2026-08-03 Sprint 4.5 inserted before Sprint 5 — Intake/Job aggregate morph, ADR-012; prior: 2026-07-28 Sprint 4 closed + archived → Sprints 0-4 archive; live file now Sprint 5 onward)
---
# Module 1 — Sprint plan (execution)
Small, assignable tasks per sprint — sized for a junior dev to pick up one at a time. The
[[Roadmap]] is the higher-level slice map; this is the build order. Each sprint is a thin vertical
that ends in something demoable.

## Conventions (read once)
- **Vanilla Rails MVC.** Models (AR: associations, validations, scopes) · controllers (thin, RESTful)
  · views (ERB + Hotwire) · routes (RESTful). No extra layers.
- **The one exception:** all state changes go through a **door** — `JobActions` for a repair,
  `IntakeActions` for the visit (`app/services/`), plain POROs. This is **ONE DOOR, per level**
  ([[Design laws]] #7). *(Renamed from `JobService` 2026-07-12, before any code existed: in
  workshop language a "service" is the repair itself — the old name confused the builder, the
  domain expert, and collided with a likely future `Service` catalog entity. One class,
  granular verb methods — builder rejected a per-action class split to keep the shared guards
  and the door auditable in one file. **⚠ 2026-08-14, [[ADR-013 The door decomposed]]:
  reversed once building it showed the two levels have genuinely different verb shapes — the
  door split by level, creation left for `CreateIntake`/`CreateJob`, and the shared guards left
  for a new `Permissions` class, checked at the controller boundary rather than inside the
  door.)*
- **No gem without justifying why Rails' built-in isn't enough** — currently Devise only ([[Agent guide]]).
- **Tenancy:** every workshop-owned table has `workshop_id`; never query it bare — always scope by
  `Current.workshop` ([[Design laws]] #2).
- **Tests (set 2026-07-08):** two layers, hollow middle. (1) **Minitest model unit tests** are
  the *priority* — calculations live in models ([[Design laws]] #9), so they're fast, isolated,
  and consistent everywhere reused. (2) **Capybara system tests** (`test/system/`, already
  gemmed) for end-to-end flows — written once the pages exist. Controller/request tests
  **deferred** (add when pages are present; low priority — pages are isolated, shared logic isn't).
  **⚠ 2026-08-14: layer (2) is suspended.** The Sprint 4.5 aggregate split retired the views the
  system tests drove; the suite was deleted rather than left red against a UI that's due a real
  rework. See the warning callout below. Layer (1) is unaffected and green.
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

> [!warning] The app has no working UI path right now (2026-08-14)
> Sprint 4.5's backend restructure retired `Customers::JobsController` and the routes it served;
> `workshops#show` and `customers#show` both raise on route helpers that no longer exist, and
> `customers/jobs/new.html.slim` is orphaned. **No tasks are scheduled to fix this** — UI work is
> deliberately parked until a real rebuild against the Intake-first flow (Sprint 6). Stated here
> as fact so it isn't mistaken for a regression to chase down.

---

## Sprints 0–4 — closed
Setup, tenancy spine, the job engine, cold-start intake, blockers, and acknowledgement-as-
visibility are built, tested, and shipped. Full task detail (ticks, dated annotations, commit
hashes) moved to [[Sprint plan (Sprints 0-4, closed)]] to keep this file focused on current work.

## Tenant-spine collapse — done before Sprint 3 *(2026-07-21)*
A prerequisite, not a numbered sprint: `WorkshopEmployment` + `WorkshopOwnership` collapsed into
`WorkshopStaff` + `WorkshopStaffRole` ([[ADR-010 WorkshopStaff supersedes the edge split]]) so
every tenant-scoped sprint from here builds on a DB-tenant-checkable actor. Schema squash +
reseed (no prod data), 72/72 green incl. new composite-FK isolation tests. Sprint 3's blocker
event tables adopt `created_by`/`acknowledged_by → workshop_staff` from birth — no rework.

---

## Sprint 4.5 · Intake/Job aggregate morph *(design pass + migration)*
**Goal:** split the overloaded `Job` into **Intake** (the car's visit) → **Job** (one repair). **Exit:**
the two-level aggregate is live and green, so the Sprint 5 board is built on the right unit.
**Why here, not later** — [[ADR-012 Intake-Job two-level aggregate]]: the S5 board groups by the *car*,
so it must sit on `Intake`; and with **no production data** this is the cheapest the schema change will
ever be (schema squash + reseed, like the tenant-spine collapse — never a data migration). Building S5
first would mean rebuilding it.

- [x] **S4.5.1** Design pass — model locked with the builder → [[ADR-012 Intake-Job two-level aggregate]]
      (aggregate, stored-vs-derived status, receivers across two levels, blocker split, the four
      sub-decisions). *(2026-08-03)*
- [x] **S4.5.2** Migration + model **Intake** (`workshop_id, vehicle_id, customer_id, status`;
      `has_secure_token :token` and `registered_by` move here from Job) + **IntakeStatusTransition**
      (append-only, `created_by`, composite `(actor_id, workshop_id)` FK). Busy-vehicle guard lands
      here as `index_intakes_one_open_per_vehicle` (partial unique, `WHERE status = 0`) — the R5
      index moves off `jobs` ([[Risk ledger]] R5). *(2026-08-14, `380ebbc`)*
- [x] **S4.5.3** **Job** morph: `belongs_to :intake`; enum re-split `registered → unassigned` and
      **drop `delivered`** (terminals become `done`/`cancelled`); customer/vehicle reached through the
      intake. Schema **squash + reseed** — `git diff structure.sql` on unchanged tables is the net.
      *(2026-08-14, `380ebbc`+`2070ed9`)*
- [x] **S4.5.4** **Stored `status` + derived `ready?`** (ADR-012 §2): `status` enum
      `{open, delivered, cancelled}` written **only** by a door, which reconciles it after every
      job-terminal move (`reconcile_intake!`); **`Intake#ready?`** stays a pure query — still `open`,
      all jobs terminal, ≥1 done ([[Design laws]] #3). Unit-tested in isolation. *(2026-08-14)*
- [x] **S4.5.5** ~~`JobActions` intake verbs (ONE DOOR stays): `register_job!` opens-or-finds the
      vehicle's open intake then creates the Job~~ **⚠ built differently — see
      [[ADR-013 The door decomposed]].** Creation left the door entirely: **`CreateIntake`** opens a
      visit (+ its first repair, one transaction, never empty); **`CreateJob`** adds a repair to an
      intake it is *given*. A new **`IntakeActions`** class (not `JobActions`) holds **`deliver!`**
      (requires every job terminal + ≥1 done, **sweeps `acknowledge_pending!` across all its jobs**)
      and **`cancel_intake!`** (cancels the remaining *open* jobs, done survives, terminal derives).
      Busy-vehicle guard → **one open intake per vehicle**, same as designed. `register_job!` is
      **deleted**; its opens-or-finds behaviour survives only as a test helper. *(2026-08-14, `33f426d`)*
- [x] **S4.5.6** **Blocker split by `blocks`**: `blocks: done` stays a `JobBlocker` vetoed in
      `Job#transition!`; **`blocks: delivered`** becomes an **`IntakeBlocker`** (+ `IntakeBlockerTransition`,
      note chain, **no ack pair** — non-acknowledgeable, per ADR-012) vetoed in intake `deliver!`. Shared
      `Blocker` catalog; widen `FORWARD_STAGES` / the DB `CHECK`. Reseed HFP as an intake blocker.
      *(2026-08-14, `380ebbc`+`33f426d`)*
- [x] **S4.5.7** Tests: derived status · deliver requires-all-terminal + the delivery sweep closes
      done-notices · `cancel_intake!` cascade (all-cancelled → cancelled; mixed → ready) · blocker veto
      on the correct door · composite-FK isolation on the new intake event tables. Also migrated every
      `register_job!` call site and extracted the role-gating assertions that no longer belong in the
      door's own tests into a new `permissions_test.rb` (see S4.5.9). 129 runs, 0 failures, 0 errors.
      *(2026-08-14, `636d47b`)*
- [x] **S4.5.8** Concept-note reconciliation: [[Stage model]], [[Job]], [[Blocker]], [[Event log]],
      [[M1-F1 Status flow and transitions]], [[Data model]], [[Job visibility]] (re-anchored to
      Intake — token/customer_id moved, Sprint 7 builds against the corrected version), plus
      **[[Overview]]** and **[[Open questions]]**, not on the original list but equally stale, and a
      **new [[Intake]]** note. (Roadmap slice 6 and [[Risk ledger]] R5 annotated 2026-08-03 during
      the vault sweep — done, not part of this task.) Annotated, not rewritten — the ADR-010/011
      pattern. [[Intake flow]] intentionally left alone: its content is the S6 UI spec, unaffected by
      the backend split. *(2026-08-14)*
- [x] **S4.5.9** *(not originally planned — discovered while building S4.5.5)* Authorization moved
      out of the door to a new **`Permissions`** class, checked at the controller boundary instead of
      inside `JobActions`/`IntakeActions` → [[ADR-013 The door decomposed]]. [[Design laws]] #7
      reworded ("ONE DOOR" → "ONE DOOR per level"), #9 footnoted. Controllers that previously had
      **no controller-level gate at all** (blocker + technician controllers — authorization lived
      only inside the door) gained visible `before_action` authorization tables + named param
      methods. *(2026-08-14, `33f426d`)*
- [ ] **S4.5.10** **`Intake#timeline`** (the visit's own status + blocker events, merged) and
      **`Intake#timeline_with_jobs`** (the deep trace — also pulls every job's events, batched flat
      like `Job.pending_acknowledgements_by_job`, not N+1 per job). Needs a new
      `has_many :intake_blocker_transitions` on Intake first — the table has a direct, indexed
      `intake_id` but no association exposes it yet. A `handoff_state(event)` presentation helper
      (history / waiting / picked-up) for the merged feed, since intake events are never
      `Acknowledgeable`. Designed, not built — see [[Intake]] §Timeline. *(This is what the
      builder originally asked for when this reconciliation pass started; parked mid-session
      for the Permissions/ADR-013 work and the vault catch-up — pick up here next.)*

> [!note] Downstream sprints read against ADR-012 + ADR-013
> **S5** groups the board by **Intake** (the car), with a *Done — awaiting delivery* group for intakes
> deriving `ready`. **S6** intake flow creates Customer/Vehicle/**Intake**/Job (via `CreateIntake`, not
> `Job.create!`). **S7**'s token page is now the **intake's** token — `GET /intakes/:token`, not
> `/jobs/:token`. See the task-level notes in each sprint below for what specifically needs re-pointing.

---

## Sprint 5 · Live job list
**Goal:** answer "where is every job right now?" **Exit:** a filterable board of active jobs.

- [ ] **S5.1** Jobs index: active jobs for `Current.workshop` (`Job.active` — `unassigned`/
      `assigned`/`in_progress`; **`delivered` is no longer a Job stage to exclude**, ADR-012), showing
      stage, crew, active blockers.
- [ ] **S5.1a** *(re-pointed 2026-08-14, ADR-012 — not in the original plan)* **Group the index by
      Intake**, not a flat job list — the whiteboard's row was always the car, and a car can now have
      several jobs. A **"Done — awaiting delivery"** group for intakes where `ready?` (every job
      terminal, ≥1 done) — the founding-pain surface, now car-level. See [[Intake]].
- [ ] **S5.2** Filters: by stage, by technician (crew), by blocker.
- [ ] **S5.2a** *(added 2026-07-24, ADR-011)* **Per-job grouping — whiteboard parity**: the car,
      its stage, and how long it has been sitting. The board a workshop already draws, made live.
- [ ] **S5.2b** *(added 2026-07-24, ADR-011)* **Per-technician grouping**: load, long-running
      `in_progress` jobs, last activity. ⚠ **Frame it as the manager's diagnostic on stuck jobs,
      never a scoreboard** — the builder's "saucy one". What keeps it honest is ADR-011's
      **symmetry**: the counter is measured identically (an unconfirmed `done` ages exactly like an
      unconfirmed assignment), so this is not a lens aimed at the floor. Stays inside the
      positioning invariant because it aggregates **jobs**, never real-time technician activity.
- [ ] **S5.3** *(Product-gap #2)* aging highlight: flag jobs sitting in a stage beyond N days.
- [ ] **S5.4** *(optional)* Turbo Streams for near-live updates — a plain refresh is fine first.
- [ ] **S5.5** PC layout primary; a clean "my jobs" view readable on a technician's phone.
- [ ] **S5.6** Tests: tenant scoping of the index · filters · aging flag.
- [ ] **S5.7** *(carried from Sprint 4)* **Ageing colour on the waiting pin** — under 1h neutral ·
      1h–1d amber (the palette's existing *Waiting / aging*) · over 1d red ("stuck, act now", whose
      two causes are a blocker and an unclaimed handoff — the chip text says which). **Colour the
      chip, never the row.** One workshop-wide constant, one place. Bands + the overnight-hours wart:
      [[Deferred design]].

---

## Sprint 6 · Intake + digitized jobsheet
**Goal:** register a car end-to-end — see [[ADR-003 Digitized jobsheet in V1]], [[Data model]].
**Exit:** full intake creates Customer/Vehicle/**Intake** (+ its first Job) + jobsheet answers in
one flow. *(⚠ 2026-08-14, ADR-012: was "…+ Job" — the visit is the thing created; a Job is its
first repair, via `CreateIntake`.)* This is also where **UI work resumes** — see the Sprint plan's
top-of-file warning: the current app has no working intake-creation UI at all.

- [ ] **S6.1** Migration + model **JobSheet** (`workshop_id`, one per workshop).
- [ ] **S6.2** Migration + model **JobSheetField** (`job_sheet_id, label, kind:integer(checkbox/text),
      position`).
- [ ] **S6.3** Migration + model **JobSheetFieldValue** (`intake_id, job_sheet_field_id, value`).
      *(⚠ 2026-08-14, ADR-012: was `job_id` — the jobsheet is the car's intake form, filled once
      per visit, not once per repair. See [[Data model]], [[Intake]].)*
- [ ] **S6.4** Field-admin: owner **adds** fields (reorder ok; **no destructive delete** — [[Data model]]).
- [ ] **S6.5** Intake flow: pick/create Customer → pick/create Vehicle (lookup by
      registration number) → **`CreateIntake`** (opens the visit + its first repair) → fill jobsheet
      field values + complaints. One screen/flow. *(⚠ 2026-08-14, ADR-012/013: was "→ create Job" —
      creation goes through `CreateIntake`, not a bare `Job.create!` or a door verb.)* **Spec:
      [[Intake flow]]** (full SA decision tree, both lookup keys, two-branch mismatch confirm —
      designed 2026-07-15; unaffected by the backend split, still the UI spec to build against).
- [ ] **S6.6** *(Product-gap #1)* ETA: add `promised_ready_at` to **Intake**; SA sets at intake;
      show on the visit. *(⚠ 2026-08-14, ADR-012: was "to Job" — a promised-ready time is a per-visit
      commitment to the customer, not a per-repair one; several repairs on one visit share one ETA.)*
- [ ] **S6.7** *(Product-gap #9)* vehicle history: on the vehicle/intake screen, list that vehicle's
      prior **intakes**.
- [ ] **S6.8** Plate normalization + existing-vehicle lookup.
- [ ] **S6.9** Tests: intake creates the full graph · jobsheet values saved · ETA persists · history shows.

---

## Sprint 7 · Owner status page ⚠️ *needs design pass*
**Goal:** the no-login owner view. **Do a short design pass first** (like we did for the model).
**Exit:** owner opens the token link on mobile, sees status, cannot write anything.

- [ ] **S7.1** Design pass: what the owner sees (stage, ETA, are blockers hidden?), token lifecycle
      (expire? revoke?).
- [ ] **S7.2** Public route `GET /intakes/:token` (no auth); find intake by `token`. *(⚠ 2026-08-14,
      ADR-012: was `/jobs/:token` — the token moved to Intake; the owner's page is the car's visit,
      which may show several repairs, not one job. See [[Job visibility]].)*
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
- [[Roadmap]] · [[M1-F1 Status flow and transitions]] · [[Data model]] · [[Intake]] ·
  [[Design laws]] · [[Product gaps]] · [[ADR-012 Intake-Job two-level aggregate]] ·
  [[ADR-013 The door decomposed]]
