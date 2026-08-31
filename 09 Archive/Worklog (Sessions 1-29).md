---
type: archive
module: M1
archived: 2026-07-21 (Sessions 12–25 added 2026-07-28; Sessions 26–28 added 2026-08-14; Session 29 added 2026-08-17)
---
# Worklog archive — Sessions 1–29
Relocated from [[worklog]] to keep the live log focused on recent sessions. Content is
unchanged from the live file at archive time. Covers vault/Rails setup through Sprint 2.5, the
job engine and blockers groundwork, the tenant-spine collapse (ADR-010), Sprint 3 (blockers),
Sprint 4 (acknowledgement as stored visibility), and the Sprint 4.5 aggregate design pass
(ADR-012, Session 29) — Sessions 1–29.
See [[worklog]] for Session 30 onward.

---

## 2026-08-03 · Session 29 — the Intake/Job aggregate morph: design pass + Sprint 4.5

**Summary.** Took the routing/screen-map session's chip — split today's overloaded `Job` (car +
visit + one repair + one stage + one technician) into a two-level aggregate — through a full design
pass with the builder, wrote **[[ADR-012 Intake-Job two-level aggregate]]**, and sequenced it as
**Sprint 4.5** *before* the S5 board. Then built it: 4 new tables, the door grown to two levels,
130 green (122 unit/service + 8 system). **Not committed yet** — codebase sweep pending.

**Why it goes before Sprint 5, not after.** Two reasons that compound: the S5 board groups by the
**car**, so it has to sit on `Intake` or be rebuilt; and there is **no production data**, so this is
the cheapest the migration will ever be — a schema squash + reseed, the ADR-010 play, never a data
migration. Building S5 first would have meant building it twice.

**The lighter alternatives were ruled out first, and the premise recorded.** The Sprint-6 jobsheet
already covers multi-item-per-visit — but a checklist has no per-item stage, crew, blocker, or
acknowledgement, so a car with a blocked brake job and a done aircon job reads as one lie. Bare
many-jobs-per-vehicle (no `Intake`) gives independent repairs but loses the **visit**: delivery
smears across N jobs, HFP has no honest home, and the board has nothing to group by. The split's
load-bearing premise is written into the ADR rather than assumed: *it earns its keep only because
repairs need independent technicians **and** independent blockers.*

**The four sub-decisions, settled with the builder.** (1) An all-cancelled car that leaves derives
**`cancelled`**, never `delivered` — delivery means "a car with real work done was collected".
(2) Cancel-the-car cancels the remaining **open** jobs only; done work survives (law #8), and the
terminal falls out rather than being chosen — 0 done → cancelled, ≥1 done → still collectible.
(3) **`mark_done!` keeps pinning the registering SA** — the builder pushed back on dropping it, and
was right: the gap wasn't the pin, it was that `deliver!` moving to the intake left the pin with no
closing action. Fix is one line — **intake `deliver!` sweeps `acknowledge_pending!` across all its
jobs**, clearing only what's addressed to the actor delivering (a technician's open pin stays
honestly open — stamping it would be the append-only lie ADR-011 forbids). (4) **HFP moves to the
Intake** as an intake blocker: payment is a per-visit hold, and hanging it on an arbitrary one of
the car's repairs would be a domain lie.

**Intake blockers are three records but NOT acknowledgeable — the builder's catch.** The first pass
gave them the full mirror "for symmetry". The builder pushed: *"i dn think intake blockers are
acknowledgeable"* — correct, and [[Event log]] already says why: **the ack pair belongs to a
direction**. HFP is the counter holding its own car until paid (SA raises, SA clears); nobody is
ever waiting on anyone. So they keep catalog + item + events (the **note chain** is real — payment
back-and-forth) but carry **no `receiver_id`/`acknowledged_at` at all**. Identical-but-dead columns
would have been cosmetic symmetry that misleads the next reader. Recorded trigger: a cross-role
`blocks: delivered` type (e.g. manager sign-off) would be directional — additive when it lands.

**A refinement the schema forced: intake `status` is stored, not fully derived.** ADR-012 §2 first
stored only `delivered_at` and derived `cancelled`. That broke against **R5**: a Postgres partial
unique index can only key on a stored column of its own table, so a derived `cancelled` left the
busy-vehicle guard unable to tell a live-open visit from a dead-cancelled one — a fully-cancelled
car would block its vehicle's next visit. Ruled with the builder: store the terminal **position**
(`status` enum, exactly like `Job.stage`), keep **`ready?`** derived — that's the genuine
about-the-children question. The door is the only writer (`reconcile_intake!` after every
job-terminal move), so `status` is a door-owned memoized derivation, not a second source of truth.
Law #3 intact; the guard stays DB-enforced. **R5 is now moved *and inverted*** ([[Risk ledger]]): a
second **job** per vehicle is legal and expected, a second **open intake** is the violation.

**Also settled:** the **jobsheet attaches to the Intake** (builder ruling — it's the car's intake
form, one per visit), and ETA follows it there; vocabulary locked before any code (**intake** = the
visit, **job** = a repair) to avoid a repeat of the `JobService`→`JobActions` confusion.

**One scare worth logging.** Mid-verification, Bash/`git status` stopped seeing this session's
edits — `git status` read "working tree clean" while the files were plainly there via the editor,
minutes after `bin/rails test` had run green against them. Stopped and flagged rather than running
any git recovery; it resolved itself on its own and all 40 files were intact. **Lesson: a
`git status` that contradicts work you just verified is a reason to pause, not to "fix" the tree.**

**Next:** codebase sweep, then commit. S4.5.8 (concept-note reconciliation) still open — and now
also covers [[Data model]] + [[Job visibility]], since the token moved to `Intake` and Sprint 7's
RLS design reads those.

---

## 2026-07-28 · Session 28 — Sprint 4 reshaped and built: acknowledgement as stored visibility

**Summary.** Went to plan Sprint 4 on Session 27's *holder* model, and a chip-out study of the
proposed `handoff` concern reshaped the whole answer.
**[[ADR-011 Acknowledgement as stored visibility]] revised in place** (it was never committed):
the holder dies, the receiver is **stored
at write time**, and the feature is **visibility, not an inbox**. Then built it: `982f7e9` (write side + board), then
`8fad8c9` (the done-awaiting-delivery pin + a coherency pass). 102 unit + 9 system green.

**Why the holder was dropped.** The holder settled the gap but over-built the answer: there's no v1
consumer for "who holds this" — useless when the job's healthy, misleading when it's blocked, blind
when it's stalled. *"Who hasn't picked up yet"* — a **stored `receiver_id`**, stamped by the door as
each event is written — is truer and more useful, and settles #5 the same way (the person who sees
the stuck pass is the sender/manager, an adopter by definition). The bomb metaphor was a good teacher,
not a feature; it's kept in ADR-011's Rejected alternatives.

**The bug that made "stored" non-negotiable.** A receiver *derived* from a blocker's
`cleared_by_role` silently re-points every already-open handoff the moment someone edits that catalog
type (shipped Sprint 3, `8a27e1d`). A log whose rows change meaning when config is edited **is not
append-only**. Storing intent at write time is what makes the append-only claim true — and it
**restores ADR-005's original stored `to_user`**, which a 2026-07-16 footnote had removed in favour
of a read-time `.pending_ack` predicate. That predicate — which existed *only* because the column was
removed — is now deleted; `receiver_id IS NULL` *is* "not a handoff", so no classifier is needed.

**Fully implicit — the one carve-out retired.** No confirm button in v1, so the only thing that fills
`acknowledged_at` is `JobActions.acknowledge_pending!` (acting on a job clears what you owe), run
under `job.with_lock` so a refused verb rolls its sweep back. Session 27's "blocker-resolve echo is
tapped, never inferred" exception **had to go**: with no button, exempting it would leave it
permanently unacknowledgeable. It sweeps like everything else; its verification content lives in the
`note` column, which carries it better than a tap.

**Three forks, settled by one reframe — there is no message/inbox concept, it's all visibility.**
(1) **raise pins nobody** — a raised blocker is already visible and `cleared_by_role` routes clearing;
B2 stays deferred, the column's there to pin later in one line. (2) **terminal verbs untouched** —
stamping an ack on delivery would be a lie in an append-only log; instead the board is scoped to
`Job.active`, so delivered/cancelled rows stay honestly NULL and just never surface. (3) **keep
`registered_by`** as the done-receiver — weakest of the six, fine at 1–4 SAs, one-line change if it
ever bites. And **done is not terminal**: the finished car nobody collected — the founding pain —
gets its own "Done — awaiting delivery" group on the board, since `Job.active` excludes `done`.
Verified in the browser at desktop and 375px.

**A naming correction from the builder.** The code had drifted back into retired vocabulary —
`handoff?`, `open_handoffs_by_job`, a fresh `pin` verb. Renamed to trace straight off the columns:
`has_receiver?`, and `pending_acknowledgements_by_job` / `WorkshopStaff#pending_acknowledgements`
(matching `acknowledged_at`/`acknowledged?`/`acknowledge!`/`unacknowledged`). No invented nouns.

**A DB-rebuild gotcha, logged so it isn't rediscovered.** `db:seed` crashed writing `receiver_id`
even though the migration was corrected. Cause: with `schema_format = :sql`, `db:migrate` on an empty
DB **loads `db/structure.sql` instead of running migration code** — and the dump was stale. Fix:
remove the dump so all migrations run fresh, then it re-dumps clean. (The dormant columns were renamed
in the source migrations in place, since nothing had ever populated them.)

**Also this session:** `as_tenant` now sets `Current.workshop` too (mirroring `set_current_context`),
so `for_current_workshop` scopes resolve in model tests. A **code-coherency pass** caught two things
(fixed in `8fad8c9`): `Job#registered_by` targeting "first registered" instead of the birth row
explicitly (`remove_technician!` writes a second `registered` row), and a dead
`WorkshopStaff#pending_acknowledgements` left over from the cut inbox. A third was a builder call —
the `left` crew event keeps its receiver (uniform with `joined`) rather than pinning nobody; the
lingering-pin consequence is recorded in ADR-011's stated limits. **Next:** the pin's ageing colour
(**S5.7**, deferred).

---

## 2026-07-24 · Session 27 — the Sprint 4 gate: Product-gap #5 settled as ADR-011

**Summary.** A vault/code **consistency audit** first (findings below), then the Sprint 4 gate:
**[[Product gaps]] #5 — "the ack model assumes full adoption"** — settled in a teaching-then-deciding
chip-out and written up as **[[ADR-011 Acknowledgement as stored visibility]]**, which *extends* ADR-005 rather
than superseding it. Sprint 4 is now unblocked and re-shaped; no code was written this session.

