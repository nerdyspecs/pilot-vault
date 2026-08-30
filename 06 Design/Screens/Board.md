---
type: screen-decision
screen: Board
route: GET /workshop → workshops#show
updated: 2026-08-27 (Session 38 — first cut built; everything under §First cut is TBD)
---
# Board

The landing surface after sign-in: **one board for everyone**, answering *"what needs a move, and
whose?"* What the screen contains today, and its gates, are in [[Screen map]] §3 — this note is only
the reasoning. Conventions: [[Screen decisions]].

> [!warning] A first cut exists, and **all of it is TBD** *(2026-08-27, branch `s5a-sass`,
> uncommitted)*
> The screen was re-cut against the structure below and **renders green**, but nothing in
> §First cut is ruled — it is a shape to react to, not a decision. The rulings in §Settled are
> separate and do stand.

## Settled

**One board for everyone, not a dashboard per role** *(2026-08-27)*. Viewpoints are **scopes** on it,
never separate screens. Decided on the schema, not on cost: `WorkshopStaff#titles` returns a *list*
and leaves precedence to the caller, so a dashboard per role would need a `role → screen` function
the edge-collapse schema deliberately refuses to hold — and the towkay who owns the shop and works
the floor is the normal case, not an edge case. Full ruling and the two supporting arguments:
[[Design system]] §L1.

**Its content is safe for every role** *(2026-08-27, builder)*. Nothing on the board is hidden per
role. Data is uniform; **actions stay gated**. What is not safe for everyone does not become a
per-role panel here — it becomes its own screen. Rule and its four clauses: [[Design system]] §L1.

**It is called the Board** *(2026-08-27)*. The whiteboard is the artefact Knot replaces, and the code
already said it (`jobs/_board_row`). *Dashboard* was retired — generic, unsaid at a workshop, and it
had become the name of the design just rejected. [[Screen flow]]'s two "Dashboard" landings were
corrected the same day.

**Route stays `GET /workshop` → `workshops#show`** *(2026-08-27)*. See *Rejected* below — this was
argued to a move and back again.

**Structure is archetype 1, using the split section** *(2026-08-27)*: frame → page head → metric
section → **split (`minmax(0,1fr)` + `--rail`)**. The arrangement was reached by drawing it, but the
archetype and the `--rail` token already existed unused — the wireframe rediscovered a recorded
structure rather than inventing one. Archetype: [[Design system]] §Layout archetypes.

**Five information regions, all from data that exists**: a stage funnel (where cars pile up), the
car table (the bulk, quiet), and in the rail — needs-attention, blockers-by-type, crew load.
The funnel replaced a four-number strip because it shows *flow*, which is what the whiteboard's
columns already show.

**"Done — awaiting delivery" folds into needs-attention** *(2026-08-27)*. It is not a different kind
of thing: a finished car nobody has collected is a car waiting on a person. `WorkshopsController`'s
own comment already made this argument — *"a finished car nobody has collected is exactly what the
'waiting on the SA' pin exists to raise."*

**No primary action on the board; check-in starts from the vehicle** *(2026-08-27, builder)*. This
**reverses** the recorded L1 line that put the front door here, and **narrows** the page pattern's
"every screen, without exception… one primary action". Both footnoted in [[Design system]]. The gain
is structural rather than cosmetic: starting check-in from a found vehicle makes the search-first
rule *enforced* instead of advisory, which is precisely what the step-6 dedup note worries about.
The board becomes a monitoring surface with no create path.

**Search and filters live in the panel's list toolbar**, not the page head — the list toolbar is
already a named component carrying search left and the count right.

## First cut — built 2026-08-27 · **all TBD**

Built to see the shape, not to settle it. Renders green; no commit.

**Structure.** `page head → section stack (gap-3) → metric strip → split(main, rail)`. The only new
CSS in the whole build is `.split` — `grid-template-columns: minmax(0,1fr) var(--rail)`; every gap
and padding is a Bootstrap utility in the template, none in the stylesheet.

**The row is the car.** A new `intakes/_board_row` replaced the job row — which means **there is no
single stage to show**, so a car renders **one badge per repair**, reusing `jobs/_stage_badge` and
`jobs/_waiting_pin` untouched. A car showing `Done` beside `In progress` is the two-level aggregate
becoming visible for the first time. **This pulled the intake regroup forward** out of Sprint 6, on
cost-of-late-change: styling a job row that is about to become a car row is work done twice.

**What the screen carries.** Strip: cars in shop · unassigned · assigned · in progress · ready for
collection. Main: every open visit. Rail: needs-attention (*ready, not collected* / *nobody has
picked this up*) then crew load.

**Deliberately not built, each for a reason:**

