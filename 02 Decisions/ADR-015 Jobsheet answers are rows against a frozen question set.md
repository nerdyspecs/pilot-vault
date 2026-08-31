---
id: ADR-015
type: decision
status: accepted
date: 2026-08-19
extends: ADR-014 (Jobsheet is a fixed product-defined inspection)
supersedes:
superseded_by:
---
# ADR-015 — Jobsheet answers are rows against a frozen question set

From working [[Inspection jobsheet — design brief]] to completion with the builder (Session 34).
[[ADR-014 Jobsheet is a fixed product-defined inspection]] settled *that* the jobsheet is fixed
and product-defined; it explicitly deferred *how* the fixed fields and their answers are stored.
This ADR makes that call — it's a schema commitment other work (S5.5's fill step, S6's board,
future reporting) will build against, the same tier as ADR-012/ADR-013.

## Decision

Three layers, none of them the discarded EAV shape:

```
TEMPLATES (code, no DB row)     app/inspections/car_routine.rb …
  key · section · position · label · answer_type · options · unit · required · retired

jobsheets (db, 1:1 with Intake)
  workshop_id · intake_id (unique) · inspection_type · item_keys[]

jobsheet_answers (db, field-level)
  workshop_id · jobsheet_id · item_key
  choice · measurement · not_applicable · note · inspected_by_id
  unique(jobsheet_id, item_key)
```

**Templates are Ruby constants**, one file per inspection type (`car_routine.rb` seeded now;
`pdi.rb`/`lorry_routine.rb` later — content only, no schema change). Each item carries
`answer_type` (`rating | boolean | enum | numeric`), which decides which of the two value columns
an answer uses and how the UI renders it. This is the mechanism `answer_type` implies, per
[[ADR-014 Jobsheet is a fixed product-defined inspection]] — the field list stays code, versioned
in git, never owner-CRUD.

**Two value columns, not four.** `rating`, `boolean`, and `enum` all reduce to the same question
— *which one of a fixed set of options was picked* — so they share `choice` (string). Only
`numeric` items (tyre tread/pressure, pad thickness) need `measurement` (decimal), because they're
the only type anything does arithmetic or ordering on. Splitting further (a column per answer
type) was tried in discussion and rejected: it buys nothing a catalog-level `answer_type` doesn't
already give the renderer, at the cost of validation and querying against four columns instead
of two.

**`item_keys` is frozen on the jobsheet at creation**, re-snapshotted only while the sheet has
zero answers, immutable the instant the first answer lands. This is the single mechanism that
answers three separate questions at once: what a printed sheet legitimately asked (§Why), which
`item_key`s an answer row may use (the load-bearing consequence, below), and what "complete" means
for this specific sheet, forever, regardless of later template edits.

**Printing reads from answers** (iterate `item_keys`, resolve each label via the catalog);
**a blank form to fill reads from the template** (iterate non-retired items). These are
deliberately different sources — see §Why.

**No door, no state machine.** Any staff member may write any field, concurrently — the SA
checking exterior with the customer and a technician checking the engine bay at the same time is
the normal case, not a race to guard against. Sections are catalog grouping and a UI hint only;
no role owns one. Answers are mutable (a later inspector may correct an earlier value) until the
intake leaves `open` (`delivered`/`cancelled`), at which point the sheet is history and stops
accepting writes.

**`not_applicable` is an explicit boolean**, orthogonal to `choice`/`measurement` (a numeric item
can be N/A too — clutch fluid on an automatic). Blank means only "not yet answered." Completion is
**derived**, never stored: every required, non-`not_applicable` key in `item_keys` has an answer
row — and it is only asked of **open** intakes; a delivered or cancelled sheet is history, not a
to-do, so a template growing after the fact can never retroactively mark it incomplete.

**Answer rows carry `inspected_by`, not `created_by`** — a deliberate departure from the house
convention (see Consequences).

**The customer's complaint is not a jobsheet field.** It stays free-form, per-visit text on
**Intake** — the jobsheet is the standardized inspection only, filled by staff, in fixed
vocabulary. Conflating the two would force free text into a form built for fixed answer types.

## Why

