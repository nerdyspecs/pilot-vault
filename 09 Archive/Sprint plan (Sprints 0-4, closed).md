---
type: archive
module: M1
archived: 2026-07-21 (Sprint 3 added 2026-07-24; Sprint 4 added 2026-07-28)
---
# Sprint plan — closed sprints (0, 1, 2, 2.5, 3, 4)
Relocated from [[Sprint plan]] to keep the live roadmap focused on current/upcoming work.
Content is unchanged from the live file at archive time — task ticks, dated annotations,
and commit hashes all stand as originally written. See [[Sprint plan]] for Sprint 5 onward.

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
      *(2026-07-14: S1.12 long since passed, still un-deployed — builder re-parked to
      **Sprint 2's exit** instead: ship the tenancy spine together with a demoable job
      engine, one deploy instead of two. Prereqs unchanged.)*
      *(2026-07-17, Session 21 — **Heroku ruled** (builder: familiarity; the RLS axis was a
      tie — managed PG grants no superuser/BYPASSRLS anywhere, and `FORCE ROW LEVEL
      SECURITY` is already on every policed table). Shape: Procfile — not `render.yaml` as
      the task line originally said — + release-phase `db:migrate` (`structure.sql` loads
      fine); Eco dyno + Mini Postgres ≈ $10/mo floor, US/EU-region latency accepted.
      Deploy-day proof: `SELECT rolsuper, rolbypassrls FROM pg_roles WHERE rolname =
      current_user` + an unset-GUC zero-rows smoke test. Re-verify current Heroku
      pricing/docs live before deploy day.)*
      *(2026-07-17, Session 22 — **re-parked a third time**: builder held off past Sprint
      2's close after all — nothing demoable enough to show anyone yet; local dev already
      proves what Heroku would (RLS enforcement, the full job lifecycle). New trigger: a
      **stable/semi-full v1** — Sprint 3 (blockers) or Sprint 4 (acknowledged handoffs, the
      differentiator) landing is the likely moment. Heroku choice + shape stand unchanged
      from the ruling above; nothing to redo when the day comes.)*
- [x] **S0.9** First commit(s), tagged `S0`. *(2026-07-08: branches unified — `v1-db-setup`
      fast-forwarded into `main` and deleted; `main` = `fc821b7` pushed to private GitHub +
      annotated tag `S0`. Note: GitHub immediately opened 3 dependabot PRs (Rails 8.1.3 +
      two Actions bumps) — Rails stays pinned 8.0.5, handle deliberately later.)*

---

## Sprint 1 · Tenancy spine
**Goal:** the multi-tenant foundation — [[ADR-004 Multi-tenant foundation]] as revised by
[[ADR-006 Ownership separate from Employment]] and
[[ADR-007 Row-Level Security pulled into Sprint 1]]. *(Rewritten 2026-07-07 per ADR-006:
Ownership edge added, `owner` dropped from the
role enum, bootstrap split into signup vs create-workshop, landing routes by edge count. Old
S1.x numbering retired — no commits referenced it. **Renumbered again 2026-07-08 per ADR-007:**
S1.8–S1.9 inserted for Postgres RLS; old S1.8–S1.12 shifted to S1.10–S1.14 — safe, none had
commits yet. S1.1–S1.7 untouched, already committed.)*
**Exit:** signup creates a bare user; "create workshop" creates `Workshop` + owner `Ownership`
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
- [x] **S1.10** "Create your workshop" flow. *(2026-07-12: `Workshop.create_with_owner!`
      (renamed from `create_with_founder!` pre-Sprint-2 — "owner" is the ADR-006 governance term)
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
      `b892ba0`.)* **⚠ Superseded 2026-07-12 by [[ADR-008 Crew joining requires acceptance]]:**
      the "account exists → active Employment immediately" behaviour is replaced — joining is now
      bilateral (invitee accepts). Reworked in **S1.15**. `Employment#end!` (unilateral) stands.
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
      *(2026-07-13: the system test's "re-fire → active" journey was superseded by ADR-008 —
      joining now requires the invitee's accept; test reworked under S1.15. The annotation
      above records the flow as it was when ticked.)*
