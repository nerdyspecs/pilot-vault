---
type: reference
module: M1
updated: 2026-08-25 (two stale entries corrected against code — the "Add job" 500 and the two pre-ADR-012 templates were already fixed; front door chipped out; prior: 2026-08-17 Session 32 — board §3 regroup deferred to S6, ADR-012 citation)
---
# Screen map

A screen-first inventory of Knot's UI surface — every page you can land on, and under each
one every button/link/form, the request it fires, who may use it, and where it lands. Built
for a front-end designer to read, and for us to catch drift when routes change.

**How to read the statuses** (every screen and every function carries one):

- **built** — exists in code today, works.
- **partial** — exists but incomplete or mid-reshape; the note says how.
- **not-built** — should exist (design intent) but has no page yet.
- **dead** — a control that points at a route that doesn't exist; it errors when clicked or
  when its page renders. Called out loudly because it's a live bug, not a hole.

**Who's who** (the role vocabulary used below):

- **Owner** — the person who created the workshop. Passes every gate (full access).
- **Manager** — a staff member holding the *workshop manager* role. Also full access.
- **Counter staff** — owner, manager, *or* service advisor. The front-desk gate; most
  customer/visit actions require it. (A *parts advisor* is a role but is **not** counter.)
- **Technician** — a staff member assigned as crew on a specific repair. Can only move
  repairs they're on.
- Any signed-in staff member of the workshop reaches the board and the visit/repair pages;
  the *actions* on those pages are then gated per the rows below.