**The measurement-trending requirement is the tell**, same argument the brief's §4 already made:
tread depth, tyre pressure, and pad thickness need to be queryable and orderable across a
vehicle's visit history. A jsonb blob buries that behind per-query JSON extraction; storing a
number as a string sorts `"10"` before `"4.5"` and can misflag a brand-new tyre as dangerously
worn. Only real rows with a real decimal column serve that requirement cheaply — which also
settles ratings/booleans/enums into rows, for a uniform shape rather than mixing storage styles by
type.

**Absence of any veto is what keeps this out of the door.** Job and Intake are state machines —
illegal moves exist that must be refused (`Job#transition!`'s allow-list, the blocker vetoes on
`deliver!`). Nothing about filling a jobsheet has an equivalent: nobody is ever *stopped* from
recording an answer. Data collection, not a workflow — so `JobActions`/`IntakeActions`/`Permissions`
([[ADR-013 The door decomposed]]) don't apply here, and building a parallel door would be modelling
by analogy to the neighboring aggregates rather than by what this one actually needs.

**The multi-actor fill flow is what killed per-section state.** An earlier candidate design gave
each section its own stateful row (`completed_by`/`completed_at`), reasoning that the SA owns
exterior and the technician owns the rest. The builder's actual flow contradicts that: whether the
SA walks the car with the customer or a technician does the whole sheet alone is circumstantial,
not fixed, and a smaller workshop needs anyone to pick up a sheet someone else started. No role
owns a section, so no section earns stored lifecycle state — the fact that varies is *who answered
this specific item*, which is exactly `inspected_by` at field granularity, not a section-level
sign-off.

**Printing-from-answers vs filling-from-template is not an inconsistency, it's the fix for
template drift.** Say a template gains a field (a hypothetical CO2 tailpipe reading) after old
sheets were already filled. If printing walked the *current* template, an old sheet would show a
blank line for a question it was never asked — reading as staff negligence rather than as history.
Printing from the sheet's own frozen `item_keys` instead means a printed sheet shows exactly what
it actually asked, no more, no less — regardless of what the template looks like today.

## The load-bearing consequence

**`item_key` is a string handshake between code (the catalog) and data (answer rows), with no
foreign key behind it.** Nothing in the database stops a catalog key from being renamed or deleted
out from under existing answer rows — it holds only because of a discipline this ADR makes
standing:

> **Catalog keys are permanent. Retire an item (`retired: true`), never delete or rename its
> key.** A wording/typo fix to a `label` may be made in place — old sheets inherit the correction
> automatically, which is intended. If an item's *meaning* changes, it is a **new key**; the old
> key is retired, not repurposed.

`item_keys` (frozen per jobsheet) is the second half of the safety net: even if this discipline is
ever broken, an existing sheet's frozen list still names exactly which keys it depends on, so a
catalog change can be checked against every sheet that references the retired key before it's
touched. Recorded here as a known standing obligation — same posture as ADR-013's door-invariant
footnote — not a silent assumption.

## Consequences

- **`inspected_by`, not `created_by`, on `jobsheet_answers`.** Every other event/transition table
  in the house uses `created_by` because the actor is audit metadata on a log row. Here the actor
  *is* a first-class domain fact — the inspector of record for that finding — so it gets the
  domain name. Deliberate, reasoned exception; not a drift from convention.
- **Freeze-on-terminal is [[Architecture laws]] #8 applied, not a new rule.** *"A Done job is
  immutable"* already states the principle; a jobsheet going read-only once its Intake reaches
  `delivered`/`cancelled` is the same law at a different aggregate. `ready?` was considered as the
  freeze point and rejected — a ready intake is still `open` and still legitimately editable
  (a technician may yet correct a finding before delivery).
- **Derived completion is [[Architecture laws]] #3 applied** (*"dashboards are queries, not tables"*) —
  no stored completion flag, a scope over `item_keys` vs. answered rows.
- **[[ADR-014 Jobsheet is a fixed product-defined inspection]]'s "no drift because the form is
  code-defined" claim is narrowed, not wrong.** It correctly ruled out drift from *removal/rename*
  (retirement handles that). It did not anticipate drift from *addition* — a template growing new
  items after old sheets exist. `item_keys` is what actually closes that second half. ADR-014 is
  not edited (ADRs are never edited); this is the dated footnote.
