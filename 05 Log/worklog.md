---
type: log
updated: 2026-07-28 (Session 28 built; Sessions 12–25 archived → live log is 26–28)
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

---

## 2026-07-28 · Session 28 — Sprint 4 reshaped and built: acknowledgement as stored visibility

**Summary.** Went to plan Sprint 4 on Session 27's *holder* model, and a chip-out study of the
proposed `handoff` concern reshaped the whole answer. **[[ADR-011 Acknowledgement as stored
visibility]] revised in place** (it was never committed): the holder dies, the receiver is **stored
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

## Sessions 1–25 — archived
Vault/Rails setup through Sprint 2.5, the job engine and blockers groundwork, and the tenant-spine
collapse (ADR-010) — Sessions 1–25. Moved to [[Worklog (Sessions 1-25)]] to keep this file focused
on recent sessions.

---
