---
type: log
updated: 2026-08-27 (Session 38 — one-board fork ruled; content rule; named the Board; screen-decision notes; deeper-layer hard rule; board first cut built, TBD)
---
# Worklog
Running narrative of discussions, decisions, and progress. **Newest session on top.**
Each session (~one work period) opens with a **summary**, then **topic entries** underneath.
Settled decisions get formalized as ADRs in [[Decisions]]; this log is the story that links them.

---

## 2026-08-27 · Session 38 — the L1 fork is ruled: one board, and it is called the Board

**Summary.** Documentation only, no app code. Ruled the open fork in [[Design system]] §L1 —
**one board for everyone vs a dashboard per role** — which had sat unruled *and* unscheduled
upstream of the shell work already in flight. One board wins. The screen's four competing names
collapse to **Board**. Added **S5A.3a** (sidebar contents) so the sidebar's top item has something
to build against. App branch `s5a-sass` unchanged at six commits; suite green at 154.

**Why one board, given that law #3 makes role dashboards cheap.** Cost was never the question, so
the argument had to come from somewhere else, and it came from the schema. `WorkshopStaff#titles`
returns a **list** and deliberately leaves precedence to the caller — the topbar already renders
`Owner · Technician`. A dashboard per role needs a function *role → screen*, so adopting it would
force us to invent a precedence order the edge-collapse schema refuses to hold, and the towkay who
owns the shop and works the floor is the normal case in a small workshop, not an edge case. The
laws point the same way: #3 licenses a new **scope**, and UI law 7 says a new viewpoint is a
recomposition, never a second screen set — so a per-role dashboard is the one reading of #3 that
UI law 7 forbids.

**The reference's role matrix is adopted, not rejected.** Knot's nav is *already* role-scoped, by
gating rather than by forking — `workshops#show` gates Customers behind counter staff, Crew and
Blocker types behind owner-or-manager. So the matrix lands as gated nav items, plus the screen's
one primary action chosen for the viewer's role, which UI law 3 already licenses in its own
wording. Only the extra screens are dropped.

**The technician's cost, named so it cannot be quietly banked.** Under one shared board a
technician must find themselves in a shop-wide list — a genuine glance-test failure and a genuine
miss on the attention-item test's *who owns the next action*. That is **deferred with the floor
posture, by ruling and not by finding**, and when the floor posture is drawn the fix is a
mine-first **scope on this board**, never a technician dashboard.

**The Board's content rule, added by the builder once the fork was settled.** *Everything on the
board is safe for every role, so nothing on it is hidden per role.* It sharpens the ruling rather
than qualifying it, and it needed two clarifications to be checkable: **data is uniform, actions
stay gated** (otherwise "all roles see it" gets read as "all roles do it", which would contradict
both `PermissionsHelper` and the role-varying primary action ruled above), and **a scope may
reorder, never conceal** (so the deferred mine-first technician scope stays inside the rule instead
of becoming its first exception). Its real force is the second clause: what is *not* safe for
everyone does not become a per-role panel here, it becomes its own screen — which is exactly the
shape Sprint 8 already plans for attribution and health reporting, so the rule predicts existing
structure rather than fighting it. The cost was accepted out loud: pricing, margin or per-person
throughput can never live on this screen. Checked against code — after S5A.3a moves the three
outline buttons into the sidebar, `workshops#show` has **zero** role-conditional content.

**The board was built, and the row became the car.** The session moved from ruling to Slim: the
layout gained one owned element (`main.px-5.py-4` — it cannot sit on `.app-main`, which holds the
sticky topbar and its full-width rule), the page head went in **inline** rather than as a partial on
the builder's call (canonical markup moved into [[UI rollout]] instead, so duplicated markup still
has a single source), and `workshops#show` was rewritten around **intakes**. That last part pulled
S6.1a partly forward, on the builder's own cost-of-late-change criterion: styling a job row about to
become a car row is work done twice. Consequence worth seeing — a car has no single stage, so the
row renders **one badge per repair**, and the two-level aggregate becomes visible on screen for the
first time. Measured flat at **9 queries for 6 cars and 9 for 18**. Everything is marked **TBD** in
[[Board]] — a shape to react to, not a decision.

**Two model methods were written, rejected, and recorded as rejected.** `Job.crew_load` and
`Job.last_moved_at_by_job` both copied the *form* of `pending_acknowledgements_by_job` without its
justification. The builder's read was right on both counts: `crew_load`'s receiver is wrong (the
answer is about the person, not the job), and `last_moved_at_by_job` turned a per-job property into
a class-level hash to dodge an N+1 — the stuttering name announcing the deformation. Backed out;
crew load now shapes loaded rows in the controller, ageing waits for S6.7 to set its thresholds.

**A hard rule earned the hard way: propose before writing into a deeper layer.** Asked to fix
`workshops#show`, the agent also added two `Job` class methods and changed `Intake#ready?` from
`jobs.exists?` to `jobs.any?` — all defensible (preloaded associations still query on `exists?`,
and the board calls it once per car, so it was 18 queries), all disclosed immediately after. The
builder's correction: **afterwards is the wrong order.** A view or controller is reviewable by
looking at the screen; a model method is shared API, and `ready?` in particular guards
`IntakeActions`' delivery refusal and is read by three views. Recorded in [[Agent guide]]
§Hard rules with the incident attached, in the same style as the dev-server rule from Session 37.