- **The complaint/inspection split is new and real**, not implied by ADR-014. [[Intake]]'s stale
  `has_many :job_sheet_field_values` line (which glossed the jobsheet as "customer complaints +
  vehicle condition") is corrected by this ADR — complaints are Intake data; the jobsheet never
  carries free-form customer text.
- **Multiple inspection types get their mechanism now, content later.** `inspection_type` +
  per-template files means adding PDI or a lorry variant is a new file and a seed, not a schema
  change — the anticipated expansion [[Inspection jobsheet — design brief]] §5 flagged is honored
  without being built.
- **Photos, an answer edit activity log, and the exterior damage diagram remain explicitly
  deferred** — see [[Deferred decisions]]. The model is shaped to not preclude any of them
  (photos anchor to `jobsheet_answers`, not `jobsheets`, so they organize per finding) but none is
  built by this ADR.

## Rejected alternatives

- **Wide typed table** (~39+ status/note/numeric columns on one row). Rejected: every field
  add/rename/remove is a migration, and a second inspection type means a second wide table or
  NULL-riddled columns on the first — collides directly with §5's multiple-inspection-types
  expansion.
- **JSONB blob per intake.** Rejected on the measurement-trending requirement above: numeric
  aggregation/ordering across a jsonb column means unpacking JSON per query rather than a plain
  indexed `WHERE`/`ORDER BY`.
- **DB-driven template hierarchy** (`InspectionTemplate → Category → Item`, owner-editable rows,
  surfaced mid-session as an externally-sourced design). This is, table-for-table, the discarded
  EAV branch (`JobSheet`/`JobSheetField`/`JobSheetFieldValue`, `s5-jobsheet-models`) that
  [[ADR-014 Jobsheet is a fixed product-defined inspection]] already reversed — its stated
  design principle ("workshops can add/reorder/disable items") is exactly the owner-configurability
  ADR-014 rejected for print-control and drift reasons. Re-examined on its merits here rather than
  dismissed on sight; re-rejected for the same reasons. Two pieces of it were kept: `enum` as a
  distinct answer type (this ADR's `choice` column generalizes it), and photos attached to a
  *finding* rather than a whole sheet (adopted as the deferred design for photos, above).
- **Per-section fill records** (`jobsheet_sections`: one stateful row per section per intake,
  `completed_by`/`completed_at`). The natural home for the "who signed off exterior" instinct —
  rejected once the actual fill flow surfaced no role that owns a section. Would have pulled the
  jobsheet toward a state machine (a keeper of stored state, likely eventually a door verb) for a
  fact — section ownership — that turned out not to exist.
- **Label snapshot on the answer row** (freeze `label`/`section`/`unit` onto each
  `jobsheet_answer`, an idea the builder proposed independently). Re-examined on new grounds
  (not the old EAV-era reasoning in [[Deferred decisions]], already moot): printing a line needs
  label *and* section *and* position *and* unit, so one column grows to four or five, duplicated
  across ~39 rows × every intake × every workshop, forever — the exact "per-answer-label-snapshot
  mechanism" ADR-014 already priced as a real ongoing cost. `item_keys` + append-only catalog gets
  the same fidelity for one array column, because the catalog (git) already preserves label
  history for free.
- **Everything as one `value` string column**, including measurements. Rejected concretely:
  lexicographic string comparison misorders numbers (`"10" < "3"`), and casting at query time
  (`value::decimal`) fails unpredictably once the Postgres planner reorders a `WHERE` clause ahead
  of a `WHERE item_key = …` filter — a bug that passes in dev and breaks at production volume.
- **A template version column** (`car_routine`, v2, v3…), as the alternative fix for drift under
  old sheets. Rejected in favor of `item_keys` + retirement: versioning requires keeping every
  historical template body in code forever *and* a human remembering to bump the version on every
  edit — a step easy to forget, silently corrupting history when missed. A frozen key list needs
  no such discipline; it records itself at creation.

## Related
- [[ADR-014 Jobsheet is a fixed product-defined inspection]] (extended by this — the fixed/
  product-defined ruling stands; this decides the storage it deferred) ·
  [[ADR-012 Intake-Job two-level aggregate]] (the per-visit anchor, unchanged — keyed on
  `intake_id`) ·
  [[ADR-013 The door decomposed]] (the contrast this ADR draws against — no veto, no door) ·
  [[Inspection jobsheet — design brief]] (the entry point this ADR resolves) · [[Data model]] ·
  [[Architecture laws]] · [[Deferred decisions]] · [[Risk ledger]] · [[Sprint plan]]
