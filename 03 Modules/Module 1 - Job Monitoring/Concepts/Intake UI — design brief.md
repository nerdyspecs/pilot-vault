---
type: concept
module: M1
created: 2026-08-25
updated: 2026-08-25 (Session 36 — Q1's premise corrected: it was factually wrong; brief re-scoped, the screen design now lives in [[Design system]])
---
# Intake UI — design brief
**Task: design the intake flow's screens and decide the build order. This is a DESIGN session,
not a build session.** Same pattern as [[Inspection jobsheet — design brief]], which reasoned out
storage structure with the builder and produced
[[ADR-015 Jobsheet answers are rows against a frozen question set]].

**Output is written into the vault: design decisions, a build order, and an ADR if a call turns
out to be structural. Not code.** Do not create branches, migrations, controllers, or views.

> [!important] The builder drives
> This project runs in learning mode — the builder writes the code. A design session's job is to
> reason the fork out *with* them, surface options and tradeoffs, and record what gets settled.
> Bring recommendations with arguments; do not rule alone on anything structural.

---

> [!warning] Status, 2026-08-25 (Session 36) — read this before the questions below
> **Q1's premise is wrong** (corrected in place below) and the session that answered these
> questions widened past the brief. The **screen design and the build order now live in
> [[Design system]]**, which is the current plan of record. This note is kept for its §1 and
> §2 — the settled constraints and the state-of-the-app facts, which still hold — and for the
> corrected Q1. **Q2–Q6 are superseded**: the builder ruled that customers and vehicles are
> created as ordinary CRUD ahead of the visit, so intake's first cut is [[Intake flow]] §1a plus
> §1c only, and the mismatch/dedup screens those questions were about are deferred with the tree.

---

## 1. What is already settled — do not reopen

**The SA's behaviour** is fully designed in [[Intake flow]] (2026-07-15): two lookup keys
(registration number → vehicles, phone → people), one script question ("whose file should this be
under?"), and every real-world fork — §1a happy path, §1b's four mismatch branches, §2a/§2b's
first-visit dedup traps. **That note is the behaviour spec.** This session designs the *screens*
that carry it, not the behaviour itself.

**The mismatch confirm is designed in prose** and explicitly waiting on this moment.
[[Deferred design]] records it (2026-07-15, with the builder): on mismatch, show *"vehicle filed
under Lim, billing Tan"* with **two explicit choices** — `[Just this visit]` (stamp only, file
untouched — borrower or third-party payer) vs `[Vehicle changed hands]` (also
`vehicle.update!(customer:)` in the same transaction — informal sale). **Two buttons because the
two stories need different writes; one generic "confirm" would leave stale files and train
click-through.** That entry says it circles back "when the intake UI exists (Phase 4 / Sprint 5)"
— it is now this session's job to turn that prose into a screen.

**The create path** *(decided 2026-08-25 with the builder)*: plate/phone lookup as model class
methods (extending the existing `Vehicle.canonicalize` and `Customer.search`, per
[[Design laws]] #9 — logic you can't call without a request is misplaced); the SA's branch tree
resolves in the UI flow; the two mutating branches (§1b-iii vehicle changed hands, §1b-iv correct
the card's phone) run as explicit CRUD; `CreateIntake` widens by `complaint:` only.
*Rejected: find-or-create in the controller (law #9); a fat `CreateIntake` absorbing the tree
(1b-iii and 1b-iv are corrections, not births); a `RegisterVisit` orchestrator (a concept
[[ADR-013 The door decomposed]] declined to name).* If the screen design contradicts this, say so
with reasoning rather than quietly deviating.

**Binding constraints, settled elsewhere:**
- The **complaint** is free text on the `jobsheets` header — not on Intake. Intake has no
  complaint column.
- **`inspection_type` is write-once** (`attr_readonly`). No re-select path exists; a mis-typed
  sheet is deleted and recreated. So the type is committed at intake, which is a design
  consideration, not a detail.
- **Phone is mandatory** on every customer (DB `NOT NULL` + validation) — it is the person-key
  that makes the §2a dedup lookup work.
- **Registration number is canonicalized** at storage and unique per workshop.
- **One open intake per vehicle** — a partial unique index, not a hand-rolled validation. §1c
  says the lookup *reports the visit already in house* rather than offering to register.
- **Tenancy**: lookups see this workshop's book only.

---

## 2. The state of the app — facts to design against

**Every GET screen that exists today:**

```
/  ·  /workshop  ·  /workshop/new  ·  /staff  ·  /blockers
/customers  ·  /customers/:id  ·  /customers/new  ·  /customers/:id/edit
/intakes/:id  ·  /jobs/:id  ·  /invitations/new
```

**There is no `/intakes` index and no `/jobs` index** — but ~~`/intakes/:id` and `/jobs/:id` are
reachable only by knowing the id. The only lists in the app are customers, crew, and blocker
types.~~ **Corrected 2026-08-25 (Session 36): `/workshop` is a job list — it *is* the board.** It
renders every active repair plus the done-awaiting-delivery group, each row linking to
`/jobs/:id`, which links on to its visit. A missing index *route* is not a missing *list*; reading
the first as the second is what produced Q1's false premise below.
Navigation is the wordmark plus the workshop name — there is no nav bar to hang a front
door on. *(That gap is real, and it is now [[Design system]] L1.)*

**Backend that already works:** `CreateIntake` (opens Intake + birth transition + Jobsheet + one
`CreateJob`, one transaction), `Intake` (token, status enum, `ready?`), `Jobsheet` (1:1, frozen
`item_keys`), `Inspection::TYPES` (`car_routine`, `lorry_routine`), `Customer`/`Vehicle` with
canonicalizing callbacks, `Permissions` (`require_counter!` / `counter_staff?`).
`POST /intakes` and `POST /intakes/:intake_id/jobs` are the only two endpoints in the app with no
UI caller ([[Screen flow]]). Suite: 154 runs green at `2c5816f`.

**Two vestiges in the views:** `customers/new.html.slim` carries a `registration_number` hidden
field that `customers#create` ignores; `customers/show.html.slim:38` has a comment marking where
the start-a-visit link belongs (it says "S6" — stale since the 5↔6 swap). *(Line corrected from
:37 — that is the first line of the two-line comment block. Its twin lives at `config/routes.rb:35`,
which also still says S6. Neither is a vault citation.)*

---

## 3. The open design questions

### Q1 — Build order. What ships first? ~~*(the session's headline question)*~~

> [!danger] **Corrected 2026-08-25 (Session 36) — the premise below was false.**
> The brief claimed the intake flow "creates visits the app cannot list." **It does not**, and the
> claim was verified false against code at `2c5816f`:
>
> - **`GET /workshop` (`workshops#show`) *is* the board.** It loads
>   `Job.for_current_workshop.active` plus the done-awaiting-delivery group and renders
>   `app/views/jobs/_board_row.html.slim` for each.
> - **`Job.active` includes `unassigned`** (`app/models/job.rb:17`), and `CreateIntake` defaults to
>   `jobs: [{}]` — one unassigned repair. So **every new intake surfaces on the board immediately.**
> - The row links to `jobs#show`, which carries *"← This car's visit"*; `Job` delegates
>   `vehicle, :customer` to `intake`. The visit is two clicks from the board, not URL-only.
> - Separately, [[Intake flow]] §1c makes the plate field its own lookup for a car already in house.
>
> **Consequence:** build the intake front door as planned. **Do not pull Sprint 6 forward, and do
> not ship a stopgap list.** The real gap was never a list — it was the front door.
>
> **No dated footnote is owed against Session 32.** This finding *supports* the S5-before-S6
> deferral rather than reversing it, so CLAUDE.md's contradiction rule does not fire.
>
> **How the error happened, so it doesn't recur:** it was read off `bin/rails routes` — there is no
> `/intakes` index and no `/jobs` index — without then checking whether an *existing* screen
> already rendered the list. A missing index route is not the same fact as a missing list.

~~The finding above is the tension: **the intake flow creates visits the app cannot list.** An SA
registers a car, lands on the visit page, navigates away — and that visit is unreachable without
the URL. The board that would list them is Sprint 6, which Session 32 deliberately deferred
*behind* Sprint 5.~~

~~Options to weigh (not exhaustive): ship a minimal "in house today" list alongside intake; move
S6 ahead of S5 after all; accept landing on the visit page as sufficient for a first cut and
list later.~~ **All three are moot — the board already lists them.**

### Q2 — Screen decomposition
Wizard (one question per screen), one page with progressive reveal, or a page re-rendering per
answer? [[Intake flow]] is a branching tree; nothing says how many screens that is.
Bound by: ~90% of visits are §1a (plate hit, phone matches) and must clear in **under a minute**
([[User stories]]); keyboard-only on PC (UI law 10 — tab/Enter, autofocus the plate field); PC
buys density not size (law 9); exactly one primary action per screen (law 3).

### Q3 — The mismatch confirm as a screen
Turn [[Deferred design]]'s two-branch prose into a real surface. Note law 2: **a mismatch is not
an error** — status colours are reserved words, so no red. [[Intake flow]] principle 3:
"mismatches are decisions, not errors."

### Q4 — The silent-compare constraint
[[Intake flow]] §1: the SA asks *"can I have your phone number to check?"* and compares
**silently** — the file's number is never read aloud or shown, because that leaks the
file-holder's phone to whoever is holding the keys. A screen that simply renders the customer
card violates this. How does the UI let the SA compare without displaying?

### Q5 — Complaint and inspection type
Where do they sit in the flow, given the type can never be changed afterwards? Same screen as
creation, or a deliberate step?

### Q6 — Where the front door lives
No nav bar exists. Workshop dashboard button, a persistent nav item, or something else — and
this interacts with Q1 (if a list ships, the list is probably the home for "New intake").

---

## 4. Out of scope

- **Writing any code.** Including exploratory branches.
- **The jobsheet fill screen** (39 items) — its own design problem, different user and device,
  waiting on the three flagged catalog content calls in `car_routine_inspection.rb`.
- **ETA / `promised_ready_at`** (S5.6), **vehicle history** (S5.7), **add-a-repair to an open
  intake**, **owner notification channel** ([[Open questions]] — v1 is a manual copy-paste
  message), the **owner token page** (Sprint 7).
- `brought_in_by` — an accepted v1 gap ([[Intake flow]] §1b-i). Do not propose the column.

---

## 5. What this session should produce

1. A **build order** answering Q1, with reasoning — and a dated footnote if it moves against
   Session 32's S5/S6 sequencing.
2. A **screen design** answering Q2–Q6: how many screens, what each shows, what each asks, how
   the forks are reached, where the flow starts and ends.
3. Written **into the vault** — this note updated in place, or a new concept note, plus a
   [[Sprint plan]] task breakdown for whatever the build order says comes first.
4. An **ADR** only if a call turns out to be structural. Design decisions that aren't structural
   belong in the notes, not a numbered ADR.
5. Anything deliberately left open, named as open.

---

## 6. Read before designing

`CLAUDE.md` · [[Intake flow]] (the behaviour spec — read in full, the forks are easy to
under-read) · [[Deferred design]] §the job↔vehicle customer-match entry (the two-branch confirm)
· [[Design system]] §UI laws · [[User stories]] §Design constraints · [[Screen map]] ·
[[Screen flow]] · [[Sprint plan]] Sprint 5 · [[Data model]]

## Related
[[Intake flow]] · [[Inspection jobsheet — design brief]] (the pattern this follows) ·
[[Sprint plan]] S5.5 · [[Screen map]] · [[Screen flow]] · [[Design laws]] · [[Design system]] ·
[[ADR-012 Intake-Job two-level aggregate]] · [[ADR-013 The door decomposed]] ·
[[ADR-015 Jobsheet answers are rows against a frozen question set]]