> *Notes for us (skip if you're the designer): gates are verified against
> `app/services/permissions.rb`, `app/helpers/permissions_helper.rb`, and each controller's
> `before_action` table. "Counter" = `Permissions.counter?` (owner ∨ service_advisor ∨
> workshop_manager). "Manager" gate = `require_manager!` (owner ∨ workshop_manager). Colours,
> spacing, and the status palette are **not** defined here — see [[Visual theme]]. The
> reshape from a job-grouped board to an Intake-grouped board is ADR-012/ADR-013; models are
> [[Intake]] / [[Job]].*

---

## 1. Home / personal landing — `GET /` → `home#index`

**Purpose** — The first thing you see. Signed out: the marketing page. Signed in: your
personal hub — pending invitations, and a way into your workshop(s).

**Who reaches it** — Everyone (public). Content branches on whether you're signed in.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Open your workshop" (signed out) | `GET /users/sign_up` | — | Public | Sign-up page | built |
| "Accept" (invitation card) | `POST /invitations/:id/accept` | `:id` | The invited person | The board, now in that workshop | built |
| "Decline" (invitation card) | `POST /invitations/:id/decline` | `:id` | The invited person | Back to `/` | built |
| "Create your workshop" (no workshops) | `GET /workshop/new` | — | Signed-in | New-workshop form | built |
| Workshop name button (your workshops) | `POST /workshop/select/:id` | `:id` | Signed-in member | The board for that workshop | built |

**Auto-route** — If you belong to exactly one workshop, have no pending invitations, and
haven't selected a workshop yet, `/` sends you straight to the board.

**States to handle** — *No workshops yet* (create-workshop card) · *pending invitation*
(highlighted card, must be seen — the auto-route won't skip it) · signed-out hero. Colours
per [[Visual theme]].

**Data shown** — Your accessible workshops; your fired (awaiting-you) invitations.

---

## 2. Create workshop — `GET /workshop/new` → `workshops#new`

**Purpose** — Name a new workshop; you become its owner.

**Who reaches it** — Any signed-in user.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Create workshop" | `POST /workshop` | `workshop[name]` | Any signed-in user | The board (you're now owner) | built |

**States to handle** — *Validation error* (blank/dup name) re-renders the form with the
message inline.

**Data shown** — Empty form.

---

## 3. The board — `GET /workshop` → `workshops#show`  ⭐

**Purpose** — The daily case. Every active repair, plus finished repairs still awaiting
delivery, at a glance. This is the app's home screen once you're in a workshop.

**Who reaches it** — Any active staff member (all roles).

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Customers" | `GET /customers` | — | Counter staff only | Customers list | built |
| "Crew" | `GET /staff` | — | Owner/manager only | Crew page | built |
| "Blocker types" | `GET /blockers` | — | Owner/manager only | Blocker-type catalog | built |
| Board row (a repair) | `GET /jobs/:id` | `:id` | Any staff | That repair's page | built |
| **"New job"** | `GET new_job_path` | — | Counter staff | — | **dead** |

> ⚠ **"New job" is dead.** It points at a route that doesn't exist. Because it renders
> inside the counter-only block, the board page itself **500s for counter staff** right now.
> Its intended destination is the (unbuilt) start-a-visit flow — see *Screens that should
> exist* below. Non-counter staff don't see the button, so their board renders fine.

**States to handle** — *Empty* ("No active jobs" with a prompt) · *active list* · *"Done —
awaiting delivery"* group (shown only when finished-but-uncollected repairs exist) · each row
can carry an *aging / waiting* pin. Aging thresholds and pin colours per [[Visual theme]].

**Data shown** — Active repairs and done-awaiting-delivery repairs (with their car + customer
via the visit), plus unacknowledged handoffs per repair.

> *Partial:* the board is still grouped by individual repair. ADR-012 reshapes it to group by
> **Intake** (the car), with the done-awaiting-delivery group keyed on `intake.ready?`. That
> regroup is **deferred (S6)** — the current repair-grouping stands until then.

---

## 4. Add crew (invite) — `GET /invitations/new` → `invitations#new`

**Purpose** — Bring a person onto the crew by email, with a starting role.

**Who reaches it** — Owner/manager only.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Add to crew" | `POST /invitations` | `invitation[email]`, `invitation[role]` | Owner/manager | Crew page | built |
| "Back to crew list" | `GET /staff` | — | Owner/manager | Crew page | built |

**States to handle** — *Validation errors* render inline. On submit, two outcomes (both land
on the crew page with a flash): account exists → invitation *fired*, awaiting their accept;
no account → *pending* until they sign up.

**Data shown** — Empty form; role picker (the four staff roles).

---

## 5. Crew — `GET /staff` → `workshop_staff#index`

**Purpose** — Everyone on the workshop and their roles; add/end roles; manage pending invites.

**Who reaches it** — Any active staff member. (The role-management controls only render for
owner/manager.)

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Add crew" | `GET /invitations/new` | — | Owner/manager | Invite form | built |
| "Add role" (per member) | `POST /staff/:workshop_staff_id/roles` | `workshop_staff_id`, `workshop_staff_role[role]` | Owner/manager | Back to crew | built |
| "End {role}" (per role badge) | `DELETE /staff/:workshop_staff_id/roles/:id` | `workshop_staff_id`, `:id` | Owner/manager | Back to crew | built |
| "Invite again" (pending row) | `POST /invitations/:id/refire` | `:id` | Owner/manager | Back to crew | built |
| "Remove" (pending row) | `DELETE /invitations/:id` | `:id` | Owner/manager | Back to crew | built |

**States to handle** — *No crew yet* · *Active* table · *Pending* table (only when invites
exist), each pending row labelled *awaiting acceptance* / *declined* / *no account yet*.

**Data shown** — Active staff with roles (owner badge + role badges); non-accepted invitations.

---

## 6. Blocker-type catalog — `GET /blockers` → `blockers#index`

**Purpose** — The workshop's library of blocker types — for each: its label, who raises it,
who clears it, and which stage it blocks. There is no page-per-blocker; add + edit happen
inline here.

**Who reaches it** — Owner/manager only.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Back" | `GET /workshop` | — | Owner/manager | The board | built |
| "Save" (per row) | `PATCH /blockers/:id` | `:id`, `blocker[raised_by_role, cleared_by_role, blocks]` | Owner/manager | Back to catalog | built |
| "Add" (new type) | `POST /blockers` | `blocker[label, raised_by_role, cleared_by_role, blocks]` | Owner/manager | Back to catalog | built |

> No delete: once a type has been used it stays in the catalog forever (only add + edit).

**States to handle** — *Empty catalog* · *catalog table* (each row an inline edit form) ·
*validation error* flashes at the top. `blocks` options are the forward stages
(in progress / done / delivered).

**Data shown** — All blocker types for the workshop.

---

## 7. The visit (Intake) — `GET /intakes/:id` → `intakes#show`  ⭐

**Purpose** — One car's visit: its list of repairs, the whole-car moves (deliver / cancel),
and holds that stop the car leaving. The car's page.

**Who reaches it** — Any active staff member. (Whole-car moves are counter-only, gated below.)

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Repair #N" (per repair) | `GET /jobs/:id` | `:id` | Any staff | That repair's page | built |
| "Deliver to customer" | `POST /intakes/:id/deliver` | `:id` | Counter staff | Back to the visit | built |
| "Cancel this visit" | `POST /intakes/:id/cancel` | `:id` | Counter staff | Back to the visit | built |
| "Hold" (place a hold) | `POST /intakes/:intake_id/blockers` | `intake_id`, `blocker_id`, `note` | Whoever holds the hold's *raise* role | Back to the visit | built |
| "Release" (on a hold) | `POST /intakes/:intake_id/blockers/:id/resolve` | `intake_id`, `:id`, `note` | Whoever holds the *clear* role | Back to the visit | built |
| "Add note" (on a hold) | `POST /intakes/:intake_id/blockers/:id/note` | `intake_id`, `:id`, `note` | Anyone on the hold's raise/clear side | Back to the visit | built |

> "Deliver to customer" only appears once every repair is terminal with at least one done
> (`ready?`). The hold picker only lists holds this user is allowed to raise.

**States to handle** — Header badge shows *Ready for collection* (open + ready), else the
stored status (open→*in progress*, *delivered*, *cancelled*) — status colours per
[[Visual theme]]. *No holds* ("Nothing holding this car") vs an active-holds list. There is
**no** "add a repair to this visit" control yet (see *Screens that should exist*).

**Data shown** — The vehicle, the customer, the visit's repairs (each with stage + assigned
tech), and the visit's active holds.

---

## 8. The repair (Job) — `GET /jobs/:id` → `jobs#show`  ⭐

**Purpose** — One repair on a visit: its technician, the floor moves (start / done / send
back / cancel), its blockers, and its history.

**Who reaches it** — Any active staff member. Actions split: floor moves = the repair's crew;
assign/remove/send-back/cancel = counter.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "← This car's visit" | `GET /intakes/:id` | `:id` | Any staff | The visit page | built |
| "Assign" (technician) | `POST /jobs/:job_id/technician` | `job_id`, `staff_role_id` | Counter staff | Back to the repair | built |
| "Remove technician" | `DELETE /jobs/:job_id/technician` | `job_id` | Counter staff | Back to the repair | built |
| "Start work" | `POST /jobs/:id/start_work` | `:id` | This repair's crew | Back to the repair | built |
| "Mark done" | `POST /jobs/:id/mark_done` | `:id` | This repair's crew | Back to the repair | built |
| "Send back to assigned" | `POST /jobs/:id/send_back` | `:id` | Counter staff | Back to the repair | built |
| "Cancel job" | `POST /jobs/:id/cancel` | `:id` | Counter staff | Back to the repair | built |
| "Ready — deliver on the visit page →" | `GET /intakes/:id` | `:id` | Counter staff | The visit page | built |
| "Add crew" (no techs yet) | `GET /invitations/new` | — | Counter staff | Invite form | built |
| "Add a role to yourself" | `GET /staff` | — | Counter staff | Crew page | built |
| "Raise" (a blocker) | `POST /jobs/:job_id/blockers` | `job_id`, `blocker_id`, `note` | Whoever holds the blocker's *raise* role | Back to the repair | built |
| "Resolve" (a blocker) | `POST /jobs/:job_id/blockers/:id/resolve` | `job_id`, `:id`, `note` | Whoever holds the *clear* role | Back to the repair | built |
| "Add note" (a blocker) | `POST /jobs/:job_id/blockers/:id/note` | `job_id`, `:id`, `note` | Anyone on the blocker's raise/clear side | Back to the repair | built |

> Which floor buttons appear depends on the repair's stage (assigned → Start; in progress →
> Mark done / Send back) and on crew vs counter. Delivery is never here — a finished repair
> points you to the visit page, because delivering is the whole car's move.