| Not built | Why |
|---|---|
| **Ageing / stalled highlight** | The bands and the overnight-hours question are undecided. Building the query before the threshold is deciding half a feature in the dark |
| **"Needs *my* attention"** | It reverses the same-day shop-wide content rule. `Acknowledgeable#awaiting_from?` already exists if it becomes a *mine-first sort* rather than a filter — which is the form the content rule permits |
| **Blockers by type** | Dropped by the builder; it overlapped needs-attention anyway |
| **Stage funnel as a funnel** | Shipped as five plain tiles. Proportional widths would show the pile-up better but want colour, and colour is spoken for |

**Two model methods were written and backed out** — recorded so they are not re-added by reflex.
`Job.crew_load` returned `{ WorkshopStaff => count }`, so its receiver was wrong: the subject of the
answer is the person, not the job. `Job.last_moved_at_by_job` turned a per-job property into a
class-level hash purely to dodge an N+1 — a performance concern deforming the model, and the
stuttering name said so. Both had copied the *form* of `pending_acknowledgements_by_job` without its
justification (that one unions three tables and has no single-model home). Crew load now shapes
already-loaded rows in the controller; ageing waits.

**One model change did land:** `Intake#ready?` uses `jobs.any?` rather than `jobs.exists?`, because
`exists?` queries even on a preloaded association and the board calls it once per car. Behaviour is
identical and the existing assertions cover it. Measured: the whole board is **9 queries at 6 cars
and 9 at 18** — flat.

## Open

- **Can an attention card carry all four questions at `--rail` width?** 320px leaves ~280px of card.
  If *whose move* and *how long* do not fit, the rail is the wrong home and attention returns to the
  main column. This is Bay's recorded "not to be copied" failure, live.
- **A quiet day renders an empty rail.** No blockers, nobody stuck, crew idle — three empty cards on
  the shop's best day. Law 8 work, and more than one sentence.
- **Blockers-by-type and needs-attention read the same rows**, aggregated differently — items vs
  pattern. Defensible, but decide it rather than ship both by accident.
- **The funnel: equal segments or proportional?** Proportional shows the pile-up far better but
  wants colour, and colour is spoken for.
- **A dense table row has no token.** `--row-min` is `5rem`, right for two-line attention rows,
  too tall for the car table. Same shape as the seventh dimension already recorded for
  `--sidebar-collapsed`.
- **The waiting pin renders an email**, because `User` carries no name. The only personal datum on
  the board, and visibly the ugliest thing on the page once drawn.
- **The technician's mine-first scope** — deferred with the floor posture, by ruling not by finding.
- **`jobs/_board_row` is now dead** — nothing references it since the row became the car.
- **Cancelled repairs still render a badge** on a live car. Honest, or noise — undecided.
- **Crew load prints an email**, the third place this screen has been forced to. The case for a
  name on `WorkshopStaff` is now made three times over.

## Rejected, with reasons

| Rejected | Why |
|---|---|
| **A dashboard per role** | Roles here are plural and simultaneous; law #3 licenses a new *scope*, and UI law 7 forbids a second screen set. The reference's role matrix is adopted as gating + a role-varying primary action instead |
| **"Dashboard" as the name** | Generic, unsaid at a workshop, and it names the design just rejected |
| **`/board` + a `BoardController`** | A controller earns its keep through actions and the board has none; the queries are model scopes under law #9, so `workshops#show` stays thin. Splitting it would be rewriting working code into a better pattern. **Tripwire:** revisit if the board grows its own verbs (acknowledging or dismissing inline) or needs board-only private methods |
| **`/workshop/board`** | The path has no id in it, so it implies a hierarchy the app does not have — every other workshop-scoped screen resolves the tenant from the session. It also argues its own case weakest: if the board *is* the workshop's front page, `/workshop` says that more directly than a sub-path does |
| **`intakes#index` as the board** | The board reads jobs, intakes, blockers, transitions and staff in one page — it is not the index of any one resource. `/intakes` should stay free for a real visit history. *(The stale "board index lands here" comment in `routes.rb` predates this.)* |
| **Three stacked full-width bands** *(the first mock)* | `col-lg-6` in a new costume — a stretched phone layout at 1120px, with air inside every band. Law 9 says PC buys density |
| **A per-technician workload panel** | "Who is on what" is a column in the table and a filter (S6.2). As a panel it becomes the scoreboard S6.2b forbids |
| **Money, bookings, tickets, meetings** | No columns exist for any of them, and pricing is the standing example of what can never be board content |

## Related
- [[Design system]] §L1 — the ruling, the content rule, the name · [[Screen map]] §3 — what exists
- [[UI rollout]] S5A.9 — the build spec · [[Sprint plan]] — status · [[worklog]] Session 38