**A new note type: [[Screen decisions]].** The builder asked for a place to keep the discussion and
conclusions per screen, and the ask was well-timed — by then the Board's reasoning was spread across
[[Design system]], [[Screen flow]], [[Sprint plan]] and this log, findable only by date. One note
per screen now holds **why the screen is the way it is**: settled calls, open questions, and
**rejected alternatives with reasons** — the section that stops an option being re-argued. The
boundary is the whole point, so it is stated in the hub: rules stay in [[Design system]], spec in
[[UI rollout]], status in [[Sprint plan]], what-exists in [[Screen map]]. A screen note never
carries a tick, a file list, or a rule other screens must obey. Added to the build reading list in
[[Agent guide]], since a note nothing points at is a note nobody reads.

**Two footnotes the Board note forced.** The builder cut *Check in a vehicle*, *stage filter* and
*search plate* from the board's head — filters and search have a better recorded home anyway (the
list toolbar), but the front door **reverses** L1, which explicitly put it on the board and argued
law 1 against a nav item. Taking it off is the stronger call: check-in starting from a found vehicle
makes the search-first rule *enforced* rather than advisory, which is exactly the step-6 dedup trap.
That leaves the board with no primary action at all, which contradicts the page pattern's "every
screen, without exception" — recorded as a **narrowing**: one is a ceiling, not a quota, and a
monitoring screen may carry none.

**The route was argued to a move and back.** `/board` with its own `BoardController` was proposed on
a settings collision and a naming argument, then dropped on the builder's objection that a
controller earns its keep through *actions* and the board has none — the queries are model scopes
under law #9, so `workshops#show` stays thin, and splitting it would be rewriting working code into
a better pattern. `/workshop/board` was also weighed and rejected: no id in the path, so it implies
a hierarchy the app does not have. **Nothing changed**, which means the morning's ruling was right as
written and needed no footnote. A tripwire was recorded instead — revisit if the board grows its own
verbs.

**The name.** Four names for one screen — *Dashboard*, *the board*, `/workshop`, `workshops#show`.
It is the **Board**: the whiteboard is the artefact Knot replaces, the code already says it
(`_board_row`, `knot-board-desktop.html`), and it survives the ADR-012 regroup, since a board of
cars is still a board. *Dashboard* is retired for two reasons at once — nobody at a workshop says
it, and it is now the name of the design just ruled against. **The URL and the controller do not
move:** the name settled is a UI-surface fact, and renaming `workshop_path` across nine call sites
would buy nothing a user sees.

**S5A.3a, lettered rather than numbered.** The sidebar contents had no task at all: S5A.3 shipped
the frame empty when the builder scoped it that way, and [[UI rollout]] listed the contents under a
task that had already closed. Numbering it `S5A.4` would renumber nine live ids — the exact rot the
5↔6 swap caused — so it takes a letter, the same move Sprint 5A itself made.

**One open call narrowed by build rather than by argument.** The sidebar's dependency was recorded
as "Stimulus *plus* an icon source". Session 37 shipped `sidebar_controller.js`, so only the icon
source is open — and it is inseparable from the collapse, because UI law 6 (words over icons) means
a collapsed rail without icons gives up its labels for nothing.

---

## 2026-08-26 · Session 37 — Bootstrap moves to Sass, and the shell goes up empty

**Summary.** First build session of Sprint 5A. Coherence check clean. Ended with six app commits on
branch `s5a-sass` (unpushed) and the suite green at 154 throughout. The headline is a **reversal**:
Bootstrap now compiles from Sass source instead of being vendored, so the "no build step" rule is
dead — footnoted in [[Tech stack]] and [[Design system]], neither of which is edited in place.

**The reversal, and how it was decided.** The session opened intending S5A.1 against the vendored
file. The builder asked whether a Bootstrap gem existed; checking turned the question into a real
fork, and it was settled with a measurement rather than an argument. Bootstrap's blue is hardcoded
**29 times** in the compiled file, but **20 of those sit inside `--bs-*` variables** — reachable at
runtime. Only **6 are raw properties**, in three components, and we use exactly one:
`.form-check-input:checked`. So the vendored route's real gap was four lines, not a class of
problems — and the closed component inventory in [[UI rollout]] caps the unknown tail. On that
evidence the recommendation was *stay vendored*; the builder chose Sass anyway, partly to learn it.
**That call then paid off in a way the measurement had missed**: at the *source*, those six raw
values are `$primary` flowing through `$component-active-bg`, so setting one variable fixes the
exact gap that needed a hand-written rule. Measuring compiled output and reading source are not the
same thing.

**Two things Bootstrap does not let you theme from outside**, both found by reading its source
rather than guessing. `$enable-rounded: false` is **not sufficient** — `_root.scss` emits
`--bs-border-radius*` regardless of that flag and `.form-control` reads the property directly, so
all six radius values must be set. And `.btn-primary`/`.btn-outline-primary` **hardcode `#0d6efd`**;
they never read `--bs-primary-rgb`. `.form-control` likewise hardcodes `font-size: 1rem` and ignores
`--bs-body-font-size`.

