---
type: plan
module: M1
created: 2026-07-04
updated: 2026-07-12
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
- Each task links the design doc that explains the *why*.

---

## Sprint 0 · Setup (mechanical)
**Goal:** a deployed Rails app a user can log into. **Exit:** app boots locally + on PaaS, login works.

- [x] **S0.1** Strip Rails 8 defaults that fight our ADRs: remove `Dockerfile`, `.dockerignore`,
      Kamal config (no Docker — [[ADR-001 Core stack]]); disable Solid Queue/Cache/Cable (no
      background jobs v1 — [[Tech stack]]). *(2026-07-05: also dropped `thruster` + stray binstubs;
      cable → async; prod DB collapsed to one)*
- [x] **S0.2** Confirm Ruby 3.2.8 / Rails 8.0.5; `bundle install`; `bin/setup` runs clean.
- [x] **S0.3** `rails db:create db:migrate` — Postgres dev + test databases. *(2026-07-05: Homebrew
      postgresql@17 (17.5) as the local server; dbs `pilot_development`/`pilot_test` confirmed)*
- [x] **S0.4** Add Devise; `rails g devise:install`; `rails g devise User` (email + password only — no
      role, no org — [[ADR-004 Multi-tenant foundation]]). Migrate. *(2026-07-05: stock 5 modules,
      zero extra columns; commit `f08e29e`)*
- [x] **S0.5** Root route + home page: public **marketing landing** (signed out) + minimal
      signed-in stub ("Signed in as {email}"). Also delivered: branded **Devise auth pages**
      (sign in/up, forgot/reset) on a dedicated no-appbar auth layout.
      *(2026-07-06/07: outgrew the original "page behind `authenticate_user!`" wording — the
      root is public by design (marketing page); the gate moved off the root. The auth-gated
      destination becomes the workshop dashboard once S1.10's edge-count landing exists; until
      then the signed-in stub stands in. Slim + Bootstrap adopted along the way — see
      [[Tech stack]] + worklog Sessions 6–7. 2026-07-07: commit `c8134bb`.)*
- [x] **S0.6** Move the Obsidian vault into the repo as `docs/`. *(2026-07-07: done with a
      privacy revision — `docs/` is **gitignored** in the app repo and carries its **own private
      git history** (root commit `ca72c49`; `.obsidian/`, `.claude/`, `.DS_Store` excluded).
      App-repo trace: gitignore commit `e68b71b`. Obsidian re-points to
      `~/teckhong/pilot/docs`; Claude sessions open at the repo root. 2026-07-08: folder
      renamed `docs/` → **`vault/`** (builder preference); gitignore + CLAUDE.md pointers
      updated, app-repo commit `fc821b7`.)*
- [x] **S0.7** Generate `CLAUDE.md` at repo root from [[Agent guide]]. *(2026-07-07: written —
      task-scoped reading lists into `docs/`, working mode, hard invariants incl. ADR-006 +
      Slim/Bootstrap rules, tracking conventions, dev quirks. **Untracked by design** (private,
      references private docs), so no commit hash — it lives on disk only.)*
- [ ] **S0.8** Deploy skeleton to Render/Heroku (managed Postgres, env vars, Procfile/`render.yaml`);
      verify the deployed login page loads. *(2026-07-08: builder decided — **deferred to
      Sprint 1's exit**. Deploy the skeleton+tenancy together; revisit when S1.12 passes.
      Prereq done: private GitHub remote `nerdyspecs/pilot` exists, `main` pushed.)*
- [x] **S0.9** First commit(s), tagged `S0`. *(2026-07-08: branches unified — `v1-db-setup`
      fast-forwarded into `main` and deleted; `main` = `fc821b7` pushed to private GitHub +
      annotated tag `S0`. Note: GitHub immediately opened 3 dependabot PRs (Rails 8.1.3 +
      two Actions bumps) — Rails stays pinned 8.0.5, handle deliberately later.)*

---

