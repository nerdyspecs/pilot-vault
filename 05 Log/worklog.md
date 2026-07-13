---
type: log
updated: 2026-07-13 (Session 13)
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

---

## 2026-07-12 · Session 13 — Sprint 2 kickoff (owner rename); ADR-008 crew acceptance

**Summary.** Started executing the approved Sprint 2 plan. Phase 0 done: `create_with_founder!`
→ `create_with_owner!` ("founder" named a status the schema never tracks; the row confusion was
a symptom of a misnamed concept, same species as the JobService problem). App commit `55951f4`,
vault `f82b19a`, 15 model tests green. Then a design detour that turned into a real decision.

**ADR-008 — crew joining requires acceptance.** Reviewing the S1.11 add-crew flow, the builder
questioned why the `Invitation` row exists at all when the invitee is never contacted. Digging in
surfaced the actual issue: the flow *drafts* a crew member (creates an active `Employment` the
instant a matching email is entered) with no consent — which (a) risks a typo silently employing a
stranger, and (b) contradicts Knot's own handshake philosophy. Builder's resolution, and the
keystone: **joining is bilateral (invitee accepts), termination is unilateral (no ack)** — consent
to bind, no veto on unbind, exactly like real employment. Auto-fire-on-signup was considered and
rejected as RLS-hostile (cross-tenant, email-keyed lookup at signup); manual "Invite again" stays
because it runs inside the admin's own tenant context. Nice side effect: with an accept step, the
name `Invitation` becomes *honest* again — the naming itch was the missing handshake all along, so
no rename needed; `pending → fired → accepted/declined` becomes a status enum.

Two things flagged and worked through before deciding: (1) the invitee needs an **own-rows RLS
policy** on `invitations` to see their own `fired` row pre-membership — same chicken-and-egg as
S1.12-pre, not optional. (2) This is a **Positioning-level** change — but narrower than first
stated: the *passive veto* in Positioning is about **adoption** (crew stop updating), which an
accept step doesn't touch; what changes is the S1.11 "no invitee-facing UI" interpretation, plus a
new *membership consent* promise. Kept the operational handshake (ADR-005) and the membership
accept explicitly distinct so "handshake" doesn't come to mean two things in copy.

**Recorded (vault-first, before touching the build plan):** ADR-008 written; Decisions index +
ADR-006 "superseded in part" footnote; Positioning updated (adoption-veto vs membership-consent,
handshake disambiguation); Sprint plan S1.11 footnoted + new **S1.15** task added.

**S1.15 built and closed (2026-07-12/13, sequencing settled: acceptance BEFORE the job
engine — builder call, correct the invitations while in them).** Four sub-phases, each verified
before moving on — full detail in the [[Sprint plan]] tick: `2bd38ca` (state machine + own_rows
policy), `598b6c4` (fire/accept/decline — `SET LOCAL` inside the model transactions is what lets
an edgeless invitee's policed writes through), `66d3137` (landing Accept/Decline card; the
1-workshop auto-route now also waits for zero fired invites), `6e027b8`, `179671b` (seeds +
4 model tests + system journey reworked). The whole loop was driven live in the browser twice —
invite → fired → card → accept → dashboard → end → lockout. Mid-flight audit (builder-requested)
found the vault and code fully aligned except two stale lines (fixed in this close-out) and
produced two observations that became small fixes: a misleading predicate name became
`User#employed_at?(workshop)`, and S1.14's annotation got a dated supersession footnote.

**Also recorded this session (was floating unwritten):**
- **Law #9 reaffirmed — no `WorkshopActions` command layer.** Builder proposed making workshop
  creation a service for symmetry with `JobActions`, then pushed further: "methods in models
  should strictly be about calculations." Worked through why that instinct (a real principle —
  command–query separation) doesn't demand a new layer: in Rails, persistence *is* the model's
  job, and CQS is honored by **naming** — bang commands (`create_with_owner!`, `end!`,
  `accept!`) vs predicate/calculation queries (`employed_at?`, `active?`). The reusable door
  test, now also footnoted on [[Design laws]] #9: a separate action class is justified only
  when **cross-model orchestration + must-not-drift permissions + a mandatory audit log
  co-occur** (JobActions: 3/3; workshop governance: 0/3 — stays on the model). Revisit trigger:
  governance gaining an audit requirement.
- **Style rule (builder):** never endless methods (`def foo = expr`) — always explicit
  `def`/`end`, even one-liners. Recorded in CLAUDE.md working mode + assistant memory.
- Naming hygiene wrap-up: the last living-doc "founder" (Sprint 1 exit criteria) fixed
  (`c6dd549`); remaining occurrences are inside dated rename-records only, by design. The
  retired Sprint 1 plan file (pre-rename vocabulary) deleted from the plans directory after it
  kept resurfacing as a ghost in the desktop preview.

**Open:** `S1` tag — Sprint 1 + S1.15 now fully closed, awaiting the builder's explicit go.
S0.8 (deploy) — parked since "revisit when S1.12 passes"; overdue for a decision. Next build:
Sprint 2 job engine, Phase 1 (Customer/Vehicle/Job spine), plan already approved.

---

## 2026-07-12 · Session 12 — Sprint 2 design talk; `JobService` → `JobActions`