**The ruling, and the analogy that produced it.** ADR-005 already said ownership isn't transferred
until acknowledged — nobody had read that sentence literally. Read literally: an unconfirmed pass
**didn't complete**, so the **sender still holds the job**. The builder's own model carried it:
**every job is a bomb** — accountability, not liability; always in exactly one pair of hands and
never set down. SA holds it at registration, passes to the technician on assignment, back to the
counter at done, defused at delivered/cancelled; each open blocker is a **fragment** held by its
`cleared_by_role` counterpart. That framing **replaced an earlier "the receiver owes a debt"
framing of mine, and did real design work in the process**: it *deleted* the "orphaned debt /
pending, unheld role" edge case. Under debt-thinking, a receiving role nobody holds resolved to
nobody and needed a special state plus a never-vanish rule; under holding-thinking the fallback is
never nobody, so the special state is gone (the *display* reason stays — the chip still says why
the pass can't complete). One less concept, from a better metaphor.

**Why it answers the gap.** The holder is computable **whether or not anyone acknowledges
anything**. A counter-only shop generates zero acknowledgement traffic by the existing same-role
rule and still gets a correct holder and a full audit trail; a partially-adopting shop gets
completed passes moving and unconfirmed ones sitting visibly with the sender — who is, by
construction, an adopter. "Partial use must still provide value" is met literally, not
approximately. Four mechanisms: derived holder · the open item belongs to its sender (a
first-class sender-side view) · **acting on a job confirms it** (one shared helper called by every
door verb — builder's ruling, since three event tables × eight verbs is where duplication drifts),
with the **blocker-resolve echo tapped-never-inferred** because it doubles as verification ·
colour at a threshold, nothing more. Surviving law: **never deleted, never faked**.

**Two of my framings were wrong and got corrected.** (1) I proposed a stored
**ack-participation flag** on `WorkshopStaff`; the builder argued it down — declaring someone a
non-participant *excuses* them, the opposite of accountability, and the phone-less technician needs
no flag because he shows up truthfully as "0 of 41 confirmed". (2) I framed unconfirmed handoffs as
**noise to suppress**; that was wrong — they **are** the capture, and the framing deflated to a
missing `group_by` (the manager wants them *grouped*, not hidden). Also rejected: a per-workshop
on/off switch (all-or-nothing fails the constraint's actual word, *partial*), threshold/snooze as
*the* answer (a knob only picks when a permanent false alarm starts), and "jobs with an update" as
unread-state (needs per-viewer `last_seen` rows — Design law #3 violation, and it breaks on a
shared screen).

**Colour bands, reconciled with the sacred palette.** Builder asked green 1h / yellow 4h / red 1d.
**Green dropped** — it means Done/success and would say "finished" about a job nobody has started.
**The 4h band dropped too**: the palette affords only amber and red for ageing without breaking the
reserved words, and the resolution a third band would add is already carried by the printed age —
colour conveys urgency, the number conveys detail. Final: neutral under 1h · amber 1h–1d · red over
1d, **on the chip never the row**, with red recorded as a **clarification** of its existing word
("stuck — act now", two causes: a blocker, an unclaimed handoff; the chip text says which).

**Sprint placement.** **S4.2 ruled without a build** — removal is only legal at `assigned` before
work starts and already rolls the job back to `registered` (`job_actions.rb:75`), returning it to
the SA by itself; the `left` event owes a confirmation only if the technician confirmed the
`joined` (you can't tell someone they're off a job they never knew they were on). **S4.3 absorbs**
the old `.pending_ack` predicate into the holder derivation — one query, not two. **S4.4 is
inbox + sender view as one unit**, deliberately not splittable: holding is symmetric, so shipping
one side leaves the symmetry true in the model and invisible in the product. **S4.8 (directed
blocker notes) deferred again** — still additive, and the base loop should meet a real workshop
before a second traffic generator lands. Sprint 5 gains per-job (whiteboard parity) and
per-technician grouping, the latter framed as **the manager's diagnostic on stuck jobs, never a
scoreboard** — kept honest by the symmetry, and inside the positioning invariant because it
aggregates *jobs*, not people's minutes.

**The audit that opened the session.** A read-only pass over vault + code. Real findings: the
docs still described the pre-ADR-010 "point crew history at the employment period" rule as if it
stood (it doesn't — the holder is now the permanent `WorkshopStaff`, and the periods live on the
role rows), and the word **"stint"** was being used for *two different tables* in different files.
Fixed: "stint" removed from code entirely (it named the staff record or the role row, never a
period), **"mechanic" → "technician"** across tests and seeds to match the app code, `M1-F1` given
the ADR-010 read-against note its sibling docs already had, top-of-doc pointers added to
[[Data model]] and [[Event log]] where the dead-table picture came first, and four stale `updated:`
dates reconciled. Code-side: `authenticate_user!` unified across the four tenanted controllers that
had been relying on `require_workshop!` to imply it (no security hole — `Current.workshop` can't be
set for a signed-out user — but a signed-out visitor was getting "select a workshop" instead of the
sign-in page). App commit `075357e`; 89 unit + 9 system green throughout. Two nits consciously
skipped: duplicated phone canonicalisation in `Customer` (the lesson `Vehicle.canonicalize` already
learned) and an unrescued `create!` in `WorkshopStaffRolesController` that 500s on a duplicate role.
Also chipped from the audit: **staff off-boarding** — there is no single "sign this person off"
action (roles are ended one at a time, so a multi-role person can be left half-off-boarded), plus
the parked question of whether `workshop_staff` needs its own `ended_at`. Reasoned through and
**leaning no**: a single `ended_at` can't even survive a rehire (it holds one departure; rehire
nulls it), whereas append-only role rows record unlimited leave/return gaps — the periods already
live in the right place.

---

## 2026-07-24 · Session 26 — Sprint 3 (Blockers) built end to end + the RLS lockout fix

**Summary.** Shipped **Sprint 3 — the overlay axis** across three coding plans (A: records + door +
scope; B: controller + views; C: catalog admin), preceded by a design deep-dive that settled the
hard parts before any code. Blockers are now three records (`Blocker` catalog + `JobBlocker` item +
`JobBlockerTransition` events), raise/resolve/note through the door, a mobile-friendly Blockers card
on the job page, and owner/manager catalog admin. Along the way a **pre-existing RLS lockout bug**
surfaced (system tests aren't in `bin/rails test`, so it had gone unnoticed) and was fixed. **89 unit
+ 9 system green.** [[Blocker]] rewritten to the built reality (the "Restructure pending" warning is
gone).

**The design deltas that shaped the build (deep-dive, then confirmed).** Three decisions turned the
Session-17 sketch into buildable code:
1. **The `blocks` stage-guard resolved the Hold-For-Payment collision.** The old ruling "open
   blockers stop `→ done`" would trap HFP forever (the car *is* done, you just can't release it).
   Fix: each catalog type carries a `blocks` column naming the one forward stage it vetoes —
   work blockers guard `done`, **HFP guards `delivered`**. Constrained to `in_progress`/`done`/
   `delivered` (DB `CHECK` + validation) so the veto — which lives **once**, in the door's
   `transition!` — provably can't bite `send_back!`/`cancel!`/`assign`. A blocked job stays
   cancellable, which falls out for free.
2. **The raise/resolve guard is crew-aware.** A `technician`-side check requires *this job's* crew
   (mirrors `start_work!`/`mark_done!`), not merely holding the role somewhere; other roles = hold
   the role; manager/owner override. This drew the same floor-vs-counter line the door already uses.
3. **Raise refuses past the guarded stage** — no raising a `done`-guard blocker on an already-done
   car (the veto only stops *entering* a stage, never pulls a job back). Keeps `active_blockers`
   honest.

**Design B shipped.** One `JobBlocker` item carries a whole incident — `raised` once, `resolved` at
most once, any number of `noted` events between (subcon A fails → subcon B works = one item, the
bounce recorded as notes; you never resolve-then-reopen). `noted` ships in v1 (verb + mobile UI, the
"B1" half). **B2 — routing an individual note to an inbox** (receiver flips mid-thread) — deferred to
Sprint 4 and pinned there as **S4.8**; it's additive (one nullable `directed_to_role`), no S3 rework.
Acks stay dormant on every event row until Sprint 4.

**Build shape: model-first paid off.** Plan A concentrated every subtle decision (the guard, the
veto, the birth-row idiom, `active_blockers`) in a fully unit-tested model layer — green before any
UI. Plan B was then thin wiring + the Blockers card, and Plan C ordinary tenant CRUD (no door verb —
a catalog type isn't job state), append-only like `WorkshopStaffRole` (no delete: a type referenced
by an item can't vanish). Two real defects were caught building the admin UI: an unscoped `form_with`
that would've missed `params.require(:blocker)`, and duplicate DOM ids across the per-row edit forms.

**The RLS lockout bug — and finally getting hands dirty in `Current`/RLS.** `crew_management_test`
failed: after an owner ended a mechanic's only role, the mechanic still saw the workshop listed on
their landing page. Root cause was **two access checks that had drifted** — `User#access_for` (the
door) filters by `WorkshopStaff#active?` and correctly denied, but `User#accessible_workshops` never
filtered, so it listed the dormant husk. The trap: the naive one-line fix (add `active?`) *locks out
live non-owner staff*, because `active?` reads `workshop_staff_roles`, which only had the
`tenant_isolation` RLS policy — and on the landing page **no workshop is selected**, so
`app.workshop_id` is blank and `active?` silently collapses to `owner?`-only. Real fix: a read-only
**`own_rows` policy on `workshop_staff_roles`** (reached through `workshop_staff` via subquery, since
roles has no `user_id`), making `active?` evaluable pre-workshop-selection; then `accessible_workshops`
filters by the *same* `active?` predicate the door uses, so the two can't drift again. Safe because
every app read of the roles table is already association-keyed or `.for_current_workshop`, so widening
its RLS can't leak across tenants. This falsified the tenancy squash's "own_rows on roles would be
dead weight" note — recorded as a dated footnote on [[ADR-007 Row-Level Security pulled into Sprint 1]]
(app commit `a283a27`). A chip-out teaching session first walked the RLS/`Current`/access-door
machinery end to end so the fix was understood, not just applied.

---

## 2026-07-21 · Session 25 — Tenant-spine collapse: WorkshopStaff + WorkshopStaffRole (ADR-010)

**Summary.** Built the tenant-spine collapse designed across three chip-out sessions
(2026-07-19/21): `WorkshopEmployment` + `WorkshopOwnership` → one **`WorkshopStaff`** record
(`owner` boolean) + append-only **`WorkshopStaffRole`** rows, and every tenant-local
actor/holding column moved off the global `User` onto `WorkshopStaff` with a composite
`(actor_id, workshop_id)` FK. New [[ADR-010 WorkshopStaff supersedes the edge split]]; ADR-006
§1 and [[Data model]]'s "actions → person" both superseded by dated footnotes. No production
data, so it shipped as a **schema squash + reseed** (6 clean migrations born in final shape),
not a data migration. **72/72 green** (up from 64), incl. a new composite-FK cross-tenant
isolation test class. Also archived the closed sprints + old worklog sessions into a new
`09 Archive/` folder to keep the live docs focused.

**How it landed (the design, chronological).** The trigger was a plain review question — "is
there an ack for job assignment?" — that surfaced ADR-005's ack columns exist but are unwired
(Sprint 4 work). That led into whether *actions* should point at the workshop relation instead
of the thin `User`. First answer was a polymorphic actor (`WorkshopEmployment | WorkshopOwnership`);
the builder's objection to polymorphism was the tell that the real problem was **two tables**.
Collapsing them removes the polymorphism (one FK target) AND solves the owner-has-no-employment
hole that forced actions onto `User` in the first place — a pure owner is just a `WorkshopStaff`
with `owner: true` and no roles. Settled: **Path B** (owner grants operational power, keeps the
door's owner branch — less delta than making owners hold a role); holdings AND actions both
point at the person; roles are append-only rows, multi-role on for v1 (no stamp-precedence
needed once Path B simplified the door).

**Why the composite FK is the point.** A global-`User` actor FK can only be tenant-checked in
Ruby — a bug or forged id could stamp an event with another workshop's actor and the DB wouldn't
care. `(actor_id, workshop_id) → workshop_staff(id, workshop_id)` makes that a foreign-key
violation. This is the actor-integrity twin of ADR-007's read isolation. Proven by a dedicated
test (a persisted staff row from workshop A, rejected as `created_by` on a workshop-B event).

**Two real things caught during the build** (neither by unit tests alone):
1. **`as_user` test helper leaked tenant visibility.** `Workshop.create_with_owner!`'s
   `SET LOCAL app.workshop_id` runs in a savepoint (the test's wrapping transaction), so it
   outlives that inner block — an `as_user` block right after, with no intervening `as_tenant`,
   still saw the workshop's rows. Fixed by `RESET app.workshop_id` at the top of `as_user`,
   mirroring the app's own top-of-request double wipe.
2. **The `crew` route hit the same singular==plural quirk** as the model (`crew`/`staff` are
   their own plurals) — a hand-rolled `get`. Resolved in review (below) by folding it into the
   `workshop_staff` resource.

**Review round (builder, post-build).** Four points, all handled: (1) `CrewController` →
**`WorkshopStaffController`**, routes consolidated into one `resources :workshop_staff` block
(index = crew list, roles nested); (2) `before_action :set_technician` in the technicians
controller — `find_by` + nil-guard turns a latent `RecordNotFound` 500 on a stale
`staff_role_id` into a friendly flash; (3) a naming convention settled — **`@<subject>_options`**
for any controller ivar that populates a `<select>` (`@technician_roles` → `@technician_options`;
pinned in the Sprint-plan conventions, applies to S3's `@blocker_options`); (4) confirmed
`Job#timeline`'s reach to `.user` is correct — the actor *is* `WorkshopStaff`, and `.user.email`
is only the display label (thin `User` is the only place a human name lives). A `delegate :email,
to: :user` on `WorkshopStaff` to flatten the view was offered and deferred.

**Live-verified in-browser** (seed DB + a second workshop): the wrenching-towkay job lifecycle
(register → assign self → start work); the pure-owner **empty-state** ("No technicians yet —
add crew, or add a role to yourself"); and **cross-tenant isolation** at the HTTP layer (a
Seed-Workshop-One user hitting another workshop's job URL → `RecordNotFound`, not a leak).

**Vault housekeeping.** New `09 Archive/` folder (rules in its index: closed sprints + old
worklog sessions belong; ADRs, Deferred design, Open questions, Rejected alternatives, and
Concepts notes never do). Sprints 0–2.5 and Sessions 1–11 relocated verbatim; live `Sprint plan`
and `worklog` trimmed with pointer notes. Sprint 3 is now unpaused — its blocker tables build on
the settled actor shape.

**Also touched:** `CLAUDE.md` tenancy invariant rewritten (WorkshopEmployment/Ownership →
WorkshopStaff + WorkshopStaffRole) — untracked/gitignored, edited on disk in the same pass.

**Next:** Sprint 3 (blockers) — the design carry-back is already in hand (three records:
catalog + `JobBlocker` item + `JobBlockerTransition` events; the `blocks` enum resolving the
`→ done`/Hold-For-Payment collision; sectioned `JobActions`).

---

## 2026-07-18 · Session 24 — Sprint 2.5 built: cold-start intake / customer–vehicle CRUD

**Summary.** Built the Session 23 design in full — S2.5.1–S2.5.7, four commits
(`fb710e2`, `33583fd`, `cea8a66`, plus the migration folded into the first). The cold-start
hole is closed: a fresh 0-customer workshop can now take in its first car end to end, no
console needed. Both entry paths (plate-first search, customer-first via the new Customers
index) live-verified in-browser; the exit criterion is also an executable system test.
69/69 suite green (64 model/service + 5 system).

**What landed.** `CustomersController` (index/show/new/create/edit/update, all
`for_current_workshop`-scoped and `require_counter!`-gated — a new guard delegating to
`JobActions.counter_staff?`, the same one-source-of-truth pattern as `PermissionsHelper`).
`Customer` gained `total_jobs`/`last_visit`/`active_job_count` (activity only, no money —
ADR-002) and a `.search` scope (name ILIKE or the canonicalized phone). Two additive
columns, `address` + `contact_person`. `Vehicle.canonicalize` extracted from the model's
`before_validation` so the new plate-search reuses the exact normalization storage uses —
search and storage can never disagree. `Customers::JobsController` is the slice's one door
touch: finds or births the vehicle under the customer (which is *why* `job.customer ==
vehicle.customer` at birth in every S2.5 path — the S2.5/S6 line holds structurally, not by
convention). `JobsController#new`/`#create` reworked from S2.11's dropdown into a plate
search: hit-with-active-job redirects to the open job (Intake flow §1c), hit-with-free
redirects to the add-job confirm screen, miss redirects to customer-first creation carrying
the plate through.

**Two real bugs caught by live browser verification, not by the test suite** (both fixed
before committing) — worth naming because they're the kind unit tests structurally can't
catch:
1. **`@customer.vehicles.create!(registration_number: ...)` silently left `workshop_id`
   nil.** Rails only auto-populates the FK of the association being traversed
   (`customer_id`); `Vehicle`'s separate `belongs_to :workshop` isn't inferred from
   `customer.vehicles`. This tripped `belongs_to`'s presence validation — which my own
   rescue then *mislabeled* as "already in the book" (a real collision message
   masking a completely different failure). Fixed by passing `workshop: @customer.workshop`
   explicitly. The log trace (`workshop_id IS NULL` in the uniqueness-check SQL) is what
   gave it away — a symptom worth remembering: a rescue that catches too broadly can hide
   the wrong bug behind a right-sounding message.
2. **The typed plate didn't survive the miss→create-customer→add-job hop.** The customer
   form carried `registration_number` through as a hidden field correctly, but the
   downstream add-job view never read it back into the "new vehicle" text field — so the SA
   would land on an empty field after typing the plate twice already. Fixed by threading
   `params[:registration_number]` into the field's `value`. Caught by reading the DOM's
   actual `.value` (not just its accessible-name label, which showed the placeholder either
   way) — a reminder that accessibility-tree reads can mask a genuinely empty field.

**Regression caught in the same session:** S2.12's `job_lifecycle_test.rb` (Session 22)
still drove the vehicle dropdown this sprint replaced — `select "WLC1234", from:
"job_vehicle_id"` no longer exists. Updated to the plate-search flow it now needs; full
suite re-confirmed green before commit.

**Verified live, beyond the automated suite:** all three plate-search branches (miss,
hit-with-active-job via §1c, hit-with-free-vehicle after pushing a seeded job to
`delivered`); the customer index's search scope; the activity panel's counts against a real
two-job customer; `require_counter!` correctly refusing a technician (redirect + flash, no
500) exactly the way the door's own guards refuse.

**Open, carried forward.** Sprint 3 (blockers) kickoff — still needs its own design pass
(the three-record respec, the Hold-For-Payment/`→done` collision). S0.8 deploy stays
re-parked to "a stable/semi-full v1." Sprint 6 inherits the full disambiguation tree, the
mismatch confirm, and vehicle reassignment — none of which S2.5 touched, by design.

---

## 2026-07-17 · Session 23 — Sprint 2.5 designed: cold-start intake / customer–vehicle CRUD

**Summary.** Design-only session (no code). Spotted a hole while reviewing where "taking in
jobs" sits: **there is no UI to create a Customer or Vehicle at all** — the models shipped
S2.1/S2.2 but rows only entered via seeds/console, so a fresh 0-customer workshop can't take
in its first car (S2.11's new-job form is a dropdown of *existing* vehicles → dead end from
empty). Converged with the builder on a new **Sprint 2.5** slice (between the closed Sprint 2
and unstarted Sprint 3) — customer CRUD + a customer-first intake motion — recorded in the
[[Sprint plan]]. Full task list + the hard S2.5/S6 boundary live there; this entry keeps the
reasoning chain.

**Vehicle stays a first-class entity — the decisive argument named.** The question "do we
even need a vehicle database?" got a real look. History and R5 are fakeable with a
registration-number string; the clincher is **ownership change over time** — a car outlives
its owners, and only an entity can carry `vehicle.customer` as a mutable-present pointer while
`job.customer` stays the frozen stamp (the sold-car design, [[Data model]] §Resolved).
Framing to keep: **the vehicle is the persistent identity; the customer is the changing
relationship.** Corollary that shaped the whole slice: **a vehicle alone is meaningless** —
it's born at intake beside its first job, never standalone. So "Add job" is the verb, a
vehicle is the byproduct; no orphan "add vehicle" CRUD.

**Job is the point of contact, not the vehicle** (builder's deliberate trap, confirmed):
frozen `job.customer` = who we bill/call *this visit*; `vehicle.customer` = the mutable
present owner. History accrues on the frozen stamps — so the v2 "pleasant surprise" vehicle
history needs **zero v1 work** (already accruing; v2 is an additive column + the hard,
deferred cross-tenant RLS).

**Routing, not ownership** (the company ruling). Person-vs-company (`kind` toggle on the flat
`Customer`) picks the *file/phone to route to*, never legal ownership. The boss's lorry and
the boss's private car are **two cards, correctly** — different phones/payers. v1 ships
companies as flat records only: **no** Company org entity, logins, `customers.user_id`, or
person→company edge — all v2, gated behind the unsolved cross-tenant RLS ([[Data model]] v2 /
Session 14). Three traps the builder set and we caught and declined: (a) "company works like a
workshop" (= v2 cross-tenant, ruled out Session 14); (b) "encapsulate users in v1" (inert
column = false readiness); (c) "person as a company's owner" link edge (v2 claim machinery
through the side door). Builder *won* one against initial resistance: flat additive fields
`address` (both kinds) + `contact_person` (company) — legit workshop contact info, not v2
schema.

**The S2.5 / S6 line, stated hard.** 2.5 = **lookup** ("does this exist?"); S6 =
**disambiguation** (phone verify, the four forks, the two-branch mismatch confirm,
changed-hands reassignment). The structural tell: **2.5 has no "who's paying?" override** —
the stamp defaults silently (`vehicle.customer` on a plate hit, the just-created customer on a
miss), so `job.customer == vehicle.customer` at birth in *both* branches → a mismatch is
**unrepresentable** in 2.5. That's *why* the deferred match-validation / confirm stays cleanly
parked ([[Deferred decisions]]) until the plate-first override arrives at S6 — no contradiction,
no rework. Dedup is the real v2-surprise-spoiler; plate-first screening + name/phone customer
search are the cheap v1 mitigations, the full phone-first dedup tree is S6.

**Open for the build:** the reg-collision stand-in (rescue `RecordNotUnique` → "pick from the
list"), `customers#show` primary/secondary layout ("Add job" vs quiet "Maintain customer"),
and the plate-search entry replacing S2.11's dropdown — all in the [[Sprint plan]] S2.5 tasks.
Next: build on the builder's go (no app code yet).

---

## 2026-07-17 · Session 22 — Sprint 2 closed: Design B + edge rename built, S2.12 shipped, S0.8 re-parked

**Summary.** The close-out plan from Session 21's addenda was built in three commits, each
verified live before the next: `c692451` (Design B crew restructure + technician
vocabulary), `7a7881d` (edge rename, full sweep), `bee2c2a` (the ten-case S2.12 test
batch). 64/64 tests green (61 model/service + 3 system). Builder then held off Item 4 —
deploying to Heroku — a third time: nothing demoable enough yet to justify a live URL, and
local dev already proves what the deploy would (RLS enforcement, the full lifecycle).
**Sprint 2 is closed** on its own exit criterion, decoupled from deploy. Next: Sprint 3
kickoff (the blocker design pass), whenever the builder calls it.

**Design B, built.** One migration (`rename_job_mechanics_to_design_b`): `rename_table` ×2
(RLS policies and FKs ride the rename — ADR-007's footnote proved out in practice, not just
theory); transitions gained `job_id`/`workshop_employment_id`, backfilled via one UPDATE
join through the old `job_mechanic_id` FK before it was dropped; membership rows whose
engagement had already `left` were deleted, so present-tense semantics started true on a
seeded dev DB, not just an empty one. **Gotcha found mid-migration:** both crew tables carry
`FORCE ROW LEVEL SECURITY` (every tenant table does, ADR-007) — which polices even the
table owner, so the backfill UPDATE silently touched zero rows under the migration's own
connection until `NO FORCE`/`FORCE` bracketed the data steps. A second, smaller gotcha:
Rails' `rename_table`/`rename_column` already auto-rename dependent indexes — the plan's
explicit `rename_index` calls were redundant and had to be dropped when they errored on
already-renamed index names. Console-verified after: assign → remove leaves membership at
zero while `job.timeline` still shows both `joined`/`left` events plus the compensating
`assigned → registered` rollback — the whole point of the split, proven, not just asserted.

**Edge rename, built.** Two `rename_table` calls; the real work was the Ruby sweep —
models, `Current`, the access door, both crew models' `belongs_to :workshop_employment`
(class name now matches, no override needed), `WorkshopEmploymentsController` (URL kept
`/employments` — this session's ruling), seeds, and every test file. **Tooling gotcha,
twice:** macOS `sed` doesn't support `\b` word-boundary regex — patterns like
`s/\bEmployment\b/WorkshopEmployment/` silently matched nothing, so an early pass looked
complete (no errors) but left ~15 bare `Employment`/`Ownership` references across five test
files, surfaced only when the suite ran. Switched to Python's `re` for the real thing.
Second-order lesson: compound identifiers (`EmploymentTest`) have no boundary between the
words for regex purposes either way — those two class names needed a direct string
replace. Full suite + a live browser walk (dashboard, crew page, full job lifecycle) both
confirmed clean before commit.

**S2.12, built — ten cases, two real test-writing lessons worth keeping.** (1) System
tests must use `assert_text`, never a raw `page.text` read, to check state after a
navigation — `page.text` doesn't wait for Turbo/redirects, so an early debugging pass
misdiagnosed two real app-flow issues (tech and SA both need to navigate to a job page
after sign-in; the dashboard only lists `Job.active` jobs, so a job that's reached `done`
correctly disappears from it) as sign-in failures, because the raw read was racing the
page. This is the sign-in-side twin of the S1.14 async-sign-out race — same family of bug,
opposite direction. (2) **Flash is read-once**, and a forged same-origin XHR that follows
its own redirect consumes it: a synchronous XHR to an illegal verb redirects to the
dashboard with a flash, and if the test then does a *second* `visit`, Rails has already
popped and cleared that flash during the XHR's own response cycle — the assertion has to
read the XHR's own `responseText` instead, which **is** the rendered, redirected page.
Once both were understood, all ten cases passed cleanly; case 2's stage×verb loop is now
the allow-list's executable spec, and case 5's `assert_no_difference` proves the
three-rows-one-transaction claim rather than just asserting the happy path.

**S0.8 — re-parked a third time, on the record.** Builder: hold off Heroku until a
stable/semi-full v1 exists — nothing to demo yet, and local dev already proves the RLS
guarantee and the full lifecycle a deploy would. New trigger: Sprint 3 or Sprint 4 landing.
The Heroku choice and shape (Procfile, release-phase migrate, the deploy-day RLS proof)
stand unchanged from Session 21 — nothing to redo when the day comes.

**Open, carried forward.** Sprint 3 kickoff (blocker design pass: full respec, the
Hold-For-Payment/`→done` collision, Roadmap slice 4's descriptor) is next, unbuilt, on the
builder's call per standing discipline.

---

## 2026-07-17 · Session 21 addendum 3 — the vault half of the close-out plan, landed

**Summary.** The rulings from addenda 1–2 are now written into the docs they bind, ahead of
any app code (house discipline: the spec is true before the build starts). One commit.
- **Design B** recorded as a dated partial supersession of Session 17: [[Event log]] (the
  careful one — "entities written once" narrowed to *crew membership is a present-tense read
  model; append-only binds the event log, always*; In-Rails shapes for `job_technicians` +
  self-contained `job_technician_transitions`), [[M1-F1 Status flow and transitions]] (crew
  bullet + verb renames), [[Job]], [[Data model]] (diagram + trackers bullet).
- **Edge rename** (`WorkshopEmployment`/`WorkshopOwnership`): dated footnotes on ADR-004/
  006/007/008 (decisions unchanged, names only; ADR-007 notes policies ride `rename_table`);
  [[Data model]] entities + diagram; CLAUDE.md invariants. Design laws needed nothing —
  law #1 names no tables. **Pinned in [[Data model]] §Resolved: "holdings point at the
  stint, actions point at the person."**
- Also landed: [[Deferred decisions]] crew entries updated (the `lead` flag lives on membership
  rows; swap verb names); Sprint plan — S2.6/S2.9 tick annotations get supersession pointers
  (history preserved, current truth linked), **S2.12 respecified to the ruled ten cases**,
  S4.1/S4.2 names + the new ⚠ S4 design-pass item (technician removed before acking their
  `joined` — decide if the debt dies with removal), **S0.8 Heroku ruling** recorded
  (Procfile not render.yaml, release-phase migrate, deploy-day RLS proof).
- Fixed in passing: [[Data model]]'s diagram still listed `owner` in the role enum — stale
  since ADR-006 removed it; caught while redrawing.
- Open for the coding plan: the Mechanic-vs-Technician **user-facing label** question
  (internals say technician; screens currently say "Mechanic") — builder rules at plan time.

---

## 2026-07-17 · Session 21 — S2.11 built: the job engine gets its first UI

**Summary.** S2.11 (controller + views for create-job / show-job / stage buttons) built and
live-verified in one session. Four app commits: `16c750a` (foundations), `f6c4bb4`
(routes + controllers), `3dd4cc6` (views), `e2c30a0` (mobile fix). 51/51 suite green
throughout; full lifecycle walked live in the browser under all three personas. Sprint 2 now
stands at everything-built-except-tests: S2.12 (test batch) and S0.8 (first deploy, parked
twice to this exit) are all that remain before sprint close.

**Housekeeping first — the Session 20 audit carried into code.** Before any S2.11 work, the
uncommitted `create_with_founder!` half-rename was **reverted** per the audit ruling
(vocabulary stays *owner* — ADR-006's term; no third word). Then the seeds finding became
S2.11's opening tick exactly as the new planning convention prescribed (current state vs
suggested fix): `db/seeds.rb`'s bare `Job.create!` calls rerouted through
`JobActions.register_job!`/the door verbs, so seeded jobs now carry real birth rows and
timelines (`16c750a`).

**The intake-flow discussion that re-shaped `jobs#new`.** The naive version — dropdown of
all vehicles — was rejected in discussion: [[Intake flow]] 1c says a vehicle with an active
job cannot take another (Risk ledger R5), so offering busy vehicles just manufactures
refusals. Ruling: `jobs/new` lists **eligible vehicles only** (no active job), and the door
itself gains a matching **busy-vehicle guard in `register_job!`** — a readable refusal ahead
of the R5 index's last-resort `RecordNotUnique`. Defense in depth: the view filters, the
door refuses politely, the index backstops. `Job.active` (stage 0/1/2) landed as the named
scope both sides share.

**Permissions: door predicates + a delegating helper — one source of truth.** "Which buttons
does this user see" (the CanCanCan trigger watched for since Session 19) resolved without a
gem: `JobActions` gained boolean predicates `counter_staff?`/`job_crew?` — the same logic
its `ensure_*!` guards raise on — and a `PermissionsHelper` that **only delegates** to them.
Views ask the door what's allowed; the door remains the single authority; a forged POST hits
the same rule as a hidden button. Role-gated buttons verified live: SA sees counter verbs,
technician-on-crew sees start/mark-done, parts advisor sees no verbs.

**Views + the stage→color ruling.** `jobs/show`: stage badge, crew card, role-gated verb
buttons, merged timeline (the S2.6 `Job#timeline` rendering for the first time).
`_stage_badge` partial + badge CSS on the sacred palette; ruled at build:
registered/assigned/cancelled = **neutral**, in_progress = **info blue**, done/delivered =
**success green** — amber stays reserved for aging (S5.3), red for blockers (S3). Dashboard
gained the active-jobs list. `send_back!` correctly appears on no technician screen.

**Verification.** 51/51 green after every commit. Live browser walk: full
register→assign→start→done→deliver lifecycle; forged POST (technician firing a counter
verb) → refusal flash, not a 500; `turbo_confirm` guards mark-done; 375px mobile check
caught a timeline wrap bug, fixed in `e2c30a0`.

**Deferred:** JSON responses for the door mutations (ADR-001's "every mutation available as
JSON") — builder ruling: HTML first; ~2 lines per action whenever a non-browser consumer
appears. Entry in [[Deferred decisions]].

**Next:** Sprint 2 completion review (aims vs built, how the engine carries the later
sprints, v2/v3 receptiveness audit), then S2.12 scope + S0.8 deploy research.

**Addendum — Sprint 2 completion review (builder rulings).**
The review ran in full: aims-vs-built (exit met, everything built beyond spec; S2.12 tests +
S0.8 deploy are all that remain), the engine-completes-Knot trace (every later v1 feature
lands additively on the Sprint 2 rails), and the v2/v3 receptiveness audit (**Design law #5
clean** — lead-flag backfill, employment-stint split, dormant ack pairs, fleet-customer
claims, v3 skill tracking all additive-confirmed against the schema as built; the one risk
is process, not schema: the S3.2 blocker respec + HFP→done collision must be resolved at
Sprint 3 kickoff before the first blocker migration). Rulings, in discussion order:

1. **S2.12 scope approved** — the ten-case list: legal path with exact rows · illegal moves
   exhaustively (the allow-list under test) · Done freeze · role gating incl. the
   manager/owner escape hatch · assignment three-rows-one-txn + refused-assign-writes-zero ·
   birth row + busy-vehicle guard · removal legality (responsibility rule, `started_work?`
   as history) · acks-live-on-events schema assertion · the sprint's model tests
   (canonicalization, `Job.active`, `started_work?`, `timeline`, current-crew read) · one
   Capybara journey (register→assign→start→done→deliver + a forged-POST refusal → flash).
2. **Deploy target = Heroku** (S0.8). RLS axis is a tie — managed Postgres on either
   platform never grants superuser/BYPASSRLS, and every policed table already carries
   `FORCE ROW LEVEL SECURITY`, which binds RLS onto a table-owning role (the S1.8 local
   problem structurally can't recur). Builder picks Heroku on familiarity; accepts US/EU
   region latency (Render's Singapore region was the counter-argument, on record). Procfile
   + release-phase `db:migrate`; Eco + Mini Postgres ≈ $10/mo floor; deploy-day proof =
   `SELECT rolsuper, rolbypassrls FROM pg_roles WHERE rolname = current_user` + the
   unset-GUC → zero-rows smoke test. S0.8's `render.yaml` wording to be corrected to
   Procfile when built.
3. **Edge-table rename ruled — both edges, one scheme:** `Employment` →
   **`WorkshopEmployment`**, `Ownership` → **`WorkshopOwnership`** — organisation-prefixed
   edges, mirroring v2's `CompanyEmployment` / `CompanyOwnership`-style pair (no
   half-applied scheme; the founder-incident lesson). **No code this session** — parked to
   the carry-back; the main session composes the plan (current-state-vs-fix per item, app +
   vault + ADR-004/006/007/008 dated footnotes landing together). Reaffirmed alongside:
   the actor/holder rule stands — **holdings point at the stint, actions point at the
   person** (owners act but hold no employment).
4. **Crew tracker restructure ruled — "Design B", supersedes the Session 17
   engagement-permanence half.** `job_mechanics` becomes a **pure membership table** (the
   crew *right now*): rows created on assign, **deleted on remove**; unique
   `(job_id, employment_id)`. `job_mechanic_transitions` becomes **self-contained history**:
   gains direct `job_id` + `employment_id`, drops the `job_mechanic_id` FK; keeps `action`
   (`joined`/`left`), `created_by`, the dormant ack pair, `workshop_id`. Both tables keep
   their own `id`. Consequences accepted with eyes open: `JobMechanic.current` disappears
   (the table read IS the crew — the builder's readability motivation, and it makes crew
   symmetric with the stage tracker's mutable-entity + event-log shape); past stints become
   event *pairs*, not rows (Sprint 8 attribution reconstructs by pairing; the deferred
   `lead` flag will live on membership rows — current stints only); the tracker pattern
   goes non-uniform vs Sprint 3's blocker items (which stay undeleted — they carry
   notes/attribution). Safety property that makes deletion honest: removal is only legal
   before work starts (responsibility rule), so only never-worked mistake-assignments are
   ever deleted, and their receipts survive in the events. Why `.first` was wrong under the
   old shape, on record so it isn't re-derived: engagements were append-only history — after
   assign→remove→assign, the bare association's first row was the *removed* mechanic, and a
   crew-emptiness check against it would have refused every re-assignment forever;
   `.current` (no-left-event subquery) was the history/present separator. Design B removes
   the trap by making the table present-only. Doc supersessions (Event log, M1-F1,
   Data model, Deferred-design lead entry) travel with the build plan, dated notes.

Review findings: finding 1 (the mechanics controller's single-mechanic `.current.first`)
is **mooted by Design B** — `job.job_mechanics.first` becomes plainly correct. Finding 2
(`Job#timeline` merges/sorts in Ruby — fine per job, must never be called in a loop; Sprint
8 reports get their own SQL per Design law #3) — proposed as a Sprint-plan pin; **builder
ruled: skip** — left to the S8 design pass. The Sprint-4 edge (a mechanic removed before
acking their `joined` event) exists in both designs — note for the S4 design pass.

**Addendum 2 — plan-readiness rulings (the five naming/scope decisions closed).**
1. **FK columns follow the rename:** `employment_id` → `workshop_employment_id` on the two
   crew tables (the only tables carrying it post-restructure; nothing references
   `ownerships`). Zero `foreign_key:` overrides left behind.
2. **Full Ruby sweep, no aliases:** associations (`user.workshop_employments`),
   `Current.workshop_employment` / `Current.workshop_ownership`, every call site — one
   vocabulary everywhere; table-only renames would recreate the founder-incident smell.
3. **Crew vocabulary unified: technician.** The builder's `job_technicians` sketch surfaced
   a pre-existing split — the role enum and assignee guard said *technician*
   (`role: :technician`, `ensure_active_technician!`) while the crew tables and verbs said
   *mechanic* (`JobMechanic`, `assign_mechanic!`). Ruled while the table is being rebuilt
   anyway: **align to technician** — `job_technicians` + `job_technician_transitions`
   tables, `JobTechnician` / `JobTechnicianTransition` models, door verbs
   **`assign_technician!`** / **`remove_technician!`**, `Jobs::TechniciansController`.
   M1-F1 / Event log / Data model wording gets dated notes with the build. (Open one-liner
   for plan time: user-facing view labels — keep "mechanic" as natural workshop language
   over technician internals (the Pilot/Knot precedent), or say technician everywhere.)
4. **No membership→events association:** under Design B, events relate to a membership row
   only by composite `(job_id, employment)` — no fake FK association; events are read via
   `Job#timeline` or the employment.
5. **Sequencing accepted:** crew restructure → edge rename → S2.12 (against the final
   shape) → S0.8 Heroku deploy. Rename-after-restructure means the sweep also catches the
   new transition columns.
No code this session; the full carry-back for the main session's coding plan was produced
at session close.

---

## 2026-07-17 · Session 20 — Vault coherence sweep: stale docs re-aligned with existing rulings

**Summary.** A full coherence sweep found documents lagging behind decisions already made —
**no new decisions**, every fix just makes a doc agree with a ruling recorded elsewhere.
Fixes applied, worst first:
- **[[Open questions]] · one-active-job-per-vehicle** (HIGH — stated the *opposite* of shipped
  reality): the old "leaning yes, `WHERE stage NOT IN (delivered, cancelled)`" text replaced with
  the actual 2026-07-15 ruling ([[Risk ledger]] R5, commit `2c5ca91`): index is
  `jobs(vehicle_id) WHERE stage IN (0,1,2)` — a Done job does **not** block a new job;
  follow-up-after-done is legal and [[Intake flow]] 1c depends on it.
- **[[Open questions]] · single-vs-multiple assignees**: "✅ resolved: multiple, one `primary`"
  rewritten to the superseding 2026-07-16 ruling — v1 single mechanic, no flag; future flag is
  `lead`, not `primary` ([[Deferred decisions]] + M1-F1 Settled 2026-07-16).
- **[[Sprint plan]] S4.1**: ack columns corrected to event rows only —
  `JobStageTransition` / `JobMechanicTransition` / `JobBlockerTransition`; engagements
  (`JobMechanic`) carry no ack columns post-restructure. S4.2–S4.7 checked — S4.3 already
  carried the `.pending_ack` correction; nothing else stale.
- **[[Roadmap]]** (untouched since 07-06): slice 4 status honest-ed to "⚠ needs respec at
  Sprint 3 kickoff" (per [[Blocker]]'s restructure-pending warning — three records); slice 5
  inbox note corrected from bare `acknowledged_at IS NULL` (overcounts) to the `.pending_ack`
  handoff predicate ([[Event log]]).
- **[[Rejected alternatives]] · RLS-first tenancy**: rejection stands; the *why* updated to
  cite ADR-007 (RLS live as the Sprint-1 backstop) instead of ADR-004's superseded
  "additive hardening" clause.
- **[[M1-F1 Status flow and transitions]]**: acceptance draft drops "primary"; the matrix's
  technician cell gains the crew-gate qualifier (current engagement on that job) so the table
  can't mislead when quoted alone.
- Cosmetic: [[Product overview]] Stage line refreshed to mid-Sprint-2; [[Stage model]] ASCII
  diagram redrawn so Cancelled branches from the active stages, never Done.

Explicitly **not** touched (recorded elsewhere, need builder decisions, not doc fixes):
Product-gaps #3/#4/#7/#8 and the S2.11 JSON-endpoint reminder.

**Addendum — code audit (read-only) of the Sessions 18–19 build against the vault.**
The door, models, and schema all match the rulings: eight verbs exactly, allow-list
structural, `with_lock` on every job-taking verb, `registered↔assigned` crew-method-private,
engagement→employment stint, `started_work?` a history query, `Employment` append-only
(restrict + FK backstop), RLS `tenant_isolation` on all three new event/crew tables (policy
born with each table), R5 partial index `WHERE stage IN (0,1,2)`, no endless methods, no bare
tenant queries. **Proof:** `bin/rails test` 51/51 green; dev-console lifecycle
register→assign→start→(send_back→restart)→done→deliver on BMT8056 wrote 8 timeline rows,
three refusals fired correctly (remove at in_progress, responsibility rule after `send_back!`,
cancel-from-done), rows visible under the workshop GUC and invisible after `RESET`. Two
findings, neither in committed Sprint-2 code: (1) the builder's **uncommitted**
`workshops_controller.rb` diff calls `Workshop.create_with_founder!(founder:)` but the model
still defines `create_with_owner!(owner:)` — create-workshop 500s while the diff is applied,
and tests stay green (they call the model directly), so the half-rename is invisible to the
suite; finish the rename (model + 12 test files + invitation comment) or revert. (2)
`db/seeds.rb` still writes jobs with bare `Job.create!` + a now-stale "door arrives Phase 3"
comment — seeded jobs carry no birth rows, so the in_progress seed job has an empty timeline
and `started_work?` = false; cheap to reroute through `JobActions` before S2.11 views render
dev data. Low note: `ensure_job_crew!` inlines `ended_at: nil` instead of merging
`Employment.active` — fine today, drifts if "active" ever grows a second condition.

**Addendum 2 — audit carry-back (builder rulings on the three findings).**
The audit findings were discussed with the builder; verdicts, in finding order:

1. **Vocabulary stays *owner* — the half-rename is reverted, not finished.** The uncommitted
   `workshops_controller.rb` diff calling `Workshop.create_with_founder!(founder:)` goes back
   to `create_with_owner!(owner:)`. "Founder" would be a third word for a concept ADR-006 and
   the whole vault call *owner*/Ownership; the codebase stays consistent with the vault's
   vocabulary. (Builder to apply the one-line revert in their working tree — no audit-session
   code changes.)
2. **New planning convention (standing, applies from S2.11 onward): every coding plan opens
   with *current state of the code* vs. *suggested fix*.** A plan must first state what the
   code does today (file:line where useful), then the proposed change — so the delta is
   explicit and reviewable before any code is written. First application: the S2.11 plan
   should open with the seeds finding in exactly this shape (current: `db/seeds.rb` writes
   jobs with bare `Job.create!`, no birth rows, stale pre-Phase-3 comment; fix: reroute the
   three seed jobs through `JobActions` as the session's first tick).
3. **`ensure_job_crew!` keeps its inline `ended_at: nil`.** Ruled fine as-is — swapping it
   for a `merge(Employment.active)` later is cheap, and touching working door code for a
   one-condition duplication isn't worth it. Parked, not a debt.

**Session summary.** This session was a read-only audit of the Sessions 18–19 job engine
against the vault (addendum above: everything matches, 51/51 green, console lifecycle +
RLS proof, three findings). The discussion then settled fix-timing: the half-rename is the
only blocker (dirty working tree next to where S2.11 code will land, invisible to the test
suite); the seeds reroute becomes S2.11's opening task rather than a pre-task; the guard
duplication is accepted. Next session: plan S2.11 under the new current-state/suggested-fix
convention.

---

## 2026-07-16 · Session 19 — Phase 3 designed: the `JobActions` verb surface (rulings swept)

**Summary.** The Phase 3 design discussion ran in a chip session and closed with builder
verdicts on every open question; this session sweeps them into the vault before any code.
Headline rulings: **named verbs, no generic `change_stage!`** — the door's public surface is
**eight** bang class methods (`register_job!`, `assign_mechanic!`, `remove_mechanic!`,
`start_work!`, `mark_done!`, `deliver!`, `send_back!`, `cancel!`; a chip-session doc said
"seven," miscount), each its own `def`/`end` block with a first-line stage guard, so the
allow-list IS the verb set and the Done-freeze is structural (nothing accepts `done` but
`deliver!`; nothing accepts the terminals). Refusals raise `JobActions::Refused` with a human
message; every verb wraps read-check-write in `job.with_lock` — the **row-lock deferral
woken** (one shared line, the "wait until it hurts" trade lost its upside) —
[[Deferred decisions]] superseded with a dated note. Next: build S2.7–S2.10 against this spec, on the
builder's go.

**The rulings, in brief** (full detail: [[M1-F1 Status flow and transitions]] Settled
2026-07-16 (Phase 3) + the respec'd S2.7–S2.10).
- **`swap_mechanic!` dropped for v1** (amends Session 17's "the swap is the only in-progress
  crew motion" — v1 now has none). Sick tech shows truthfully until done/cancelled.
  **Escape hatch, recorded explicitly:** the manager/owner exemption in the crew gate means
  a manager can still drive a stuck job to `done` — the workshop is never trapped, the job
  just keeps naming the sick tech. New [[Deferred decisions]] entry; trigger = first real
  mid-job handover need.
- **Tech moves crew-gated:** `start_work!`/`mark_done!` need active technician employment
  AND a current engagement on that job (manager/owner exempt).
- **"Touched `in_progress`" = `Job#started_work?`**, a history query on
  `job_stage_transitions` (`to_stage: :in_progress`), never stage-based — `send_back!`
  keeps it true. Session 17's ⚠ sanity-check on that edge: **checked, consistent** — after
  `send_back!` the job is assigned + crew + started, both remove and assign refuse, ruling
  applied uniformly.
- **`registered ↔ assigned` crew-method-private:** those stage rows are written only inside
  `assign_mechanic!`/`remove_mechanic!` transactions — "assigned ⟺ crew exists" unbreakable.
- **`send_back!` kept** (counter-only) but reframed: rare compensating correction, never on
  technician screens (S2.11 note). Internal name only.
- **Crew freezes on terminals** — `cancel!`/`deliver!` write no synthetic `left` events.
- **Accepted edge:** `mark_done!` has no compensating path (done's only exit is `delivered`);
  soften with an S2.11 confirm dialog.
- **Positioning pin:** Knot tracks job statuses, never real-time tech activity — many
  `in_progress` jobs per tech is normal, no pause/resume ever joins the stage axis; per-tech
  workload is a query feeding S5.2, zero Phase 3 work.

**Sweep.** [[Deferred decisions]]: row-lock entry superseded (woken), new `swap_mechanic!`
entry, self-join entry's swap sentence dated-corrected.
[[M1-F1 Status flow and transitions]]: new Settled 2026-07-16 (Phase 3) section + dated
corrections on the
responsibility rule's swap wording, the resolved ⚠ edge, and the concurrency-deferral line.
Sprint plan S2.7–S2.10 respecified to the verb surface (guards: `ensure_counter!` ·
`ensure_crew_technician!` · `ensure_technician!` + per-verb stage checks; S2.10 =
`register_job!`, customer defaults to `vehicle.customer`). Correction on record: `JobActions`
does **not** exist yet — no `app/services/` — Phase 3 creates it from scratch;
`register_job` was spec'd but never coded.

**Build addendum (same day) — Phase 3 built: S2.7–S2.10 ticked.** The door exists:
`app/services/job_actions.rb`, eight verbs + three guards + two helpers in one
`class << self` block, `Job#started_work?` on the model. Three app commits: `045f5c1`
(stage verbs, guards, `Refused`), `36bb90e` (crew verbs), `5592291` (`register_job!`).
Two decisions taken at plan review with the builder:
- **Checks inside `with_lock`** — the ruling's "stage check → role gate → lock" order
  would check stale state (the lock reloads the row; a pre-lock check is decorative).
  One check per verb, inside the lock, same reading order, actually race-safe. Explained
  via the Siti-cancels-while-Ah-Boy-marks-done race; builder chose it over a duplicate
  pre-check + re-check shape.
- **Guards renamed for plainer vocabulary**: `ensure_counter_staff!` / `ensure_job_crew!` /
  `ensure_active_technician!` (from the chip session's `ensure_counter!` /
  `ensure_crew_technician!` / `ensure_technician!`). Kept `ensure_` over the controllers'
  `require_` — those redirect, these raise. **CanCanCan raised and rejected**: door
  permissions are verb+stage+crew-membership business rules, not resource permissions;
  an Ability class would split ONE DOOR across two files. Revisit only if v2 grows
  truly dynamic per-workshop permissions (the Blocker catalog's role fields already
  handle the data-driven case).
Also ruled: **S2.12 tests wait for the sprint-close batch** — verification this session was
a 22-check console script (happy path end-to-end, every refusal, the post-`send_back!`
edge live-checked, untouched-assigned removal with compensating rollback, refused-assign
atomicity), all green, plus the suite (51/51) after each commit. One observation recorded
on S2.9: v1's "crew already full" refusal is shadowed by the stage guard (`assigned`
implies crew) — kept as belt-and-suspenders.

**Addendum 2 (same day) — the actor/holder split: engagements point at Employment
(`6799438`).** Post-build, the builder asked the honest question: shouldn't the schema
reference Employment instead of `user_id` — more accurate, keeps User bare? The discussion
ran the full arc. First position (defended): keep `user_id` everywhere — the **owner gap**
(owners act through the door but hold no Employment, ADR-006) makes an employment FK
unrecordable for actor columns; polymorphic actors and auto-created owner-employments were
both examined and rejected (the latter would reintroduce the `owner` role as data and
quietly rewrite the Employment-OR-Ownership access rule); role-at-action-time stays
derivable from `(user_id, workshop_id, created_at)` because employments are append-only.
Then the builder's real motivation surfaced — **"what did this guy do while employed at
X"**, jobs grouped per stint — and that flipped the answer *for one FK family*: on
`job_mechanics`, the owner gap doesn't exist (every legal assignee holds an active
technician employment, by the door's own guard), and an engagement genuinely is a
responsibility held by a *stint*. **Ruled: the split** — `job_mechanics.employment_id`
(holder = stint; `employment.jobs` direct; re-employment attributes per stint; sick-tech
engagement pointing at an ended employment reads truthfully), while `created_by`/
`acknowledged_by` on all event tables stay User (actions are taken by people; owners act).
Rider ruling: **employments are append-only** — `end!` never delete, role change = end +
new row — now load-bearing (engagements FK to stints; `restrict_with_error` + DB FK
enforce it). Built same session: migration on the empty table, `ensure_active_technician!`
returns the stint, crew lookups join through `employments` (`ensure_job_crew!` simplified —
the engagement's employment is technician by construction), 3 test files updated, 51/51
green, 12-check console verification (stint attribution, re-employment onto the new stint,
destroy refused, off-crew refusal through the join) all pass. Full reasoning:
[[Data model]] §Resolved.

**Open, carried forward.** Sprint 3 blocker respec + the Hold-For-Payment/`→done` collision
(unchanged); S2.11 controllers/views (confirm dialog, `send_back!` counter-only); S2.12
test batch at sprint close; S0.8 deploy at sprint exit. Standing parked items unchanged.

---

## 2026-07-16 · Session 18 — Phase 2 built: JobStageTransition + crew split (S2.5/S2.6)

**Summary.** Straight implementation of Session 17's design — schema, models, associations,
`Job#timeline`, tests. Nothing writes these tables yet (`JobActions` is Phase 3); this was
deliberately a contained, demo-free chunk so any schema surprise surfaced in isolation.
None did: the build matched the vault spec exactly, no design changes mid-build. Commit
`30f3a10`. Sprint plan S2.5/S2.6 ticked. Next: Phase 3 (S2.7–S2.10, the `JobActions` door)
— on the builder's call, per standing discipline.

**What landed.** Three migrations (`20260716090000/090100/090200`), each bundling its RLS
`tenant_isolation` policy in the same file (ADR-007 gotcha 1), same shape as every prior
tenant table. `JobStageTransition`'s `from_stage`/`to_stage` enums **reuse `Job.stages`**
rather than redeclaring the integer mapping — one source of truth, so Risk ledger R5's index
predicate (`stage IN (0,1,2)`) and the transition table can never drift apart; `prefix: true`
on both keeps `from_stage_registered?` and `to_stage_registered?` from colliding on the same
method name. `JobMechanic.current` implements Event log.md's promised query ("engagements
with no left event") as a `where.not(id: ...)` subquery — deliberately the plain/readable
form over anything cleverer. `Job#timeline` merges `job_stage_transitions` +
`job_mechanic_transitions` (reached via `has_many :through`) by `created_at`, `includes`-ing
the author/acknowledger associations now even though nothing renders them until Sprint 4 —
cheap to add while touching the method, expensive to retrofit later.

**Verification.** 8 new unit tests (birth-row shape, enum-prefix independence, current-scope
behavior across joined/left, `#timeline` ordering) + the existing 43 = **51/51 green**.
Migrated clean in dev; `db/structure.sql` regenerated with the three new
`CREATE POLICY tenant_isolation` blocks. Live spot-check in `bin/rails console`: a row
created under `SET app.workshop_id` was visible, then invisible after `RESET` — the
fail-closed backstop re-verified on new tables, as every prior tenant table got. `db:seed`
re-run clean and idempotent (these tables aren't seeded yet, by design — nothing to seed
until Phase 3 gives them a writer).

**Open, carried forward.** Phase 3 (`JobActions`, S2.7–S2.10) is next but unplanned — needs
its own discussion before a plan, per standing discipline. It inherits two named questions
from Session 17: the "touched `in_progress`" edge in the removal-legality rule (sanity-check
when the allow-list matrix is written) and the Sprint 3 blocker respec (full [[Blocker]]
rewrite + the Hold-For-Payment/`→done` collision). Standing parked items unchanged
(launch.json cleanup, R3 GUC hardening, R7 invitation index, vehicle-normalizer punctuation,
match-validation circle-back, Company×RLS v2 pass, PDPA trigger).

---

## 2026-07-16 · Session 17 — Tracker restructure: entity + event log; crew split

**Summary.** Pre-Phase-2 reasoning session, run partly across side discussions (RLS-style
"chip out" sessions kept the main thread from bloating). Reasoned through what a job
transition actually is in real life (Siti assigns Ah Boy, the handshake, the limbo case),
which surfaced that a single-table `JobMechanic` can't give a *leave* its own ack receipt —
one ack pair can only shake hands once. That reasoning generalized into a restructure of
every tracker: **entity + event log**, the same idiom `Job` + `JobStageTransition` already
used. Vault updated and committed (`f3c253f`) before any Phase 2 code is written (house
discipline). Next: draft the Phase 2 build plan against the new shape, on the builder's call.

**The restructure.** Trackers split into an entity (written once, never carries ack columns)
and its event table (append-only, each row: one author, one derived receiver, one dormant
ack pair). Crew becomes `JobMechanic` (the engagement — "Ah Boy on WVK 3721") +
`JobMechanicTransition` (`joined`/`left` events); old `assigned_by`/`removed_at` dissolve
into the joined/left rows. Blockers (Sprint 3) will mirror it: catalog + `JobBlocker` items +
`JobBlockerTransition` events, with a `noted` action for state-neutral annotation and item
identity replacing the never-quite-stated "no double raise" rule. Post-split, the ack pair is
the *only* designed mutation on any tracker row anywhere — append-only becomes easier to hold,
not harder. [[Event log]] rewritten to state the pattern generally; [[Job]]'s tracker
associations and [[Data model]]'s entity diagram follow it. [[Job]] also had a stale
"double-stamped" line fixed to triple-stamped (workshop + vehicle + customer) — drifted since
Phase 1 shipped `2c5ca91`, caught in this sweep.

**Ack theory closed out (the four framings, now canonical in [[Event log]]).** The ack pair
belongs to a *direction*, not a table — a row's NULL means either "dropped handoff" or
"never was a handoff," and only a derived receiver tells them apart (the trap the builder hit
twice mid-discussion). Receivers are always derived by query, never stored — stage
transitions target the `service_advisor` role, crew events target whichever party didn't
act, blocker raises/resolves target the catalog's role or the raiser. Ack-owed is therefore a
*role-level* test: a creator holding the receiving role owes nothing (Siti's own
`registered → assigned` is silent by design). The assignment handshake rides the `joined`
crew event, not the stage transition — freeing the stage tracker's pair for its real job:
catching an unacked `in_progress → done`, a finished car nobody at the counter has noticed,
named as the product's founding pain expressed as a single NULL. Sprint 4 gains two named
design-pass items: **the handoff predicate** (`.pending_ack`, one shared scope encoding
not-self-caused + role-resolution + aging, so every inbox/board/limbo view agrees) and
**orphaned debts** (a role that resolves to nobody renders "pending, unheld role," never
vanishes). [[ADR-005 Acknowledged handoffs in V1]] gets a dated footnote only — the decision
(ack lives on the event record, no handoff table) is unchanged, arguably strengthened; its
"accepts the job" wording is corrected to "acknowledges" (ack is a receipt, not a consent
gate — a proposal to make assignment consent-based was raised and set aside: unilateral
assignment matches how a workshop floor actually runs, and ADR-008's crew-joining consent is
a different species — it crosses an org boundary, this doesn't).

**Three rulings, reached via a dedicated clarity side-session.**
1. **Crew split: yes**, deliberately (it rewrites settled S2.6/S2.9 spec, so it got a real
   look rather than a nod). Reasoning above.
2. **Removal legality — ruled, not parked: the "responsibility, not presence" principle.**
   An engagement asserts who's *responsible*, not who's physically present right now — same
   shape as `in_progress` meaning "this is the active stage," not "work is happening this
   exact second." At `assigned`, before `in_progress` is ever touched, removing the last
   mechanic is legal (compensating `assigned → registered` rollback). Once a job has ever
   touched `in_progress`, the last mechanic is **never removable** — the only crew motion is
   the reassignment swap (old `left` + new engagement/`joined`, one door call). A sick tech
   with no replacement keeps showing on the job, truthfully, until someone swaps in — the
   board isn't lying, it's naming who's accountable. v1's primary-only crews make the swap
   the only `in_progress` crew motion; true standalone removability is conceptually a
   helpers-era (v2) capability. Non-binding soft note: if the floor wants to distinguish
   "working slowly" from "responsible tech absent," a `Blocker` catalog entry ("no
   manpower") is the natural v1-compatible escape hatch, not a schema change. Lands in
   Phase 3 with the `JobActions` allow-list; the "touched `in_progress`" edge (a job moved
   backward to `assigned` via the reassignment move) needs a sanity check when the matrix is
   written. [[M1-F1 Status flow and transitions]] gains a Settled 2026-07-16 section
   recording this plus the SA/manager/owner-only `ensure_counter!` guard and the new
   **open blockers stop `→ done`** rule (with the Hold-For-Payment collision flagged
   unresolved for the Sprint 3 deep-dive: HFP as written would block Done forever if used as
   "don't release until paid" — needs redefining as pre-work, or a per-blocker
   stage-it-guards field).
3. **`primary`/`lead` flag: deferred entirely, not renamed.** S2.6 ships with no flag — v1
   crews are single-mechanic, so "primary" has nothing to distinguish yet. Recorded in
   [[Deferred decisions]]: when helpers arrive, the flag lands as `lead` (unreserved SQL,
   genuine workshop vocabulary, no quoting tax on the project's habitual raw-`psql` audit
   sweeps) with `default: true` honestly backfilling every v1 engagement as having been the
   lead. Naming settled now so it isn't re-litigated when it actually ships.

**Open, carried forward.** Phase 2 build plan — not yet drafted, next on the builder's call.
Sprint 3 kickoff needs: full [[Blocker]] respec (already noted inline), the HFP/`→done`
collision, and the `noted`-action detail. Phase 3's allow-list write-up needs: the
"touched `in_progress`" sanity check for removal legality. Standing parked items unchanged
(launch.json cleanup, R3 GUC hardening, R7 invitation index, vehicle-normalizer punctuation,
match-validation circle-back at the intake UI, Company×RLS v2 pass, PDPA trigger).

---

## 2026-07-15 · Session 16 — Phase 1 kickoff gates ruled; ready to plan the spine

**Summary.** Pre-plan reasoning session ("what's on our plate"). All Phase 1 gate decisions
ruled; vault recorded before the plan is drafted (house discipline). Next: the Phase 1 plan
(S2.1–S2.4) on its own page.

**Gate 2 — customer stamp CONFIRMED, on strengthened reasoning (the builder's own).** The
stamp records a past-tense fact ("who brought the car in / who pays for this visit") that a
present-tense pointer (`vehicle.customer_id`) can't answer honestly once vehicles change
hands. Builder independently derived the `owner_read` query shape: *persons query jobs from
the person inward through the frozen stamp; the vehicle is supporting information derived
from the job, never the path* — canonized in [[Job visibility]]. Both privacy directions
noted: sellers keep their history, buyers never inherit it. Validation is `on: :create` only
(post-sale divergence is the correct data, not drift).

**Gate 1 — R5 ruled: REFUSE (one active job per vehicle).** The per-visit definition of Job
already implies it — a vehicle can't be in two visits at once; the partial unique index
(`jobs(vehicle_id) WHERE stage IN (0,1,2)`) enforces the definition rather than adding
policy. Edges cooperate: "waiting for parts" = a blocker (Sprint 3); the post-Done follow-up
job stays legal (done/delivered aren't in the active set — law #8's correction flow).
Consequence named: the stage integers become frozen **by schema**, not just convention.
[[Risk ledger]] R5 → decided.

**Two quieter rulings (old plan snippets predated ADR-009):** (1) **Customer/Vehicle
deletion = restrict, not cascade** — ADR-009's refusal-first one level down: a customer with
vehicles / a vehicle with jobs refuses deletion (the workshop's history); mistake-entries
with nothing attached delete freely; PDPA-style requests are the parked workshop-side
anonymization tool's job. (2) **`Job#timeline` + tracker associations move to Phase 2**
beside the tables they read — sequencing hygiene, no design change.

**Also this session (pre-gate):** the audit findings closed — Rails pinned exactly 8.0.5
(`6c73972`), `main` pushed, Risk ledger footer fixed (`31079cc`), PWA files kept
deliberately, launch.json cleanup explained and left to the builder.

**Addendum — plan review refinements (pre-build).** Three builder calls on the drafted
Phase 1 plan: (1) `plate` → **`registration_number`** (verbose JPJ term); (2) canonical
storage collapses **all** whitespace + upcases ("WVK 3721"/"wvk3721" → `WVK3721` — makes the
unique index actually catch dupes); (3) the **customer↔vehicle match validation is deferred**
— "pretty rigid… give it a recovery path… push it back, circle back": legitimate payer≠file
cases exist (borrowed car, informal sale, third-party payer) and a hard block at intake is
workflow poison. The stamp + the door's default-copy stand untouched; only the law forbidding
an explicit different choice is parked ([[Deferred decisions]], revisit at intake UI with a
soft-override shape). [[Data model]] amended with a dated note.

**Addendum 2 — intake flow designed and documented ([[Intake flow]], new concept note).**
Builder's structure (registration lookup → existing/new × phone verify) expanded into the
full SA decision tree with every discussed case (courier, third-party payer, sold vehicle,
new-phone dedup trap, second-car attach, company van, R5 in-house surfacing). Also from this
discussion: the **household/shared claims** v2 feature (rides Company claim machinery —
[[Data model]] v2 list) and the ask-inbound phone-verify form (never read the file's number
aloud). Phase 1 schema already carries the whole flow; one candidate delta raised — phone
canonicalization at storage, same lesson as registration_number.

**Addendum 3 — Phase 1 BUILT (S2.1–S2.4, commit `2c5ca91`).** The spine exists: Customer /
Vehicle / Job, three migrations each carrying ENABLE+FORCE+`tenant_isolation` in-file
(policy inventory now 6 tables: 3 edge ×2 policies + 3 spine ×1). Everything the session
ruled is in the schema: `registration_number` canonical (whitespace-collapsed, upcased),
contact keys canonical on Customer (phone digits-only — the person-key), double restrict on
Customer (vehicles + jobs directly), triple-stamped Job with **no** match validation
(deferred, [[Intake flow]]), R5's `index_jobs_one_active_per_vehicle` live (stage ints now
schema-frozen), token minted at birth. Tests 28→**42** green (14 new: canonicalization
collisions, vehicle-less payer restrict, stamp independence + freeze-through-sale, R5
refuse + law-#8 follow-up, RLS backstops ×2, cross-workshop same-registration legality).
Seeds idempotent (Lim + Speedy, 3 vehicles, 3 jobs across stages); no-GUC runner reads 0
rows on all three tables. [[Risk ledger]] R5 → fixed. Next: Phase 2 (trackers + timeline).

**Addendum 4 — post-build full sweep + the phone ruling (`0e4204d`, pushed).** Sweep
all-clean: 43 tests green, live `pg_policies` = docs exactly, R5 index predicate verified,
fail-closed proven live (no GUC → 0 rows ×3), seeds idempotent, all 36 cited hashes real,
links clean, shipped models = plan verbatim. **One genuine code find:** the Customer
normalizers stored `""` for blank phone/email — at Sprint 6, `find_or_create_by(phone: "")`
would have glued every phone-less walk-in onto one card. Fixed with `.presence` (blank →
nil). On the back of it the builder ruled: **no customer exists without a phone** — it's
the person-key AND the notification channel; presence validation + NOT NULL migration
(tests 43 green). Also synced S6.5 (registration number; [[Intake flow]] named as its spec)
and pushed `main` (was one commit behind). Parked observation: vehicle normalizer keeps
punctuation ("WVK-3721" ≠ "WVK3721") — candidate for the intake pass.

**Open (carried).** Company×RLS = v2 design pass; S0.8 deploy at Sprint 2 exit; R7 index
decision (with R4's rescue); launch.json tidy-up (builder's call); Phase 2 next.

---

## 2026-07-14 · Session 15 — Company×RLS drill-down; two misconceptions corrected; job stamps its customer

**Summary.** Continuation of Session 14's flagged discussion (how RLS serves Company reads
across tenanted workshops). Pure design session, no code. Three concepts straightened, one
data-model decision recorded ([[Data model]] Resolved).

**The Company×RLS drill-down.** Why Company is inherently cross-tenant: one real fleet =
one per-tenant Customer row *per workshop it visits*; the v2 `Company` (global, like User)
claims them via `customers.company_id`. A fleet manager's core query ("all my vans, all
workshops") is unservable under `tenant_isolation` alone — no single `app.workshop_id` fits,
and no GUC set means zero rows. The v2 shape is the own_rows species: an **additional
permissive `FOR SELECT` policy** keyed through `app.user_id` → company membership, ORed next
to `tenant_isolation`. Two genuinely open v2 problems: (1) policies whose `USING` subqueries
read other *policed* tables (policy-chasing-policy; escapes = SECURITY DEFINER helper or
denormalizing the claim onto readable rows); (2) the projection question — companies probably
get a narrow slice (stage/ETA/vehicle, the token-page shape), not `SELECT jobs` wholesale;
what a workshop consents to expose to a customer who also visits competitors is positioning
as much as engineering.

**Two builder misconceptions corrected (the useful kind — both imagined *more* separation
than exists).** (1) "Jobs have no RLS / the DB is effectively public" — wrong: every tenant
table incl. jobs gets ENABLE+FORCE+`tenant_isolation` (Sprint 2 plan Phase 1). RLS fails
**closed**: cross-org reads return *zero rows silently*, never leak — v2's work is punching
deliberate read-only holes, not plugging accidental ones. (2) "customer id 1 can exist in
both workshop A and B" — wrong: shared tables, one global PK sequence; per-tenant ID
namespaces are the schema-per-tenant world ADR-007 rejected. `workshop_id` is a stamp on
globally-unique rows, not a namespace. Per-tenant uniqueness exists only on business keys
(`unique(workshop_id, plate)`). Shared real-world identity is expressed by claim columns
(`customers.company_id`), never by colliding IDs. Also re-taught from the shipped code: the
two-key door (tenant_isolation OR own_rows, edge tables only; jobs = one key in v1) and the
landing-page motivation for own_rows.

**Decision — job stamps its customer (recorded in [[Data model]] Resolved; re-confirm at S2
Phase 1 kickoff alongside R5).** Builder caught the sold-vehicle hole: `job.vehicle.customer`
derives history through a mutable pointer — selling the car rewrites who past jobs were for.
Cure = the law-#8 move: stamp the fact when it's true. Job becomes triple-stamped
(`workshop_id + vehicle_id + customer_id`), customer written once at registration, one
creation-time validation (must equal the vehicle's customer at birth; divergence only ever
arises later, by reassignment). Also pins mid-job notification targets. `VehicleOwnership`
period-edges rejected for v1. Sprint 2 plan file amended so the decision surfaces at the Job
migration.

**Addendum — the visibility map written down ([[Job visibility]], new concept note).**
Follow-up discussion enumerated the complete set of parties querying `jobs` — six scenarios,
no seventh: crew (tenant_isolation, Sprint 2, the only one built now), landing (zero jobs,
correct), private owner (`owner_read`, v2), company (`company_read`, v2), token holder
(`token_read`, Sprint 7), stranger (fail-closed default). The walk-in is a *customer-record
state* (unclaimed), not a query path — reached only via crew board + token link. Along the
way: permissive policies OR per command; `FOR SELECT` keys are never consulted for writes;
every read key resolves from the person inward (`app.user_id` → claim → the job's frozen
`customer_id` anchor). Sprint 7's token-page×RLS wrinkle (unauthenticated page can't even
look up the job under fail-closed RLS) flagged in [[Deferred decisions]].

**Addendum 2 — audit pass + the risk ledger promoted to a file.** Re-ran the consistency
ritual: vault clean (all wikilinks resolve — one false positive, a literal backticked
`[[link]]` in Session 14's own prose; "founder" rule holds everywhere; frontmatter current),
code↔vault exact (22 tests green; live `pg_policies` = the documented 3 edge tables × 2
policies, ENABLE+FORCE all true; cited commits real). One genuine gap found: the vault never
recorded *why* `workshops`/`users` are un-policed — closed with a dated ADR-007 footnote
(they're what tenancy is made of; sign-in and claim-verification must read them noteless;
app-layer guard is necessary and sufficient; `users` thin-by-law caps the worst leak at
email-existence). The audit + builder's "who can query what?" question surfaced four new
risks → **[[Risk ledger]] created** (R-numbers were load-bearing but lived only in Session
14's narrative; now a lookup table, R1–R10): R7 duplicate fired invitations across an email
change (uniqueness is workshop+email, not workshop+user → two Accept cards, second = 500),
R8 role-swap under an open card, R9 accept-vs-destroy 404, R10 account-enumeration oracle in
the invite flash messages. All backlog-grade; no code changed.

**Addendum 3 — ADR-009 written: all three edges ruled.** The builder ruled each edge:
(1) **trapped owner** → refusal itself settled (last owner can never delete while the
workshop stands — derives from the lifetime invariant); the *escape routes*
(add-owner-then-leave vs delete-workshop-first; real vs soft delete vs disable) are
functional design, parked in [[Deferred decisions]] (`321bf26`). (2) **Placement** → the
invariant "a workshop cannot exist without an Ownership" is a *design decision*, so it lives
**inside ADR-009**, not Design laws (laws stay at 9); the [[Decisions]] index line names the
invariant so it's findable without opening the ADR. (3) **PDPA** → the whole question waits
for v1 to be up ("a decision is early only if something we're about to build could contradict
it — nothing can; it's all additive"); parked in [[Deferred decisions]] with the trigger
(*anonymization must exist before the first real workshop's data enters — the moment Knot
becomes a processor of someone else's customers*) plus the two preserved insights (data-user
vs processor split; anonymization = two features, Knot-side for users + workshop-side for
customers). **[[ADR-009 Account deletion is refusal-first]] accepted**; index updated (Open
questions now empty); [[Risk ledger]] R1 → decided, guard code pending. Next code step: the
~10-line destroy guard (builder drives, spec to follow).

**Addendum 4 — S1 tagged; S0.8 re-parked; S2.0b built and verified (commit `0e135da`).**
Tag `S1` cut and pushed (`bd7e079`, includes S1.15 + the R2 guard) — mirrors `S0`. S0.8
deploy re-parked a second time: S1.12 passed long ago with no deploy following, so the
builder moved it to **Sprint 2's exit** deliberately (spine + a demoable job engine in one
deploy) rather than let it drift further.

Built the R1 guard: `User#bare?` (no ownerships, no employments — active or ended) gates a
`before_destroy` with `prepend: true` (must run ahead of `dependent: :destroy`'s own
callbacks, or the edges vanish before the guard can see them); `RegistrationsController`
overrides Devise's destroy for a friendly refusal, model guard as backstop. **Refined
mid-implementation** (plan review): invitations dropped from the bare-ness check — an offer
isn't a commitment, only the Employment it might create is, and the ADR-008 mistyped-email
stranger must never be hostage to someone else's typo. Matched invitations now *release* to
`pending`/`user: nil` (new `Invitation#release!`) instead of blocking. ADR-009 footnoted
with the refinement (ADRs never edited). Live-verified in preview both ways: a bare
throwaway account destroys cleanly to the marketing root; the seeded owner
(`workshop.owner@seed.local`) gets the exact refusal copy from the ADR, account and session
intact. 28/28 tests green. [[Risk ledger]] R1 → fixed.

**Addendum 5 — pre-Sprint-2 audit: five findings, all small, none blocking.** Thorough
vault↔code scan before Phase 1. Verified exact: links, founder rule, all 33 cited commit
hashes, tests (28+1 green), live RLS = docs, seeds idempotent, 9/9 migrations, Bootstrap
vendored, every code artifact vault-documented. Findings + rulings: (1) Risk ledger footer
still said "ADR-009 pending" → fixed (`31079cc`). (2) `main` was ahead of GitHub → pushed
(now `6c73972`). (3) Gemfile said `~> 8.0.4` while docs say "pinned 8.0.5" → **pinned
exactly at 8.0.5** (builder: "decide directly, get it out of the way"; commit `6c73972`).
(4) `app/views/pwa/` — Rails 8 default PWA templates (manifest + service worker; would let
phones "install" Knot like an app), routeless and inert. Builder: **keep for now**; possibly
relevant later (technicians live on phones). (5) `launch.json` housekeeping (stale dead
entries from pre-move vault) — explained to builder, pending their call.

**Open (carried).** Company×RLS = v2 design pass (framed, not designed); next build =
Sprint 2 Phase 1, on its own plan (kickoff decisions: R5 one-active-job-per-vehicle + confirm
the customer stamp).

---

## 2026-07-13 · Session 14 — Post-close audit: risk pass, R2 guard, deletion design (ADR-009 pending)

**Summary.** Builder-requested full audit after S1.15 closed: vault currency, reasoning
coherence, code↔vault connectivity, then an honest risk review of everything built. Vault
passed with three mechanical defects, all fixed same session (`f89dc06`): four wikilinks broken
by line-wrapping (Obsidian can't resolve a split `[[link]]`), a *genuinely stale* Open-questions
item still describing the pre-ADR-006 bootstrap (missed by two revision sweeps), and ADR-006 §4
lacking a footnote for S1.15c's landing-guard refinement. Code↔vault: all cited commits exist,
RLS policy inventory matches the docs exactly (3 tables × 2 policies), invariants hold.

**Risk review produced six findings (R1–R6), ranked.** R2 fixed immediately (`bd7e079`):
`Invitation#accept!`/`#decline!` carry their own RLS key (`SET LOCAL`), so row security cannot
catch a caller misusing them — the state rules now live *inside* the bypass (refuse any
non-fired invitation) plus a validation that every non-pending status carries its matched user.
Principle worth keeping: **any method that bypasses RLS must enforce its own preconditions.**
Backlogged/parked: R3 (GUC writes are interpolated SQL — integer-sourced today, harden the
pattern before it grows), R4 (double-accept race — DB index holds, unrescued 500), R5 (decide
"one active job per vehicle" at S2 Phase 1 kickoff), R6 (invitation email display can drift
from a renamed account — cosmetic).

**R1 — account deletion — became a design discussion (ADR-009 pending, see [[Decisions]]).**
Devise's stock `registerable` ships a live `DELETE /users`; nobody had decided what deletion
*means*, so today it means `dependent: :destroy` accidents: an ownerless workshop (breaks
ADR-006), erased employment history (breaks append-only), and an FK crash from any fired
invitation. Two builder moves shaped the answer:
- **The invariant reword (right, and load-bearing):** "a workshop can never exist without a
  user" (birth-time, ADR-006 §2) strengthened to **"a workshop cannot exist without an
  Ownership" — a lifetime invariant**. The last-owner rule then *derives* instead of being
  policy: deletion refused not because we chose to refuse, but because the invariant forbids
  the result. Enforcement is app-level at the only two gates that ever remove Ownership rows
  (account deletion now; owner-removal/transfer when built).
- **The organisation pattern (right, with a named trap):** Workshop and Company (v2) share one
  shape — organisation ─ membership edges ─ people, governance on top — so Company gets its own
  governance edge in v2 ([[Data model]] updated). The trap, caught in discussion: same shape ≠
  same entity. Workshop IS the tenant/RLS boundary; Company is a *customer* whose vehicles
  cross those boundaries. A merged "organisation" table would tangle tenancy at the root.
- **The incompleteness, patched in reasoning:** the invariant covers governance only — a
  technician holds no Ownership, so protecting *their* history needs the second principle
  (append-only history). Deletion rule = refuse when **either** principle would be violated
  ⇒ bare users delete freely, everyone else refused; anonymization is the PDPA-era upgrade
  (the User row survives for every FK; the person inside it leaves).

**Open (for ADR-009):** the trapped-owner edge — v1 has no workshop-delete and no ownership
transfer, so a solo owner cannot delete their account at all; acceptable for v1 but must be
stated. Law vs ADR placement for the invariant. A concrete PDPA/anonymize trigger. **Next
discussion (builder-flagged): how RLS serves Company reads across tenanted workshops** — noted
in [[Data model]]'s v2 sketch; ADR-007 already hints at the answer (shared tables were chosen
partly *because* physical separation makes cross-tenant reads worse).

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
  test, now also footnoted on [[Architecture laws]] #9: a separate action class is justified only
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
   door, cost of waiting doesn't compound. New dated entry in [[Deferred decisions]] with the
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

**Also same day — testing philosophy locked + [[Architecture laws]] #9.** Builder set the test
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
identity. The discussion that followed produced
**[[ADR-006 Ownership separate from Employment]]** — the biggest model revision since the
foundation session.

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
  the [[Design system]] tokens + branding-only classes. Chosen for expansion speed; supersedes
  the "no CSS framework" stance in [[Tech stack]] and [[Design system]] (both updated today).
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
  index — points to sources of truth (never duplicates [[Design system]] or [[Positioning]]),
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
note, [[Design system]]. Closed with a vault audit (connectivity + consistency).

**Sprint 0 progress** (see [[Sprint plan]] ticks):
- **S0.1 ✓** stripped Rails 8 defaults — Docker/Kamal/Thruster gone, Solid Queue/Cache/Cable gone,
  cable → async, prod DB collapsed to one. Root commit `b9bfa02` (81 files, 102 gems from 118).
- **S0.2 ✓ / S0.3 ✓** Ruby 3.2.8 / Rails 8.0.5 confirmed; local Postgres = Homebrew
  `postgresql@17` (17.5), dormant `@14` noted; `pilot_development`/`pilot_test` exist.
- **S0.4 ✓** Devise `User` — thin (email + password only, stock 5 modules, zero extra columns,
  per [[ADR-004 Multi-tenant foundation]]). Commit `f08e29e`.
- **S0.5 in flight** — home `index` behind `authenticate_user!`; code specified (tokens CSS +
  layout shell + view), builder implementing.

**Decisions (design, recorded in [[Design system]]):**
- **Theme locked:** industrial & confident · light, high-contrast · one theme for both clients.
  Brand steel blue `#22456B` (scarce), action blue `#2D5E94`, blue-tinted neutrals, all-light
  chrome ("Option A"). **No secondary hue** — deliberately.
- **Status colors are sacred:** gray/blue/amber/red/green = job state only, identical everywhere.
- **Typography: system font stack** (0KB, can't fail); Inter noted as the escape hatch.
- **10 UI laws** + posture note (tech = phone, SA/manager = PC, owner = phone read-only) —
  the UI twins of [[Architecture laws]].
- **Dark mode deferred** → derive surfaces from brand steel blue ([[Deferred decisions]]).

**Vault audit:** 0 broken links · stale-term scan clean (persona/ADR-footnote hits legitimate) ·
fixed: [[Design system]] orphan (now in [[Agent guide]] reading list + [[Tech stack]]),
7 stale `updated:` frontmatter dates, this missing session entry.

**Open:** S0.5 proof drive + commit, then S0.6–S0.9 (vault → `docs/`, `CLAUDE.md` — must now
include [[Design system]] — deploy, tag `S0`).

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
  table (would double-record; the inbox is a query, per [[Architecture laws]] #3).
- Added [[Architecture laws]] #8 — **a Done job is immutable**; corrections open a new job. Vehicle
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
blockers, immutable-on-Done), [[Deferred decisions]] (handshake removed → ADR-005; jobsheet snapshot
added), [[Roadmap]] (slice 5 designed), [[Overview]] (roles, features), [[User stories]] (persona→role
mapping), [[ADR-002 V1 scope]] (dated footnote only — accepted ADRs aren't rewritten).

**Fixed (stale refs from Session 2):** `Event log` "StageChanges", `Roadmap` "Contact", `Overview`
old role names + "handshakes backlogged", `Job` "Warehouse" role.

**Open:** primary/helper `JobMechanic` permissions · owner notification channel (WhatsApp/email vs
manual) · exact `Blocker` admin UI · paper_trail (later) — all in [[Open questions]] / [[Deferred decisions]].
**Next:** resume slice 0/1 build on this model.

---

## 2026-07-04 · Session 2 — Reconcile 2026-07-03 foundation handoff

**Summary.** Compared a foundation-design handoff (claude.ai, 2026-07-03) against this vault.
Adopted its tenancy/user foundation — it directly answers the "User model + tenancy" pause from
Session 1. Kept the vault's jobsheet. Backlogged the handshake pattern and lock_version. Settled
the v1 role set. Rewrote the affected concepts/feature so the vault has one consistent voice.

**Decisions:**
- Adopted [[ADR-004 Multi-tenant foundation]] — Workshop tenant, thin User, Employment edges, session re-verification every request.
- Added [[Architecture laws]] (7 invariants) and [[Rejected alternatives]] (dead ends, do not re-propose).
- **Jobsheet stays in v1** ([[ADR-003 Digitized jobsheet in V1]]), now tenant-scoped (`JobSheet belongs_to :workshop`).
- **v1 role set:** technician / service_advisor / parts_advisor / workshop_manager / owner (on Employment). Foreman dropped, front_desk folded into service_advisor. Parts *role* is v1; parts *module* stays v2.
- v2 Company roles: owner / fleet_manager / driver ([[Data model]]).
- **Backlogged (not dropped):** the two-role handshake/confirmation pattern, and `lock_version` optimistic locking → [[Deferred decisions]].
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