## Sprint 1 · Tenancy spine
**Goal:** the multi-tenant foundation — [[ADR-004 Multi-tenant foundation]] as revised by
[[ADR-006 Ownership separate from Employment]] and [[ADR-007 Row-Level Security pulled into
Sprint 1]]. *(Rewritten 2026-07-07 per ADR-006: Ownership edge added, `owner` dropped from the
role enum, bootstrap split into signup vs create-workshop, landing routes by edge count. Old
S1.x numbering retired — no commits referenced it. **Renumbered again 2026-07-08 per ADR-007:**
S1.8–S1.9 inserted for Postgres RLS; old S1.8–S1.12 shifted to S1.10–S1.14 — safe, none had
commits yet. S1.1–S1.7 untouched, already committed.)*
**Exit:** signup creates a bare user; "create workshop" creates `Workshop` + founder `Ownership`
atomically; a 2nd user joins via employment; ending an employment kills access next request;
**a query with no `WHERE workshop_id` still cannot see another tenant's rows** (RLS backstop).

- [x] **S1.1** Migration + model **Workshop** (`name`, timestamps). *(2026-07-08: `name`
      `null: false` + presence validation. Commit `6a7e82a`.)*
- [x] **S1.2** Migration + model **Ownership** (`user_id`, `workshop_id`; unique per pair) —
      the governance edge ([[ADR-006 Ownership separate from Employment]]). Not `CompanyOwner` —
      "Company" is reserved for the v2 fleet entity. *(2026-07-08: composite unique index +
      model-level uniqueness validation; `User#owned_workshops`, `Workshop#owners` associations.
      Commit `6a7e82a`.)*
- [x] **S1.3** Migration + model **Employment** (`user_id`, `workshop_id`, `role:integer`,
      `ended_at`); `belongs_to :user, :workshop`. *(2026-07-08: `scope :active`. Commit `6a7e82a`.)*
- [x] **S1.4** `role` enum on Employment: `technician / service_advisor / parts_advisor /
      workshop_manager` — **no `owner`** (ADR-006; the wrenching towkay holds both edges).
      *(2026-07-08: commit `6a7e82a`.)*
- [x] **S1.5** Partial unique index: one **active** employment per `(user_id, workshop_id)` where
      `ended_at IS NULL`. *(2026-07-08: DB partial index + matching Ruby validation
      (`conditions: -> { active }`) — belt and suspenders. Smoke-tested: ended employment +
      new active one for the same pair coexist correctly. Commit `6a7e82a`.)*
- [x] **S1.6** `Current` (`ActiveSupport::CurrentAttributes`) with `workshop` + `employment`
      (+ `ownership` when acting as owner). *(2026-07-08: `app/models/current.rb`, 3 attributes.
      Commit `efb919d`.)*
- [x] **S1.7** **The access door**: ONE method resolving a user's access to a workshop —
      active `Employment` OR `Ownership` — called from an `ApplicationController` before_action
      that sets `Current` from session, **re-verified every request**; reject if no edge
      ([[ADR-004 Multi-tenant foundation]] / ADR-006 §3). *(2026-07-08: `User#access_for` +
      `set_current_context` before_action + `require_workshop!` helper (unused until S1.8-10
      wire a gated controller). Smoke-tested: no edge → nil; ownership+employment both resolve;
      ending employment while ownership remains still grants access; wrong workshop → nil.
      Also fixed two unrelated pre-existing bugs found while running the full suite: stale
      `users.yml` fixture (blank emails collided on the new unique index, dating to S0.4) and
      `home_controller_test.rb`'s leftover `home_show_url` (dating to S0.5's index rename).
      Commits `c3f0d36`, `efb919d`.)*
- [x] **S1.8** Postgres RLS setup: non-superuser, table-owning app DB role `pilot_app` (LOGIN);
      `config/database.yml` dev/test point at it. *(2026-07-08: superusers **silently bypass**
      RLS — the connection now runs as `pilot_app` (verified `superuser: false`) instead of the
      macOS superuser, so RLS actually bites. Transferred DB/schema/table ownership; dropped
      fixtures (`fixtures :all`) since fixture loading needs superuser privilege the app role
      correctly lacks. Suite green as `pilot_app`. Commit `94f9d7e`.)*
