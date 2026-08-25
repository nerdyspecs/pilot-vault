---
type: concept
module: M1
created: 2026-08-25
updated: 2026-08-25
---
# Intake UI — build brief
**Task: S5.5, the counter half.** Build the screens that open a visit — plate in, visit out.
This is the app's missing front door: `POST /intakes` and `POST /intakes/:intake_id/jobs` are
the only two endpoints in the whole app with no UI caller ([[Screen flow]] §Issues surfaced).

This brief is **self-contained**. It carries what is already decided, what is genuinely open,
and what is explicitly out of scope. Read [[Intake flow]] (the SA behaviour spec) alongside it —
that note is the decision tree; this note is the build.

> [!important] UI work resumes here
> The last several sprints were backend. Nothing about the intake path renders today. Expect to
> write Slim views, not just controllers.

---

## 1. Decisions already made — do not reopen

**The create path** *(decided 2026-08-25 with the builder)*. Three pieces, no new orchestrator
class:

1. **Lookup is model class methods.** Plate and phone lookup extend what already exists —
   `Vehicle.canonicalize` and `Customer.search` (which canonicalizes phone the same way storage
   does). Unit-testable without a request, per [[Design laws]] #9.
2. **The SA's branch tree resolves in the UI flow.** Its *output* is a resolved Customer +
   Vehicle. The two mutating branches — 1b-iii (vehicle changed hands) and 1b-iv (correct the
   card's phone) — run as ordinary explicit CRUD on `Vehicle`/`Customer`, each confirmed by the
   SA. They are corrections that happen at the counter, not part of a birth.
3. **`CreateIntake` widens by exactly one argument: `complaint:`.** It writes to the `Jobsheet`
   that `CreateIntake` already creates inside its own transaction.

*Rejected, with reasons — do not re-propose:*
- **Find-or-create inside the controller** — plate canonicalization + lookup + the
  ownership-transfer branch is logic, and [[Design laws]] #9 forbids logic you can't call
  without a request.
- **A fat `CreateIntake` that absorbs the decision tree** — 1b-iii mutates an existing vehicle
  and 1b-iv edits an existing customer. Neither is a birth. Folding them in turns an authoring
  class into a decision tree with side effects.
- **A new `RegisterVisit` orchestrator** — adds a concept [[ADR-013 The door decomposed]] didn't
  name, for a tree the UI already has to walk screen by screen anyway.

*Accepted consequence:* a resolved Customer/Vehicle is created **before** the intake transaction
opens, so an abandoned intake can leave a customer with no visit. This is correct, not a leak —
a person who walked in is a real record, and [[Intake flow]] §2a (the reverse-wife trap) wants
that card to exist next time. Do not add cleanup.

**Settled elsewhere, still binding here:**
- The **complaint lives on `jobsheets.complaint`**, a free-text header column — *not* on Intake.
  Intake has no complaint column. (Session 35 build footnote narrowing
  [[ADR-015 Jobsheet answers are rows against a frozen question set]]; see [[Data model]] §The jobsheet.)
- **`inspection_type` is write-once** (`attr_readonly` on `Jobsheet`, alongside `item_keys`).
  There is no re-select path: a mis-typed sheet is deleted and recreated. So the type must be
  chosen *at intake*, deliberately, on the screen that opens the visit.
- **Phone is mandatory** on every customer — DB `NOT NULL` plus a presence validation. No card
  without a phone; this is what makes the §2a lookup reliable ([[Intake flow]] principle 1).
- **Registration number is canonicalized at storage** (whitespace stripped, upcased) and unique
  per workshop. Search must canonicalize identically — `Vehicle.canonicalize` exists for exactly
  this.
- **One open intake per vehicle** — partial unique index `WHERE status = 0`. A second open visit
  on the same car is a DB violation, not a validation to hand-roll. Surface it as
  [[Intake flow]] §1c does: the lookup *reports the visit in house*, it doesn't offer "register".
- **Creation is authoring, not a door verb** ([[ADR-013 The door decomposed]]). `CreateIntake`
  and `CreateJob` write directly; `JobActions`/`IntakeActions` are not involved in a birth.
- **Tenancy**: never query bare. `Customer.for_current_workshop`, `Vehicle.for_current_workshop`.
  Lookups see this workshop's book only — other workshops' vehicles do not exist here.

---

## 2. What exists in code today

**Models** — `Customer` (kind person/company, canonicalizing `before_validation` on phone+email,
`search` scope), `Vehicle` (`canonicalize` class method, canonicalizing `before_validation`,
`belongs_to :customer`), `Intake` (`has_secure_token :token`, status enum open/delivered/cancelled,
`ready?`, `registered_by`), `Jobsheet` (1:1 with intake, `complaint` column, `item_keys` frozen
from the template on create, `complete?`/`editable?` derived), `JobsheetAnswer`.

**Authoring** — `app/services/create_intake.rb`:

```ruby
CreateIntake.new(vehicle:, acting_user:, jobs: [ {} ], customer: nil,
                 inspection_type: Inspection::DEFAULT_TYPE).call
```

One transaction: `Intake` + its birth `IntakeStatusTransition` + `Jobsheet` + one `CreateJob` per
entry in `jobs:`. Defaults to one unassigned repair. **Takes no `complaint:` — that's the widening
this task makes.**

**Inspection catalog** — `app/inspections/`. `Inspection::TYPES` is `%w[car_routine lorry_routine]`,
`DEFAULT_TYPE` is `car_routine`. `Inspection.items(type)` returns non-retired `InspectionItem`s
sorted by position. The type picker reads `Inspection::TYPES`; do not hard-code the two names.

**Controller** — `IntakesController` has `create` / `show` / `deliver` / `cancel`, gated
`authenticate_user!` → `require_workshop!` → `require_counter!`. `create` currently does
`Vehicle.for_current_workshop.find(params.require(:vehicle_id))` — it assumes the vehicle already
exists and was chosen elsewhere. **There is no `new` action and no `GET /intakes/new` route.**

**Authorization** — gate with `require_counter!` (already on this controller); hide with
`counter_staff?` in views. Both delegate to `Permissions`, so they can't drift. Do not write a
new check.

**Views that exist** — `intakes/show` (the landing target: plate, customer, repair list, holds,
deliver/cancel), `customers/index` (with a `q` search box), `customers/new`, `customers/show`.

**Traps in the existing code:**
- `app/views/customers/new.html.slim` carries a `registration_number` hidden field and a
  "Registering X under this new customer" line — a vestige of the pre-ADR-012 flow.
  `customers#create` **ignores it**. Either wire it into the new flow deliberately or remove it;
  don't leave it half-live.
- `app/views/customers/show.html.slim:37` has a comment saying the start-a-visit entrypoint is
  "S6" — stale since the Sprint 5↔6 swap, and it marks the spot where the link belongs. Fix the
  comment when you add the link.
- The workshop dashboard (`workshops/show`) offers Customers / Crew / Blocker types and no way to
  start a visit. That's the natural home for the front door.

**Suite:** 154 runs, 0 failures at `2c5816f`. Keep it green commit by commit.

---

## 3. What to build

- **`GET /intakes/new`** — plate-first entry. Canonicalized lookup against this workshop's book.
- **The resolution flow** — the branches in [[Intake flow]] §1 and §2, reaching a resolved
  Customer + Vehicle. Every fork in that note must be reachable; none may be silently skipped.
  §1c (vehicle already has an open visit) is a *report*, not an error page.
- **Complaint capture** — free text, written to `jobsheets.complaint`.
- **Inspection type picker** — from `Inspection::TYPES`, on the screen that opens the visit,
  because it can never be changed afterwards.
- **`CreateIntake` widened by `complaint:`**, written inside the existing transaction.
- **`intakes#create` reworked** to accept what the flow produces instead of a bare `vehicle_id`.
- **Entry points restored** — from the workshop dashboard, and from the customer page where the
  stale comment marks the spot.
- **Tests** — the graph is created whole (Customer/Vehicle/Intake/Job/Jobsheet); complaint lands
  on the jobsheet; both lookup keys canonicalize; the four §1b forks; §2a (existing card, new
  vehicle); §1c (open visit blocks a second); tenancy isolation on both lookups. This is
  Sprint plan **S5.9**'s first half — tick what you cover.

---

## 4. The one open design question — screen decomposition

**Undesigned, and yours to decide.** [[Intake flow]] is a behaviour spec: two lookup keys, one
script question, four mismatch forks. It says nothing about how many screens that is, or whether
the branches are a wizard, one page with progressive reveal, or a page that re-renders per answer.

Constraints that bind the answer ([[Visual theme]] §UI laws):
- **The SA is PC-primary.** Law 9 — bigger screens buy density, not size. Law 10 — the flow must
  complete on tab/Enter alone, autofocus the plate field, no click-only widgets.
- **Law 3 — one primary action per screen.** Whatever the decomposition, each step has exactly
  one solid `--action` button.
- **Law 8 — empty states and errors tell the truth.** "No vehicle with that plate" must say what
  happens next, not dead-end.
- **Law 6 — words over icons.** The forks are questions with honest answers; label them as the
  script speaks them.
- **Law 2 — status colours are reserved words.** A mismatch confirm is not an error state; don't
  reach for red. It's a decision, per [[Intake flow]] principle 3.
- **Under a minute** for the happy path ([[User stories]] §Design constraints) — ~90% of visits
  are §1a, plate matches, phone matches. That path must stay frictionless and must never show a
  warning.

Bootstrap 5.3.3 vendored, brand layer in `application.css`, tokens per [[Visual theme]]. **No
Bootstrap JS is loaded** — if a component needs it, say so in the carry-back rather than adding
an importmap entry unprompted.

---

## 5. Explicitly out of scope

- **The jobsheet fill screen** (39 items) — different user, different device, and blocked on the
  three flagged catalog content calls in `car_routine_inspection.rb`. The sheet is *created* here,
  empty; it is not filled here.
- **ETA / `promised_ready_at`** — Sprint plan S5.6.
- **Vehicle history on the intake screen** — S5.7.
- **Adding a repair to an already-open intake** — a separate authoring path; the intake page
  already notes the gap.
- **Owner notification channel** — [[Open questions]]; v1 is a manual copy-paste message and
  nothing about it is built here.
- **The board** (Sprint 6) and the **owner token page** (Sprint 7).
- `brought_in_by` — an accepted v1 gap, named in [[Intake flow]] §1b-i. Do not add the column.

---

## 6. Required first steps

1. Read `CLAUDE.md`, then this brief, then [[Intake flow]] in full — the forks matter and are
   easy to under-build.
2. Branch off `main`. Never work on `main`.
3. Decide §4 (screen decomposition) and state the decision in the carry-back with its reasoning.
4. Build in layers, one commit each, task ids in the messages (`S5.5: …`).
5. Green suite at every commit. Do not push.
6. Before closing: `git log --oneline main..<branch>` must reflect your work.

---

## Related
[[Intake flow]] · [[Sprint plan]] S5.5 · [[Screen map]] · [[Screen flow]] · [[Data model]] ·
[[Visual theme]] · [[Design laws]] · [[ADR-012 Intake-Job two-level aggregate]] ·
[[ADR-013 The door decomposed]] · [[ADR-015 Jobsheet answers are rows against a frozen question set]] ·
[[Inspection jobsheet — design brief]] (the fill half, not this task)