**`--action-hover` ruled.** Bootstrap derives button states inside `button-variant()` from
percentage shade amounts, turning one action blue into three — `#2121B8` on hover, `#1F1FAE` on the
hover border, another again when active. None is a design-system colour. The builder ruled the
system colour wins as a whole, tweakable later, so `.btn-primary` overrides those four states back
to `--action-hover`. Links needed no override: `$link-hover-color` is a real variable.

**The shell went up empty, after going up too full.** S5A.3 was built with sidebar contents, the
twelve-screen law-9 dedent and all of S5A.2 folded in — because [[UI rollout]] lists those under the
same task. The builder had scoped the step to *"an empty topbar and sidebar"*. **The spec says where
a task ends up; the builder says what this step is, and the builder wins.** Reverted to the frame
alone, then the topbar was built as its own agreed step.

**Three layouts, because one was serving three things.** The gate is authentication only; the
signed-out marketing page became its own layout and its own design track (builder's call — marketing
and the system are separate flows); everything signed in gets the shell. The
signed-in-but-no-workshop state stays inside the shell rather than becoming a fourth archetype,
since [[Design system]] already rules workshop selection out of the gate.

**`WorkshopStaff#titles`.** Roles are genuinely plural — the towkay owns the shop and works the
floor — so it returns a list and leaves precedence to the caller. Owner is a boolean, not a role
enum value, so it is added separately. The topbar renders `Owner · Technician`, which the static
reference mock could never have shown.

**A rule earned the hard way.** Driving the in-app browser to check a layout put two stray
`WorkshopStaffRole` rows on a seeded persona — element refs went stale after the sidebar collapsed
and the clicks landed on the Crew screen — and twice left an orphaned server holding the port.
[[Agent guide]] §Hard rules now says **running the app is the builder's call, not the agent's**. The
two stray rows are left in place for now, by ruling.

**CLAUDE.md is no longer tracked.** It was slimmed first — reading lists removed (they duplicate
this vault and had already gone stale once), vault citations cut from 29 to one, machine-local facts
moved to an untracked `CLAUDE.local.md` — then committed and untracked. Recoverable at
`git show 31c36ef:CLAUDE.md`. **Consequence worth holding:** durable rules must live in the vault
now, because the file that auto-loads them has no history and dies with the machine.

**Open, carried forward.** `--sidebar-collapsed` is a **seventh** component dimension where
[[Design system]] records six — flagged in the stylesheet, not quietly adopted. `p-3` leaves only
24px of content in a 56px collapsed rail. Four names exist for one screen (Dashboard / the board /
`/workshop` / `workshops#show`) and the sidebar nav will have to pick one. And **one board vs a
dashboard per role** remains unruled, which is what actually decides where sign-in lands.

---

## 2026-08-25 · Session 36 — the UI design plan, a visual re-lock, and the design system on Bootstrap

**Summary.** Opened as the chipped-out intake-screen design session and **widened, on the builder's
call, into the UI design plan for the whole app**. Ran the coherence check first (clean: 0 broken
links, 0 citation leaks, no stranded branches), read the intake brief and behaviour spec, then
verified every load-bearing claim against code at `2c5816f` before writing anything. Output is
documentation only — **no app code was written**, per the chip's constraint. Plan of record is now
**[[Design system]]**.

**The brief's headline question rested on a false premise, and it was verified false.** The brief
claimed the intake flow "creates visits the app cannot list." It does not: `GET /workshop`
(`workshops#show`) **is** the board — it loads `Job.for_current_workshop.active` plus
done-awaiting-delivery and renders a row for each; `Job.active` includes `unassigned`
(`app/models/job.rb:17`) and `CreateIntake` defaults to `jobs: [{}]`, one unassigned repair, so
**every new intake surfaces on the board immediately**. The row links to `jobs#show`, which carries
"← This car's visit". So: no Sprint 6 pull-forward, no stopgap list, and — because the finding
*supports* Session 32's S5-before-S6 deferral rather than reversing it — **no dated footnote is
owed**. Corrected in place in [[Intake UI — design brief]] §2 and Q1, including *how* the error
happened: it was read off `bin/rails routes` (no `/intakes` or `/jobs` index) without then checking
whether an existing screen already rendered the list. **A missing index route is not a missing
list.**

**Why the session widened.** Designing the intake screens first would have invented a header and a
component set the other eleven screens then had to match. Looking at the whole view layer instead
turned up five problems that all sit *above* the screen level, none of them intake's:

1. **No navigation model.** The layout is a wordmark, workshop name, email, Sign out. Every screen
   invents its own way back — four different wordings for the same affordance. `/customers` is
   reachable from exactly one place.
2. **Three postures designed, one has screens.** Every built view is the desk layout.
   `app/views/jobs/show.html.slim` *is* the technician's screen and is a 106-line PC page; UI law 5
   has never been exercised by anything.
3. **UI law 3 broken where it matters** — `jobs/show` carries four solid `btn-primary`,
   `home/index` three. Those screens never decided their one next step.
4. **UI law 2 broken in the layout, so on all 27 templates** — flashes render as Bootstrap's
   `.alert-info` cyan and `.alert-warning` yellow, neither remapped, both wearing reserved status
   hues. (The badges are correct — `.stage-badge`, one source, sacred palette.)