- [x] **S1.9** RLS policies + wiring. *(2026-07-08: `ENABLE` + `FORCE ROW LEVEL SECURITY` +
      `CREATE POLICY tenant_isolation` landed together in one migration on `ownerships` and
      `employments` (never policed-without-a-policy, which would deny everyone) — `users`/
      `workshops` stay unpoliced by design, since the access door reads them before a tenant is
      known. Policy: `workshop_id = NULLIF(current_setting('app.workshop_id', true), '')::bigint`
      — the `NULLIF` guards a real gotcha: `RESET` on a custom GUC restores `''`, not `NULL`,
      and a bare `::bigint` cast on `''` raises instead of denying.
      Chose **session-local `SET`, reset-first every request** over transaction-local
      `set_config(..., true)`: Rails doesn't wrap requests in one transaction (each bare
      statement autocommits), so a transaction-local setting would die before the next query in
      the same request. Session-local persists correctly within a request; the pool-leak risk is
      closed by resetting `app.workshop_id` at the very top of `set_current_context` before
      anything else runs, so a pooled connection can never carry a stale tenant into a new
      request even if the prior one errored. The GUC is set to the *candidate* workshop **before**
      `access_for`'s Ownership/Employment lookup (those tables are now policed — an unset GUC
      would hide real access, not just wrong-tenant rows), then rolled back if access isn't
      confirmed. Switched `config.active_record.schema_format = :sql` first — `schema.rb` has no
      `CREATE POLICY` vocabulary and would silently drop RLS from the test DB snapshot.
      Verified: unset GUC → 0 rows though data exists; GUC set to seed workshop → exactly its
      rows (5 employments, 2 ownerships); reset → back to 0; superuser `psql` still sees all
      (documented escape hatch); all four seed personas (pure owner, pure employment, both-edges
      towkay, edgeless bystander) resolve correctly through the live access door. Suite green.
      Commit `87c78b8`.)*