**States to handle** — Technician present vs *no technician yet* vs *no technicians in the
workshop at all* (offers Add crew / Add role). *No blockers* vs active-blocker list. History
is always shown (stage/technician/blocker events, newest handling per [[Visual theme]]).

**Data shown** — The repair's stage, car, customer, technician, active blockers, the
technician-role picker, the raiseable-blocker picker, and the full event timeline.

---

## 9. Customers — `GET /customers` → `customers#index`

**Purpose** — Find a customer; start a new one.

**Who reaches it** — Counter staff only.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "New customer" | `GET /customers/new` | — | Counter staff | New-customer form | built |
| Search | `GET /customers` | `q` | Counter staff | Filtered list | built |
| Customer row | `GET /customers/:id` | `:id` | Counter staff | Customer page | built |

**States to handle** — *No customers yet* vs a list; search filters name/phone.

**Data shown** — Customers (with vehicle counts).

---

## 10. New customer — `GET /customers/new` → `customers#new`

**Purpose** — Add a customer (person or company).

**Who reaches it** — Counter staff only.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Add customer" | `POST /customers` | `customer[kind, name, phone, email, address, contact_person]` | Counter staff | The new customer's page | built |

**States to handle** — *Validation errors* render inline. If reached carrying a
`registration_number` param, it shows a "registering {plate} under this customer" hint (a
future create-a-visit hand-off — the receiving flow isn't wired yet).

**Data shown** — Empty form; kind picker (person / company).

---

## 11. Maintain customer — `GET /customers/:id/edit` → `customers#edit`

**Purpose** — Edit an existing customer's details.

**Who reaches it** — Counter staff only.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Save" | `PATCH /customers/:id` | `:id`, `customer[kind, name, phone, email, address, contact_person]` | Counter staff | The customer's page | built |
| "Back to {name}" | `GET /customers/:id` | `:id` | Counter staff | The customer's page | built |

**States to handle** — *Validation errors* render inline.

**Data shown** — The customer's current details, pre-filled.

---

## 12. Customer — `GET /customers/:id` → `customers#show`

**Purpose** — One customer: their activity summary, their vehicles, and (intended) a way to
start a job for them.

**Who reaches it** — Counter staff only.

**Functions**

| Label | Fires | Params | Who | Lands | Status |
|---|---|---|---|---|---|
| "Maintain customer" | `GET /customers/:id/edit` | `:id` | Counter staff | Edit form | built |

> ⚠ **Corrected 2026-08-25 — this note was stale.** It described an "Add job" button pointing at
> the vanished `new_customer_job_path` and claimed the page **500s for every counter user**. That
> button was already **removed in code** with the aggregate split; the page renders fine. A
> code comment at `app/views/customers/show.html.slim:37` marks the spot where the start-a-visit
> link belongs once the front door exists — see [[Intake UI — design brief]].

**States to handle** — *Activity* summary tiles · *Vehicles* list (only when any exist).

**Data shown** — Customer name/kind/phone, activity counts (vehicles / jobs / active / last
visit), and the customer's vehicles.

---

## Screens that should exist but don't (the create-path hole)

The two dead buttons above both point here. **Every create action is screenless** — the
POST endpoints exist and work, but nothing renders a form to reach them:

| Intended screen | Would feed | Status | Notes |
|---|---|---|---|
| **Start a visit / "New intake"** (`GET /intakes/new`) | `POST /intakes` (`vehicle_id`) | **not-built** | The plate-first entry flow. **Chipped out 2026-08-25 — [[Intake UI — design brief]]** (Sprint plan S5.5). Target of the board's "New job". |
| **Add a repair to an intake** (`GET /intakes/:intake_id/jobs/new`) | `POST /intakes/:intake_id/jobs` | **not-built** | No way in the UI to add a second repair to an open intake yet; the intake page even notes the gap. *(2026-08-16: create nested under its intake — `ed2595c`.)* |
| **Customer status page** (public, by token) | — read-only — | **not-built** | Each visit carries a share token for an owner-facing "how's my car?" page. No route yet. |

> *Notes for us:* ~~two stale templates survive from before the Intake split~~ — **corrected
> 2026-08-25: both are gone.** `app/views/jobs/new.html.slim` and
> `app/views/customers/jobs/new.html.slim` were deleted with the aggregate split; there is
> nothing left to rebuild around Intake, and the front door starts from scratch. One vestige does
> survive: `app/views/customers/new.html.slim` still carries a `registration_number` hidden field
> that `customers#create` ignores — flagged in [[Intake UI — design brief]] §2.

---

## Not covered here

Devise-owned account screens (sign in / sign up / password reset, `/users/edit`) use the
separate focused auth layout and aren't part of the workshop UI — excluded on purpose until
restyled.

---

## How to keep this true

The **built** half of this note is a reflection of code. Re-sync it from `bin/rails routes`
(filter out devise/rails/active_storage/asset noise; split GET = screens from
POST/PATCH/DELETE = the buttons) **whenever routes change** — and re-verify each gate against
`app/services/permissions.rb` and the controllers' `before_action` tables. Make it a
per-sprint ritual, same as the test-review pass. The **intended** half (not-built rows) moves
only when design intent changes.
