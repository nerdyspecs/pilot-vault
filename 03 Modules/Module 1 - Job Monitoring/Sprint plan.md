---
type: plan
module: M1
created: 2026-07-04
updated: 2026-08-25 (Session 36 — design source of truth consolidated into [[Design system]] (renamed from Visual theme); S5.5a/b re-pointed and widened after verifying the Bay prototype live; prior: S5.5b now carries the Visual theme re-lock and S5.5a the /design-preview route, after the Bay comparison; S5.5c absorbs the law-9 width fix; S5.5 rewritten as the UI build order (S5.5a–i): four cross-cutting fixes before any screen, then add-vehicle and the happy-path intake front door; S5.8 superseded/split; plan of record is [[Design system]]; prior: Session 35 — S5.1a/S5.1b built + ticked; S5.1c ticked (no-op), S5.1d core done (value-type/completion + R11 deferred); build footnotes: re-snapshot dropped (attr_readonly write-once), complaint moved to jobsheets header; prior: Session 34 — jobsheet storage decided, ADR-015; S5.1 (rev) split into four build tasks S5.1a–d; S5.5 split (complaints vs. inspection fill); S5.9 retargeted; prior: Session 33 — jobsheet reversed to fixed/product-defined, ADR-014; old S5.1–S5.4 EAV tasks struck/dropped, replaced by S5.1 (rev) pointing at the design brief; prior: 2026-08-17 Session 32 Sprint 5 ↔ 6 renumbered; prior: 2026-08-14 Sprint 4.5 built + ticked, S4.5.5 rewritten, S4.5.9/.10 added — ADR-013;
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

> [!note] Build order *(2026-08-17, Session 32)*
> **Build order: 4.5 (closing) → 5 (next) → 6 (deferred) → 7 → 8.** The **intake vertical
> (Sprint 5) goes next, led by the jobsheet** — it's the create-path the whole app is missing
> (`bin/route-orphans` finds two orphan endpoints, both intake creates) and it's genuine model work.
> The **board (Sprint 6) is deferred** — query-over-existing-tables plus heavy UI, best built once
> real intakes flow and the designer has [[Screen map]] / [[Screen flow]].
> *(Sprint 5 ↔ 6 hard-swapped 2026-08-17: the intake vertical was Sprint 6, the board Sprint 5.
> Neither is built, so no commit ids broke; ADR-011/012 keep their original numbers with a
> forward-pointer footnote. See [[worklog]] Session 32.)*

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
the two-level aggregate is live and green, so the Sprint 6 board is built on the right unit.
**Why here, not later** — [[ADR-012 Intake-Job two-level aggregate]]: the S6 board groups by the *car*,
so it must sit on `Intake`; and with **no production data** this is the cheapest the schema change will
ever be (schema squash + reseed, like the tenant-spine collapse — never a data migration). Building S6
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
      for the Permissions/ADR-013 work and the vault catch-up — the one open S4.5 task; it completes the
      aggregate's own read surface but does **not** block Sprint 5, so land it when convenient.)*

> [!note] Downstream sprints read against ADR-012 + ADR-013
> **S6** groups the board by **Intake** (the car), with a *Done — awaiting delivery* group for intakes
> deriving `ready`. **S5** intake flow creates Customer/Vehicle/**Intake**/Job (via `CreateIntake`, not
> `Job.create!`). **S7**'s token page is now the **intake's** token — `GET /intakes/:token`, not
> `/jobs/:token`. See the task-level notes in each sprint below for what specifically needs re-pointing.

---

## Sprint 5 · Intake + fixed inspection jobsheet
**Goal:** register a car end-to-end — see [[ADR-014 Jobsheet is a fixed product-defined inspection]]
(supersedes [[ADR-003 Digitized jobsheet in V1]]'s owner-configurable core), [[Data model]].
**Exit:** full intake creates Customer/Vehicle/**Intake** (+ its first Job) + jobsheet answers in
one flow. *(⚠ 2026-08-14, ADR-012: was "…+ Job" — the visit is the thing created; a Job is its
first repair, via `CreateIntake`.)* This is also where **UI work resumes** — see the Sprint plan's
top-of-file warning: the current app has no working intake-creation UI at all.

> [!note] ▶ Next *(2026-08-19, Session 34)* — jobsheet storage decided; build is what's left
> [[ADR-015 Jobsheet answers are rows against a frozen question set]] resolves what
> [[Inspection jobsheet — design brief]] chipped out: code-defined templates + a thin `Jobsheet`
> header (1:1 per Intake, carrying `inspection_type` + a frozen `item_keys` snapshot) +
> field-level `JobsheetAnswer` rows (`choice`/`measurement` split, `inspected_by`, explicit
> `not_applicable`). No door — nothing to veto, concurrent multi-actor fill. What remains is
> **S5.1a–d below** (migration → model/catalog → seed → tests, each its own commit, per house
> pattern). See [[worklog]] Session 34.

> [!note] Superseded 2026-08-19 — jobsheet reversed to fixed/product-defined; storage chipped out
> [[ADR-014 Jobsheet is a fixed product-defined inspection]] reverses ADR-003's owner-configurable
> core: the jobsheet's fields are set by the product (versioned in code), not owner-CRUD at
> runtime. The EAV build (`JobSheet`/`JobSheetField` template models, branch
> `s5-jobsheet-models`, 4 commits) is **discarded** — see old S5.1–S5.4 below, all struck. A
> concrete 39-item field list (5 sections, mixed answer types) surfaced and reopens the storage
> question (wide typed table vs. code-defined catalog + answer rows vs. jsonb) — **not decided
> here**, chipped out to [[Inspection jobsheet — design brief]] for a future session to design and
> build from scratch off `main`. Two fill-layer decisions from Session 32 are folded into that
> brief rather than parked separately: the freeze condition, and jobsheet-in-the-intake-form vs. a
> later step. See [[worklog]] Session 33. **Storage decided, Session 34 — see the note above.**

- ~~**S5.1** Migration + model **JobSheet** (`workshop_id`, one per workshop).~~
- ~~**S5.2** Migration + model **JobSheetField** (`job_sheet_id, label, kind:integer(checkbox/text),
      position`).~~
- ~~**S5.3** Migration + model **JobSheetFieldValue** (`intake_id, job_sheet_field_id, value`).~~
- ~~**S5.4** Field-admin: owner **adds** fields (reorder ok; **no destructive delete**).~~

  **Dropped 2026-08-19, [[ADR-014 Jobsheet is a fixed product-defined inspection]].** S5.1/S5.2
  were built (branch `s5-jobsheet-models`, discarded) then reversed once configurability was
  judged the source of the print-control/versioning/drift problems; S5.3 (EAV values) and S5.4
  (owner field-admin) never applied to begin with once the fields aren't owner-CRUD. Replaced by:

  ~~**S5.1 (rev)** — fixed inspection jobsheet: design storage structure + build model/migration/
      tests (see [[Inspection jobsheet — design brief]]). Supersedes old S5.1–S5.4 per
      [[ADR-014 Jobsheet is a fixed product-defined inspection]].~~

  **Storage designed 2026-08-19, [[ADR-015 Jobsheet answers are rows against a frozen question set]].**
  Split into the four-layer build, each its own commit, per house pattern:

- [x] **S5.1a** *(2026-08-19, `6041a0b`)* Migration — `jobsheets` (`workshop_id, intake_id` unique,
      `inspection_type`, `item_keys` string array, `complaint` text — see the S5.5 footnote) +
      `jobsheet_answers` (`workshop_id, jobsheet_id, item_key` unique
      per jobsheet, `choice, measurement, not_applicable, note, inspected_by_id`). RLS +
      `workshop_id` + composite FK on `inspected_by_id`, per house pattern (see
      `app/models/concerns/workshop_scoped.rb` and the intake/blocker migrations for the concrete
      shape). → [[ADR-015 Jobsheet answers are rows against a frozen question set]].
- [x] **S5.1b** *(2026-08-19, catalog `c5b8977` + models `a1889a3`)* Models + catalog — `Jobsheet` (`has_many :jobsheet_answers`, snapshots
      `item_keys` from the template on create/re-select-while-empty), *(⚠ 2026-08-19 — re-snapshot
      dropped at build: `inspection_type` and `item_keys` are both `attr_readonly`, write-once on
      create, so there is no re-select-while-empty path. A mis-typed sheet is deleted and
      recreated. Narrows ADR-015's re-snapshot rule; the frozen question set itself is unchanged.
      See [[Data model]] §The jobsheet for the full footnote.)* `JobsheetAnswer`
      (`item_key` inclusion-validated against its jobsheet's own `item_keys`; only the value
      column matching the catalog's `answer_type` populated), and
      `app/inspections/car_routine_inspection.rb` + `lorry_routine_inspection.rb` — the
      code-defined catalog modules (lorry mirrors car for now), fronted by an `Inspection` registry.
- [x] **S5.1c** *(2026-08-19 — n/a)* Seed — moot: the catalog is code (`c5b8977`), not DB rows, so
      the 39-item list ships in `car_routine_inspection.rb`, nothing to seed. Kept as a ticked
      no-op so the four-layer numbering stays intact.
- [~] **S5.1d** *(2026-08-19, `4b77664` — core done, two rules deferred)* Tests: `item_key` bounded
      to the jobsheet's frozen list · both DB CHECKs (one value column; `not_applicable` orthogonal)
      · frozen once the intake leaves `open` · one-per-item · RLS/tenancy + composite-FK isolation,
      same shape as the blocker tests. **Deferred** (need catalog *content*, the churny part): the
      `value_matches_answer_type` (numeric→measurement/else→choice) and `complete?` (required set)
      rules — write once the flagged content calls settle, or via a registry stub. **R11 orphan
      protection is NOT a unit test** — it's a deploy-time data-integrity check (see [[Risk ledger]]).
- [ ] **S5.5** ~~Intake flow: pick/create Customer → pick/create Vehicle → `CreateIntake` → complaint
      → jobsheet, one screen/flow.~~ **Rewritten 2026-08-25 (Session 36) into the UI build order —
      plan of record: [[Design system]].** The design session widened past the intake screen:
      the problems that block *every* screen turned out to sit above the screen level (no nav
      model, one posture of three, UI law 3 broken on `jobs/show`/`home/index`, law 2 broken in the
      layout so on all 27 templates, law 7 broken by duplicated `_blocker_item` twins). **The rule
      driving the order below: do the things that change every screen before drawing any screen.**
      Sub-tasks run in order; each leaves the app walkable one notch further than before.
      *(Prior footnotes preserved: ⚠ 2026-08-19 ADR-015 — the complaint is free text, never a fixed
      inspection item; ⚠ corrected at build 2026-08-19 — it ships on the `jobsheets` header, not on
      Intake; ⚠ 2026-08-14 ADR-012/013 — creation goes through `CreateIntake`, never a bare
      `Job.create!` or a door verb.)*
- [ ] **S5.5a** *(cross-cutting)* **Component inventory** — write it into [[Design system]] §Components,
      **build the `/design-preview` route** (a page rendering every component once, in every state),
      and **merge the `_blocker_item` twins** (`app/views/intakes/_blocker_item.html.slim` and
      `app/views/jobs/_blocker_item.html.slim` differ by six lines, all naming). UI law 7 has no
      enforceable meaning until the components are named; the review route makes it *checkable*
      rather than asserted, and it is where the re-derived status chips get the sample comparison
      [[Design system]] says they still owe. **`/design-preview` is the third layer of the design
      source of truth** (note → `application.css` → rendered page); the reference implementation
      declared it and never built it, so Knot is the first to actually populate it.
- [ ] **S5.5b** *(cross-cutting)* **The visual re-lock** — apply [[Design system]]'s 2026-08-25
      restyle in `application.css`, all through Bootstrap variables and still never forking the
      vendored file. **Largely a transcription** — `08 Experiments/knot-board-desktop.html` already
      carries the intended `:root` block ([[UI experiments]] §3). Contents: `--font-sans` →
      Helvetica/Arial · `--action` `#2727D9` (`--brand` retired) · `--page`/`--surface` **white**,
      `--hover` `#F4F4F2` · two border weights (`--border` `#D8D8D4`, `--rule` `#E5E5E1`) · the muted
      ink ramp · `--bs-border-radius` family → `0` · `--bs-body-font-size` → `.875rem` · negative
      tracking in `em` · the six component dimensions · the three **floating-layer shadows** (inline
      structure stays flat) · **`.stage-badge` re-cut from two-part to border + fill + text** ·
      **flashes moved off `.alert-info`/`.alert-warning`**, which were wearing reserved status hues
      (UI law 2). **No spacing values** — those come from Bootstrap utility classes in the templates
      ([[Design system]] §Sizing and spacing). A token-level restyle, not a rewrite: only the badge
      and the flash need markup changes.
- [ ] **S5.5c** *(cross-cutting)* **The shell** ([[Design system]] §Components + L1): one header, one back
      affordance (four wordings exist today), reachability for Customers/Crew/Blocker types (which
      exist only as buttons on the board), **and page width** — every workshop screen is currently
      `col-12 col-md-8 col-lg-6`, a centred half-column, which is the stretched phone layout UI
      law 9 forbids. Width and sidebar-vs-centred are **one decision**. **The one item here that is
      genuinely expensive to retrofit** — every screen's header is downstream of it. ⚠ If the
      sidebar pattern is taken, it drags in Stimulus (no Bootstrap JS is loaded) and an icon source
      for the rail — one coupled call, recorded as open.
- [ ] **S5.5d** *(cross-cutting)* **UI law 3 audit**: `jobs/show` carries four solid `btn-primary`,
      `home/index` three. Each picks its one next step. Deciding it changes those screens' layout,
      so it precedes any redesign of them. **Settle the rule itself here too** — "one primary per
      *screen*" (Knot) vs "per *local area*" (Bay): `jobs/show`'s four primaries are four genuinely
      distinct regions, so either the law gains a density qualifier or those screens restructure.
      Open call in [[Design system]] UI laws.
- [ ] **S5.5e** **Customers — the search-first rule.** `Customer.search` already matches name *or*
      canonicalized phone. Make "New customer" reachable only *through* an empty search result,
      never as a peer button beside the search. This is the dedup guard that S5.5f makes necessary
      (see the [[Intake flow]] §2a consequence recorded in [[Design system]] L3).
- [ ] **S5.5f** **Add-vehicle screen — NEW.** No `VehiclesController`, no route, no screen exists;
      today a vehicle can only be born inside the intake flow. Builder ruling 2026-08-25: it becomes
      ordinary CRUD under a customer. The first of the two genuinely new screens, and the smaller.
- [ ] **S5.5g** **`Vehicle.find_by_plate`** + unit tests — the lookup Screen A needs, beside the
      existing `Vehicle.canonicalize` ([[Design laws]] #9: logic you can't call without a request is
      misplaced). Absorbs the live half of S5.8 below.
- [ ] **S5.5h** **Intake front door — NEW, happy path only.** `GET /intakes/new`: one autofocused
      monospace plate field, one primary "Look up", reached from the board's reserved front-door
      slot (`app/views/workshops/show.html.slim` already carries the comment; every sibling control
      there is `btn-outline-secondary`, so one solid `--action` button *satisfies* law 3 rather than
      straining it). Three outcomes: **found** → open the visit ([[Intake flow]] §1a);
      **found with an open intake** → report the car already in house and link to the visit
      (**§1c — this is not polish**: `index_intakes_one_open_per_vehicle` is a partial unique index,
      so keying an in-house plate without it raises `RecordNotUnique`, a 500);
      **not found** → a *routed* dead end to S5.5f, saying what to do next (UI law 8). Widen
      `CreateIntake` and `IntakesController#create` by **`complaint:` only** — not by `customer:`,
      since §1b is deferred (the controller is currently narrower than the service: it does
      `params.require(:vehicle_id)` while `CreateIntake` already accepts `customer:` and
      `inspection_type:`). Keyboard-only on tab/Enter (law 10).
- [ ] **S5.5i** **Tests.** Model/unit coverage per house convention. ⚠ **Open call, not ruled:**
      whether to un-suspend the Capybara system-test layer (suspended 2026-08-14 when the aggregate
      split retired the views it drove; `test/system/` and `application_system_test_case.rb` are
      gone, but **capybara and selenium-webdriver are still in the `Gemfile`** — so it costs one
      harness file, no new dependency). These are the first real pages since, and "does Enter on the
      plate field reach the visit" is exactly what a controller test cannot see.
- [ ] **S5.6** *(Product-gap #1)* ETA: add `promised_ready_at` to **Intake**; SA sets at intake;
      show on the visit. *(⚠ 2026-08-14, ADR-012: was "to Job" — a promised-ready time is a per-visit
      commitment to the customer, not a per-repair one; several repairs on one visit share one ETA.)*
- [ ] **S5.7** *(Product-gap #9)* vehicle history: on the vehicle/intake screen, list that vehicle's
      prior **intakes**.
- [ ] ~~**S5.8** Plate normalization + existing-vehicle lookup.~~ **Superseded 2026-08-25
      (Session 36) — it was half-done and mis-ordered.** The *plate-normalization* half already
      shipped, well before S5.5, as `Vehicle.canonicalize` (called from a `before_validation`, with
      the per-workshop unique index keying on one spelling). The *existing-vehicle lookup* half is a
      **prerequisite of the intake screen, not a follow-on** — folded into **S5.5g**. The phone
      side that once sat here — extracting the canonicalization duplicated between `Customer`'s
      `before_validation` and its `search` scope, plus a `Customer.find_by_phone` — is **not needed
      for the happy path** and moves with the deferred §2a/§2b dedup tree ([[Deferred design]]).
      Extract it before a third caller lands.
- [ ] **S5.9** Tests: intake creates the full graph (incl. its auto-created `Jobsheet`) · complaint
      saved on Intake · jobsheet answers saved (see S5.1d for the model-level answer tests) · ETA
      persists · history shows.

---

## Sprint 6 · Live job list
**Goal:** answer "where is every job right now?" **Exit:** a filterable board of active jobs.

> [!warning] ⏸ Deferred *(2026-08-17, Session 32)* — board pushed back behind the intake vertical (S5)
> Built after Sprint 5. When picked up it's mostly UI over **new** queries not yet written (batched
> `Intake.ready` scope, aging clocks, two-level blocker/pin roll-up, filter scopes) — decompose it
> then, against real intakes + the designer's work off [[Screen map]] / [[Screen flow]]. **S6.2a is
> stale** (a car has no single stage post-ADR-012; it overlaps S6.1a — fold its "sitting time" into
> S6.1a at build time). Do the aging-clock / two-level-display design pass at pick-up, not now.

- [ ] **S6.1** Jobs index: active jobs for `Current.workshop` (`Job.active` — `unassigned`/
      `assigned`/`in_progress`; **`delivered` is no longer a Job stage to exclude**, ADR-012), showing
      stage, crew, active blockers.
- [ ] **S6.1a** *(re-pointed 2026-08-14, ADR-012 — not in the original plan)* **Group the index by
      Intake**, not a flat job list — the whiteboard's row was always the car, and a car can now have
      several jobs. A **"Done — awaiting delivery"** group for intakes where `ready?` (every job
      terminal, ≥1 done) — the founding-pain surface, now car-level. See [[Intake]].
- [ ] **S6.2** Filters: by stage, by technician (crew), by blocker.
- [ ] **S6.2a** *(added 2026-07-24, ADR-011)* **Per-job grouping — whiteboard parity**: the car,
      its stage, and how long it has been sitting. The board a workshop already draws, made live.
- [ ] **S6.2b** *(added 2026-07-24, ADR-011)* **Per-technician grouping**: load, long-running
      `in_progress` jobs, last activity. ⚠ **Frame it as the manager's diagnostic on stuck jobs,
      never a scoreboard** — the builder's "saucy one". What keeps it honest is ADR-011's
      **symmetry**: the counter is measured identically (an unconfirmed `done` ages exactly like an
      unconfirmed assignment), so this is not a lens aimed at the floor. Stays inside the
      positioning invariant because it aggregates **jobs**, never real-time technician activity.
- [ ] **S6.3** *(Product-gap #2)* aging highlight: flag jobs sitting in a stage beyond N days.
- [ ] **S6.4** *(optional)* Turbo Streams for near-live updates — a plain refresh is fine first.
- [ ] **S6.5** PC layout primary; a clean "my jobs" view readable on a technician's phone.
- [ ] **S6.6** Tests: tenant scoping of the index · filters · aging flag.
- [ ] **S6.7** *(carried from Sprint 4)* **Ageing colour on the waiting pin** — under 1h neutral ·
      1h–1d amber (the palette's existing *Waiting / aging*) · over 1d red ("stuck, act now", whose
      two causes are a blocker and an unclaimed handoff — the chip text says which). **Colour the
      chip, never the row.** One workshop-wide constant, one place. Bands + the overnight-hours wart:
      [[Deferred design]].

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
  [[ADR-013 The door decomposed]] · [[ADR-014 Jobsheet is a fixed product-defined inspection]] ·
  [[ADR-015 Jobsheet answers are rows against a frozen question set]] ·
  [[Inspection jobsheet — design brief]]
