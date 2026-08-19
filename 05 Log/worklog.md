---
type: log
updated: 2026-08-19 (Session 35 — jobsheet backend built (S5.1a/b): catalog + migration + models; two build-time footnotes narrowing ADR-015; prior: Session 34 — jobsheet storage decided, ADR-015; prior: Session 33 — jobsheet reversed to fixed/product-defined, ADR-014; EAV branch discarded; storage chipped out; prior: 2026-08-17 Session 32 — Sprint 5 ↔ 6 swapped)
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

---

## 2026-08-19 · Session 35 — jobsheet backend built (S5.1a/b): catalog, migration, models

**Summary.** Received Session 34's carry-back cold, ran the ADR-015 coherence check against the
vault (clean — the red-flag sweep found only historical/struck references), then built the jobsheet
backend as three commits on `main`: the code catalog (`c5b8977`), the migration (`6041a0b`), the
models (`a1889a3`). Suite green throughout (139 runs). No UI — that's S5.5. **S5.1c** (seed) and
**S5.1d** (model tests, incl. the R11 orphan-scan) are still open.

**Two build-time deviations from ADR-015, both recorded as dated footnotes (ADRs are never edited).**

1. **Re-snapshot dropped.** ADR-015 allows a jobsheet's `item_keys` to be re-snapshotted while it has
   zero answers (so a mis-typed `inspection_type` could be corrected before filling). At build this
   became the design's only genuinely tricky rule — a conditional callback plus a state-dependent
   validation — for an edge case. Simplified to `attr_readonly :inspection_type, :item_keys`:
   write-once at creation, and Rails raises `ReadonlyAttributeError` on a later write
   (`load_defaults 8.0`). A mis-typed sheet is deleted and recreated. The frozen-question-set
   decision — the load-bearing part — is unchanged and actually strengthened. Footnote in
   [[Data model]] §The jobsheet and [[Sprint plan]] S5.1b.

2. **Complaint moved to the jobsheets header.** ADR-015 §Decision placed the customer's complaint on
   **Intake** ("the complaint is not a jobsheet field"). At build the builder moved it to a free-form
   `complaint` column on the `jobsheets` header instead. Framed as a **narrowing, not a reversal**:
   the ADR's real argument — keep free text out of a fixed-answer form — is fully honoured, because
   the complaint is a plain header column, never an `InspectionItem`; `ANSWER_TYPES` stays
   `rating|boolean|enum|numeric`. Only *which table* holds the text changed. It sits with the sheet
   because the walk-around and the complaint are one act at intake, and both freeze together when the
   intake goes terminal. So **Intake has no complaint column**. Footnotes in [[Data model]],
   [[Intake]], and [[Sprint plan]] S5.5.

**The shape as built.** `Inspection` (registry, `TYPES`-guarded lookup) + `InspectionItem` (value
object) + `CarRoutineInspection`/`LorryRoutineInspection` (39 items each, lorry mirrors car for now,
duplicated not shared so a car edit can't rewrite lorry sheets). `Jobsheet` (1:1 with Intake,
auto-created inside `CreateIntake`, freezes `item_keys` from the template on create; `complete?` /
`editable?` derived, never stored). `JobsheetAnswer` (validates `item_key` against the sheet's own
frozen list, not the live catalog; value column must match the item's `answer_type`; upsert write
path settles `inspected_by` as last-writer). Both tables RLS-policed; `inspected_by` carries the
composite `(id, workshop_id)` FK to `workshop_staff` (ADR-010). Three content calls flagged in
`car_routine_inspection.rb` to revisit before real sheets exist (battery voltage / pad thickness /
pedal free play typing) — keys are retire-only, so getting them right now is cheaper than later.

**R11 note.** The `item_key` code↔data handshake still has no FK behind it (accepted, [[Risk ledger]]).
The first automated net — a test scanning every answer's key back to the catalog — lands with
**S5.1d**, not yet written.

---

## 2026-08-19 · Session 34 — jobsheet storage decided: catalog + narrow answer rows, two value columns, a frozen question set

