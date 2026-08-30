---
type: reference
module: M1
updated: 2026-08-27 (Session 38 — the landing screen is named the Board, not Dashboard; prior: 2026-08-25 Session 36 — flows 6 and 7 re-shaped: vehicles get their own create screen, intake ships happy path only; plan of record [[Design system]]; prior: flow 7 chipped out)
---
# Screen flow

The **flows** — task journeys walked in order, each with the screen it lives on, the API it
hits, its components `(C)`, actions `(A)`, and what each action **triggers next**. This is the
"how a workshop lives through the app" view; per-screen component detail is in [[Screen map]],
and the feature each flow realises is in [[Features overview]].

Cross-cutting flows (Monitor, Owner status) are parked for now — this covers **Setup → Records
→ Daily loop**.

**Legend:** ✅ built · ⚠️ endpoint exists, no UI yet · ❌ not built · 🔒 Devise (exempt from redesign)

*(2026-08-27: the two **Dashboard** landings above now read **Board** — one screen had four names
and this note held the retired one. Ruling and reasoning: [[Design system]] §L1.)*

## Setup
| # | Flow | Screen | Api | Components (C) | Actions (A) | → triggers |
|-|-|-|-|-|-|-|
| 1 | Sign up | Sign up 🔒 | `GET /users/sign_up` · `POST /users` | email + password | Create account | → **Landing** → *Workshop reg* / *Team accept* |
| 2 | Workshop registration | Create workshop ✅ | `GET /workshop/new` · `POST /workshop` | workshop name | Create workshop | → **Board** (now owner) → *Team invitation* |
| 3 | Team invitation | Add crew ✅ | `GET /invitations/new` · `POST /invitations` | email · role | Add to crew `[mgmt]` · Invite again `…/refire` · Remove `DELETE` | → **Crew list** → *Team accept* |
| 4 | Team accept | Landing (invite card) ✅ | `POST /invitations/:id/accept` · `…/decline` | invite card (workshop · role) | Accept · Decline | Accept → **Board** (now crew) · Decline → *stays* |

## Records
| # | Flow | Screen | Api | Components (C) | Actions (A) | → triggers |
|-|-|-|-|-|-|-|
| 5 | Customer create | New customer ✅ | `GET /customers/new` · `POST /customers` | kind · name · phone · contact · address | Add customer `[counter]` | → **Customer show** → *Add vehicle* |
| 6 | Add vehicle | Add vehicle ❌ | *(no route, no controller)* — today born only inside *New intake* | registration # | Add vehicle `[counter]` | → **Customer show** |

## Daily loop
| # | Flow | Screen | Api | Components (C) | Actions (A) | → triggers |
|-|-|-|-|-|-|-|
| 7 | New intake | New intake ⚠️ | `GET /intakes/new` ❌ · `POST /intakes` | plate field · complaint · inspection type | Look up `[counter]` | found → **Intake show** · in house → *that visit* · not found → **Add vehicle** |
| 8 | Add repair | on Intake show ⚠️ | `POST /intakes/:intake_id/jobs` | add-repair form *(no UI yet)* | Add repair `[counter]` | → **Job show** |
| 9 | Assign technician | Job show ✅ | `POST /jobs/:id/technician` · `DELETE` | technician picker | Assign · Remove `[counter]` | → *stays* (unassigned→assigned) |
| 10 | Work the repair | Job show ✅ | `…/start_work` · `/mark_done` · `/send_back` · `/cancel` · `/blockers` | stage · technician · blockers · timeline | Start · Mark done `[crew]` · Send back · Cancel `[counter]` · Raise/clear blocker `[role]` | moves → *stays* · blocker → **Blocker show** |
| 11 | Deliver / cancel | Intake show ✅ | `POST /intakes/:id/deliver` · `/cancel` · `/blockers` | header · repairs · holds | Deliver · Cancel intake `[counter]` · Place/release hold `[role]` | Deliver/Cancel → *stays* · hold → **Blocker show** |

## Issues surfaced
- ~~**Job create should nest under Intake.**~~ **Fixed 2026-08-16 (`ed2595c`).** Was flat
  `POST /jobs` with `intake_id` in the request body; now `POST /intakes/:intake_id/jobs`
  (helper `intake_jobs_path`), matching every other child. Member verbs stay flat at
  `/jobs/:id/...` — once a job exists its own id addresses it, and deeper nesting would push the
  blocker routes to four levels while letting the URL carry an `intake_id` that can disagree with
  the job's real parent. The security boundary here is the **workshop** scope, not the intake, so
  nesting adds no safety.
- **The front door is the S5 hole — and it's provable from the route side.** A route-orphan check
  (`bin/route-orphans`) shows **exactly two endpoints in the whole app with no UI caller**:
  `POST /intakes` (`intakes#create`) and `POST /intakes/:intake_id/jobs` (`jobs#create`). Both
  creates are implemented and tested; nothing in the view layer can reach them. Every other
  endpoint has at least one caller. New intake has no `GET /intakes/new` form, Add repair has no
  UI on Intake show, and Add vehicle has no standalone screen (a vehicle is only born when you
  type an unknown plate during New intake). Setup + Records are otherwise fully built; the daily
  loop is built from Assign-technician onward.
  *(2026-08-25: flow 7 — the counter half of the front door — is **chipped out** to
  [[Intake UI — design brief]], Sprint plan S5.5. Flow 8 (add a repair to an open intake) and the
  jobsheet fill screen are **not** in that chip.)*
- **Flows 6 and 7 re-shaped 2026-08-25 (Session 36).** Ruled with the builder: a vehicle gets its
  own create screen under a customer (flow 6, still unbuilt — there is no `VehiclesController`), and
  flow 7 therefore ships **happy path only** — [[Intake flow]] §1a plus §1c, with the §1b mismatch
  forks and the §2a/§2b dedup tree deferred. Flow 7's not-found outcome is a *routed* dead end into
  flow 6, not a refusal (UI law 8). The dedup consequence this creates, and the search-first guard
  that answers it, are recorded in [[Design system]] L3. Build order for both: [[Sprint plan]]
  S5.5a–i.
- **Considered and declined: renaming the blocker-type catalog route.** `/blockers` (catalog)
  sits beside `/intakes/:id/blockers` and `/jobs/:id/blockers` (applied). Renaming the catalog to
  `/blocker_types` was tried and reverted: the nested controllers already disambiguate, this
  codebase already accepts label≠route (the crew page is "Crew" at `/staff`), and renaming the
  route without the `Blocker` model would trade one asymmetry for another. If it ever matters, do
  the full model rename instead.

## How to keep this true
Re-sync from `bin/rails routes` whenever routes change — same per-sprint ritual as [[Screen map]].
This note tracks *flows* (task journeys), so it changes when a step is added, removed, or
re-pointed, or when an endpoint's shape changes (like the job-nesting fix above).

## Related
[[Screen map]] · [[Features overview]] · [[Overview]] · [[Roadmap]]