**Summary.** Pre-plan design conversation for Sprint 2 (the job engine). Walked the risks
before writing any plan: the service's API shape is a one-way door (raise vs return; who the
actor is, given roles live on Employment but owners have none); ONE DOOR is convention, not
DB-enforced; "Done freezes" really means the allow-list from `done` has exactly one exit
(`delivered`); every new tenant table needs its RLS policy in its own creation migration;
enum integers are forever. Five design questions left open for the builder (creation through
the door? · who may move `in_progress → assigned`? · assignment eligibility · cancel-from-done
· row lock in the service now vs later).

**Decision — `JobService` renamed `JobActions` (builder-driven, before any code).** The
builder — a workshop manager — flagged that "service" in workshop language means the repair
itself (oil change, maintenance), so `JobService` read as "the service performed on the job"
and had been quietly confusing all along. It would also collide with a plausible future
`Service` catalog entity. Builder first proposed splitting into per-action classes
(`JobCreate`, `JobTransition`, …) for granular self-describing names; rejected after talking
it through — every action shares the same guards (actor-role resolution, Done-freeze check,
change+log in one transaction), so a split means either duplicating the guards or building a
base-class mini-framework, and the door stops being one auditable file. Resolution: one
class, granular **verb methods** (`change_stage`, `assign_mechanic`, …) — the granularity
lives in the method names. Renamed across Sprint plan, Design laws #9, CLAUDE.md; no ADR
names the class, so no footnotes needed. Extract-when-it-hurts noted for any action that
later outgrows the class.

**All five questions settled (builder answers, same session — recorded in M1-F1's new
"Settled" section + sprint-plan task notes):**
1. Job creation goes **through `JobActions`** — already multi-record (Job + `nil→registered`
   transition), will grow (jobsheet Sprint 6, possible parts list later).
2. `in_progress → assigned` = **service_advisor** (+ manager/owner); it's a handoff, so it
   acknowledges like the rest once Sprint 4 lands.
3. Assignment: **active `technician` employments only, primary-only in v1** — the `primary`
   column exists but helpers stay dormant.
4. **No cancel from `done`** — two terminals, `delivered` (success) and `cancelled` (died
   before done); done's only exit is delivered.
5. **Stage-change row lock deferred to backlog** (builder call, brain-budget honest). Talked
   through the real-life concern — SA and tech "editing the same job" concurrently — and why
   the model dissolves most of it: append-only trackers mean most edits are inserts into
   different tables, which can't collide; the mutable surface is basically the stage column;
   worst case is loud (double transition in the log), fix is one `with_lock` line in the one
   door, cost of waiting doesn't compound. New dated entry in [[Deferred design]] with the
   revisit trigger.

**Open:** Sprint 2 plan drafting is next. `S1` tag still pending explicit builder confirmation.

---

## 2026-07-12 · Session 11 — Sprint 1 closed: S1.10–S1.14 (flows, scoping, tests)

**Summary.** Closed out Sprint 1 in one planned run, pausing for approval after each phase.
Builder revised the add-crew design mid-plan (accept-step → admin-side pending reminders,
passive crew veto preserved) and cross-checked the plan against three RLS gotchas learned in
another session before approving — two were already covered, the reasoning for the third
(session-local `SET`, not transaction-local) got its first proper writeup. Every flow was
driven live in a real browser before being called done — the plan called for it, and it paid
off: a double-render bug in `InvitationsController` and two system-test-only bugs (an async
Turbo race, a native-dialog auto-dismiss) were all caught by actually running the thing.

**Built (full detail on each in [[Sprint plan]]):**
- **S1.12-pre (unplanned):** own-rows RLS policies — landing routing needs to list a user's
  own edges before any workshop is selected, and the existing tenant policy alone hid them
  (chicken-and-egg). Second permissive policy, `FOR SELECT` only, keyed to a new
  `app.user_id` GUC. ADR-007 got its first dated footnotes here — including the overdue
  correction that S1.9 built session-local `SET`, not the transaction-local pattern the ADR's
  ops note had called mandatory.
- **S1.10:** `Workshop.create_with_owner!` (later renamed from `create_with_founder!`) — the pair created atomically, `SET LOCAL` inside
  the transaction so the RLS-policed Ownership insert can see its own policy without any
  manual reset. The dashboard shell.
- **S1.11:** admin-side add-crew — found account → instant Employment; no account → a
  persistent pending reminder with "Invite again". `Employment#end!` soft-ends (history
  stays, access dies via the existing door). One bug found and fixed the same session: a
  double-redirect in the invitations controller.
- **S1.12:** landing routing by edge count — 0/1/>1, the 1-edge auto-route being the daily
  case, the personal home doubling as the context picker for >1. This is what finally let the
  full S1.10–S1.12 journey be exercised together in the browser, not phase by phase.
- **S1.13:** `WorkshopScoped` concern (`for_current_workshop`) — actually wired into the
  Phase 2 controllers, not left declared-and-unused.
- **S1.14:** 16 model/unit tests plus one real end-to-end Capybara system test in headless
  Chrome. The RLS backstop test does what ADR-007 promised back in Session 8: a bare query
  with no `WHERE workshop_id` at all still can't see another tenant's row.