- [x] **S1.12-pre** Own-rows RLS policies (unplanned, surfaced by S1.12's needs).
      *(2026-07-12: landing routing must list a user's own edges before any workshop is
      selected — `tenant_isolation` alone hid them (no `app.workshop_id` yet → 0 rows → every
      user looked edgeless). Added a second permissive policy `own_rows FOR SELECT` on
      `ownerships`/`employments`, keyed to a new `app.user_id` GUC set at sign-in. Postgres ORs
      permissive policies — visible if in-workshop-context OR about-you. `FOR SELECT` only:
      reads about yourself, never writes. Verified as `pilot_app`: no notes → 0/0; each seed
      persona's note → exactly their own edges; INSERT with only the user note → denied.
      ADR-007 annotated with two dated footnotes (this addition + the S1.9 session-local-`SET`
      deviation, on record for the first time). Commit `b19c14e`.)*
- [x] **S1.10** "Create your workshop" flow. *(2026-07-12: `Workshop.create_with_founder!`
      creates the pair in one transaction, using `SET LOCAL app.workshop_id` inside the
      transaction so the RLS-policed Ownership insert can see its own policy — dies
      automatically at commit/rollback, no manual reset. `WorkshopsController` (new/create/show)
      behind singular `resource :workshop`; `require_workshop!` (built S1.7, unused until now)
      gates `#show`. Dashboard shell takes over the "No jobs yet" card from `home/index`.
      Verified live: signed-in edgeless user → create → both rows persisted together (confirmed
      via direct SQL) → redirected to dashboard. Commit `a3be58b`.)*
- [x] **S1.11** Admin-side add-crew with pending reminders — **flow revised by the builder
      2026-07-09**, replacing the original "person signs up, pending edge attaches" idea:
      admin enters email+role; account exists → active `Employment` created immediately;
      no account yet → an `Invitation` row persists as a visible pending reminder on the crew
      page with "Invite again", admin re-fires once the account exists. No invitee-facing UI
      at all — the crew veto stays fully passive ([[Positioning]]). *(2026-07-12:
      `invitations` table + its `tenant_isolation` policy land in the same migration that
      creates the table (ADR-007 gotcha 1 — never policed-without-a-policy). `Employment#end!`
      (soft-end, `ended_at`) — access dies on the crew member's next request via the existing
      access door, not by deleting history. New `require_manager!` guard
      (`Current.ownership || workshop_manager?`). One live bug caught and fixed in this same
      session: `InvitationsController#create`/`#refire` double-redirected because
      `add_or_invite` already redirects on both branches → `DoubleRenderError`. Commit
      `b892ba0`.)*
- [x] **S1.12** Landing routing by edge count (ADR-006 §4). *(2026-07-12:
      `User#accessible_workshops` — explicit two-step (`ownerships | active employments`),
      `|` de-dups the wrenching towkay's double edge; works pre-context via S1.12-pre's
      own-rows policy. `home#index`: `0` edges → personal home with "Create your workshop" CTA
      + "ask your boss" note; `1` edge on a fresh sign-in → auto-selected into session,
      straight to the dashboard (the daily case); `>1` → the same personal home lists every
      workshop as a select button (no separate picker page — UI still assumes ≤1 in v1).
      The `session[:workshop_id].nil?` guard on the 1-edge branch is what makes "step back to
      personal home" possible without a redirect loop. `WorkshopsController#select` verifies
      the id against `accessible_workshops.find` before writing the session claim; the access
      door still re-verifies every request regardless (ADR-004/006 §3 — the session is never
      trusted alone). Verified the **full S1.10–S1.12 journey live in-browser**: sign-in →
      auto-route to dashboard → Crew page → invite nonexistent email → pending →
      account created → refire → active → end → sign back in → correctly locked out to the
      0-edge personal home. Commit `f6a76b0`.)*
- [x] **S1.13** Scoping convention. *(2026-07-12: `WorkshopScoped` concern —
      `for_current_workshop` scope, explicit not `default_scope` (Design laws #2 — hidden
      magic surprises on create/unscope/joins; nil `Current.workshop` → matches no real row,
      fails closed). Included in `Employment`, `Ownership`, `Invitation`; switched the S1.11
      controllers' in-context reads to it so the convention is live, not declared-and-unused.
      Confirmed `find_or_create_by!` on the scoped relation auto-populates `workshop_id` via
      Rails' scope-attribute inference — checked via direct SQL, not assumed. Commit
      `d9ac032`.)*
- [x] **S1.14** Tests, two layers. *(2026-07-12: `as_tenant`/`as_user` test helpers set/reset
      the respective RLS GUC around a block — model tests run outside a request, so nothing
      else would. **Model unit:** create-workshop creates the pair atomically (and a forced
      Ownership failure rolls the Workshop back too, proven via `assert_no_difference`) ·
      signup alone → no edges · `access_for` resolves employment/ownership/both/none across
      four personas · ending employment blocks access while ownership alone still grants it ·
      `accessible_workshops` de-dups the towkay · **the RLS backstop test**: two tenants
      seeded, a bare `Employment.count` with no `WHERE workshop_id` at all still can't see the
      other tenant's row, and sees zero rows with no tenant note set — the test ADR-007
      promised, proving the backstop backs something up rather than re-testing the app filter.
      **System (Capybara, real headless Chrome):** the full journey — sign up → create
      workshop → invite pending → second person signs up separately (not auto-added) →
      re-fire → active → signs in straight to dashboard → owner ends employment → locked out
      next sign-in. Two bugs only this level of test caught: an async Turbo sign-out race
      (`click_button` returns before the fetch completes — fixed with a wait-for-signed-out
      helper) and headless Chrome auto-dismissing the "End" button's native confirm dialog
      (used Capybara's `accept_confirm` to drive the real dialog rather than stripping it).
      16 tests + 1 system test green; seeds still idempotent. Commit `4a61ce4`.)*

---

## Sprint 2 · Job engine (ONE DOOR)
**Goal:** the job spine + state machine — see [[Job]], [[Stage model]], [[M1-F1 Status flow and transitions]].
Includes **minimal** Customer/Vehicle (Job must belong to a Vehicle; rich intake is Sprint 6).
**Exit:** move a job through stages via `JobActions` (illegal moves rejected); assign a mechanic; Done freezes the job.

- [ ] **S2.1** Migration + model **Customer** (minimal): `workshop_id, kind:integer(person/company),
      name, phone, email`. Scoped.
- [ ] **S2.2** Migration + model **Vehicle** (minimal): `workshop_id, customer_id, plate, vin`;
      `belongs_to :customer`; normalize plate (strip/upcase); unique `(workshop_id, plate)`.
- [ ] **S2.3** Migration + model **Job**: `workshop_id, vehicle_id, stage:integer, token`;
      `belongs_to :workshop, :vehicle`; `has_secure_token :token`.
- [ ] **S2.4** `stage` enum: `registered / assigned / in_progress / done / delivered / cancelled`.
- [ ] **S2.5** Migration + model **JobStageTransition**: `workshop_id, job_id, from_stage, to_stage,
      created_by_id, acknowledged_at, acknowledged_by_id`; `belongs_to :job`.
- [ ] **S2.6** Migration + model **JobMechanic**: `workshop_id, job_id, user_id, primary:boolean,
      assigned_by_id, removed_at, acknowledged_at, acknowledged_by_id`.
- [ ] **S2.7** **`JobActions`** (ONE DOOR) — `change_stage(job, to:, by:)`: enforces the allow-list,
      role gate, and lock-on-Done (reject edits once `done/delivered/cancelled`); writes the transition.
- [ ] **S2.8** Allow-list transition map + role gate, per the [[M1-F1 Status flow and transitions]] matrix.
- [ ] **S2.9** `JobActions#assign_mechanic` — creates a primary `JobMechanic` **and** moves
      `registered → assigned` in one motion.
- [ ] **S2.10** Log the `nil → registered` transition at job creation (clean entry timestamp).
- [ ] **S2.11** Controller + views: create a job (pick vehicle), show a job (stage + crew), stage-advance
      buttons calling `JobActions`. **Technician-facing buttons mobile-friendly** ([[Tech stack]]).
- [ ] **S2.12** Tests (service-heavy): legal/illegal transitions · role gating · Done freeze ·
      assignment one-motion · `nil→registered` logged.

---

## Sprint 3 · Blockers
**Goal:** the overlay axis — see [[Blocker]]. **Exit:** raise/clear blockers per role; a job shows its (possibly several) active blockers.

- [ ] **S3.1** Migration + model **Blocker** (catalog): `workshop_id, label, raised_by_role,
      cleared_by_role`. **Seed "Hold For Payment"** (`raised_by/cleared_by: service_advisor`).
- [ ] **S3.2** Migration + model **JobBlockerTransition**: `workshop_id, job_id, blocker_id,
      action:integer(raised/resolved), note, created_by_id, acknowledged_at, acknowledged_by_id`.
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

- [ ] **S4.1** `JobActions#acknowledge(record, by:)` — sets `acknowledged_at` + `acknowledged_by` on a
      `JobStageTransition` / `JobMechanic` / `JobBlockerTransition`.
- [ ] **S4.2** Receiver logic (a query, not stored): stage change → service advisor · mechanic added →
      that mechanic · blocker raised → the `cleared_by_role` holder(s).
- [ ] **S4.3** "Waiting on me" query across the three trackers where `acknowledged_at IS NULL` and
      receiver = current user.
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
- [ ] **S6.5** Intake flow: pick/create Customer → pick/create Vehicle (lookup by plate) → create Job →
      fill jobsheet field values + complaints. One screen/flow.
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
      workshop health).
- [ ] **S8.2** Scopes/queries per metric — **queries, not tables** ([[Design laws]] #3).
- [ ] **S8.3** Manager dashboard (PC); owner mobile health summary ([[Tech stack]] device posture).
- [ ] **S8.4** Tests: each metric query is correct against seed data.

## Related
- [[Roadmap]] · [[M1-F1 Status flow and transitions]] · [[Data model]] · [[Design laws]] · [[Product gaps]]