**Summary.** Picked up [[Inspection jobsheet — design brief]] cold, as designed — the entry point
Session 33 chipped out. Worked the storage-structure fork (brief §4) to a full decision with the
builder, recorded as [[ADR-015 Jobsheet answers are rows against a frozen question set]]. Three
things surfaced along the way that the brief hadn't anticipated: a **concrete SA/technician fill
flow**, an **externally-sourced design (ChatGPT) worth comparing against**, and a **template-
evolution case** (adding a field after old sheets exist) that exposed a gap in ADR-014's
"no drift" claim. All three fed the final shape.

**The six-way survey, narrowed to a real fork.** Every candidate splits into two independent
axes — *where the form lives* (code vs. owner-editable DB rows) and *how answers are stored*
(wide columns / narrow rows / jsonb). Axis one was already settled by ADR-014 (code). On axis
two: wide typed columns die to §5's multiple-inspection-types expansion (a second type forces a
second wide table or NULL-riddled columns on the first); jsonb dies to the numeric-trending
requirement (§3 — string comparison misorders numbers, and casting at query time fails
unpredictably once Postgres reorders a `WHERE` clause). That leaves narrow answer rows as the only
survivor, matching the brief's own lead candidate.

**An externally-sourced design (ChatGPT) turned out to be the discarded EAV branch, table for
table.** The builder brought a `InspectionTemplate → Category → Item → Inspection → Result`
design to compare against. Mapped onto this project's vocabulary, its top three tables are
exactly `JobSheet`/`JobSheetField`/`JobSheetFieldValue` — the branch ADR-014 already reversed —
and its own stated design principle ("workshops can add/reorder/disable items") is the owner-
configurability ADR-014 rejected for print-control and drift reasons. Re-examined on its merits,
not dismissed on sight; re-rejected for the same reasons. Two pieces were worth keeping: `enum`
as a genuinely distinct answer type (folded into this ADR's `choice` column, which generalizes
rating/boolean/enum together), and photos attached to a *finding* rather than the whole sheet
(adopted as the deferred photo design — [[Deferred design]]).

**The SA/technician fill flow killed per-section state, and simplified the answer shape.** An
intermediate design gave each *section* its own stateful row (who signed it off, when), reasoning
the SA owns exterior and the technician owns the rest. The builder's actual flow — the SA walks
the customer through exterior *only if* they happen to go out with them; otherwise a technician
covers the whole sheet alone; anyone can be interrupted and someone else picks up the sheet —
contradicts that. No role owns a section, so nothing section-level earns stored lifecycle state.
What *does* vary per row is who answered a specific item — which resolved as `inspected_by` on
each `jobsheet_answer`, a deliberate departure from the house `created_by` convention because the
actor here is a first-class domain fact, not audit metadata on a log row. This also settled the
door question: nothing about filling a sheet has an *illegal move* to veto (unlike Job/Intake's
transition machinery), so the jobsheet stays outside `JobActions`/`IntakeActions`/`Permissions`
([[ADR-013 The door decomposed]]) entirely — data collection, not a state machine.

**Four value columns collapsed to two.** The brief's answer types (rating/numeric/boolean) plus
the ChatGPT design's `enum` all reduce to the same shape except numeric: "pick one of a fixed set
of options" (rating, boolean, enum) vs. "a number you order/aggregate on" (tyre tread, pressure,
pad thickness). `choice` (string) covers the first three; `measurement` (decimal) is split out
alone, because it's the only type anything does arithmetic or ordering on — drawn exactly where
string storage stops working, confirmed by walking a concrete failure: storing "10mm" and "4.5mm"
as strings sorts the new tyre as more worn than the old one, and casting a mixed column at query
time (`value::decimal`) errors unpredictably once the query planner reorders which row it casts
first — a bug invisible in dev, live in production.