**What the live/system testing actually caught, not just what it confirmed:**
- `InvitationsController#create`/`#refire` called `redirect_to` after `add_or_invite` had
  already redirected on both branches → `DoubleRenderError`, caught the first time the invite
  form was submitted in the browser.
- The system test's own sign-out step failed first: `click_button "Sign out"` is an async
  Turbo submission, and the next `visit` was racing ahead of it, landing on a
  still-authenticated "You are already signed in." page. Fixed with an explicit
  wait-for-signed-out helper.
- Headless Chrome dismisses native `confirm()` dialogs by default, which would have silently
  no-op'd the "End employment" button under system test. Used Capybara's `accept_confirm`
  block to actually drive the dialog, rather than stripping the confirm out of the UI to make
  testing easier.

**Open:**
- Sprint 1 is functionally complete. Exit check + `S1` tag still pending explicit builder
  confirmation before cutting it.
- Sprint 2 (job engine, ONE DOOR) is next up in the Sprint plan.
- S0.8 deploy remains parked.
- Vault still has no offsite git remote.

---

## 2026-07-09 · Session 10 — Post-S1.9 codebase audit vs vault requirements

**Summary.** Builder asked for a read-only sweep of the whole app against the vault's Design
laws and working-mode rules (readability, no smart solutions, vanilla Rails) — a check, not a
build session. Found one real bug and several small cleanups; all confirmed and fixed with
sign-off per item rather than as a bundled diff.

**Real bug — `db:seed` broken by RLS.** The `tenant_isolation` policy (S1.9) is `cmd = ALL`
with an empty `WITH CHECK`, so Postgres falls back to the `USING` clause for inserts too.
Seeds run outside a request, so nothing sets `app.workshop_id` — every `Ownership`/`Employment`
insert was silently denied by RLS the moment S1.9 landed, breaking the idempotency the seeds
file claimed. Fixed: seeds set the GUC once after the workshop exists, reset at the end.
Verified `bin/rails db:seed` twice, identical counts. **Lesson, now on record**: any
non-request write path (seeds, console, future background jobs, S1.14 tests) must set the GUC
itself — RLS doesn't know about Rails request boundaries, only Postgres sessions.

**Cleanup (builder approved each item individually):**
- `application.css` comment still pointed at `docs/01 Context/...` — stale from the earlier
  `docs/` → `vault/` rename. Fixed.
- Deleted dead scaffold: `hello_controller.js` (Stimulus demo, unused; app is declared
  no-JS), `home_helper.rb` (empty generated module).