5. **UI law 7 broken concretely** — `intakes/_blocker_item` and `jobs/_blocker_item` are twins
   differing by six lines of naming, out of only four shared partials in the app.

**Builder rulings (all recorded in [[Design system]]).** Clean slate is **design-level**, not a
teardown; the rebuild-or-keep verdict is taken **per screen, when reached** — deliberately not
recorded now. Scope is every surface including the marketing page, but ordered **bread-and-butter
first**. **Desk-first**; floor and owner postures planned later, with the `jobs/show` trap named so
it isn't mistaken for a finding that they're unnecessary. The plan lives inside [[Screen map]],
restructured into two halves — the risk of mixing intent with the half re-synced from
`bin/rails routes` was raised, accepted, and contained by a two-part split. *(Superseded later the
same session — see the consolidation entry below: the plan moved out of [[Screen map]] entirely.)*

**The ordering rule, which is the actual deliverable:** *do the things that change every screen
before drawing any screen.* Four cross-cutting fixes first (component inventory + merge the twins ·
flash recolour · the shell · the law-3 audit), then the spine in the order a real workshop lives
through it — because that is the only order in which the product can be walked end to end after
**every** step. **Only two genuinely new screens exist in the whole gap**: add-vehicle, and the
intake front door. Most of the twelve built screens are right and merely unshelled.

**The ruling that reshaped intake: customer → vehicle as ordinary CRUD, intake happy-path only.**
Vehicles get a real create screen (there is no `VehiclesController` at all today — a vehicle can
only be born inside the intake flow), so the front door ships [[Intake flow]] **§1a plus §1c** and
the §1b mismatch tree and §2a/§2b dedup forks wait. Three consequences put on the record rather than
discovered later:

- **§1c cannot go with the rest of the tree.** `index_intakes_one_open_per_vehicle` is a partial
  unique index (`… ON intakes (vehicle_id) WHERE (status = 0)`); keying an in-house plate without
  handling it raises `RecordNotUnique` — **a 500, not a branch**.
- **The dedup trap moves rather than disappears.** With a standalone add-vehicle, an SA who can't
  find the customer creates a new one and hangs the car off it — [[Intake flow]] §2a's reverse-wife
  trap promoted from edge case to *primary* path. The guard is a screen rule, not new logic:
  `Customer.search` already matches name **or** canonicalized phone, so "New customer" must be
  reachable only *through* an empty search result (S5.5e).
- **Screen A's not-found outcome is a routed dead end, not a refusal** — it must say what to do next
  and take the SA to add-vehicle (UI law 8). And `IntakesController#create` stays narrow: widened by
  `complaint:` only, not by `customer:`.

**A real finding against the recorded mismatch design.** [[Deferred design]]'s two-branch confirm
(`[Just this visit]` / `[Vehicle changed hands]`) answers only the *payer* question, but
[[Intake flow]] §1b has **four** forks. A screen showing just those two silently drops **1b-i**
(bill the file as usual) and **1b-iv** ("that's my old number" — the dedup trap that splits one
person into two cards). So the mismatch surface is **three** choices and the recorded two-branch
confirm is the **tail of the third**, reached only after the payer's own phone is looked up. Two
buttons stays right *at that point*, for the reason originally recorded. Written as a dated addendum
on that entry, together with the deferral. Also recorded there so it isn't re-derived: the §1
silent-compare rule means the phone comparison is **server-side on submit** — no client-side
compare, no hidden field, no `data-` attribute — while the file-holder's **name** may be shown
(builder ruling: a narrower exception, since the SA says it aloud at the counter anyway).

**Sprint plan.** S5.5 rewritten as the UI build order, **S5.5a–i**, in the order they run. **S5.8
superseded and split** — it was half-done and mis-ordered: the plate-normalization half already
shipped as `Vehicle.canonicalize`, the existing-vehicle-lookup half is a *prerequisite* of the
intake screen (now S5.5g), and the phone half moves with the deferred dedup tree.

**Left open, named as open.** Floor and owner postures. The §1b/§2 tree. Rebuild-or-keep per screen.
Add-a-repair, the jobsheet fill screen, the owner token page. Whether to restyle Devise. And
**S5.5i's system-test call** — un-suspending the Capybara layer suspended 2026-08-14 costs one
harness file, since capybara and selenium-webdriver are still in the `Gemfile`; these are the first
real pages since, and "does Enter on the plate field reach the visit" is what a controller test
cannot see. Not ruled — the session was redirected before it was put.

**Late in the session: the Bay system reference, and a visual re-lock.** The builder brought in an
external document — the *Bay system reference*, a **parallel design of the same product** (same
domain, same roles, decision dates 2026-08-04/08-09) — first to mine for UI ideas, then with the
ruling that **Bay's style becomes Knot's style, so the two read as one house**. Triage,
provenance, and every accept/reject sits in the new note
[[Bay system reference — external comparison]] (`06 Design/`). It is explicitly **not binding** —
where Bay conflicts with a recorded Knot decision, Knot wins and the conflict is written down
rather than quietly resolved.

