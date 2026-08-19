---
type: concept
module: M1
updated: 2026-08-19 (Session 34 — resolved by ADR-015, annotated not rewritten; prior: created — chipped out from Session 33's jobsheet reversal)
---
# Inspection jobsheet — design + build brief

**Entry point for a future session.** Self-contained: everything needed to decide the storage
structure and build the fixed inspection jobsheet lives here, without depending on this
conversation. Start fresh from `main` (see §7).

> [!note] Resolved 2026-08-19 by [[ADR-015 Jobsheet answers are rows against a frozen question set]]
> §7's steps 1–2 (decide the storage structure; consider whether an ADR is needed) are **done** —
> see the ADR for the full reasoning, including two things this brief didn't anticipate: the
> SA/technician **concurrent multi-actor fill flow** (which ruled out per-section state), and the
> **frozen `item_keys` snapshot** (which closes a drift-on-addition gap this brief's storage fork
> didn't surface — see §4's annotation). Step 3 (build: migration → model → seed → tests) is what
> remains, on the builder's call. This note is kept as the reasoning trail, per house pattern —
> not rewritten.

## 1. The decision already made
[[ADR-014 Jobsheet is a fixed product-defined inspection]] settled this much, and it is **not
open for re-litigation** here:

- The jobsheet is a **fixed, product-defined** inspection form. Its fields are set by the
  product and versioned in code — an owner never adds, edits, or removes one at runtime. This
  supersedes [[ADR-003 Digitized jobsheet in V1]]'s owner-configurable core.
- The filled sheet is **per-visit** — one inspection per **Intake** (not per Job), unchanged from
  [[ADR-012 Intake-Job two-level aggregate]].
- **S5.4 (owner field-admin UI) is dropped entirely.** There is nothing left for an owner to
  administer.
- The old EAV model (`JobSheet` template / `JobSheetField` / `JobSheetFieldValue`) that was
  briefly built against the reversed decision is **gone** — see §7.

What this brief exists to decide: **how** the fixed form and its answers are actually stored.
That is genuinely open.

## 2. The drafted field list (TENTATIVE)
Drafted quickly by the builder during the reversal discussion — **not finalized**. Treat the
exact items, wording, and grouping as a strong starting point, not gospel. 5 sections, 39 items.

**EXTERIOR — 12**
1. Windscreen
2. Front bumper
3. Bonnet
4. Left body
5. Right body
6. Left side mirror
7. Right side mirror
8. Front lights
9. Front signals
10. Rear lights
11. Brake lights
12. Rear bumper / boot

Purpose: existing damage + obvious lighting/safety issues.

**TYRES — 8**
1. Front-left tread depth
2. Front-right tread depth
3. Rear-left tread depth
4. Rear-right tread depth
5. Front-left pressure
6. Front-right pressure
7. Rear-left pressure
8. Rear-right pressure

Purpose: safety + objective measurements.

**ENGINE BAY — 6**
1. Engine oil level
2. Coolant level
3. Brake fluid level
4. Clutch fluid level (N/A for some cars)
5. Battery condition/voltage
6. Visible fluid leaks

**BRAKES — 5**
1. Brake pedal free play
2. Brake pedal feel
3. Brake master pump
4. Brake pad thickness/condition
5. Brake disc/drum condition

**INTERIOR — 8**
1. Dashboard warning lights
2. Horn
3. Wipers
4. Washer spray
5. Seat belts
6. Air conditioning
7. Power windows
8. Central locking

## 3. The core modelling problem
Answer types are **not uniform**, so a naive single-shape table is the wrong model:

- **Ratings** — most EXTERIOR/ENGINE BAY/BRAKES/INTERIOR items: ok / attention / damage + a free
  note.
- **Numeric measurements** — all of TYRES (tread depth mm, pressure psi) and brake pad thickness
  (mm). These are wanted **queryable and trendable over time**, across a vehicle's visit
  history — e.g. "show this car's tread depth across its last 5 visits." That requirement is
  the tell: it rules out formats that bury numbers where they can't be aggregated cheaply.
- **Booleans** — a smaller set (e.g. visible fluid leaks, dashboard warning lights present).

"One wide table of booleans" was the shape the old EAV model avoided by making fields rows; a
fixed form doesn't need EAV's owner-editability, but it still needs to not collapse mixed types
into one column shape that serves none of them well.

## 4. The storage-structure fork — decide this
| Option | Shape | Trade-offs |
|---|---|---|
| **Wide typed table** | ~39 status + note columns, plus ~9 numeric columns (tread×4, pressure×4, pad×1), all on one per-intake row (or a 1:1 child of Intake). | Simple queries (`SELECT` one row, done). Wide and rigid — adding/renaming/removing an item is a migration, and 39+ columns on one table is a lot of surface. |
| **Code-defined catalog + answer rows** (lead candidate) | The item catalog (id, section, label, answer type) lives in a **Ruby constant** — fixed, versioned in git, and not owner-editable (so no drift risk, unlike the old EAV table). The DB stores **narrow answer rows** per intake: `item_key, status, value, note`. | Narrow and print-controlled (the catalog is code, so what prints is exactly what the product ships); an item change is code-only, no migration; still queryable/aggregable across visits since answers are real rows, not a blob. Reconciles "fixed + controlled" (the whole point of ADR-014) with "narrow + queryable" (the numeric-trend requirement). |
| **JSONB blob** | One `answers: jsonb` column per intake, keyed by item. | Narrowest schema of the three. Weakest for the numeric-measurement requirement — trending tread depth or pressure across visits means unpacking JSON in every query rather than querying a column; also weaker for any future reporting (§8's aggregate reports read the same event-log-style rows other models use). |

**The measurements are the tell.** If every item were a rating, jsonb would be perfectly
reasonable. The requirement to trend numeric values across a vehicle's history is what pushes
away from pure jsonb and toward something with real columns or real rows for at least the
numeric fields — which is most of the argument for the catalog + narrow-rows shape being the
right single answer for everything, ratings included, rather than mixing jsonb for some items
and columns for others.

> [!note] Decided 2026-08-19 — [[ADR-015 Jobsheet answers are rows against a frozen question set]]
> **Catalog + narrow answer rows won**, refined past what this table sketched: ratings, booleans,
> and enums collapse into one `choice` string column (they're all "pick one from a fixed set");
> only `measurement` (decimal) is split out, because it's the only type anything orders or
> aggregates on. A thin `jobsheets` header (1:1 per Intake) carries `inspection_type` plus a
> **frozen `item_keys` snapshot** — not anticipated by this table — which is what actually
> prevents a template edit from reinterpreting an old sheet (printing reads the sheet's own frozen
> list, not the live template) and what bounds which keys an answer row may use. See the ADR for
> the full reasoning and the rejected alternatives, including a DB-driven template hierarchy
> surfaced mid-session and re-rejected as the discarded EAV shape by another name.

## 5. Anticipated expansions — design for, don't build
Both are explicitly **"still thinking," not committed scope**. Don't build either now — but the
storage decision in §4 must not hard-assume a single universal sheet that would preclude them
later.

- **Multiple inspection types.** Beyond this one form: Pre-Delivery Inspection (PDI), Used-Car
  Inspection today; Lorry and Passenger-car variants later. This likely reshapes the model into
  a small **catalog of product-defined inspection types**, each carrying its own fixed field
  set, with an Intake picking which type applies. If the chosen storage structure bakes in "the
  one universal 39-item sheet" as a schema-level assumption (e.g. 39 named columns), this
  expansion becomes a rebuild instead of an addition — worth weighing when scoring §4's options.
- **Exterior damage diagram.** A base vehicle image (car and lorry variants) that the technician
  "dots" to mark cracks/nicks, with an optional photo upload per mark. This is **its own
  sub-design**, separate from the other four sections and from the rating/numeric/boolean model
  above — it implies a coordinate/annotation model plus attachments, not just a per-item rating.
  Flag it as a distinct piece of scope; do not fold it into the general answer-row shape without
  a dedicated design pass.

> [!note] Status 2026-08-19 — [[ADR-015 Jobsheet answers are rows against a frozen question set]]
> **Multiple inspection types**: mechanism built now (per-template Ruby files + `inspection_type`
> on the jobsheet header), content added later with no schema change — the model doesn't hard-
> assume a single universal sheet, as this section required. **Exterior damage diagram**: still
> fully deferred, still its own design pass — not touched by ADR-015, tracked in
> [[Deferred design]]. One design-for decision made in its favor: photos anchor to `jobsheet_
> answers` (a finding), not to the jobsheet as a whole — so when the diagram is designed, it has
> a per-item home to attach to rather than a flat pile.

## 6. Housekeeping
The earlier EAV attempt at this (branch `s5-jobsheet-models`, models `JobSheet` +
`JobSheetField`, a seeded default template, model tests — 4 commits) has already been
**discarded**. Its two migrations were rolled back while the files still existed so the dev DB
dropped the tables cleanly, then the branch was deleted. `main` was never touched by it. Start
this work fresh from `main` — there is nothing to rebase or recover.

## 7. Required first steps for this future session
1. **Decide the storage structure** — pick from §4 (or a variant), scored against §3's
   requirement and §5's expansions.
2. **Consider whether a follow-up ADR is needed** to record the storage choice, if it turns out
   structural (it likely is — it's a schema commitment other work will build against).
3. **Build model + migration + tests**, four-layer like the discarded S5.1/S5.2 were originally
   built — migration → model → seed → tests, each its own commit:
   - **RLS + `workshop_id` + `WorkshopScoped`**, per house pattern — read
     `app/models/concerns/workshop_scoped.rb` and an existing migration under `db/migrate/` that
     sets up RLS (e.g. the intake or blocker migrations) for the concrete shape to follow.
   - **Keyed on `intake_id`**, `references :intake` — per [[ADR-012 Intake-Job two-level aggregate]],
     not `job_id`.

## Related
- [[ADR-014 Jobsheet is a fixed product-defined inspection]] · [[ADR-003 Digitized jobsheet in V1]]
  (superseded core) · [[ADR-012 Intake-Job two-level aggregate]] · [[Data model]] · [[Sprint plan]]
  · [[Intake]]
