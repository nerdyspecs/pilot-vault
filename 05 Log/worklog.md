---
type: log
updated: 2026-07-07
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

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