**The CO2-tailpipe case exposed a real gap in ADR-014, and `item_keys` is the fix.** ADR-014
claimed a code-defined form has "no drift to snapshot against." True for removal/rename
(retirement handles that — keys are permanent, only labels may be corrected in place). Not true
for *addition*: a template gaining a field after old sheets exist means printing from the live
template would show a blank line for a question that sheet was never asked — reading as staff
negligence rather than history. Fix: **`Jobsheet.item_keys` is frozen at creation** (re-snapshotted
only while the sheet has zero answers), and printing/completion read from that frozen list, never
the live template. This closes drift from both directions and, as a side effect, makes a template-
version column unnecessary — versioning would need every historical template kept in code forever
*and* a human remembering to bump it; the frozen list records itself.

**A proposed label-snapshot design was examined and re-rejected on new grounds.** The builder
independently proposed freezing `label` (or more) onto each answer row — the classic EAV-adjacent
fix for drift. Re-examined against the concrete printing requirement: a usable line needs label
*and* section *and* position *and* unit, so one snapshot column becomes four or five, duplicated
across ~39 rows × every intake × every workshop, forever — the exact per-answer-snapshot cost
ADR-014 had already priced as not worth paying. `item_keys` + an append-only catalog gets the same
fidelity for one array column, because the catalog lives in git, which already preserves label
history for free. The [[Deferred design]] entry marking this "moot" (2026-08-19, ADR-014) is
footnoted rather than rewritten — moot for the old reason, re-rejected here for a new one.

**The complaint separated cleanly from the inspection.** Working through what a jobsheet answer
row actually needs surfaced that "customer complaint" and "staff inspection finding" had been
informally conflated (visible in [[Intake]]'s stale `has_many :job_sheet_field_values` line,
glossed as "customer complaints + vehicle condition"). Resolved: the complaint is free-form,
customer-reported, per-visit text that belongs on **Intake**; the jobsheet is the standardized,
staff-recorded inspection only, in fixed vocabulary. Folding the complaint into the jobsheet would
have forced free text into a form built for fixed answer types.

**Recorded as [[ADR-015 Jobsheet answers are rows against a frozen question set]]**, extending
(not superseding) ADR-014. Vault sweep: [[Decisions]], [[Inspection jobsheet — design brief]]
(annotated per house pattern, not rewritten), [[Data model]], [[Sprint plan]] (S5.1 (rev) split
into S5.1a–d, the four-layer build; S5.5 split to separate the complaint from the inspection
fill), [[Intake]] (stale association + gloss corrected), [[Roadmap]], [[Features overview]],
[[Overview]], [[Deferred design]] (four new entries: photos, an answer edit activity log, multiple
inspection types, the damage diagram), [[Risk ledger]] (new R11 — the `item_key` code↔data
handshake has no FK behind it, held by discipline not the database), and [[Product gaps]] (#7's
EAV-era description corrected, photos now have a designed home). ADR-014, ADR-012, ADR-003 not
edited — every reconciliation is a dated footnote or annotation, per the ADR-010/011 pattern.
**Not built this session** — the build (migration → model → seed → tests, S5.1a–d) is a separate,
future step, on the builder's call.

---

## 2026-08-19 · Session 33 — jobsheet reversed: fixed, product-defined, not owner-configurable

**Summary.** Picked up exactly where Session 32 left off — "spec the `JobSheet` + `JobSheetField`
migration + models (S5.1–S5.2)" — and built them on branch `s5-jobsheet-models` (migrations,
models, a seeded default template, model tests; 4 commits). Then, discussing the design with the
builder, the core assumption underneath it came apart: an **owner-configurable** jobsheet
([[ADR-003 Digitized jobsheet in V1]]) means the product can never guarantee what a printed
sheet looks like, and it was the single upstream cause of every hard question downstream —
template versioning, per-answer label snapshots, drift once an owner edits/renames a field a old
answer already used. The resolution: **the jobsheet is a fixed, product-defined inspection form**
— fields set by the product, versioned in code, never owner-CRUD at runtime — recorded as
[[ADR-014 Jobsheet is a fixed product-defined inspection]], superseding ADR-003's core. The
per-visit anchor from [[ADR-012 Intake-Job two-level aggregate]] (one inspection per Intake) is
unchanged; only *who defines the fields* moved.