- [x] **S1.15** Crew acceptance flow ([[ADR-008 Crew joining requires acceptance]]) — reworks
      S1.11's instant-employment into a consent step. `Invitation` gains a `status` enum
      (`pending → fired → accepted/declined`) + nullable `user_id` (set when `fired`). Admin
      "add crew": account exists → `fired`; no account → `pending`; manual "Invite again"
      flips `pending → fired` once the account appears (no signup auto-fire — RLS-aligned, ADR-008
      §5). Invitee sees `fired` invites on their landing page (Accept → active `Employment`;
      Decline → `declined`, shown to admin). Own-rows RLS policy on `invitations`
      (`user_id = app.user_id`) so the invitee can read their own row pre-membership (mirrors
      S1.12-pre). `Employment#end!` unchanged (unilateral). *(Built 2026-07-12/13, **before**
      the Sprint 2 job engine — builder call: correct the invitation flow while in it. Commits:
      `2bd38ca` (state enum + user ref + own_rows policy), `598b6c4` (fire/accept/decline —
      the RLS wrinkle: `accept!`/`decline!` `SET LOCAL` the tenant GUC inside their transaction,
      since the invitee holds no edge yet and both policed writes would otherwise be denied),
      `66d3137` (landing Accept/Decline card + crew status labels + the 1-workshop auto-route
      now also requires no fired invites), `6e027b8` (`User#employed_at?(workshop)` replaces a
      misleading controller predicate — employment-only on purpose: a pure owner stays
      invitable as crew), `179671b` (seeds across states + 4 model tests incl. accept-atomicity
      and own-rows isolation + system journey reworked). Accept/decline live in the same
      `InvitationsController` with `except:`-scoped guards (builder call). Full flow verified
      live in-browser twice; suite green: 19 model + 1 system.)*

---

## Sprint 2 · Job engine (ONE DOOR) — ✅ **closed 2026-07-17 (Session 22)**
**Goal:** the job spine + state machine — see [[Job]], [[Stage model]], [[M1-F1 Status flow and transitions]].
Includes **minimal** Customer/Vehicle (Job must belong to a Vehicle; rich intake is Sprint 6).
**Exit:** move a job through stages via `JobActions` (illegal moves rejected); assign a mechanic; Done freezes the job.
**Exit met** — S2.0b–S2.12 all built and tested (S2.6/S2.9 in their final Design B +
technician-vocabulary shape); S0.8 deploy **decoupled from sprint exit** and re-parked a
third time (see its own annotation) — the code exit criterion above doesn't need a live
URL to be true.

