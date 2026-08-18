---
id: ADR-014
type: decision
status: accepted
date: 2026-08-19
supersedes: ADR-003 (owner-configurable jobsheet core)
superseded_by:
---
# ADR-014 — Jobsheet is a fixed, product-defined inspection

From building **S5.1/S5.2** ([[Sprint plan]]) — the `JobSheet`/`JobSheetField` template models
were built on branch `s5-jobsheet-models` (4 commits) before the builder interrogated the
design underneath them and reversed it. This ADR records the reversal and discards the branch.

## Decision
The jobsheet is a **fixed, product-defined inspection form** — its fields are set by the
product and versioned in code, not owner-configurable at runtime. This reverses
[[ADR-003 Digitized jobsheet in V1]]'s core: "an owner-configurable inspection form... the owner
edits in admin." Owners no longer author their own field list; the product ships one.

The filled sheet stays **per-visit**, unchanged from [[ADR-012 Intake-Job two-level aggregate]]
— one inspection per **Intake**, not per Job.

## Why
Configurability was the single upstream cause of the whole rabbit hole, not an independent
feature pulling its weight:

- **No print control.** An owner-editable field list means the product can never guarantee what
  a printed sheet looks like — every workshop's sheet is potentially a different shape. A fixed
  form returns that control to the product.
- **Template versioning + drift.** Owner-editable fields force a template-versioning story (what
  does an old filled sheet mean once the owner has since added/renamed fields?) and a
  per-answer-label-snapshot mechanism to keep history honest as the template drifts underneath
  it. A fixed, code-defined form has no drift to snapshot against — the form standing today
  behind an old answer *is* the form that was live for it, because it never moves except by a
  product release.
- **The adoption trade is real, and consciously taken.** ADR-003's bet was that digitizing *the
  workshop's own paper sheet* is the wedge — matching what staff already do beats asking them to
  learn something new. That bet is not free: a well-designed **standard** sheet is judged the
  better v1 bet, on balance, once configurability's cost (above) is counted against it. Losing
  "your sheet, digitized" is accepted, not overlooked.

## Consequences
- **The EAV three-table model is dropped**: `JobSheet` (template) / `JobSheetField` (owner-CRUD
  fields) / `JobSheetFieldValue` (answer) no longer exist. The built branch
  (`s5-jobsheet-models`) is discarded — see [[Sprint plan]] for the S5.1/S5.2 tick reversal.
- **S5.4 (owner field-admin UI) is dropped entirely.** There is nothing left for an owner to
  administer — the field list ships in code, changed only by a product release.
- **The filled sheet is per-visit, per-Intake**, exactly as ADR-012 already ruled — this ADR
  changes *who defines the fields*, not *what the filled sheet hangs off*.
- **The storage structure is undecided and explicitly deferred** to
  [[Inspection jobsheet — design brief]]. A concrete 39-item field list (5 sections, mixed
  answer types — ratings, numeric measurements, booleans) surfaced after this reversal and
  reopens the question of *how* a fixed form is stored (wide typed table vs. a code-defined
  catalog + narrow answer rows vs. jsonb). Also deferred there, as design inputs only, not
  decisions: multiple inspection types (PDI / used-car / lorry / passenger) and an exterior
  damage diagram (dot-annotated vehicle image + optional photo). None of this is decided by this
  ADR — only that the form is fixed and product-defined.

## Rejected alternatives
- **Keep ADR-003's owner-configurable core, add versioning to fix the drift.** Considered
  implicitly by continuing to build S5.1–S5.3 as EAV. Rejected once the shape of the fix became
  visible: per-answer label snapshots, a template-version column, and admin UX for reordering/
  soft-deleting fields — real, ongoing complexity purchased to preserve a customization power
  no concrete workshop had asked to exercise yet. The configurability was speculative; the
  complexity it would have bought was not.
- **Keep configurability, but freeze the template once any answer exists.** A middle path where
  a workshop could still shape its own form, just not after first use. Rejected for the same
  reason as full configurability once the field list itself turned out to need genuinely mixed
  answer types (numeric measurements, not just checkbox/text) — a per-workshop form-builder
  capable of expressing that is a bigger build than the fixed form it would replace, for a
  capability nothing in the product's actual customer base has demonstrated needing.

## Related
- [[ADR-003 Digitized jobsheet in V1]] (superseded, core) · [[ADR-012 Intake-Job two-level aggregate]]
  (the per-visit anchor, unchanged) · [[Inspection jobsheet — design brief]] (the deferred storage
  decision) · [[Data model]] · [[Sprint plan]]