**It exposed a sixth systemic finding, and this one is Knot's own.** Bay's "desktop content uses the
available canvas width" prompted a check: **every workshop screen is `col-12 col-md-8 col-lg-6`** —
a centred half-column on a PC, across the board, the visit, the repair, customers, crew and blocker
types. That is the stretched-phone layout **UI law 9 forbids**, and it is arguably the largest of
the six because it caps what the board can ever show. Added to [[Design system]] as finding 6,
and folded into S5.5c — page width and sidebar-vs-centred are one decision, not two.

**[[Design system]] re-locked** (the 2026-07-06 lock's *structure* survives; the *surface* changed).
Adopted: Helvetica/Arial · ink `#101010` · warm-achromatic neutrals replacing the blue-undertoned
family · a **new Geometry section** (square corners, visible 1px borders, restrained elevation, no
pills — Knot had **no shape rule at all** and was on Bootstrap's rounded defaults by accident) · a
**new Interaction and accessibility section** (desktop ~40px / mobile ≥48px, visible focus,
colour-independence, `prefers-reduced-motion`, safe-area). **Knot's identity is not part of the
adoption** — wordmark, K tile and brand steel blue stay ours; Bay's `#2727D9` and `bay.svg` were
offered and declined, so `--action` remains `#2D5E94`.

**The status palette was re-derived — the builder over-ruled the recommendation to keep it, and
that is recorded as their call.** Bay has *no* status colour system, so there was nothing to copy;
the five were re-derived against the new white surface and `#101010` ink. **The hue families did not
move** — that is what keeps the reserved words reserved. What changed is structure: with a visible
1px border the fill can lighten, so a badge is now **border + fill + text**, square-cornered.
Values are flagged in [[Design system]] as a **first derivation that has not been sample-compared** —
the 2026-07-06 palette was locked by comparing rendered samples, and these deserve the same before
being called final. **No footnote is owed against [[ADR-011 Acknowledgement as stored visibility]]:**
it says explicitly *"Colour — deferred, not decided here."*

**Best single idea taken: `/design-preview`** — a route rendering every component once, in every
state. It turns UI law 7 from an assertion into something checkable, it would have caught the
`_blocker_item` twins on day one, and it is where the re-derived chips get their sample comparison.
Folded into S5.5a. Also adopted into L1: Bay's IA rule (*durable work areas in primary navigation;
detail, create flows and item actions reached contextually*), the sidebar/drawer pattern **with its
cost named** (no Bootstrap JS is loaded, so collapse needs Stimulus, and a rail needs icons — one
coupled call, deferred), and the four-question attention test.

**Loudest rejection: `Employment`.** Bay authorises on an `Employment` edge — a concept Knot
*retired* on 2026-07-21 when `WorkshopEmployment`/`WorkshopOwnership` collapsed into `WorkshopStaff`
([[ADR-010 WorkshopStaff supersedes the edge split]], Session 25). Importing Bay's role table would
have resurrected a dead vocabulary into notes that took a session to clean. Also declined: the
stack (Rails 8.1/ERB/SQLite/Propshaft/Docker/Netlify), `lucide-rails` (deferred with the sidebar,
not refused), the `/` role chooser, signed jobsheet snapshots (ADR-014/015 went elsewhere), and
Bay's entire "Still Proposed" list — Knot has already decided nearly all of it.

**Corroborations worth keeping.** Bay independently reaches "age the *stalled state*, not the
creation date" (ADR-011), "urgency in text, not a competing colour rule" (ADR-011's chip rule),
intake under a minute ([[User stories]]), and "surface exceptions rather than search every job"
(Design law #3). Two independent designs landing on the same reasoning is evidence it holds.

**Verified the Bay reference against the running prototype, and it changed things.** The pasted
document turned out to be a **thin summary**. Walked `bay-qwyg.onrender.com` — login, component
inventory, all four role dashboards, a destination view, the mobile posture — and measured the real
values. **Two things this session had already written down were wrong:**

1. **"Restrained elevation" is not "no shadows".** Inline cards genuinely carry none, but every
   *floating* layer has a specified shadow (drawer, tray, floating control, scrim). The rule is
   **elevation means "above the page", never "important"**.
2. **One border token and one muted ink are both wrong — each is a ramp.** Container edges
   `#D8D8D4` (which had been derived independently and matched exactly), inner rules `#E5E5E1`–
   `#E8E8E5`; muted ink steps down with size across four values; plus a third surface `#F7F7F5`
   for toolbars. A dense table reads heavy without the lighter inner rule — exactly the doubt
   raised when the sample board mock was reviewed, now confirmed as a real token.

**The largest omission was the type treatment**, which the document reduces to "Helvetica / Arial".
It is a **1.6 modular scale at weight 400 with proportional negative tracking** and display
line-height *below* 1.0 — regular weight, set tighter than its own size. Also absent: the eyebrow
component, the page pattern (eyebrow → display heading → one primary top-right), the full nav spec
(42px rows; **selected = full-bleed solid accent**, not a tint; account chip pinned to the sidebar
footer), the role matrix, the inverted `#101010` context panel, square bullets and avatars,
border-collapsed stat tiles, the list toolbar, and the responsive rules. All now in
[[Design system]].

**Confirmed by absence:** no `badge`/`chip`/`pill`/`status` class exists anywhere in Bay's markup —
so the ruling to *re-derive* Knot's status palette was made on accurate information. **And
`/design-preview` is an empty shell**: all nine sections return empty Turbo frames. The idea stays
the best one taken from Bay, but it is unproven there — Knot will be the first to populate it.

**One divergence recorded rather than absorbed:** Bay puts "Add customer" as the top-right primary
beside the search, the opposite of Knot's search-first rule (S5.5e). Knot's reason still holds; the
difference is now deliberate. Two patterns explicitly **not** copied: list rows as grid `div`s
rather than a `<table>` (an accessibility regression on a dense board), and attention rows dropping
the owner and waiting columns on mobile — which discards two of the four questions the component
exists to answer.

**Consolidation: one source of truth for design (builder ruling).** [[Design system]] — **renamed
from *Visual theme***, 51 links updated across 15 notes — now holds *everything* design: tokens,
type scale, geometry, elevation, the component inventory, UI laws, interaction/accessibility, **and
the whole design plan** (findings, rulings, build order, L0–L3) that had been living in
[[Screen map]] Part I. [[Screen map]] goes back to being purely **a reflection of code** — what
screens exist — with its not-built rows the one piece of intent it still carries.

I argued for a narrower split (system in one note, per-sprint plan in another, because half of
Screen map is mechanically re-synced from `bin/rails routes`); the builder ruled for full
consolidation, and it is recorded as their call. The re-sync ritual is protected by an explicit rule
at the foot of [[Screen map]]: a re-sync that finds code contradicting [[Design system]] has found
**drift, not an update**.

**The rename touched a link inside [[ADR-011 Acknowledgement as stored visibility]] and two closed
archive notes.** Link text only — no decision content altered — but recorded here because ADRs are
never edited and the archive is closed.

**What makes the source of truth actually true** is three layers that must agree: the note
(authoritative), `application.css` (implements it), and `/design-preview` (renders it, so drift is
visible instead of a doc-versus-code guess). That third layer is S5.5a.

**Then the design system got built, compared, and ruled on.** A working desktop mock of the board
was built from the settled tokens, iterated against the live reference five times, and used to settle
the open calls by *looking* rather than arguing. It now lives at
`08 Experiments/knot-board-desktop.html` and is documented in [[UI experiments]] — unlike the earlier
experiments it is not exploratory, it is **the reference implementation** [[Design system]] was
written from.

**Sizing and spacing ruled onto Bootstrap's ladder.** The mock carries a **scale switch** (pure CSS,
`:has()` plus a checkbox, no JavaScript) flipping between the values measured off the reference and
Bootstrap 5.3.3's own — same markup, only the token block swaps, both sets verified to hold identical
token names. Three of six type sizes were already identical; the rest moved 1–4px. **Bootstrap won on
the builder's call** ("easy win, Bootstrap sizing looks way better"). The payoff is that spacing stops
being ours: every padding and gap value can live as a Bootstrap utility class in the Slim templates,
so **no spacing number lives in our CSS at all**. What remains in the brand layer is colour, negative
tracking (Bootstrap has no letter-spacing utilities), six component dimensions, and five components.

**Four reversals and corrections, all recorded.**

1. **The accent reversed.** `--action` is now `#2727D9`, and **`--brand #22456B` is retired** — the
   builder leaned the design the rest of the way toward the reference, and a steel-blue mark beside
   an indigo button reads as broken. The first pass had explicitly declined that accent to keep
   Knot's identity; the wordmark, K tile and square avatar stay ours, but the hue is shared.
2. **The canvas is white, and the earlier note was wrong.** It recorded `#F4F4F2` as a "deliberate,
   reasoned decline" of the reference's white-canvas line on the 2026-07-06 glare argument. The
   shipped CSS sets `#f4f4f2` then **overrides it to `#fff`** — the operational canvas always was
   white, as the reference's own spec said. The grey's real job is the **hover** tone; retaining it
   as canvas gave it a job it never had and left hover with none.
3. **Two border weights, not three.** The reference uses three; two are indistinguishable. The
   distinction that earns its keep: metric-strip dividers take the full `--border` so the strip reads
   as a grid, list rows take `--rule` so a list reads as a list.
4. **The "no status classes anywhere" claim was too strong** — based on dashboard markup alone. The
   stylesheet does carry `.status*` and `.label-chip`, but they are monochrome and blue, flat and
   borderless. No semantic hue vocabulary, so the substance held and the re-derivation ruling stands.

**Scope narrowed to desktop, deliberately** ("for simplicity and sanity, everything on monitor
screens first"). This **confirms** rather than reverses the desk-first sequencing in L2 — law 5 is
dormant, not deleted. One consequence recorded so it isn't discovered later: Bootstrap's `.display-6`
and `.fs-1`–`.fs-4` are **fluid below the xl breakpoint**, so they resolve exactly only at ≥1200px.
That is the first thing to re-check when the phone posture returns.

**Two structural bugs the mock exposed**, both fixed and both now in the spec: the sidebar is
**fixed, full viewport height, with the topbar inside the content column** (not a full-width bar
above the shell), and the page head is **bottom-aligned** so the primary action sits level with the
title's baseline. Also caught by the builder from a screenshot: a sub-nav item styled as an indent
with **no parent above it** — sub-navigation exists only under a parent, with a vertical rule.

**[[Design system]] gained a §Page template** — the four-part skeleton every desktop screen follows
(frame → topbar → page head → section stack), with only two section types and one panel anatomy.

**The gate — and a design failure worth recording.** Started the sign-in / sign-up screens as
[[Design system]] L3 step 1, noting first that they can legitimately jump the queue ahead of the
cross-cutting X1–X4 because the auth layout has **no shell** — X3 does not block them.

Two facts corrected on the way in. The vault said Devise views "stay unstyled until the theme grows";
they are in fact **fully styled, to the old theme**, so this is a restyle with live breakage. And
retiring `--brand` is one of those breakages: `.auth-badge` reads `var(--brand)` on all four gate
screens, and `--brand` appears in five CSS rules also covering `.wordmark` and the marketing home.
That has to be handled in S5.5b regardless of any design decision.

**The failure: the first sign-in mock broke five of the rules the session had just written.** Built
it, then audited it when the builder asked plainly whether it followed the design language. It did
not — off-ladder spacing (`.75rem`, `2rem`), an off-scale font size, a raw hex outside the tokens, a
hand-rolled focus-ring `box-shadow`, an invented `--canvas-alt` duplicating `--hover`, and no eyebrow.
Worst of all it **hand-rolled the input, button and checkbox** that Bootstrap already ships — the
exact duplication the single-source-of-truth ruling existed to end, on a screen that is mostly a form.

**Root cause was structural, not carelessness: the gate had no archetype.** §The page template covers
app screens only — frame → topbar → page head → sections — and the gate has none of those. The gap
was flagged, the builder redirected off a split layout, and the screen then got built with nothing in
the system defining what it should be. A screen with no spec cannot be checked against one.

**Three fixes, in that order.**

1. **[[Design system]] §The page template became §Layout archetypes**, with **two**: the app page and
   **the gate**. The gate's spec is explicit — grey canvas so a white card lifts without a shadow,
   one 25rem centred column, mark → eyebrow → title, one card, and the action pair (solid primary,
   rule, outlined cross-link). It carries the boundary too: **the gate's job ends at authentication**;
   workshop creation and invitation acceptance stay on `home#index`. The section opens with the rule
   that would have prevented all of this — *a screen fitting neither archetype is a new archetype,
   recorded before it is built.*
2. **A new §Bootstrap is the default answer** — a table of which Bootstrap component to use for which
   need and which `--bs-*` variable themes it, plus the smell test: **a padding or gap value written
   in `application.css` is the smell**, because spacing belongs in utility classes in the templates.
3. **`CLAUDE.md` now carries seven checkable design rules** *(builder's call — "we probably need to
   change it into claude.md so it follows the design rules we came up with")*. It is loaded every
   session, where the vault is only read when someone opens it. The rules are deliberately short and
   testable: Bootstrap-first · the spacing ladder ("12px and 32px do not exist") · six type steps,
   display at weight 400 · tokens only, no raw hex · square, shadows only on floating things · one
   solid primary · every screen is a recorded archetype.

**The rebuilt mock loads the repo's actual vendored `bootstrap.min.css`** rather than approximating
it, so what it renders is what Rails will render — the earlier mocks were bespoke CSS that only
looked Bootstrap-ish. Its brand layer is ~40 lines and holds nothing Bootstrap can express, which is
the target shape for `application.css` after S5.5b. It passes all seven rules on audit. Kept in
scratch rather than promoted to `08 Experiments/` — the board mock earned that place by being ruled
on; this one has not been.

**Still open on this screen:** forgot-password and reset-password reuse the gate archetype but are
not drawn, and the confirm-password field on sign-up is a call not yet made.

**Sprint 5A · Design system rollout — the UI work promoted to its own phase.** `S5.5a–i` was a
sub-task list inside the intake vertical, but it had grown to cover the auth gate and a restyle of all
twelve existing screens — neither of which is intake work. Promoted to **Sprint 5A** (lettered, not
numbered: `S5.1`–`S5.9` are live ids and another renumber would rot citations the way the 5↔6 swap
did). **S5.5 narrows to the two genuinely new screens** — add-vehicle and the intake front door —
both now marked as depending on 5A. Roadmap gains slice 5.75 for the same reason.

**The builder named a sequencing rule, and it caught a real error in my plan.** I had opened Sprint 5A
with `/design-preview`, on the reasoning that it is how everything else gets verified. The builder's
instinct was to build the shell first — "creating pages/components that are shared more first" — and
asked what the pattern is called.

It is **outside-in** / **shell-first** development; formally a topological order over the dependency
graph, and the deliberate opposite of Atomic Design's bottom-up atoms → pages (right for a component
library you publish, wrong for an application, where you end up with good buttons and no idea whether
the page holds together).

**The rule is now in [[Sprint plan]] §Conventions as "sequence by fan-in"**, with three criteria in
strict order: fan-in, cost of late change, verification leverage — *"this earns early, it never earns
first."* My error was sorting by the third axis and letting it win the first slot: `/design-preview`
has **zero fan-in**, nothing depends on it, and it cannot render until the tokens exist, so it was a
leaf placed before its own root. The principle was already recorded in [[Design system]] as "do the
things that change every screen before drawing any screen" — the failure was that it was implicit, and
an implicit rule loses to whichever axis feels urgent. Sprint 5A reordered on it: tokens → shell →
rulings → `/design-preview` → gate → screens. Two things fell out of the reorder that were not there
before: `_page_head` became its own task (identical on twelve screens, so a partial from day one or
twelve screens invent it), and the **law-3 ruling split from the law-3 audit** — the ruling binds every
screen and costs nothing to decide now, while applying it is per-screen work.

**New note [[UI rollout]]** *(builder: the UI phase "deserves a beefier space")*. I argued against a
second sprint plan — two plans means two places to tick and eventually a disagreement about what is
built, and `S5.5f`/`S5.5h` depend on 5A so that dependency would cross a file boundary. The vault
already had the convention to reach for: **a sprint task points at a working note** (`S5.1` →
[[Inspection jobsheet — design brief]]). Applied at phase scale. The note names, per task, the **files,
components or style rules** you actually build — written so a junior can pick it up cold — plus the
canonical ten-component set and the seven-rule definition of done as a checklist. The sprint entry
dropped from 90 lines to a grouped tick list.

**Cleanup pass on [[Sprint plan]], including two faults of my own.** A heading had been corrupted to
`## Sprint 5A · e` — the likely mechanism is running several overlapping `s.index()` slice edits on one
file in sequence, which silently eats content when a marker lands unexpectedly; headings need verifying
after bulk edits, and the link check does not see them. A task list had been written run-together
(`- [ ] S5A.1  - [ ] S5A.2 …`), which renders as two list items of literal text rather than fourteen
checkboxes — and duplicated a table listing the same fourteen tasks. Also: the `updated:` frontmatter
had grown to **1,800 characters** carrying every change back to Sprint 3, now 205 with a note that
history lives in this worklog and `git log`; and Sprint 5 opened with **27 lines of superseded callouts
and struck tasks before the first live task**, now parked under *Superseded in Sprint 5* at the end of
the section. Nothing deleted, 56 tasks intact. **The same frontmatter bloat exists vault-wide** — a
convention problem, not a one-file one, and deliberately left alone for now.

**Two stale code comments to fix when their screens are touched** (neither is a vault citation):
`config/routes.rb:35` and `app/views/customers/show.html.slim:38` both still say the front door
lands with "S6", stale since the 5↔6 swap. The customer-page one marks a TODO the builder asked to
raise **when that page is reached**, not now.

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

**Model-core tests (S5.1d core, `4b77664`).** Built the durable mechanics for both models, kept
deliberately independent of catalog *content* — they touch only `Inspection::DEFAULT_TYPE`/`TYPES`
and read `item_keys` back off the row, and answers are built value-less so nothing depends on an
item's `answer_type`: freeze, write-once (`attr_readonly`), one-per-intake / one-per-item, both DB
CHECKs, RLS + composite-FK isolation. **Deferred** (need catalog content, the churny part):
`value_matches_answer_type` and `complete?`. The builder flagged wanting to redo the test structure
later — this is the core, not the final shape. S5.1c (seed) is a ticked no-op: the catalog is code,
nothing to seed.

**Recovered a month-old orphaned note during the sweep.** A vault-wide coherence pass turned up
three stale git worktrees under `.claude/worktrees` (gitignored, invisible to `git status`, and
tripling every grep). Two were detached at commits already in `main` and were pruned. The third
held **three unmerged commits from Session 23 (2026-07-18)** carrying a 256-line
[[V2 - Customer and company dashboards]] note — the v2 read-surface vision, the **claim-flow
token-bridge** (with its own critique: a bearer link proves possession, not identity), **QR
self-enrollment** for company crew, and a **strategic fence** ruling daily driver↔truck tracking
out of Knot entirely (it's fleet-ops software on a second tenant). None of that thread existed on
`main` or was referenced anywhere. Restored the file, left the body as the faithful Session 23
record, and added a drift banner covering five corrections since (ADR-010 killed the
`WorkshopEmployment`/`WorkshopOwnership` naming its down-payment argument rested on; ADR-012 moved
the token and the frozen stamp to Intake; sprint numbers predate the 5↔6 swap; the one down-payment
it asked for — canonicalizing `vehicle.vin` like the plate — is still undone). Linked from
[[Deferred design]], and gave [[ADR-008 Crew joining requires acceptance]] a forward-pointer
footnote so its QR-reopen flag isn't dangling. **Nothing was lost, and nothing v1 built depends on
it** — the note's own §"What v1 should do NOW" asked for *"Schema: nothing."*

**Also fixed in the sweep:** 11 wikilinks wrapped across newlines (Obsidian rendered them as plain
text, not links) and the crew entry in [[Open questions]] still saying `JobMechanic` a month after
the technician rename.

**R11 net is a rake task, not a unit test — corrected.** Earlier this session assumed the orphan
scan would be an S5.1d test; walking it through showed it can't be. The test DB is empty (no
fixtures), so there are no persisted answers to orphan — a catalog rename stays green in the suite.
The real net is a deploy-time data-integrity check against real data (scan `item_key`/`item_keys`,
resolve each via `Inspection.item`). Deferred to first deploy; [[Risk ledger]] R11 updated to say so
rather than implying tests will catch it.

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
remainder), a comment-noise pass, and a naming/comment-discipline standard added to the
[[Agent guide]]. Then, reviewing the service layer, the builder pushed back hard on `JobActions`
doing too
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