- [x] **S2.0b** ADR-009 destroy guard — replace Devise's stock `DELETE /users` path: bare
      account (no edges, no history) → delete; anything else → refuse with the derived reason
      ([[ADR-009 Account deletion is refusal-first]]; closes [[Risk ledger]] R1). Independent
      of the spine — can land any time before Sprint 2 exit. *(added 2026-07-14)*
      *(2026-07-14: built + live-verified in preview — `User#bare?` (edges only, active or
      ended; invitations excluded — refined during implementation, see ADR-009 footnote)
      gates a `before_destroy` guard with `prepend: true` (must run before
      `dependent: :destroy`'s own callbacks); a matched invitation releases back to
      `pending`/`user: nil` via new `Invitation#release!` rather than blocking. Friendly
      refusal in a new `RegistrationsController#destroy`; model guard is the backstop.
      Bare-account and owner-refusal paths both live-verified. 28/28 tests green. Commit
      `0e135da`.)*
- [x] **S2.1** Migration + model **Customer** (minimal): `workshop_id, kind:integer(person/company),
      name, phone, email`. Scoped. *(2026-07-15: built, commit `2c5ca91`. Two additions from
      the session's rulings: **double restrict** — vehicles AND jobs directly, since payer≠file
      makes vehicle-less payers possible; **contact keys canonicalized** — phone digits-only,
      email stripped/downcased ([[Intake flow]]: phone is the person-lookup key).)*
- [x] **S2.2** Migration + model **Vehicle** (minimal): `workshop_id, customer_id,
      registration_number, vin`; `belongs_to :customer`; unique
      `(workshop_id, registration_number)`. *(2026-07-15: built, commit `2c5ca91`. Renamed
      `plate` → **`registration_number`** (builder: verbose JPJ term); canonical form collapses
      ALL whitespace + upcases — "WVK 3721"/"wvk3721" are one vehicle. `has_many :jobs`
      restrict.)*
- [x] **S2.3** Migration + model **Job**: `workshop_id, vehicle_id, customer_id, stage:integer,
      token`; `belongs_to :workshop, :vehicle, :customer`; `has_secure_token :token`.
      *(2026-07-15: built, commit `2c5ca91`. R5 index live:
      `index_jobs_one_active_per_vehicle` — partial unique on `jobs(vehicle_id) WHERE stage
      IN (0,1,2)`. No match validation, per the deferral below. Trackers + `#timeline` land
      Phase 2.)*
      *(2026-07-14: `customer_id` added — the triple-stamp, written once at registration with a
      creation-time must-equal-vehicle's-customer validation; [[Data model]] §Resolved, the
      sold-vehicle decision.)* *(2026-07-15: the match **validation** deferred at plan review —
      too rigid without a recovery path (payer≠file cases are legitimate); stamp + default-copy
      stand; circle back at the intake UI — [[Deferred design]].)* **Kickoff gate — both ruled 2026-07-15:**
      [[Risk ledger]] R5 → **refuse** (per-visit definition; partial unique index on
      `jobs(vehicle_id) WHERE stage IN (0,1,2)`); customer stamp **confirmed** on
      strengthened reasoning (persons query jobs through the frozen stamp, never through
      the vehicle — see [[Job visibility]]). Also ruled at kickoff: Customer/Vehicle
      deletion is **restrict, not cascade** (ADR-009's refusal-first, one level down);
      `Job#timeline` + tracker associations move to Phase 2 beside their tables.
- [x] **S2.4** `stage` enum: `registered / assigned / in_progress / done / delivered / cancelled`.
      *(2026-07-15: built with S2.3, commit `2c5ca91` — integers now frozen **by schema**
      (the R5 index predicate references 0/1/2).)*
- [x] **S2.5** Migration + model **JobStageTransition**: `workshop_id, job_id, from_stage, to_stage,
      created_by_id, acknowledged_at, acknowledged_by_id`; `belongs_to :job`.
      *(2026-07-16: built, commit `30f3a10`. `from_stage`/`to_stage` are enums that **reuse
      `Job.stages`** rather than redeclaring the mapping — one source of truth for the frozen
      integers; `prefix: true` on both keeps `from_stage_registered?`/`to_stage_registered?`
      from colliding. `to_stage` presence-validated; `from_stage` nullable for the future
      birth row (S2.10, Phase 3). No writer yet — `JobActions` is Phase 3.)*
- [x] **S2.6** Migrations + models for **crew — split into two tables** *(respecified
      2026-07-16, builder ruling: entity + event log, replacing the settled single-table
      shape — see [[Event log]] and worklog Session 17)*: **JobMechanic** (engagement):
      `workshop_id, job_id, user_id` — no ack columns, **no `primary`/`lead` flag in v1**
      (deferred entirely, [[Deferred design]]). **JobMechanicTransition** (events):
      `workshop_id, job_mechanic_id, action:integer(joined/left), created_by_id,
      acknowledged_at, acknowledged_by_id`. Old columns dissolve into events: `assigned_by`
      → the joined row's `created_by`; `removed_at`/`removed_by` → the `left` row. Current
      crew = engagements with no `left` event (a query).
      *(2026-07-16: built, commit `30f3a10`. `JobMechanic.current` scope implements the
      no-left-event query as a `where.not(id: ... select(:job_mechanic_id))` subquery.
      `Job#timeline` merges `job_stage_transitions` + `job_mechanic_transitions` (via
      `has_many :through`) by `created_at`, `includes` on the author/acknowledger to head
      off N+1 before Sprint 4's views exist. 8 new tests + 51/51 suite green; RLS fail-closed
      spot-checked live (visible under the tenant note, invisible after reset); seeds still
      idempotent, untouched by these tables.)*
      *(**Amended 2026-07-16 later, Session 19, commit `6799438`:** `job_mechanics.user_id`
      → **`employment_id`** — an engagement is held by a *stint*, not a login; actor/audit
      columns stay User. The actor/holder split + the append-only-employments pin:
      [[Data model]] §Resolved.)*
      *(**⚠ Superseded in part 2026-07-17, Session 21 "Design B":** engagement → present-tense
      **membership** `job_technicians` (deleted on remove); events → self-contained
      `job_technician_transitions` (direct `job_id` + `workshop_employment_id`, no FK to
      membership; `.current` scope deleted). This tick's annotation stands as the record of
      what S2.6 built; the current shape is [[Event log]]'s supersession note.)*
      *(**Built 2026-07-17, Session 22, commit `c692451`:** one migration
      (`rename_job_mechanics_to_design_b`) — `rename_table` ×2 (RLS + FKs ride the rename);
      transitions gain `job_id`/`workshop_employment_id`, backfilled via one UPDATE join
      through the old `job_mechanic_id` FK, then that FK dropped; membership rows whose
      engagement had already `left` are deleted (present-tense semantics start true).
      Backfill gotcha: both crew tables carry `FORCE ROW LEVEL SECURITY` — even the table
      owner is policed, so the backfill UPDATE saw zero rows until `NO FORCE`/`FORCE`
      bracketed the data steps. Models: `.current` scope deleted (the table IS current);
      `JobTechnicianTransition` has no membership association (self-contained by design).
      Console-verified: assign→remove shows membership at 0 while `job.timeline` still
      carries both `joined`/`left` events + the compensating rollback. 51/51 green.)*
- [x] **S2.7** **`JobActions`** (ONE DOOR) — the class + the stage verbs *(respecified
      2026-07-16, Phase 3 rulings: **named verbs, no generic `change_stage!`** — each move is
      its own `def`/`end` block with a first-line stage guard, so the allow-list IS the verb
      set and the Done-freeze is structural: no verb accepts `done` except `deliver!`, none
      accepts `delivered`/`cancelled`)*. Plain PORO, **class methods**, one file
      (`app/services/job_actions.rb`): `start_work!(job, acting_user:)` ·
      `mark_done!` · `deliver!` · `send_back!` · `cancel!` (the crew verbs are S2.9, creation
      S2.10 — **eight** public methods total). Refusals raise `JobActions::Refused <
      StandardError` with a human message; bang methods commit or raise (controllers rescue
      into flash). Every verb uniformly: stage-legality first → role gate →
      **`job.with_lock`** transaction *(the row-lock deferral **woken** 2026-07-16 —
      [[Deferred design]])* → state write + event row together → `created_by = acting_user`,
      `workshop_id` from the job. *(Settled 2026-07-12, stands: job **creation** goes through
      the door too.)* *(2026-07-16: built, commit `045f5c1`. One plan-review refinement: the
      stage/role checks run **inside** `job.with_lock` — the lock reloads the row, so a check
      outside it reads stale state, the exact race the woken lock kills; checking once inside
      keeps the ruled reading order without a duplicate pre-check. Console-verified: full
      happy path, every refusal readable, both terminals verb-less.)*
- [x] **S2.8** Role gates + the guard set, per the [[M1-F1 Status flow and transitions]]
      matrix + Settled 2026-07-16 (Phase 3). Private class methods: `ensure_counter!(user)`
      (Ownership OR active SA/workshop_manager employment) ·
      `ensure_crew_technician!(job, user)` (manager/owner pass; else technician role + a
      **current engagement on that job** — gates `start_work!`/`mark_done!`) ·
      `ensure_technician!(user)` (the assignee check inside `assign_mechanic!`) · per-verb
      one-line stage checks. **"Touched `in_progress`" = `Job#started_work?`** on the model —
      a history query (`job_stage_transitions.where(to_stage: :in_progress).exists?`), never
      stage-based (`send_back!` keeps it true). *(Settled 2026-07-12, stands:
      `in_progress → assigned` = counter — its verb is now `send_back!`, a rare compensating
      correction, never on technician screens (S2.11); no cancel from `done` — done's only
      exit is `delivered`.)* *(2026-07-16: built, commit `045f5c1`. Guards **renamed at plan
      review** — builder wanted plainer vocabulary: `ensure_counter_staff!` ·
      `ensure_job_crew!` · `ensure_active_technician!`. Kept the `ensure_` prefix, not the
      controllers' `require_`: those redirect, these raise — a shared prefix would suggest
      shared behavior. CanCanCan raised by the builder and rejected: these are
      verb+stage+crew-membership business rules, not resource permissions — an Ability class
      would split the door across two files; the M1-F1 matrix already compiles to
      three guards on eight verbs, readable in one screen.)*
- [x] **S2.9** The crew verbs — `JobActions.assign_mechanic!(job, mechanic:, acting_user:)`
      + `remove_mechanic!(job, mechanic:, acting_user:)`. Assignment = one transaction,
      **three rows**: `JobMechanic` engagement + its `joined` transition + the
      `registered → assigned` stage transition. *(Settled 2026-07-12: assignee must hold an
      active `technician` employment (`ensure_technician!`); single mechanic per job in v1 —
      `assign_mechanic!` refuses a non-empty crew.)* *(2026-07-16 rulings: one
      `ensure_counter!` guard — only SA/manager/owner assigns or removes, self-join deferred;
      `remove_mechanic!` writes the `left` event + the compensating `assigned → registered`
      in the same transaction — legal only at untouched-`assigned`, **refused** if
      `started_work?` (the responsibility rule, [[M1-F1 Status flow and transitions]]).
      Implementation gotcha: re-check crew emptiness **fresh via `JobMechanic.current`**
      after writing `left`, never a cached association — a stale cache would skip the stage
      move.)* *(2026-07-16 Phase 3: the **`registered ↔ assigned` moves are
      crew-method-private** — written only inside these two verbs' transactions, so
      "`assigned` ⟺ crew exists" is unbreakable. `swap_mechanic!` **dropped** for v1 —
      no crew motion on a started job; escape hatch + trigger in [[Deferred design]].
      Crew freezes on terminal stages: `cancel!`/`deliver!` write no synthetic `left`
      events.)* *(2026-07-16: built, commit `36bb90e`. Console-verified: 3 rows one txn;
      removal at untouched-assigned writes `left` + compensating rollback; refused after
      `send_back!` (the ruled edge — checked live, both remove and assign refuse); a refused
      assign writes zero rows. Note from verification: in v1 the "crew already full" refusal
      is shadowed by the stage guard — `assigned` implies crew, so a second assign refuses as
      "not registered" first; the crew check stays as belt-and-suspenders.)*
      *(**Amended 2026-07-16 later, Session 19, commit `6799438`:** engagements now point at
      the assignee's **Employment** — `ensure_active_technician!` returns the stint the
      engagement belongs to; crew lookups join through `employments`. `employment.jobs`
      now answers "work done this stint" directly. See [[Data model]] §Resolved.)*
      *(**⚠ Superseded in part 2026-07-17, Session 21 "Design B":** verbs renamed
      `assign_technician!`/`remove_technician!`; removal now writes the `left` event then
      **deletes** the membership row; the fullness check becomes a bare `exists?` and the
      fresh-`.current`-re-check gotcha dissolves with the scope. Responsibility rule,
      three-rows-one-transaction, and crew-method-private `registered↔assigned` all stand.)*
      *(**Built 2026-07-17, Session 22, commit `c692451`:** `assign_technician!`/
      `remove_technician!` shipped in the renamed shape; `job_crew?` predicate simplified
      to a direct membership join (no `.current`).)*
- [x] **S2.10** `JobActions.register_job!(vehicle:, customer: nil, acting_user:)` — creates
      the Job **and** logs the `nil → registered` birth transition, one transaction (clean
      entry timestamp). Counter-gated (`ensure_counter!`); `customer` defaults to
      `vehicle.customer` (the door's default-copy — [[Deferred design]] match-validation
      entry; an explicit different customer is legal, the two-branch confirm is Sprint 6 UI).
      *(respecified 2026-07-16 — creation was always through the door (Settled 2026-07-12);
      this names the verb.)* *(2026-07-16: built, commit `5592291`. Plain
      `ActiveRecord::Base.transaction` — no job row exists yet to lock. Console-verified:
      birth row `from_stage` nil, `customer` defaults from the vehicle. Suite 51/51 green
      after each of the three commits; S2.12 tests deferred to sprint close, builder ruling.)*
- [x] **S2.11** Controller + views: create a job (pick vehicle), show a job (stage + crew), stage-advance
      buttons calling `JobActions`. **Technician-facing buttons mobile-friendly** ([[Tech stack]]).
      *(2026-07-17: built + live-verified. Four commits: `16c750a` (seeds rerouted through the
      door — the Session 20 audit finding, first tick per the current-state/fix convention —
      + `Job.active` scope + a busy-vehicle guard in `register_job!` + `JobActions` predicates
      `counter_staff?`/`job_crew?` + a delegation-only `PermissionsHelper`, one source of
      truth for buttons and refusals alike); `f6c4bb4` (job routes as named member verbs +
      `JobsController` + `Jobs::MechanicsController`, one `rescue_from JobActions::Refused`
      each → flash, never a 500); `3dd4cc6` (views: `jobs/new` eligible-vehicles dropdown per
      [[Intake flow]] 1c, `jobs/show` with stage badge / crew card / role-gated buttons /
      timeline, `_stage_badge` partial, sacred-palette badge CSS, dashboard active-jobs list);
      `e2c30a0` (mobile timeline wrap fix). **Stage→color mapping ruled at build:**
      registered/assigned/cancelled = neutral, in_progress = info blue, done/delivered =
      success green — amber reserved for aging, red for blockers ([[Visual theme]] status
      colors stay sacred). Also applied first: the uncommitted `create_with_founder!`
      half-rename reverted per the Session 20 audit ruling — vocabulary stays *owner*.
      Verified: 51/51 suite green throughout; live browser walk of all personas
      (SA / technician / parts advisor) incl. forged-POST refusal → flash not 500,
      `turbo_confirm` on mark-done, full register→assign→start→done→deliver lifecycle,
      375px mobile check. JSON responses for door mutations consciously deferred —
      [[Deferred design]].)*
- [x] **S2.12** Tests *(respecified 2026-07-17, Session 21 — the ruled ten cases, written
      against the post-Design-B/post-rename shape)*: (1) full legal path
      register→assign→start→done→deliver asserting exact stage rows incl. the
      `nil→registered` birth row · (2) every verb refused at every illegal stage — loop over
      stages, the allow-list under test · (3) Done freeze: only `deliver!` accepts `done`,
      nothing accepts terminals · (4) role gating: counter verbs refuse
      technician/parts advisor, floor verbs refuse a technician not on this job's crew,
      manager/owner pass everywhere · (5) assignment = three rows one transaction; a refused
      assign writes zero rows (`assert_no_difference`) · (6) birth row + busy-vehicle guard
      · (7) removal legality: legal at untouched-`assigned` (membership deleted + `left`
      event + compensating `assigned→registered`), refused after `send_back!`
      (`started_work?` is history) · (8) schema assertion: ack pair on transition tables
      only, never membership · (9) model tests: Customer/Vehicle canonicalization,
      `Job.active`, `started_work?`, `timeline` merge order, current-crew read ·
      (10) one Capybara journey: SA registers → assigns → tech starts + marks done (confirm
      dialog) → SA delivers → timeline complete; plus one forged POST → refusal flash,
      not a 500.
      *(**Built 2026-07-17, Session 22, commits `bee2c2a`.** All ten cases landed:
      `test/services/job_actions_test.rb` (1–8) + two additions to `test/models/job_test.rb`
      (9 — `Job.active` and `started_work?` were the only real gaps; canonicalization,
      timeline order, and current-crew reads were already covered) +
      `test/system/job_lifecycle_test.rb` (10, two tests). System-test gotcha found and
      fixed: `page.text` is a raw synchronous read that doesn't wait for Turbo
      redirects — early debug output raced the page and lied; switched every check to
      `assert_text`, which polls, matching the S1.14 precedent (async sign-out race) with a
      sign-in-side twin. Forged-POST gotcha: the XHR itself follows the redirect and Rails'
      flash sweep clears it there — a later `visit` finds the flash already gone (read-once);
      asserted on the XHR's own response body instead, which **is** the rendered, redirected
      page. 64/64 green (61 model/service + 3 system, incl. the pre-existing crew journey).)*

---

## Sprint 2.5 · Minimal intake / customer–vehicle management — ✅ **built 2026-07-18 (Session 24)**
**Goal:** the front-desk UI that **creates** a Customer and a Vehicle, so a fresh
0-customer workshop can take in its first car. Forms in front of the already-built job
engine — closes the cold-start hole (models existed since S2.1/S2.2 but had no UI, only
seeds/console), and unblocks eventual dogfooding + the parked S0.8 deploy.
**Exit:** from an empty DB, a counter user creates a customer, adds a job (which births the
vehicle), and the job appears on the board — no console needed.

**Why it exists / why it's a new slice** *(designed Session 23; sits between the closed
Sprint 2 and unstarted Sprint 3)*: S2.11 shipped a deliberately-dumb dropdown of *existing*
vehicles — from 0 rows it's a dead end. This slice adds the create surface. **Ruled: the
fuller slice** (customer CRUD + intake motion), not a minimal create-only path; **new
"Sprint 2.5"** — Sprint 2 stays closed (exit met, `bee2c2a`), Sprint 3 keeps its number.

> **⚠ ARCHITECTURE BOUNDARY — read before building.** [[Design laws]] #7 ONE DOOR governs
> **job state only**. Customer and Vehicle create/edit are **ordinary Rails CRUD** (plain
> controllers + AR models) — **not** through `JobActions`. Only the job-creation step goes
> through the door (`register_job!`, already built). This slice is *forms*, no new engine.

**Entry is plate-first; creation is customer-first — two phases, not a contradiction:**
- **Plate hit** → surface the vehicle (+ the active-job guard: [[Intake flow]] §1c
  "in-house: job #N" instead of a register) → create job.
- **Plate miss** → customer-first creation (find/create the customer, then birth the vehicle
  under them) → create job.

**The hard S2.5 / S6 boundary — keep it:** 2.5 is **lookup** ("does this exist?"); Sprint 6
is **disambiguation** (phone verify, the four forks — bill husband / I'm paying / I bought it
/ old number — the two-branch mismatch confirm, changed-hands reassignment). The tell: **2.5
has NO "who's paying?" override** — the stamp defaults silently to `vehicle.customer` on a
hit, to the just-created customer on a miss. So `job.customer == vehicle.customer` at birth in
**both** branches → a mismatch is structurally unrepresentable in 2.5, which is exactly why
the deferred match-validation / two-branch confirm stays cleanly parked until the plate-first
override lands in S6 ([[Deferred design]]).

- [x] **S2.5.1** `CustomersController` + `resources :customers` (new/create): `kind` toggle
      (person | company), flat fields, `for_current_workshop`-scoped, counter-gated at the
      controller. **Additive flat fields ruled 2026-07-17:** `address` (both kinds) +
      `contact_person` (company) — legit workshop contact info, not v2 schema. *(v1 =
      **contact**, not company-structure accuracy — see the company ruling below.)*
      *(2026-07-18: built, commit `fb710e2`/`33583fd`. Migration adds both columns nullable,
      no backfill. `require_counter!` added to `ApplicationController`, delegating to
      `JobActions.counter_staff?` — one source of truth with the door's own guard, same
      pattern as `PermissionsHelper`.)*
- [x] **S2.5.2** `customers#index` — a searchable rolodex (by name / phone — the cheap dedup
      aid), **vehicle count per row** (not plucked plates — keeps the cell clean).
      *(2026-07-18: built, commit `33583fd`. `Customer.search(term)` scope: name ILIKE OR
      phone = the canonicalized term — reuses the same digits-only key contacts are stored
      under. Live-verified: search narrows correctly, `pluralize` fixed a "1 vehicles" nit.)*
- [x] **S2.5.3** `customers#show` — read page: basic info + an **activity panel** (vehicles
      owned · total jobs · last visit · active jobs — **activity only, no money** per
      [[ADR-002 V1 scope]]; a grows-into-it panel) + **"Add job"** as the one primary action
      + **"Maintain customer"** as a quiet secondary → edit. Editing is rare (create once at
      intake), which is the argument *for* the distinct quiet button (UI law 3).
      *(2026-07-18: built, commit `fb710e2`/`33583fd`. `Customer#total_jobs`/`#last_visit`/
      `#active_job_count` (the last reuses `Job.active` from S2.11) — explicit `def`/`end`,
      no logic in the view. Live-verified against a real customer with two jobs (one
      delivered, one registered): counts and last-visit date all correct.)*
- [x] **S2.5.4** `customers#edit` / `#update`. *(2026-07-18: built, commit `33583fd`.)*
- [x] **S2.5.5** **"Add job"** = customer-first vehicle+job birth → `JobActions.register_job!`
      (the one door touch in this slice). Reg-number collision on a "new" plate → rescue
      `ActiveRecord::RecordNotUnique` → friendly flash *"already in the book — pick it from
      the list"* (accepted v1 stand-in; the crude seed of S6's lookup-first).
      *(2026-07-18: built, commit `33583fd`. `Customers::JobsController` — birthing the
      vehicle under the customer is what makes `job.customer == vehicle.customer` at birth
      (the S2.5/S6 line, held). **Real bug caught live:** `@customer.vehicles.create!` only
      auto-populates the association's own FK (`customer_id`) — `workshop_id` stayed nil,
      tripping `belongs_to :workshop`'s presence validation, which the rescue then
      mislabeled as "already in the book." Fixed by passing `workshop:` explicitly. Also
      rescues `ActiveRecord::RecordInvalid` (the uniqueness validation fires before the DB
      constraint does). Live-verified: the typed plate now survives the whole
      miss→create-customer→add-job chain (the pre-fill was originally dropped between
      screens — fixed by threading `params[:registration_number]` through the view).)*
- [x] **S2.5.6** **Plate-search entry** — replaces S2.11's dumb dropdown: hit → found-vehicle
      screen (active-job guard, §1c) → job; miss → customer-first creation.
      *(2026-07-18: built, commit `33583fd`. `JobsController#new`/`#create` reworked into a
      search: `Vehicle.canonicalize` (extracted from the model's `before_validation`, so
      search and storage can never disagree) → hit-with-active-job redirects straight to
      the open job (§1c); hit-with-free-vehicle → the customer's add-job confirm screen;
      miss → `new_customer_path` carrying the plate through. All three branches
      live-verified in-browser, including pushing a seeded job to `delivered` to re-test
      the freed vehicle.)*
- [x] **S2.5.7** Tests: customer CRUD + canonicalization (model layer mostly covered by S2.1)
      · intake creates the customer→vehicle→job graph · one Capybara **cold-start journey**
      (0 customers → new customer → add job → job appears on the board).
      *(2026-07-18: built, commit `cea8a66`. `Customer#total_jobs`/`#last_visit`/
      `#active_job_count`/`.search` unit-tested; `Vehicle.canonicalize` given a direct test
      (previously only exercised indirectly). Two system tests: the plate-miss cold-start
      journey (the sprint's exit criterion, executable) and the customer-first path (add
      customer → Add job from their page). **Regression caught and fixed in the same
      commit:** S2.12's `job_lifecycle_test.rb` still drove the old vehicle dropdown this
      sprint replaced — updated to the plate-search flow. Full suite 69/69 green (64
      model/service + 5 system).)*

**Company handling (v1) — flat record, not an org** *(ruled 2026-07-17)*: companies are
flat `Customer` rows via the `kind: person | company` toggle. **No** Company org entity,
**no** logins, **no** `customers.user_id`, **no** person→company link edge — all v2, gated
behind the unsolved cross-tenant-RLS question ([[Data model]] v2 notes, worklog Session 14).
The boss's lorry and the boss's private car are **two cards, correctly** (different
phones/payers) — routing by responsible-contact, never legal-ownership; v2 claim machinery
reunites the views if ever needed. **Vehicle reassignment** ("park cars under a company")
defers to Sprint 6 with its changed-hands sibling.

---

## Sprint 3 · Blockers ✅ *(built 2026-07-24 — three coding plans A/B/C, all green)*
**Goal:** the overlay axis — see [[Blocker]]. **Exit (met):** raise/clear blockers per role; a job shows its (possibly several) active blockers.
Design deep-dive settled three deltas before build: the **`blocks` stage-guard** resolving the
Hold-For-Payment vs `→ done` collision (HFP guards `delivered`); the **crew-aware** raise/resolve
guard; and raise refused past the guarded stage. 89 unit + 9 system green.

- [x] **S3.1** Migration + model **Blocker** (catalog): `workshop_id, label, raised_by_role,
      cleared_by_role, blocks`. *(2026-07-24, `f47a4f6`.)* Built with a **`blocks` stage-guard**
      column (DB `CHECK` to `in_progress`/`done`/`delivered`) — not in the original spec, it's what
      resolved the HFP collision. **Four seeds** at `create_with_owner!`, not just HFP (Subcon /
      Parts / Technical → `done`; **Hold for payment → `delivered`**) — see [[Blocker]].
- [x] **S3.2** Migrations + models **`JobBlocker`** (item) + **`JobBlockerTransition`** (events).
      *(2026-07-24, `f47a4f6`.)* Respecified per the ⚠ note: **three records**, events carry
      `job_blocker_id` (not a direct `blocker_id`), `action` includes **`noted`**, and
      `created_by`/`acknowledged_by` → `WorkshopStaff` composite FK (ADR-010).
- [x] **S3.3** Extend `JobActions`: `raise_blocker!` / `resolve_blocker!` / `note_blocker!` + the
      stage veto. *(2026-07-24, `9fe5156`.)* Role check is **crew-aware** (a `technician` side needs
      *this job's* crew; manager/owner override); the veto lives once in `transition!`.
- [x] **S3.4** `Job#active_blockers` scope: items with no `resolved` event (a query, [[Design laws]] #3). *(2026-07-24, `4009792` — also spliced blocker events into `Job#timeline`.)*
- [x] **S3.5** Controller + views: the Blockers card — raise from catalog + note, resolve, add-note,
      show active blockers; tech raise flow mobile-friendly. *(2026-07-24, `0ae84c2` wiring +
      `eb377d0` views + `7273db9` system tests.)*
- [x] **S3.6** Blocker-catalog admin — owner/manager adds + edits types (no delete: append-only
      catalog). *(2026-07-24, `8a27e1d` + `4686bfb` tests.)* Built now rather than slipped.
- [x] **S3.7** Tests: raise/resolve/note · crew-aware role checks · multiple active items · manager
      override · the veto both directions · composite-FK actor integrity. *(Model/door units in
      `job_actions_test.rb` 9–15 + `blocker_*_test.rb`; two system flows.)*

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
