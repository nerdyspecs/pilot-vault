---
id: ADR-003
type: decision
status: accepted
date: 2026-07-01
supersedes:
superseded_by:
---
# ADR-003 — V1 includes a digitized jobsheet

Extends the scope set in [[ADR-002 V1 scope]].

## Decision
V1 includes a **digitized jobsheet**: an owner-configurable inspection form attached to each job.
The form is a **flat list of fields** (label + type) the owner edits in admin — **not** a
form-builder (no sections, conditional logic, or field-type zoo).

## Why
The jobsheet is what workshops **already fill out on paper**. Digitizing the thing they already do
means staff fill the sheet as usual and **status tracking happens as a byproduct** — instead of
monitoring being extra data entry nobody keeps up. For existing workshops this is the adoption
wedge and the single biggest reason they'd switch. It's also the natural home for the V2 parts
breadcrumbs (inspection findings, job type).

## Scope boundary (what keeps it from ballooning)
- **Flat fields only** — `label` + `kind` (checkbox | text). No sections, no conditional logic.
- **One form per workshop** in V1. Multi-workshop templating is additive/deferred.

## Model
- `JobSheet belongs_to :workshop` — one form per tenant, owner-configured.
- `JobSheetField` (the fields, owner CRUDs) → `JobSheetFieldValue` (one car's answer),
  `belongs_to :job` + `belongs_to :job_sheet_field`.
- Fields are **rows, not columns** → owner adds fields at runtime, no migration.
- Full model in [[Data model]].

## Consequences
- V1 is larger than pure monitoring (ADR-002) — the jobsheet + a simple field-admin are real build.
- Accepted, because the adoption payoff is the whole game for existing shops.
- A form-builder and multi-workshop templating are explicitly **out** of V1.

## Related
- [[ADR-002 V1 scope]] · [[Data model]] · [[Roadmap]]
