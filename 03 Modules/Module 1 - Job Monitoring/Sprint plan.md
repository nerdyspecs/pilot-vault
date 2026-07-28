---
type: plan
module: M1
created: 2026-07-04
updated: 2026-07-28 (Sprint 4 reshaped by ADR-011 stored-receiver; S4.1/S4.3/S4.4/S4.7 built 982f7e9; colour → S5.7)
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

## Sprints 0–3 — closed
Setup, tenancy spine, the job engine, cold-start intake, and blockers are built, tested, and
shipped. Full task detail (ticks, dated annotations, commit hashes) moved to
[[Sprint plan (Sprints 0-3, closed)]] to keep this file focused on current work.

## Tenant-spine collapse — done before Sprint 3 *(2026-07-21)*
A prerequisite, not a numbered sprint: `WorkshopEmployment` + `WorkshopOwnership` collapsed into
`WorkshopStaff` + `WorkshopStaffRole` ([[ADR-010 WorkshopStaff supersedes the edge split]]) so
every tenant-scoped sprint from here builds on a DB-tenant-checkable actor. Schema squash +
reseed (no prod data), 72/72 green incl. new composite-FK isolation tests. Sprint 3's blocker
event tables adopt `created_by`/`acknowledged_by → workshop_staff` from birth — no rework.

---

## Sprint 4 · Acknowledgement as visibility
**Goal:** the differentiator — see [[ADR-005 Acknowledged handoffs in V1]] as extended by
**[[ADR-011 Acknowledgement as stored visibility]]** (Product-gap #5 gate **settled**). **Exit:** every
active job — including a finished car awaiting delivery — shows *"Waiting on &lt;name&gt;"* on the
board when a handoff hasn't been picked up; acting on a job clears it.

**The frame (ADR-011):** the receiver is **stored at write time** (`receiver_id`), so "waiting on
whom" is a plain query, always answerable regardless of adoption. **No inbox, no routing, no confirm
button** — the answer surfaces on the manager's board and they walk over. The 2026-07-24 *holder*
model was studied and dropped (ADR-011 Rejected alternatives).

- [x] **S4.1** The acknowledgement writer — `982f7e9`. Rename the dormant `acknowledged_by_id` →
      **`receiver_id`** on all three event tables (never populated → no data migration). A 4-method
      **`Acknowledgeable`** concern (`has_receiver?` / `acknowledged?` / `awaiting_from?` /
      `acknowledge!` + `scope :unacknowledged`) included bare in each transition model. The door
      stamps receivers: `joined`→technician, `left`→technician, `mark_done`→intake SA, `send_back`→
      technician, `resolve_blocker`→raiser; `raised` and the terminals pin nobody.
      **Implicit acknowledgement** via `JobActions.acknowledge_pending!` — `transition!`'s first line
      plus the three blocker verbs — under `job.with_lock`, so a refused verb rolls its sweep back.
- [x] **S4.2** ~~Receiver logic~~ **RULED, folded into S4.1** *(reshaped 2026-07-28)*: `left` pins
      the technician (they should know they're off). Removal is only legal at `assigned` before work
      starts and already rolls the job back to `registered` (`app/services/job_actions.rb`), which
      **returns the job to the service advisor by itself** — no orphan, no new columns.
- [x] **S4.3** **The board read** — `982f7e9`. `Job.pending_acknowledgements_by_job(jobs)` (three
      flat queries, no N+1) and `WorkshopStaff#pending_acknowledgements`, both a plain
      `receiver_id IS NOT NULL AND acknowledged_at IS NULL`. **Replaces the old holder derivation and
      deletes the `.pending_ack` predicate** — the stored NULL check *is* "not a handoff", so no
      classifier is needed ([[Event log]]).
- [x] **S4.4** **The board surface (not an inbox)** — `982f7e9` + `8fad8c9` (done pin). A muted
      *"Waiting on &lt;name&gt;"* line on each board row (`jobs/_waiting_pin`, `jobs/_board_row`), on
      `workshops#show`. **Done is not terminal:** a **"Done — awaiting delivery"** group surfaces the
      finished-car pin, since `Job.active` excludes `done` (the founding pain). Mobile-verified at
      375px. There is no receiver "inbox" and no sender-side view — one board, read by the manager.
- [ ] **S4.5** ~~Ageing colour~~ — **moved to S5.7** *(deferred 2026-07-28)*. The pin ships muted;
      amber/red at a threshold, on the chip never the row, is a styling pass after real use. Bands +
      the overnight-hours wart parked in [[Deferred design]].
- [x] **S4.6** *(Product-gap #5 — the mechanisms)* satisfied by the above: **stored receiver read**
      (S4.3) · **board surface** (S4.4) · **acting confirms** (S4.1) · colour deferred (S5.7).
      **Surviving law: never deleted, never faked** — no auto-ack, no expiry, no purge; a terminal
      verb writes no closing lie, the open row stays honestly NULL and just leaves the board.
- [x] **S4.7** Tests — `982f7e9`. 13 new (102 unit + 9 system green): receiver correct per verb ·
      an unconfirmed pass stays put and acting clears it · a refused verb rolls its sweep back ·
      **the append-only bug regression** (editing `cleared_by_role` doesn't re-point open handoffs) ·
      a counter-only workshop generates zero confirmation traffic and still reads correctly (the
      Product-gap #5 proof, executable).
- [ ] ~~**S4.8** Directed blocker notes (B2)~~ — **deferred again** (ADR-011). Still additive (one
      nullable `directed_to_role`, no rework), and `raise` pins nobody precisely so a second traffic
      generator waits for a real workshop. See [[Blocker]] B1/B2 split · [[Deferred design]].

---

## Sprint 5 · Live job list
**Goal:** answer "where is every job right now?" **Exit:** a filterable board of active jobs.

- [ ] **S5.1** Jobs index: active jobs for `Current.workshop` (exclude `delivered/cancelled`), showing
      stage, crew, active blockers.
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
