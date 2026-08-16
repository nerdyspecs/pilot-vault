---
type: reference
updated: 2026-08-14 (B2 amended — request/reply addressing rule, derive-vs-store split)
---
# Deferred design
Consciously parked — **revisit later**, not dropped. Each is additive (won't require rewriting v1).

- **Jobsheet per-answer snapshot (level-3)** — freeze `label`/`kind` onto each `JobSheetFieldValue`
  at fill time, so even a field *rename* can't change how an old sheet reads. v1 relies on
  lock-on-Done + add-only fields instead. Add if a compliance/dispute need appears.
- **`lock_version` (optimistic locking on Job)** — guards against silent lost-updates when two staff
  edit the same job's free-text at once. Deferred: audit trail covers stage; concurrent free-text
  edits are rare in a small trusted team. Add the column + a "stale, reload" handler if it bites.
- **Stage-change row lock in `JobActions` (2026-07-12)** — two people moving the same job's stage
  in the same second (e.g. SA cancels while tech marks Done) can both pass the allow-list check
  read on stale state. Deferred by the builder pre-Sprint-2: the append-only transition log makes
  the failure *loud* (two exits from the same stage, visible, reconstructable — never silent
  corruption), and most "concurrent editing" in real workflow is inserts into different tables,
  which can't collide. The fix is one line (`job.with_lock` around read-check-write) in the one
  door, so the cost of waiting doesn't compound. **Trigger to revisit:** a workshop reports a job
  that "changed twice at once", or the log shows two transitions out of the same stage.
  **⚠ WOKEN 2026-07-16 (Phase 3 rulings, builder):** superseded before it ever bit — the
  Phase 3 design session ruled every `JobActions` verb wraps its read-check-write in
  `job.with_lock`, from day one. The cost turned out to be one shared line in the door, so
  the original "wait until it hurts" trade lost its upside. No longer deferred; kept here
  as the record of the original reasoning.
- **Trapped last owner — the escape routes from account deletion (2026-07-14).** Since a
  workshop cannot exist without an Ownership (lifetime invariant, ADR-009), the **last owner
  of a workshop can never delete their account** while the workshop stands — that refusal is
  settled. What's parked is the *escape route*, a functional-design question: builder leaning
  = either **add another owner, then remove yourself** (needs an add-owner flow — none in v1)
  or **delete the workshop first, then the account** (needs workshop deletion — none in v1;
  and whether that's a real delete, soft delete, or disable is itself TBD). Pre-launch the
  trap has zero occupants; "contact support" bridges v1. Revisit when designing workshop
  lifecycle (close/transfer) — likely v2. See [[Risk ledger]] R1 / ADR-009.
- **PDPA / anonymization — the whole question (2026-07-14, builder ruling: after v1 is up).**
  Everything is additive (scrub columns, no schema change), so deferring costs nothing
  structurally. **Trigger: anonymization must exist before the first real workshop's data
  enters the system** — that's the moment Knot becomes a *processor* of someone else's
  customers, the heavier PDPA role. Two insights preserved so future-us doesn't re-derive:
  (1) **data user vs processor split** — for accounts (users/edges) Knot is the data user and
  requests land on us; for customers/vehicles/jobs the *workshop* is the data user and Knot
  processes on its behalf, so requests land on the workshop; (2) anonymization is therefore
  **two features**: Knot-side (scrub a User, keep the edge/history skeleton — the ADR-009
  path) and workshop-side (a crew-facing tool to scrub a Customer, keep the job history as
  business records — retention principle covers that). At trigger time also re-check the 2024
  PDPA amendments (breach notification, DPO, portability, processor liability) and add a
  privacy-notice line at signup. See [[ADR-009 Account deletion is refusal-first]].
- **Job↔vehicle customer-match validation (2026-07-15, builder: "a little too far — circle
  back").** Phase 1 planned a create-time rule: the job's stamped `customer_id` must equal
  `vehicle.customer_id` at registration (refuse "Lim's van billed to Speedy's file").
  Parked as **too rigid without a recovery path**: legitimate mismatches exist (borrowed car
  where the borrower pays, informal sale nobody filed, third party covering a repair), and a
  hard block at intake with the car on the lift is workflow poison. **The stamp itself
  stands** (gate 2, [[Data model]] §Resolved) — the door still *copies* the vehicle's
  customer by default; what's parked is only the law forbidding an explicit different
  choice. Circle back when the intake UI exists (Phase 4 / Sprint 6). **Recorded leaning
  (2026-07-15, designed with the builder): a two-branch SA-facing confirm, not a generic
  warning.** Happy path: registration number → vehicle → customer auto-fills (the door's
  default-copy), no warning ever. On mismatch: "vehicle filed under Lim, billing Tan" with
  two explicit choices — **[Just this visit]** (stamp only; file untouched — borrower/third-
  party payer) vs **[Vehicle changed hands]** (also `vehicle.update!(customer:)` in the same
  transaction — informal sale; future visits auto-fill right). Two buttons because the two
  stories need different writes; one generic "confirm" would leave stale files and train
  click-through. Mechanically: `register_job(customer: ...)` defaults to `vehicle.customer`;
  cue = ids differ. Schema untouched — Phase 1 already supports it. **2026-07-15: the full
  SA decision tree (both lookup keys, the script question, all real-world forks incl.
  first-visit dedup) is now documented in [[Intake flow]]** — that note is the intake spec
  this entry's circle-back builds from. **Consequences audited
  2026-07-15:** (1) payer-with-no-vehicle became possible → Customer must restrict deletion
  on **jobs directly**, not just vehicles (patched into Phase 1); (2) no ownership-change
  log (accepted — the frozen stamps form a de facto timeline); (3) v2 nuance: a borrowed-car
  job is visible to the payer, not the vehicle's owner — person-inward being consistent;
  note it in the v2 design pass. See [[Job visibility]].
  **⚠ 2026-07-17 (Session 23) — Sprint 2.5 lands the LOOKUP half, this confirm stays parked
  to Sprint 6.** The new Sprint 2.5 intake ([[Sprint plan]]) builds customer/vehicle *create*
  + plate-first lookup, but deliberately has **no "who's paying?" override**: the stamp
  defaults silently (`vehicle.customer` on a plate hit, the just-created customer on a miss),
  so `job.customer == vehicle.customer` at birth in both branches — a mismatch is
  **structurally unrepresentable** in 2.5, which is exactly what keeps this two-branch confirm
  cleanly deferred (nothing in 2.5 can produce the state it resolves). It circles back with
  the **plate-first override** at Sprint 6, alongside the phone-first dedup tree. Accepted 2.5
  gap in the meantime: **weaker dedup** (no phone-first dedup) — mitigated cheaply by
  plate-first screening + a name/phone-searchable customer index (S2.5.2), the full tree
  still S6.
- **Crew: helpers + the `lead` flag (2026-07-16, builder ruling).** v1 crews are a single
  responsible mechanic; S2.6 ships `job_mechanics` **without** any lead/primary flag — all
  v1 technicians on a job are treated the same. When helpers arrive, the flag lands as
  **`lead`** (boolean, `default: true` — which honestly backfills v1 engagements, since every
  v1 mechanic *was* the lead). Named `lead`, not `primary`: PRIMARY is a reserved SQL keyword
  (quoting tax on the raw-psql audit sweeps run every session) and reads ambiguously next to
  "primary key". **Naming settled now so it isn't re-litigated.** See
  [[M1-F1 Status flow and transitions]] Settled 2026-07-16. *(2026-07-17, Design B: when it
  lands, the `lead` flag lives on **membership rows** (`job_technicians`) — current stints
  only; past stints are event pairs and carry no flag. `default: true` still honestly
  backfills — every membership row that exists at flag-time is a v1 single-technician crew's
  lead. Table renamed from `job_mechanics` in the same restructure.)*
- **Crew self-join / self-leave (2026-07-16).** The event shape supports it (`joined`/`left`
  events don't care who `created_by` is — the receiver derivation already handles "the party
  who didn't act"), but v1 capability is counter-only: one `ensure_counter!` guard in the
  door (SA + manager/owner). Waking self-service crew motion **supersedes M1-F1's permission
  matrix** — dated note there when it lands. Free-standing "remove" as a motion is
  conceptually for helpers (v2); v1's single-mechanic crews make the reassignment swap the
  only in-progress crew change (the responsibility rule, M1-F1). *(2026-07-16 later that
  day: the swap itself was then dropped for v1 — see the `swap_mechanic!` entry below;
  v1 has **no** in-progress crew motion at all.)*
- **`swap_mechanic!` — mid-job crew handover (2026-07-16, Phase 3 rulings, builder).**
  Dropped from v1 entirely (amends Session 17's wording that made the swap "the only
  in-progress crew motion" — v1 now has none). Consequence accepted: once a job has started
  work, its crew is fixed — a sick tech shows on the board, truthfully, as the responsible
  party until the job reaches done/cancelled. **The escape hatch that makes this safe:** the
  manager/owner exemption in the crew gate means a manager can still drive a stuck job to
  `done` themselves — the workshop is never trapped, the job just keeps naming the sick tech
  as responsible. **Trigger to revisit:** the first real mid-job handover need from the
  floor. When it lands it's one door verb (old member's `left` + row deleted, new member +
  `joined`, one transaction — Design B shape, 2026-07-17; crew verbs now named
  `assign_technician!`/`remove_technician!`) — purely additive. See
  [[M1-F1 Status flow and transitions]] Settled 2026-07-16 (Phase 3).
- **Token page × RLS — how the note reaches Postgres (2026-07-14).** The Sprint 7 status page
  is unauthenticated: no `app.user_id`, no `app.workshop_id` — under fail-closed RLS the page
  can't even look up the job by token (zero rows). Needs its own read key at build time: either
  a token-keyed `FOR SELECT` policy (`USING (token = current_setting('app.job_token', true))`,
  set before the lookup) or a `SECURITY DEFINER` lookup function. Purely additive — `token`
  is stamped at birth (S2.3). Decide at Sprint 7 kickoff. See [[Job visibility]].
- **CanCanCan — gem for permission checks (2026-07-16, builder: park, might need it).**
  Rejected for the door itself (Session 19): `JobActions` permissions are
  verb+stage+crew-membership *business rules* — an Ability class would split ONE DOOR
  across two files, and the M1-F1 matrix already compiles to three guards on eight verbs,
  readable in one screen. Where it *could* earn its keep later: **controller/view-level
  checks** — "which buttons does this user see" (S2.11+), authorizing RESTful resources
  outside the door (crew admin, blocker catalog, jobsheet fields), or v2's truly dynamic
  per-workshop permissions (the Blocker catalog's `raised_by_role`/`cleared_by_role` is
  already the data-driven case). Rails-built-in rule applies at revisit: show first why
  plain guard methods + shared predicates aren't enough. **Trigger:** permission checks
  spreading across controllers/views faster than the guard-method pattern keeps tidy —
  likely visible at S2.11 or Sprint 5's role-shaped screens. Adds a dependency — builder
  decides at trigger time.
- **JSON responses for door mutations (2026-07-17, builder ruling at S2.11: HTML first).**
  [[ADR-001 Core stack]] commits to "every mutation available as JSON", but S2.11's
  controllers ship HTML-only — `rescue_from ActionRefused` *(2026-08-14: was
  `JobActions::Refused`, [[ADR-013 The door decomposed]])* renders flash, success
  redirects. Consciously deferred, not dropped: no non-browser consumer exists yet, and all
  business logic lives in the doors, so adding JSON later is ~2 lines per action
  (`respond_to` + a status payload) with zero logic movement. **Trigger to revisit:** the
  first non-browser consumer — a mobile app or a React front-end.
- **Dark mode = steel-blue chrome (2026-07-05).** v1 ships light mode only (high-contrast, per
  device-posture decision). When dark mode comes, the dark theme's surfaces derive from the brand
  steel blue (~`#22456B` family) — chosen because the navy app-bar sample read as "dark mode" and
  was liked exactly for that. Keep status colors (gray/blue/red/amber/green) identical in both modes.

## 2026-07-24
- **Staff off-boarding — one sign-off path + does `workshop_staff` need its own `ended_at`?** Decide whether to add a single `WorkshopStaff#retire!`-style helper that ends **all** of a person's active roles atomically, so the only way to sign someone off is the one door (today roles are ended one at a time via `WorkshopStaffRolesController#destroy`, so a multi-role staffer can be left half-off-boarded — the "left the company but a role still active" contradiction). Second half of the decision: whether a **company-level `ended_at` on `workshop_staff`** is ever warranted, or whether employment periods stay wholly on the append-only role rows (leaning **no for v1** — access = holding an active role, so "here but roleless" isn't a real state; a single `ended_at` can't survive a rehire; multiple staff rows would revert [[ADR-010 WorkshopStaff supersedes the edge split]] into two nested period tables. Revisit only for owner-lifecycle or v2 tenure/rehire-eligibility). **The sign-off is a tenant-policed write — it MUST set RLS context** (`SET app.workshop_id` inside its transaction, same pattern as `Workshop.create_with_owner!` / `Invitation#accept!`/`#decline!`/`#release!`), or the role-ending updates are denied by `tenant_isolation`.

- **Directed blocker notes (B2)** *(deferred out of Sprint 4; was S4.8)*. Decide whether a `noted`
  (or `raised`) blocker event can carry a **stored `receiver_id` pinned to a specific person** — a
  parts advisor's "arrived, please verify" showing as *"waiting on"* that technician on the board —
  rather than pinning nobody as B1/Sprint-4 shipped. The hard part, and why it keeps sliding: **the
  receiver can flip mid-thread**, so it isn't one fixed role per item. Purely additive whenever it
  lands — the `receiver_id` column already exists; `raise`/`note` just start stamping it, old rows
  reading as neutral, **no rework**. **Trigger:** the base acknowledgement loop (ADR-011) meets a
  real workshop and the note chain is observed actually needing direction — deliberately not before,
  so a second traffic generator isn't added on a guess. See [[Blocker]] B1/B2 split.

  > [!note] Two advances from 2026-08-14 (Sprint 4.5 wrap session)
  > **1. A rule that reframes "flips mid-thread" as expected, not a blocker.** *Address a
  > **request** by capability, a reply by identity.* Raising/noting outbound ("waiting for
  > parts") wants *someone who can act* — a role. Resolving/replying wants the specific person
  > who's been waiting — an identity. That's the pattern the five shipped pins already follow
  > (assign/remove = the person **is** the subject; `send_back!` = the responsibility rule;
  > `mark_done!` = context continuity, the registering SA holds the customer's story;
  > `resolve_blocker!` = reply-to-requester). B2's stalling point — "the receiver can flip" — is
  > this rule working correctly, not a design flaw: a request is role-addressed, its reply is
  > person-addressed, by design.
  > **2. `raise`/`note` may need no schema change at all.** The catalog already stores
  > `cleared_by_role` on `Blocker` — so *"who should act on this active blocker right now"* is
  > already derivable, a board **query**, not a stored column. But that collides with ADR-011's
  > whole point (a stored receiver survives a later catalog edit; a derived one doesn't) — so
  > the two questions must stay separate: **"who should act now"** (present tense, over *active*
  > work) is safe to derive from current config; **"who was this addressed to"** (history) must
  > still be stored, exactly as ADR-011 already established for every other pin. Still messy:
  > a `note` from a manager/owner override has no clear "other side" to address. Doesn't change
  > the Trigger above — still waiting on a real workshop, not decided today.
- **A universal "Got it" button** *(parked 2026-07-28, ADR-011 reshape)*. A single explicit receipt
  on a board item, letting someone signal *"seen, not yet actionable"* — a state the two-column
  schema deliberately can't represent (it measures "has acted", stored as `acknowledged_at`).
  Deferred as not-important-now; would need a third state (seen ≠ acted). **Trigger:** a real
  workshop wanting to distinguish "I saw it" from "I did it".
- **Multi-technician crew vs a single receiver** *(chipped 2026-07-24, [[ADR-011 Acknowledgement as stored visibility]])*. Decide how the receiver stamping survives a crew of two: `assign_technician!` / `send_back!` / `mark_done!` each pin **one** `receiver_id`, which currently rests on `assign_technician!` refusing a job that already has crew (`app/services/job_actions.rb:44`). Two shapes on the table: **per-member receipts** (each crew member gets their own `joined` row and receiver; any/all confirm — decide which) or ride the **already-deferred `lead` flag** (the lead is the receiver, helpers aren't). Naming for the flag is already settled (`lead`, not `primary`) in the crew-helpers entry above. **Trigger:** the first real two-technician job.
- **Workshop opening hours** *(chipped 2026-07-24, [[ADR-011 Acknowledgement as stored visibility]])*. Decide whether ageing should pause outside business hours. The Sprint-5 waiting-pin colour (S5.7) will colour on raw elapsed time, so a handoff made at 6pm goes amber by 9am next morning purely from the shop being shut — a known accepted wart, not a bug. Fixing it needs opening hours per workshop, which is outside [[ADR-002 V1 scope]] and would also feed Sprint 5's aging highlight (Product-gap #2) and any future ETA maths. **Trigger:** a real workshop complaining that overnight jobs look late.

## 2026-08-03
- **✅ PROMOTED 2026-08-03 → [[ADR-012 Intake-Job two-level aggregate]]** (sequenced as **Sprint 4.5**,
  before S5). All four open sub-decisions settled with the builder: (1) all-cancelled car → derives
  `cancelled` (no deliver path); (2) cancel-cascade cancels remaining *open* jobs, outcome derives,
  confirm names both sides; (3) `mark_done!` keeps the registering SA as receiver (stored at write
  time), and intake `deliver!` sweeps its jobs to close the done-notices; (4) HFP → a **non-acknowledgeable**
  intake blocker (no direction = no ack pair), `blocks` is the single discriminator. Vocabulary locked:
  **intake = the visit, job = a repair.** The original chip is kept below for the reasoning trail.
- **Intake/Job aggregate morph — split the single `Job` into `Intake` (the visit) + `Job` (one repair)** *(chipped from the routing/screen-map session)*. Decide whether to break today's overloaded `Job` (car + visit + one repair + one stage + one technician) into a two-level aggregate: **Intake** = one car's visit (Customer → Vehicle → **Intake** → Job), **Job** = one repair carrying its own stage machine, its own `job_technicians`, and its own blockers. Motivation: a car realistically comes in for several services worked by several technicians in parallel, which the one-job-per-visit model can't represent. **Needs its own ADR and a real design pass (weight of the tenant collapse), and Sprint-plan edits to sequence it BEFORE the board/dashboard build** — no production data means this is the cheapest the migration will ever be, and the S5 screens would otherwise be built on the wrong aggregate. Working model already reasoned with the builder (not yet locked): stages split — old `registered` splits into intake `open` + job `unassigned`; `assigned`/`in_progress`/`done`/`send_back` stay on Job; `delivered` is the one stored intake fact; intake **status is derived** from its jobs (ready = all terminal w/ ≥1 done; cancelled = all cancelled), never stored (Design law #3). Deliver requires every job terminal (done|cancelled); "defer" = cancel-now + rebook as new job/new intake (laws #6/#8). Cancel-the-car = cancel all *remaining open* jobs (done work survives), terminal then derives itself. Every backward move lives on Job; Intake only steps forward to a boundary. **Open sub-decisions:** (1) all-jobs-cancelled car that leaves — delivered or cancelled? (builder lean: cancelled); (2) cancel cascade confirm UX; (3) the ADR-011 handoffs (register/assign/ready/deliver) now span two levels — receiver still **stored at write time, never derived** even though status is derived; (4) blocker `blocks` guard now spans two models (work-blockers veto job `done`, HFP vetoes intake `delivered`). **Alternative to rule out first:** if shops actually run one tech + a task checklist, the Sprint-6 jobsheet already covers multi-item-per-visit for less — the split only earns its keep if repairs need independent technicians AND independent blockers. Vocabulary to lock before any code: "intake" = the car's visit, "job" = a repair (avoid a repeat of the JobActions/JobService confusion). See [[M1-F1 Status flow and transitions]] · [[Job]] · [[ADR-011 Acknowledgement as stored visibility]] · [[Sprint plan]].

## Related
- [[M1-F1 Status flow and transitions]] · [[Blocker]] · [[Job]]