- `routes.rb`: dropped leftover generator comments that sat above the real `root` route and
  read like unresolved decisions ("`# root "posts#index"`").
- `database.yml`: dropped the generator's commented `#username: pilot` block under
  `development:` — contradicted the live `username: pilot_app` in `default:` from S1.8.
- `application.html.slim`: `notice` and `alert` rendered as identical muted text — an error
  was indistinguishable from an FYI, and inconsistent with `auth.html.slim`'s correct
  `.alert-info`/`.alert-warning`. Now matches.

**Explicitly declined by builder — left as-is:** Gemfile (`jbuilder`, `stimulus-rails` kept;
stimulus may be needed later), `~> 8.0.4` version pin. `Workshop#users` naming and empty
model-test stubs flagged as future considerations only, not changed.

**Commits:** `47e2397` (seeds fix), `84d6a55` (cruft cleanup), `15a958a` (flash styling).

**Open:** same S1.10–S1.14 backlog as Session 9; nothing new deferred by this audit.

---

## 2026-07-08 · Session 9 — Sprint 1 Phase C1: seed personas + RLS live (S1.8–S1.9)

**Summary.** Closed out RLS ([[ADR-007 Row-Level Security pulled into Sprint 1]]), executed
**explain-first** per the builder's request to actually understand the mechanism before it
landed, not just approve a diff. Seeds (Phase C0), S1.8 (app DB role), and S1.9 (policies +
wiring) all built and committed. Builder separately cross-checked the work against three RLS
gotchas learned in another session — two were already covered by the plan, one (`schema.rb`
silently dropping `CREATE POLICY`) was a real gap, caught before it shipped and folded into
Step 0.

**Built:**
- `db/seeds.rb` — one user per persona (owner, manager, 2 advisors, mechanic, the wrenching-
  towkay with both edges, two edgeless v2 personas), idempotent, dev-only. Commit `7c30b95`.
- **S1.8** — non-superuser `pilot_app` role (superusers **silently bypass RLS**, the #1
  footgun); transferred DB/schema/table ownership; `database.yml` dev/test point at it;
  dropped fixtures (needed superuser). Commit `94f9d7e`.
- **S1.9** — `config.active_record.schema_format = :sql` first (schema.rb can't spell `CREATE
  POLICY`, so it would silently vanish from the test snapshot); one migration landing `ENABLE`
  + `FORCE ROW LEVEL SECURITY` + `CREATE POLICY tenant_isolation` together (never
  policed-without-a-policy, which denies everyone) on `ownerships`/`employments` — `users`/
  `workshops` stay global by design. Wired session-local `SET`/`RESET app.workshop_id` into
  `set_current_context`, reset-first (closes the connection-pool leak) and set to the
  *candidate* workshop before `access_for`'s lookup runs (those tables are now policed — an
  unset GUC would hide real access, not just wrong-tenant rows). Commit `87c78b8`.

**A live bug the money-proof caught:** first policy used a bare `current_setting(...)::bigint`
cast. `RESET` on a custom (app-defined) GUC doesn't revert to "unset" — Postgres restores it to
`''`, and `''::bigint` raises `PG::InvalidTextRepresentation` instead of denying the row. Fixed
with `NULLIF(current_setting('app.workshop_id', true), '')::bigint`, folding both the
never-set and the reset states down to the same NULL → no match → deny. Caught by actually
running the reset step of the verification, not by reasoning about the SQL.

**Verification (all against the live seed data, as `pilot_app`):** unset GUC → `Employment.count`
== 0 though rows exist; GUC set to the seed workshop → exactly its rows (5 employments, 2
ownerships); reset → back to 0; superuser `psql` still sees all (documented ops escape hatch);
simulated the full access-door + GUC dance for all four seed personas (pure owner → granted,
pure employment → granted, both-edges towkay → granted with both edges, edgeless bystander →
denied) — all resolved correctly through live RLS, not mocked. Full suite green as `pilot_app`
against the new `structure.sql`-built test DB.

**Deviation from the original sprint-plan note:** S1.9 was originally written as
transaction-local `set_config(..., true)`. Built session-local `SET` instead — Rails doesn't
wrap a request in one transaction (each bare statement autocommits), so transaction-local would
die before the next query in the *same* request. Session-local persists correctly through a
request; the leak risk that transaction-local would have auto-closed is instead closed by
resetting at the top of every request, before anything else runs.

**Open:**
- S1.10–S1.14 (flows, scoping convention, test batch) and S0.8 deploy remain parked, back to
  builder-drive mode.
- Model tests for `Ownership`/`Employment` still don't exist — S1.14 will need a test-side seam
  for setting `app.workshop_id` (build-your-own-records tests will otherwise have their own
  inserts hidden by the very policy just enabled). Flagged, not built.
- Vault still has no offsite git remote.

---

## 2026-07-08 · Session 8 — Sprint 1 Phase A+B: tenancy models + the access door

**Summary.** Git housekeeping closed out (branches unified onto `main`, pushed to private
GitHub `nerdyspecs/pilot`, tag `S0`; vault renamed `docs/` → `vault/`). Then Sprint 1 began —
builder explicitly handed Phase A (data layer) and Phase B (request plumbing) to Claude to
execute directly (an exception to the default learning-drive mode, same as S0.1). Used
Claude Code's plan mode for this: reviewed the phased plan before approving execution.

**Built (S1.1–S1.7, all in [[ADR-006 Ownership separate from Employment]]'s shape):**
- `Workshop`, `Ownership` (governance edge, composite unique index), `Employment` (operations
  edge, role enum with **no owner**, partial unique index for "one active per pair").
- `Current` (`ActiveSupport::CurrentAttributes`) — the per-request clipboard: `workshop`,
  `employment`, `ownership`, reset automatically between requests.
- **The access door**: `User#access_for(workshop)` resolves active Employment OR Ownership
  (or both, for the wrenching-towkay case); `ApplicationController#set_current_context`
  re-verifies on every request from `session[:workshop_id]` — never trusts the session alone;
  `require_workshop!` ready for controllers to opt into once S1.8-10 build them.

**Verification:** `rails runner` smoke tests (not just generator stubs) proved every
constraint — duplicate ownership rejected, duplicate active employment rejected, ended+new
employment coexist, cross-workshop access denied, `Current` sets/resets correctly. Full
`bin/rails test` also surfaced and fixed two **unrelated pre-existing bugs**: a stale
`users.yml` fixture (blank emails, latent since S0.4, only now colliding with the unique
index) and a leftover `home_show_url` test call (should've been renamed with S0.5's
`show`→`index` swap). Suite is green.

**Open:**
- Phases C+D (now S1.10–S1.14 — see below) deferred to a future session, back to builder-drive
  mode.
- S0.8 deploy still waits on the final Sprint 1 test task passing.
- Vault still has no offsite git remote.

**Later same day — [[ADR-007 Row-Level Security pulled into Sprint 1]].** Explaining how the
S1.5 partial index works under the hood surfaced the builder's real question: wanting
schema-per-tenant/database-per-tenant for "commercial-grade" isolation. Walked through why the
shared-table model (already locked, [[ADR-004 Multi-tenant foundation]]) beats schema-per-tenant
even harder given the builder's own requirement — Job/Vehicle history must query *across*
workshops, which schema-per-tenant makes worse, not better. The real want was a database-enforced
guarantee, not physical separation — **Postgres Row-Level Security** gives that on the shared
model. Builder chose to pull it into Sprint 1 now (cheapest while only 2 tenant tables exist)
rather than leave it as ADR-004's "later hardening."

**Sprint plan renumbered again:** S1.8–S1.9 inserted for RLS (role/enable, then policies +
`set_config` wiring into the access door); old S1.8–S1.12 shifted to S1.10–S1.14 (safe — none
had commits). Exit criterion gained a clause: a bare query with no `WHERE workshop_id` still
can't leak across tenants. ADR-004 annotated (RLS clause superseded); [[Decisions]] index
updated.

**Open (updated):** Sprint 1 remaining work is now S1.8–S1.14, all deferred to a future
session, builder-drive mode. RLS work (S1.8–S1.9) should land before S1.10's create-workshop
flow so the tenant tables are guarded from the first real write, not retrofitted after.

**Also same day — testing philosophy locked + [[Design laws]] #9.** Builder set the test
strategy; corrected the literal wording (Capybara is a system/feature tool, not a unit-test
tool) into **two layers, hollow middle**: Minitest **model unit tests** are the priority
(calculations belong in models, so shared logic stays consistent and unit-testable), Capybara
**system tests** for end-to-end flows once pages exist, controller/request tests deferred
(pages are isolated; shared logic isn't). New **Design Law #9 — Calculations live in the model
layer** captures the architectural principle underneath. Added a **per-sprint test-review
ritual** to the Sprint plan conventions (evaluate model changes at each sprint's close, write
tests as a batch — not necessarily the moment code lands). S1.14 reframed into its two layers.
No gem change — Capybara + selenium already in `group :test`.

---

## 2026-07-07 · Session 7 — Auth pages styled; ADR-006 (Ownership ≠ Employment)

**Summary.** Devise's four auth pages got the Knot treatment (Slim + Bootstrap, dedicated
no-appbar auth layout — one primary action per screen). Styling the sign-up page exposed a
modeling flaw: my copy said "one account for the whole shop," which contradicts per-person
identity. The discussion that followed produced **[[ADR-006 Ownership separate from
Employment]]** — the biggest model revision since the foundation session.

**Decisions (all in ADR-006; ADR-004 annotated, [[Decisions]] index updated):**
- **Ownership is its own edge** (`user_id` + `workshop_id`), not an Employment role.
  Governance vs operations; the wrenching towkay holds both edges. Role enum loses `owner`.
  Named `Ownership`, *not* `CompanyOwner` ("Company" reserved for v2 fleet entity).
- **Onboarding split:** signup creates the bare `User`; "Create your workshop" is a
  post-signup act creating `Workshop` + founder `Ownership` in one transaction. A workshop
  can never exist without a user. Workshop = v1's only module; a bare user is the hallway.
- **Access = one door:** per-request check resolves active Employment OR Ownership through a
  single method (never sprinkled).
- **Landing routes by edge count:** 0 → personal home (create/join CTA) · 1 → workshop
  dashboard (the daily case — crew ease) · >1 → context picker. Personal-home-as-default was
  considered and rejected for v1 (empty room for 100% of v1 users).

**Build state (uncommitted, on top of Session 6's work):** four Devise views in Slim
(`sessions/new`, `registrations/new`, `passwords/new+edit`), styled error partial, `auth`
layout via `layout_by_resource`. Verified in-browser: render, validation errors, full
sign-in → home flow. Sign-up copy fixed per ADR-006 ("Create your account").

**Changes to [[Sprint plan]]:** Sprint 1 rewritten per ADR-006 (S1.1–S1.12): Ownership model
task added, enum fixed, access-door task, create-workshop flow, **add-crew flow** (was a gap —
old exit criterion mentioned a 2nd user joining but no task built the path), edge-count landing.

**Open:**
- ~~Everything since Session 6 is still one uncommitted changeset~~ → committed `c8134bb`
  (S0.5, same day) and ticked.
- ADR-006 leaves open: the exact in-app permission surface of an Ownership (beyond
  governance) — settle when the board is built.

**Later same day — S0.6 + S0.7 done (privacy-revised).** The vault physically moved into the
repo at `docs/` but is **gitignored** there (builder wants planning private) with its **own git
history** (root `ca72c49`; `.obsidian/`/`.claude/`/`.DS_Store` excluded). `CLAUDE.md` generated
at the repo root from [[Agent guide]] — also untracked. App-repo trace: gitignore commit
`e68b71b`. Claude's session memory copied to the repo's project path. **New habits:** Obsidian
opens `~/teckhong/pilot/docs`; Claude Code sessions open at `~/teckhong/pilot`; vault changes
commit to the vault's own repo. Next: S0.8 deploy (pick Render vs Heroku) → S0.9 tag `S0` →
Sprint 1 (builder drives).

---

## 2026-07-06 · Session 6 — Landing page in the app; Slim + Bootstrap adopted

**Summary.** The marketing landing page went from mockup to the real Rails app (Claude-driven —
explicitly handed over, an exception to the S0.5 learning drive). Two stack decisions along the
way, both user-made: views in **Slim**, styling on **Bootstrap 5.3.3**. Product surfaces now say
**Knot** per [[Positioning]] ("Pilot" stays internal). Work is on disk but **uncommitted** —
S0.5 stays unticked until the user reviews and commits.

**Decisions:**
- **Slim templates** (`slim-rails`) — user preference; Devise's generated views stay ERB for now.
- **Bootstrap 5.3.3** (most stable 5.3, vendored `bootstrap.min.css`, no build step, JS not
  loaded) with `application.css` reduced to a **brand layer**: Bootstrap CSS variables mapped to
  the [[Visual theme]] tokens + branding-only classes. Chosen for expansion speed; supersedes
  the "no CSS framework" stance in [[Tech stack]] and [[Visual theme]] (both updated today).
- Landing copy ported from the Session-3-era Knot mockup — already passes [[Voice and tone]].
  Title carries the pairing rule: "Knot — no job goes unseen". Dropped the mockup's "See a live
  board" CTA (nothing to show — UI law 8) and second primary button (law 3).

**Build state:** `home#index` is public (landing when signed out, welcome-back stub when signed
in); layout has the Knot wordmark appbar. Verified in-browser both states, desktop + mobile.

**Open:**
- User to review + commit (suggested: `S0.5: landing page, Slim + Bootstrap`), then tick S0.5.
- Leftovers in repo: dead `get "home/show"` route + `show.html.erb`; throwaway dev user
  `preview-check@test.local`.
- Dev quirk: port 3000 is held by the unrelated *kaffa* app — run Pilot with `rails s -p 3001`.

---

## 2026-07-06 · Session 5 — Task-scoped entry points + brand stub

**Summary.** Vault review raised a scaling question: must every session read all 11 files (e.g. a
marketing/branding task)? Answer: no — entry points should multiply as work diversifies, one per
kind of work; the reading list shouldn't grow with the vault. All changes **additive** — no
existing content removed or rewritten.

**Changes:**
- [[Agent guide]] now has **task-scoped reading lists**: the original 11-file list (unchanged) is
  the *building* default; a 4-note *branding/marketing* list added. Rule of thumb: a task reads
  its neighborhood, not the whole graph.
- New `07 Brand/` folder. [[Positioning]] (worked out in parallel, locked same day) is the
  **anchor**: name **Knot** committed, audience (owner-boss, crew veto), message hierarchy
  ("No job goes unseen"), flat per-workshop pricing posture. [[Brand overview]] is the folder's
  index — points to sources of truth (never duplicates [[Visual theme]] or [[Positioning]]),
  clarifies app-scope "not a CRM" vs marketing-the-product, grounds the workstream in
  [[Main problem list]] L3-P3 (Workshop Marketing), and tracks what's still open (logo, voice,
  final copy, landing page, assumption validation).
- [[Main problem list]] stitched into the graph: frontmatter + header marking it **raw discovery
  source material (never rewrite)** + Related links out. Body preserved verbatim (was the vault's
  only zero-outlink note).
- **Brand work (parallel, same day):** [[Positioning]] locked — name **Knot**, silent-K weakness
  named + "the name never travels alone" rule, domain unresolved (`knotapp.com` territory).
  [[Voice and tone]] locked — "good foreman" voice, neutral international English (local flavor
  deferred to ad variants), 5 voice laws (verbal twins of the UI laws).
- **Vault audit** (continuity/coherency/consistency): 0 broken links, no orphans, stage/role/ack
  vocabulary consistent throughout. Surfaced two **pending design calls** (not yet applied):
  ① blocker **resolve**-ack — [[ADR-005 Acknowledged handoffs in V1]] says the raiser acks the
  resolve, but [[M1-F1 Status flow and transitions]] + Sprint task S4.2 omit that fourth trigger;
  ② **self-initiated events** (e.g. SA's own Done → Delivered) — auto-ack or inbox? Decide both
  before Sprint 4.

**Open:** the two ack design calls above (before Sprint 4) · [[Brand overview]] "Not designed
yet" list (tagline, domain, logo, landing page, assumption validation). No impact on Sprint 0
(S0.5 still in flight from Session 4).

---

## 2026-07-06 · Session 4 — Sprint 0 execution + visual theme

**Summary.** Building started. Sprint 0 executed through S0.4 in a **learning mode**: builder
drives the commands/code, Claude navigates (explains, specifies, verifies read-only). Worked out
the entire visual identity interactively (sample boards → choices) and locked it as a new context
note, [[Visual theme]]. Closed with a vault audit (connectivity + consistency).

**Sprint 0 progress** (see [[Sprint plan]] ticks):
- **S0.1 ✓** stripped Rails 8 defaults — Docker/Kamal/Thruster gone, Solid Queue/Cache/Cable gone,
  cable → async, prod DB collapsed to one. Root commit `b9bfa02` (81 files, 102 gems from 118).
- **S0.2 ✓ / S0.3 ✓** Ruby 3.2.8 / Rails 8.0.5 confirmed; local Postgres = Homebrew
  `postgresql@17` (17.5), dormant `@14` noted; `pilot_development`/`pilot_test` exist.
- **S0.4 ✓** Devise `User` — thin (email + password only, stock 5 modules, zero extra columns,
  per [[ADR-004 Multi-tenant foundation]]). Commit `f08e29e`.
- **S0.5 in flight** — home `index` behind `authenticate_user!`; code specified (tokens CSS +
  layout shell + view), builder implementing.

**Decisions (design, recorded in [[Visual theme]]):**
- **Theme locked:** industrial & confident · light, high-contrast · one theme for both clients.
  Brand steel blue `#22456B` (scarce), action blue `#2D5E94`, blue-tinted neutrals, all-light
  chrome ("Option A"). **No secondary hue** — deliberately.
- **Status colors are sacred:** gray/blue/amber/red/green = job state only, identical everywhere.
- **Typography: system font stack** (0KB, can't fail); Inter noted as the escape hatch.
- **10 UI laws** + posture note (tech = phone, SA/manager = PC, owner = phone read-only) —
  the UI twins of [[Design laws]].
- **Dark mode deferred** → derive surfaces from brand steel blue ([[Deferred design]]).

**Vault audit:** 0 broken links · stale-term scan clean (persona/ADR-footnote hits legitimate) ·
fixed: [[Visual theme]] orphan (now in [[Agent guide]] reading list + [[Tech stack]]),
7 stale `updated:` frontmatter dates, this missing session entry.

**Open:** S0.5 proof drive + commit, then S0.6–S0.9 (vault → `docs/`, `CLAUDE.md` — must now
include [[Visual theme]] — deploy, tag `S0`).

---

## 2026-07-04 · Session 3 — Vault audit + acknowledged handoffs

**Summary.** Ran a full coherency audit of the vault (flow, stale refs, open questions) after
Session 2's reconciliation. Fixed the stale-reference bugs the audit surfaced. Then **reversed**
Session 2's handshake backlog — acknowledgement is a v1 KEY feature — and reshaped `Blocker` into
a workshop-owned catalog with per-type role permissions. Simplified the stage list and closed out
most of Module 1's open questions.

**Decisions:**
- Added [[ADR-005 Acknowledged handoffs in V1]] — every ownership handoff (stage change, blocker
  raised, mechanic added) is acknowledged by its receiver; ack lives as columns **on the event
  record** (`JobStageTransition`, `JobBlockerTransition`, `JobMechanic`) — no separate handoff
  table (would double-record; the inbox is a query, per [[Design laws]] #3).
- Added [[Design laws]] #8 — **a Done job is immutable**; corrections open a new job. Vehicle
  owners are read-only everywhere.
- **`Blocker` reshaped:** now a workshop-owned *catalog* (`label`, `raised_by_role`,
  `cleared_by_role`) — not a stateful row. All state moved to `JobBlockerTransition`, parallel to
  `JobStageTransition`. **Multiple blockers can be active on a job at once** (derived from
  raise/resolve events, not a column). Seeded blocker: **Hold For Payment**.
- **Assignment → `JobMechanic`** (not `JobAssignment`) — a membership join, not a one-time action.
  Supports **one primary + optional helpers** (`primary` bool, `removed_at` for history).
- **Stage list simplified:** Registered → Assigned → In-Progress → Done → Delivered, + Cancelled
  (dropped Assessment/In Repair/Ready-for-Collection as separate stages).
- **JobSheet clarified, not renamed:** stays `JobSheet` (form) → `JobSheetField` → `JobSheetFieldValue`
  (answers). Treated as a **record** — v1 supports adding fields only, no destructive delete.
- **Resolved from [[Open questions]]:** blocker taxonomy (→ workshop catalog); single-vs-multiple
  assignee (→ `JobMechanic`, multiple). **Still circling back:** primary/helper permissions, owner
  notification channel (v1 = manual copy-paste message).
- **Device posture (provisional):** web app, standard login on phone/PC; **technician screens
  absolutely mobile-friendly**, SA/owner PC-primary (owner needs a mobile health view); no special
  floor-device/PIN yet — closes [[Product gaps]] #6. See [[Tech stack]].

**Rewritten:** [[M1-F1 Status flow and transitions]] (acknowledgement table, new stages, blocker
permissions via catalog), [[Blocker]], [[Event log]] (three trackers), [[Stage model]] (new stages,
ack, lock-on-Done).
**Edited:** [[Data model]] (new entities + JobSheet clarification), [[Job]] (crew, multiple
blockers, immutable-on-Done), [[Deferred design]] (handshake removed → ADR-005; jobsheet snapshot
added), [[Roadmap]] (slice 5 designed), [[Overview]] (roles, features), [[User stories]] (persona→role
mapping), [[ADR-002 V1 scope]] (dated footnote only — accepted ADRs aren't rewritten).

**Fixed (stale refs from Session 2):** `Event log` "StageChanges", `Roadmap` "Contact", `Overview`
old role names + "handshakes backlogged", `Job` "Warehouse" role.

**Open:** primary/helper `JobMechanic` permissions · owner notification channel (WhatsApp/email vs
manual) · exact `Blocker` admin UI · paper_trail (later) — all in [[Open questions]] / [[Deferred design]].
**Next:** resume slice 0/1 build on this model.

---

## 2026-07-04 · Session 2 — Reconcile 2026-07-03 foundation handoff

**Summary.** Compared a foundation-design handoff (claude.ai, 2026-07-03) against this vault.
Adopted its tenancy/user foundation — it directly answers the "User model + tenancy" pause from
Session 1. Kept the vault's jobsheet. Backlogged the handshake pattern and lock_version. Settled
the v1 role set. Rewrote the affected concepts/feature so the vault has one consistent voice.

**Decisions:**
- Adopted [[ADR-004 Multi-tenant foundation]] — Workshop tenant, thin User, Employment edges, session re-verification every request.
- Added [[Design laws]] (7 invariants) and [[Rejected alternatives]] (dead ends, do not re-propose).
- **Jobsheet stays in v1** ([[ADR-003 Digitized jobsheet in V1]]), now tenant-scoped (`JobSheet belongs_to :workshop`).
- **v1 role set:** technician / service_advisor / parts_advisor / workshop_manager / owner (on Employment). Foreman dropped, front_desk folded into service_advisor. Parts *role* is v1; parts *module* stays v2.
- v2 Company roles: owner / fleet_manager / driver ([[Data model]]).
- **Backlogged (not dropped):** the two-role handshake/confirmation pattern, and `lock_version` optimistic locking → [[Deferred design]].
- **Dropped from v1:** `Contact` + `Group` entities — folded into plain `Customer` fields; richer multiplicity deferred to v2 Company/Fleet.

**Rewritten:** [[Data model]], [[Blocker]] (single-step lifecycle), [[M1-F1 Status flow and transitions]] (new role matrix, single-actor).
**Edited:** [[Job]] (workshop_id/vehicle_id double-stamp, token, ONE DOOR), [[Event log]] (→ `JobStageTransition`), [[Stage model]] (tenant-scoped, role names, kept **Cancelled**), [[ADR-003 Digitized jobsheet in V1]] (tenant note).

**Open:** blocker taxonomy vs [[Main problem list]] · exact stage enum values · single vs multi assignee · one-active-job-per-vehicle index · paper_trail (later) · WhatsApp owner-link research — all now tracked in [[Open questions]].
**Next:** resume slice 0/1 build on the reconciled model (Devise + Workshop + Employment first).

---

## 2026-07-01 · Session 1 — Vault workflow + Module 1 design + Rails scaffold

**Summary.** Set up the entire planning workflow and vault structure, designed all of Module 1
(the model spine + data model + digitized jobsheet), recorded 3 ADRs, and scaffolded the Rails
app. Design is essentially complete.

**Decisions:** [[ADR-001 Core stack]] · [[ADR-002 V1 scope]] · [[ADR-003 Digitized jobsheet in V1]] · Ruby 3.2.8 + Rails 8.0.5.
**Open threads:** User model + workshop tenancy · notification channel · handshake storage · report shapes.
**Next:** pause building → design the **User model** (which forces the tenancy decision), then resume slice 0/1.

### Vault workflow & structure
- Built the planning layer: numbered folders (`01 Context` … `05 Log`), one-ADR-per-file, this running log.
- Split the old "Who am I" note → [[Builder identity]] (identity) + [[Agent guide]] (agent instructions → future `CLAUDE.md`).
- Conventions: vault will move into the repo as `docs/`; commits reference slice/feature IDs; **never edit an accepted ADR — supersede it.**

### Module 1 — model spine
- Two axes: [[Stage model]] (Registered → Assigned → Assessment → In Repair → Ready for Collection → Delivered, + Cancelled) and [[Blocker]] (overlay; strict def = can't-progress + another role must act; co-responsible for attribution; lifecycle `open → resolved → closed`, **both ends are handshakes**).
- [[Event log]] = `StageChange` history (+ blocker timestamps) → time-in-stage & per-department attribution. No grand event table.
- **Handshake** = a general two-role confirmation pattern (assignment, completion, blocker-close); the confirm step **doubles as the cross-department notification**. See [[M1-F1 Status flow and transitions]].
- **Roles:** 5 capability-based, multi-role per user (Mechanic, Front Desk, Warehouse, Foreman, Manager). L1–L4 levels stay in the Problems doc only, not on roles.

### Data model
- [[Data model]]: `Customer` (individual | company) → `Contact` / `Group` (optional, + PIC) → `Vehicle` → `Job`. Framed as a **routing problem, not a CRM**.
- **Vehicle key:** registration = lookup key; VIN = optional identity. `make/model/year/origin` loose → seed the V2 `VehicleModel`.
- **Jobsheet:** configurable form — `JobSheet` (1/workshop) → `JobSheetField` → `JobSheetFieldValue` (on Job). Fields are **rows, not columns** → owner CRUDs at runtime. (Weighed EAV vs jsonb; jsonb is the lighter alt.)
- **Two user populations kept separate:** staff `User` ≠ customer `Contact` (privilege-escalation risk).

### V2 foresight (not built)
- Parts as an **evidence graph** (variant × job-type × parts) — learned, not cataloged; per-vehicle history for recon; aggregate demand across variants for rare-shared stock.
- Cheap V1 breadcrumbs to seed it: **job-type** + **parts-used**.

### Rails scaffold — slice 0 (partial)
- `rails new pilot -d postgresql` at `/Users/teckhong/teckhong/pilot`. Rails **8.0.5**, Hotwire default.
- Bumped Ruby **3.2.4 → 3.2.8** (latest 3.2 patch; stayed on the 3.2 line over 3.3/3.4), re-bundled — 118 gems.
- ⚠️ Rails 8 shipped **Kamal + Dockerfile** (contradicts ADR-001 = PaaS, no Docker) and **Solid Queue/Cache/Cable** (contradicts "no background jobs in v1") — inert defaults, clean up later.
- Remaining slice 0: `db:create`, Devise, vault → `docs/`, `CLAUDE.md`, first commit, deploy skeleton.

### Open / next
- **Design pause — User model:** forces the **workshop tenancy** decision (single-shop vs multi-tenant SaaS — ripples to jobsheet template, customer data, every query). Plus: multi-role storage + permission-enforcement mechanism.
- Other opens: notification channel (WhatsApp/email), handshake storage, report shapes.