**The field list surfaced, and reopened a real question.** The builder drafted a concrete
39-item list — 5 sections (EXTERIOR, TYRES, ENGINE BAY, BRAKES, INTERIOR), tentative, not
finalized — and its answer types are **not uniform**: ratings (ok/attention/damage + note),
numeric measurements (tread depth mm, tyre pressure psi, pad thickness mm — wanted queryable/
trendable across a vehicle's visit history), and booleans. That mix means "one wide boolean
table" is the wrong shape, and reopens the storage-structure question the EAV model had
answered by construction (fields are rows). This is **deliberately not decided now** — a wide
typed table, a code-defined catalog + narrow answer rows (lead candidate), and jsonb are all
live options with real trade-offs, and deciding needs its own session.

**Two forward ideas, noted not designed.** Still "thinking, not committed": (a) **multiple
inspection types** — Pre-Delivery Inspection, Used-Car Inspection, later Lorry / Passenger-car
variants, which likely reshapes the model into a small catalog of product-defined inspection
types rather than one universal sheet; (b) an **exterior damage diagram** — a base vehicle image
the technician "dots" for cracks/nicks, with optional photo upload per mark, which implies a
coordinate/annotation + attachment model just for the EXTERIOR section, separate from the other
four. Both are recorded as design inputs for the chip, not designs.

**The EAV branch discarded.** `s5-jobsheet-models` (migrations for `job_sheets` +
`job_sheet_fields`, the models, the seed, the tests) modelled the abandoned design and would
mislead if kept as "current." Rolled its two migrations back while the files still existed (dev
DB drops cleanly), discarded `db/structure.sql`'s rewrite, checked out `main`, deleted the
branch. `main` is unaffected — the branch was never merged — so this is a clean discard, not a
revert.

**Chipped out.** Rather than decide the storage structure in this session, the work is handed to
a future session via a self-contained design+build brief:
[[Inspection jobsheet — design brief]]. It carries the decision already made (ADR-014), the
verbatim 39-item list, the mixed-type problem, the storage-structure trade table, both forward
ideas, and the required first steps (decide structure → migration → model → seed → tests,
four-layer like the discarded S5.1/S5.2 were, RLS + `workshop_id` + `WorkshopScoped` per house
pattern, keyed on `intake_id`).

**Housekeeping.** [[Decisions]], [[Data model]], [[Sprint plan]] reconciled to stop describing
the EAV design as current/upcoming; old S5.1–S5.4 struck and replaced with a single
S5.1 (rev) task pointing at the chip. A stale-reference sweep caught [[Features overview]] (F6),
[[Roadmap]] (slice 6), [[Deferred design]] (the now-moot per-answer-snapshot note), and
[[Open questions]] — reconciled or footnoted, not left describing the abandoned model as live.

---

## 2026-08-17 · Session 32 — Sprint 5 ↔ 6 renumbered: intake vertical (jobsheet) runs first

**Summary.** A planning-and-reconciliation session spread across several parallel sessions (owned,
and folded back together here). The **board is deferred**; the **intake vertical, led by the
jobsheet, goes next** — and we **renumbered so the numbers match the order**: the intake vertical is
now **Sprint 5**, the board **Sprint 6**. Recorded in [[Sprint plan]], swept the ~15 docs that named
the old numbers, and fixed three lines Session 31's fresh notes carried that the change made stale.
No app code this session.

**Why swap.** The board is query-over-existing-tables plus heavy UI — best built once real intakes
flow and the designer has [[Screen map]] / [[Screen flow]] to work from. The intake vertical is the
**create-path the whole app is missing** (`bin/route-orphans`: exactly two orphan endpoints, both
intake creates) *and* it's genuine model/controller work — so it fits building backend-first, which
the board never did. Build order is now **4.5 (closing) → 5 → 6 → 7 → 8**.

**Jobsheet first, and why the template models are the clean start.** [[ADR-003 Digitized jobsheet in V1]]
locks the shape (one form per workshop; flat fields `label` + `checkbox`/`text`; values keyed on
`intake_id`; add-only, no destructive delete). The **template models** (`JobSheet` +
`JobSheetField`) are view-blind, seedable, unit-testable, and depend on none of the open fill-layer
questions — so they're the first slice. Plan: build them + **seed a default template**, **defer the
owner field-admin UI (S5.4)**; `JobSheetFieldValue` (S5.3) + the front-door form (S5.5) that fills it
come after, and S5.5 closes the create-path hole.

**Two fill-layer decisions parked, not forgotten.** (1) **Freeze condition** — [[Data model]]'s own
`[!question]`: "answers freeze once the job is Done" no longer parses (a visit has several repairs,
no single Done moment); freeze on the *intake* going terminal, or on `ready?` — undecided.
(2) **Jobsheet in the intake form vs a later step** — where the values get written. Both bite only
at the value layer, so deferred until we build `JobSheetFieldValue`.

**Renumber — a hard 5 ↔ 6 swap.** First recorded as an annotation (build order ≠ numbering, ids left
alone), then the builder called it: mentally the "do 6 before 5" skip didn't sit right, so we swapped
the numbers outright. Cheap now — **nothing's built, so no commit references any `S5.x`/`S6.x` id**;
the cost was a ~15-doc sweep, not code. The ADRs are the one thing we can't edit, so **ADR-011/012
keep their original S5/S6 numbers with a forward-pointer footnote** (the [[Data model]] / ADR-010
supersession pattern), not a rewrite.

**Coherence fixes (Session 31's notes were right as written; the swap made three lines stale).**
(1) [[Features overview]] F6 said "models built" — split it: customer/vehicle built, **jobsheet not
built** (its models are exactly our next task). (2) The board's "reshaping to group by car" read as
in-flight; with the board parked it now says **deferred (S6)** in F1 and [[Screen map]] §3. (3) The
group-by-car board reshape is **ADR-012** (the aggregate), not ADR-013 (the door) — citation fixed
in both notes.

**Housekeeping.** Committed Session 31's vault work as its own commit first (`b4505ec`) so the two
sessions' authorship stays clean; the stray `Untitled.canvas` (a FigJam export, not a note) was
deleted. **Open, flagged not chased:** [[Screen map]] still marks the two create-buttons *dead* /
500-ing, while Session 31's own log says they were cleaned up pre-merge — a vault-vs-code check to
run once we're on the merged `main` app tree, separate from this swap. **Next:** spec the `JobSheet`
+ `JobSheetField` migration + models (**S5.1–S5.2**).

---

## 2026-08-16 · Session 31 — the UI surface mapped, the feature model named, routes made consistent

**Summary.** A mapping session, not a building one. Wrote [[Screen map]] (every screen, its
components/actions, role-gates verified against `Permissions` and the controllers' `before_action`
tables, each carrying a build status) — which immediately paid for itself by exposing **two dead
buttons that were 500ing their pages**: the S4.5 aggregate split deleted `new_job_path` and the
nested `customer_jobs` routes without updating their callers, so `/workshop` raised for counter
staff and `/customers/:id` raised for everyone who could reach it. Cleaned those up and folded them
into the S4.5 commit before merging. Then named the **feature model** ([[Features overview]], F1–F8)
and the **flow model** ([[Screen flow]], 11 flows across Setup → Records → Daily loop), explored the
whole thing in FigJam, concluded the vault should stay the source of truth, and finished with a
routes consistency pass: job create now nests under its intake (`ed2595c`), plus a reusable
route-orphan check. Merged S4.5 to `main` and pushed (`6e0434c`); deleted 11 stale branches and 3
leftover worktrees.

**The mapping caught what the tests couldn't.** The two 500s shipped silently because Session 30
deleted the whole `test/system` suite in the same commit that removed the routes — and the model/
service suite never renders a view, so it stayed green through a broken UI. Worth stating plainly:
**this suite's green bar says nothing about whether a page renders.** The system tests that would
have caught it were themselves dropped as "stale UI"; only `cold_start_intake_test.rb` was genuinely
tied to a reworking surface, while `job_lifecycle`/`blocker_lifecycle`/`crew_management`/
`blocker_catalog_admin` covered screens that are staying. A thin request-level smoke layer (does
each surviving page render for a counter user?) would fence off exactly this regression class — not
built, flagged here deliberately.

**Per-record time analytics are not reporting.** The builder corrected the first draft of the
feature model: time-in-stage and blocked-by-department are *this car's story*, so they belong on
**Intake show / Job show / the intake timeline** (F2), not in F8. F8 shrank to **aggregate numbers
across visits** — sums and averages — and *which* reports earn their place is still undefined.
Both read the same source: the [[Event log]] renders per-record as a timeline, aggregated as
reports. That also promoted the timeline from "infrastructure" to a named surface.

**The V1 fence, restated.** Asked what [[ADR-002 V1 scope]] actually means, and correcting a wrong
claim made earlier in the session: there is **no line running through F1–F8**. ADR-002 is a fence
around *all of Module 1* — **F1–F7 are all V1, the owner status page included** (ADR-002 lists it
explicitly). Only **F8 aggregate reporting** is the fuzzy edge. What sits *outside* the fence isn't
a slice of these features but whole other domains: parts/warehouse → V2, technician skills → V3,
money (pricing/quotes/invoicing) → deferred, with "awaiting customer approval" handled as a blocker
type.

**Vocabulary: "visit" is retired, it's an intake.** The models, controllers, and routes all say
`Intake`; "visit" was a translation layer sitting on top of the code and the source of the same
job/visit/intake tangle that ADR-012 already had to untie. Docs now say **intake**. ("Deliver"
stays — you deliver the *car*.)

**FigJam explored, then deliberately demoted.** Mapped features → screens → flows on a board, plus
a per-screen card grid in a `Screen name / (C) components / (A) actions` format with role-gates on
every action. Useful for *exploring*; wrong as a home. It's disconnected from the code (goes stale
the moment routes change, which is the exact drift [[Screen map]] exists to prevent), and the Figma
MCP bridge hit its Starter-plan cap mid-session, which settled the argument. **The vault is
canonical; any diagram is a disposable projection** regenerated on demand. Figma stays for pixel
mockups later, where it's genuinely the right tool.

**Routes: one real inconsistency fixed, one rename declined.** Job create was the odd child —
flat `POST /jobs` with `intake_id` smuggled through the request body while every sibling
(intake blockers, job blockers, job technician) takes its parent from the path. Now
`POST /intakes/:intake_id/jobs`; member verbs stay flat at `/jobs/:id/...`, since once a job exists
its own id addresses it, deeper nesting would push blocker routes to four levels, and the URL could
otherwise carry an `intake_id` that disagrees with the job's real parent. **The security boundary
here is the workshop scope, not the intake** — so nesting adds path noise, not safety.
*Declined:* renaming the blocker-type catalog from `/blockers` to `/blocker_types`. Tried, then
reverted on the builder's challenge — the nested controllers already disambiguate catalog from
applied, this codebase already accepts label≠route (the crew page is "Crew" at `/staff`), and
renaming the route without the `Blocker` model trades one asymmetry for another. Do the full model
rename or nothing. *(Also declined: Rails' `shallow: true`, which would express the same URLs more
concisely — rejected because it isn't obvious to read, and routes should be legible at a glance.)*

**A route-first check, and what it proved.** Every inventory we keep is screen-first, so none can
answer "does anything actually *trigger* this endpoint?" Added **`bin/route-orphans`**, and it
found exactly two orphans in the whole app: **`POST /intakes`** and **`POST /intakes/:intake_id/jobs`**.
Both creates are implemented, tested, green — and **nothing in the view layer can reach either**.
Every other endpoint has a caller. That's the create-path hole proven from the route side rather
than asserted, and it's now a standing per-sprint check (exits non-zero when orphans exist).

**Housekeeping.** Merged `s4.5-intake-job-aggregate` → `main` fast-forward and pushed
(`8fad8c9..6e0434c`). Deleted 11 local branches (8 fully merged; 3 pre-squash duplicates whose only
unique files were the orphan templates and system tests we'd deliberately removed) and 3 stale
worktrees under `.claude/worktrees/`. Routes audited clean in both directions — no controller action
without a route, no route without an action, no broken path helper in any view. 129 runs, 0 failures.

**Next session — the front door (S6).** The whole daily loop is built from *assign technician*
onward; what's missing is the way in. Three screens, in dependency order:
1. **New intake** (`GET /intakes/new`) — the plate-first entry: registration number → find-or-create
   vehicle → find-or-create customer → open the intake. The one create gateway, reached from both
   the Intake board and a Customer. This unblocks everything else.
2. **Add repair** — a form on Intake show posting to `intake_jobs_path`; today an open intake can
   never gain a second repair through the UI.
3. **Add vehicle** — currently a vehicle only comes into being as a side effect of typing an unknown
   plate during intake; it has no screen of its own.
Design questions still open going in: what the plate-miss branch does (customer-first vs
vehicle-first), whether the jobsheet ([[ADR-003 Digitized jobsheet in V1]]) is part of the intake
form or a later step, and whether Add vehicle is its own page or an inline affordance on Customer
show. Orientation for that session: [[Screen flow]] (flows 6–8), [[Screen map]] (the not-built
table), [[Features overview]] (F2, F6).

---

## 2026-08-14 · Session 30 — Sprint 4.5 built, split further into ADR-013, vault caught up

**Summary.** Picked up Session 29's built-but-uncommitted Intake/Job split and took it the rest of
the way: a codebase sweep, then layered commits (schema → models → services/controllers →
remainder), a comment-noise pass, and a naming/comment-discipline standard added to the [[Agent
guide]]. Then, reviewing the service layer, the builder pushed back hard on `JobActions` doing too
much — three separate concerns (state moves, authorization, creation) crammed into one class — which
produced a second decision this session, **[[ADR-013 The door decomposed]]**: creation left for
`CreateIntake`/`CreateJob`, authorization left for a new **`Permissions`** class checked at the
controller boundary, and the door split into `JobActions` (repair) + `IntakeActions` (visit).
Migrated the 29 tests still calling the removed `register_job!`, extracted the role-gating
assertions that no longer belonged in the door's own tests into a new `permissions_test.rb`, deleted
the stale system-test suite (UI is parked pending a rebuild), and — the bulk of this session — swept
the whole vault for what ADR-012 and ADR-013 broke and wrote it back to the truth. 129 runs, 0
failures, 0 errors, committed as two commits (`33f426d`, `636d47b`) on `s4.5-intake-ui`.

**Why the door split further, past what ADR-012 planned.** ADR-012 §Vocabulary had ruled the single
service object stays, intake verbs just get added to it. Building it exposed the reason that was
wrong: an Intake has no "start work" or "assign technician" verb at all — its state is *derived*
from its jobs — so a single class trying to hold both levels either lied by omission or by naming
(`JobActions.deliver!(intake)` names the whole after a part; `IntakeActions.start_work!(job)` would
be the reverse mistake). The builder's own framing carried the rest: *"job actions are doing way too
much… should this be literal actions taken to it, rather than a check on whether the action should
fire?"* — which is exactly the authorize-at-the-boundary pattern. Landed on: doors execute moves and
trust the caller; `Permissions` is where "may you?" gets answered, once, at the controller.

**One thing target-validation clarified along the way.** Not every guard on a verb is an
authorization check. `assign_technician!`'s check that the *technician being assigned* actually
holds an active technician role is validating the **argument**, not the caller — the same kind of
check as a stage guard — so it stayed on the door instead of moving to `Permissions`. The dividing
line that fell out: **authorize the actor, validate the target** — different questions, different
homes.

**A CanCanCan detour, and why it didn't take.** Introducing `Permissions` prompted the obvious
question — reach for a gem? Argued through and declined for now: the surface is small and
idiosyncratic (a handful of verbs, not a CRUD matrix), crew-awareness is a join through
`job_technicians` a static `Ability` table expresses badly, catalog rules (`blocker.raised_by_role`)
are workshop *data* not code, and — the decisive point — every guard here must **return the
`WorkshopStaff` to stamp `created_by`**, not just a boolean, so the gem would sit beside a
hand-rolled resolver rather than replace it. Revisit only if the surface grows into a large,
standard CRUD-by-role matrix, and reach for Pundit over CanCanCan then — `Permissions` is already
shaped like a policy object.

**The safe-params + naming/comment threads, folded in along the way.** Naked `params[:x]` reads
became named, presence-checked accessors (`require` for structural ids the form always sends,
`permit` for user-picked ids handled with a friendly flash) — no new params-object abstraction, just
using Rails' own strong-params deliberately. And reading the diff together surfaced the builder's own
naming signature as a real, describable pattern (verbs as actions not CRUD; sigils mean things;
parallel things get parallel names; spend characters to kill guesswork; the tell that a name is wrong
is "what does this refer to?") — written into the [[Agent guide]] alongside the comment-discipline
rule from the prior pass.

**The stashed remainder revealed a broken commit before it shipped.** Mid-session the builder asked
to squash the four-commit chain into one; before squashing, a stash-pop of parked work turned up
that the branch tip's `db/seeds.rb` still called the just-removed `register_job!` and
`JobActions.deliver!` — squashing at that moment would have baked a non-running commit into history.
Fixed first, then squashed (`git write-tree` before/after confirmed byte-identical trees — the squash
changed only history shape, not code), then a second squash-and-reopen when the builder asked to
split the migration/model/service layers back out for review.

**Two things chipped rather than resolved.** *Role-addressed blocker pins* — the builder noticed
`raise`/`note` blockers have no receiver because the target is a role, not a person, and worked out
a real rule mid-session (*address a request by capability, a reply by identity*) plus a possible
free lunch (`cleared_by_role` may already carry it, as a board query rather than a schema change).
Folded into the existing **B2** entry in [[Deferred design]] rather than left loose or duplicated —
still parked pending a real workshop, per B2's original trigger. *The Intake timeline* — the
original ask that opened this whole reconciliation thread (trace one intake's history, or the deep
trace including all its jobs) — got designed (`Intake#timeline` / `Intake#timeline_with_jobs`, the
missing `intake_blocker_transitions` association, a `handoff_state` presentation helper for the
mixed acknowledgeable/non-acknowledgeable merged feed) but not built. Recorded as **S4.5.10** in
[[Sprint plan]] — the actual next code task.

**The vault reconciliation itself.** Confirmed by grep that nothing anywhere documented
`IntakeActions`, `CreateIntake`, `CreateJob`, or `Permissions`, and that three places still asserted
the single-door shape ADR-013 reversed. Wrote a new [[Intake]] concept note (mirroring [[Job]]);
corrected the "In Rails" fact-lists in [[Job]], [[Data model]], [[Stage model]], [[Event log]],
[[Blocker]] (zero intake awareness before this pass despite ADR-012 §6 splitting blockers across
both levels), [[Job visibility]] (re-anchored from `jobs.token`/`customer_id` to `intakes` — this is
Sprint 7's design input, corrected before that sprint starts rather than after), [[M1-F1 Status flow and transitions]], plus
[[Overview]] and [[Open questions]] — stale, but missing from S4.5.8's own reconciliation list.
Ticked S4.5.2–S4.5.7, rewrote S4.5.5 (its `register_job!`-opens-or-finds framing was overtaken by
ADR-013), added S4.5.9 (retro, the Permissions split) and S4.5.10 (the timeline goal), re-pointed
S5/S6/S7's stale task text, and added a warning near the top of [[Sprint plan]] recording — as fact,
no tasks scheduled — that the app currently has no working UI path (`workshops#show` and
`customers#show` both 500 on route helpers the backend restructure removed).

**Next:** S4.5.10, the Intake timeline — the task this whole session's reconciliation thread traces
back to.

---

## Sessions 1–29 — archived
Vault/Rails setup through Sprint 4 (acknowledgement as stored visibility) and the Sprint 4.5
aggregate design pass (ADR-012) — Sessions 1–29. Moved to [[Worklog (Sessions 1-29)]] to keep this
file focused on the current arc: Session 30 onward (Sprint 4.5 built, the UI-surface map, the
Sprint 5/6 reorientation).

