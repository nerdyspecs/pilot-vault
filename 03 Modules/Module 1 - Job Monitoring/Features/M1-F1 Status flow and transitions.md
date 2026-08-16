---
id: M1-F1
type: feature
module: M1
status: settled
updated: 2026-08-14 (read-against note for ADR-012/ADR-013 — the Intake split + door decomposition)
---
# M1-F1 — Status flow & role-gated transitions

## Goal
Move a job through its [[Stage model]] stages, each transition permitted only for the right
role, **through a door per level** (`JobActions` for a repair — [[Design laws]] #7). Every
ownership handoff is **acknowledged** by the receiver ([[ADR-005 Acknowledged handoffs in V1]]).

> [!warning] Read against [[ADR-012 Intake-Job two-level aggregate]] and [[ADR-013 The door decomposed]] (2026-08-14)
> This doc predates both. Three things to hold while reading everything below:
> - **`delivered` is not a Job stage anymore.** It's the one stored fact on **Intake**; a Job's
>   terminals are `done`/`cancelled` only, and `deliver!` is an **`IntakeActions`** verb, not a
>   `JobActions` one. Where this doc says "Registered", read "Intake `open` + Job `unassigned`" —
>   the old single stage split across the two levels. See [[Stage model]], [[Intake]].
> - **The door no longer checks who's asking.** Every `ensure_*!` guard named below
>   (`ensure_counter!`/`ensure_counter_staff!`, the crew gate, the catalog-role check) moved to a
>   new **`Permissions`** class, checked at the controller boundary. The permission matrix and
>   the crew-gate rules further down are still **factually correct** — they describe *who may do
>   what* — just relocated. `JobActions::Refused` is now `ActionRefused`.
> - **`register_job!` is gone.** Creation is `CreateIntake` (opens a visit + its first repair)
>   and `CreateJob` (adds a repair to a given intake) — see [[ADR-013 The door decomposed]].
>
> Read the older callout below for the ADR-010 renames (WorkshopStaff etc.) — both apply at once.

> [!note] Read against [[ADR-010 WorkshopStaff supersedes the edge split]] (2026-07-21)
> This doc predates ADR-010 and still says **"Employment role"** throughout (and **"mechanic"**
> in the permission matrix / ack table below). Substitute as you read — the *rules are unchanged*,
> only where a role is stored moved:
> - A person's operational role now lives on an append-only **`WorkshopStaffRole`** row attached
>   to their single **`WorkshopStaff`** record — not a separate `WorkshopEmployment` edge. "owner"
>   is a **governance boolean** on `WorkshopStaff`, not a role. Who may do what per role is exactly
>   as the matrix states; only the table it's read from changed.
> - Vocabulary is **technician** everywhere in code; the "mechanic" cells below are pre-2026-07-17
>   wording kept for history.
> - The door's guard methods settled at build as `ensure_counter_staff!` / `ensure_job_crew!` /
>   `ensure_active_technician!` — this doc's `ensure_counter!` etc. are the design-time names.

## Settled 2026-07-12 (pre-Sprint-2 design session, builder decisions)
- **Job creation goes through the door too** — `JobActions` owns it. Creation is already a
  multi-record motion (Job + the `nil → registered` transition) and will grow (jobsheet
  values in Sprint 5; a possible parts list later). No `Job.create!` from controllers.
  *(⚠ 2026-08-14, [[ADR-013 The door decomposed]]: this ruling **reversed**. Creation is
  authoring, not a door move — `CreateIntake`/`CreateJob` own it now, outside `JobActions`
  entirely. The underlying point stands, just relocated: no `Job.create!`/`Intake.create!`
  from controllers, every birth row written explicitly by a service.)*
- **`in_progress → assigned` (the backward/reassignment move): service_advisor** (+
  workshop_manager/owner as always). It is a handoff, so it acknowledges like any other
  once Sprint 4 lands.
- **Assignment eligibility: active `technician` employment only; primary-only in v1.**
  The `primary` boolean column exists from day one but helpers stay dormant.
  *(2026-07-16 correction: the flag is **deferred entirely** — S2.6 ships without the column;
  when helpers arrive it lands as **`lead`**. See Settled 2026-07-16 below + [[Deferred design]].)*
- **No cancel from `done`.** Done is frozen, not active; its only exit is delivering its
  **intake**. *(⚠ 2026-08-14, ADR-012: `delivered` left Job's enum — a Job's own terminals are
  `done` and `cancelled` only. "Delivered" is the success outcome one level up, on Intake.)*
- **Concurrency guards deferred** (stage-change race + `lock_version`) — see
  [[Deferred design]]; failure mode is loud (double transitions visible in the log),
  fix is one line in the one door. *(2026-07-16: the stage-change row lock was **woken**
  before Phase 3 was built — every verb wraps read-check-write in `job.with_lock`.
  `lock_version` stays deferred. See Settled 2026-07-16 (Phase 3) below.)*

## Settled 2026-07-16 (tracker-restructure sessions, builder rulings)
- **Crew tracker split — entity + event log** ([[Event log]]): `JobMechanic` (engagement) +
  `JobMechanicTransition` (`joined`/`left` events, each with author + ack pair). Assignment =
  one transaction, three rows (engagement + `joined` + stage transition). Removal writes a
  `left` event — compensating events, never deletion. *(⚠ Superseded in part 2026-07-17,
  Session 21 "Design B": the engagement became a **present-tense membership** —
  `JobTechnician`, deleted on remove — and the events (`JobTechnicianTransition`) became
  self-contained history carrying `job_id` + `workshop_employment_id` directly.
  "Never deletion" now binds the event log only. Vocabulary unified to **technician**
  (matches the role enum). Three-rows-one-transaction and the ack shape stand. See
  [[Event log]]'s supersession note.)*
- **Only the counter touches crew in v1**: service_advisor + workshop_manager/owner
  assign/remove — one `ensure_counter!` guard in the door. Self-join/leave is
  schema-supported but **capability-deferred** ([[Deferred design]]); waking it supersedes
  this matrix with a dated note.
- **Removal legality — the responsibility rule.** An engagement asserts **responsibility,
  not real-time presence** (same principle as "in_progress is a stage, not a
  this-exact-moment status"). At `assigned`, provided the job has **never touched
  `in_progress`**: removing the last mechanic is legal and writes the compensating
  `assigned → registered` rollback. Once a job has **ever** touched `in_progress`: the last
  mechanic is never removable — the only crew change is the reassignment swap (old `left` +
  new engagement/`joined`, one door call). *(2026-07-16 Phase 3 rulings: the swap itself was
  then **dropped for v1** — no crew motion at all on a started job; see Settled 2026-07-16
  (Phase 3) below + [[Deferred design]].)* **Invariant: a job that has started work always
  names ≥1 responsible mechanic.** Sick tech with no replacement → the board keeps showing
  them, truthfully, as the responsible party until a swap. Time-in-stage deliberately does
  not split "worked slowly" from "responsible tech absent" in v1 *(soft note, non-binding:
  a "no-manpower" blocker type is a possible Sprint-3 catalog candidate if the floor demands
  the distinction)*. ⚠ Sanity-check the "touched in_progress" edge (a job moved back to
  `assigned` via the backward move) when the Phase 3 allow-list is written. *(2026-07-16:
  checked at the Phase 3 design session — consistent. After the backward move the job is
  `assigned` + crew + touched-`in_progress`, so **both** remove and assign refuse; with the
  swap dropped, that job's crew is simply fixed, uniformly with every other started job.
  "Touched `in_progress`" is a **history query** — `Job#started_work?`,
  `job_stage_transitions.where(to_stage: :in_progress).exists?` — never derived from the
  current stage, which the backward move would falsify.)*
- **Open blockers stop `→ done`.** ✅ Resolved at the Sprint 3 blocker deep-dive: per-blocker
  stage guards (`Blocker.blocks`), so Hold For Payment guards its own boundary instead of
  colliding with `done`. *(⚠ 2026-08-14, ADR-012: HFP's guarded boundary, `delivered`, is no
  longer a Job stage at all — HFP is a wholly separate `IntakeBlocker`, vetoing intake
  `deliver!`. See [[Blocker]].)*
- **`primary` flag deferred entirely** (corrects 2026-07-12 above): S2.6 ships without it;
  all v1 technicians on a job are treated the same. When helpers arrive it lands as
  **`lead`** (`default: true` honestly backfills v1 engagements — every v1 mechanic was the
  lead); named `lead` not `primary` (reserved SQL keyword — quoting tax on raw-psql audit
  sweeps). Naming settled now so it isn't re-litigated ([[Deferred design]]).

## Settled 2026-07-16 (Phase 3 design session, builder rulings)
- **Named verbs, no generic `change_stage!`.** The door's public surface was **eight** bang
  methods — `register_job!`, `assign_mechanic!`, `remove_mechanic!`, `start_work!`,
  `mark_done!`, `deliver!`, `send_back!`, `cancel!` — each its own `def`/`end` block with a
  first-line stage guard. *(2026-07-17: crew verbs renamed `assign_technician!` /
  `remove_technician!` — vocabulary unified to technician, Session 21.)* The allow-list **is**
  the verb set: no verb accepts `done` except (at the time) `deliver!`, none accepts
  `delivered`/`cancelled` — the Done-freeze is structural, not a checked rule. Refusals raise a
  refusal error (human message); every verb wraps read-check-write in `job.with_lock` (the
  row-lock deferral **woken** — [[Deferred design]]). *(⚠ 2026-08-14, ADR-012/ADR-013 — what
  shipped is not this list. `register_job!` is **gone** (see the top callout); `deliver!` moved
  to `IntakeActions` (an Intake verb, since `delivered` left Job); `JobActions`'s surface today
  is `assign_technician!`, `remove_technician!`, `start_work!`, `mark_done!`, `send_back!`,
  `cancel!`, plus the job-blocker trio — six repair-level moves, not eight. The refusal type is
  `ActionRefused`. Everything else in this bullet — named verbs, the allow-list-is-the-verb-set
  law, `job.with_lock` — is unchanged.)*
- **`swap_mechanic!` dropped for v1** (amends Session 17's "the swap is the only in-progress
  crew motion" — v1 has none). A started job's crew is fixed until done/cancelled; a sick
  tech shows truthfully as responsible. **Escape hatch:** the manager/owner exemption in the
  crew gate means a manager can still drive a stuck job to `done` — the workshop is never
  trapped. Revisit at the first real mid-job handover need ([[Deferred design]]).
- **Tech moves are crew-gated.** `start_work!`/`mark_done!` require active `technician`
  employment **and** a current engagement on that job (manager/owner exempt) — being a
  technician somewhere isn't enough; you must be *this job's* mechanic.
- **`registered ↔ assigned` is crew-method-private.** Those stage rows are written only
  inside `assign_mechanic!`/`remove_mechanic!` transactions — "`assigned` ⟺ crew exists"
  is unbreakable because no other path can write either move.
- **`send_back!`** is the verb for the 2026-07-12 backward move (`in_progress → assigned`,
  counter-only). Reframed: a rare **compensating correction**, not a workflow step — never
  surfaced on technician screens (S2.11 note). Internal name only.
- **Crew freezes on terminal stages.** `cancel!`/`deliver!` write no synthetic `left`
  events — the engagement record keeps naming who was responsible.
- **Accepted edge: `mark_done!` has no compensating path** (done's only exit is
  `delivered`). Softened with a confirm dialog in S2.11.
- **Positioning invariant (pinned):** Knot tracks **job statuses**, never real-time tech
  activity. Many `in_progress` jobs per tech is normal; no pause/resume ever joins the stage
  axis. Per-tech workload is a query feeding S6.2 — zero Phase 3 work. *(2026-07-17,
  Design B: even simpler — `joins :job_technicians`; membership IS the current crew,
  no `.current` merge needed.)*

## Permission matrix (v1)
*(Checks below are still correct as a description of who may do what; **where** they're
checked moved to `Permissions` — [[ADR-013 The door decomposed]].)*

| Role | Stage transitions | Blockers | Crew |
|---|---|---|---|
| technician | Assigned → In-Progress → Done — **only on jobs where they hold a current engagement** (crew gate, Settled 2026-07-16 Phase 3) | raises/clears per its `Blocker.raised_by_role`/`cleared_by_role` | — |
| service_advisor | create Unassigned (via `CreateJob`, not a door verb); Unassigned → Assigned; Send back; Cancel; **delivers/cancels the intake** | raises/clears per role fields, job **and** intake | assigns/removes technician (responsibility rule below) |
| parts_advisor | — | raises/clears per role fields | — |
| workshop_manager | all | all (overrides any blocker's role fields) | all |
| owner | all | all (overrides any blocker's role fields) | all |

- **Stages:** Unassigned → Assigned → In-Progress → Done, + **Cancelled** — Job's own terminals
  (see [[Stage model]]). One level up: **Intake** — Open → Delivered/Cancelled (see [[Intake]]).
- **Assignment** = one motion, three rows *(2026-07-16; names per Design B 2026-07-17)*: `JobTechnician` membership + its
  `joined` event + the Unassigned → Assigned stage transition, one transaction.
- **Blockers** — who can raise/clear which blocker type is data on the [[Blocker]] catalog
  (`raised_by_role` / `cleared_by_role`), not hardcoded here. `workshop_manager`/`owner` always
  override. The catalog is shared across both levels; `blocks` picks the table.
- **Cancelled** reachable from any active stage (service_advisor / workshop_manager / owner).
- **Vehicle owner** is **read-only** — no door access at all, either level ([[Design laws]] #8).

## Acknowledgement (the handshake)
The change takes effect immediately, but **ownership isn't transferred until acknowledged** — an
unacknowledged handoff stays visible as limbo. Three triggers:

Under [[ADR-011 Acknowledgement as stored visibility]] the receiver is **stored** at write time,
not derived, and it is a **person** (or nobody), never a role:

| Event (verb) | Passed by (`created_by`) | Receiver (stored `receiver_id`) |
|---|---|---|
| `joined` / `left` (crew) | service advisor | that **technician** — a receipt, never consent (ADR-005) |
| `in_progress → done` (`mark_done!`) | technician | the **intake SA** (`registered_by`) |
| `in_progress → assigned` (`send_back!`) | counter | the job's **technician** |
| `resolved` (blocker) | resolver | the **raiser** |
| birth · `raised` · terminals | — | **nobody** — not a handoff |

"Waiting on whom" = a query across the event tables:
`receiver_id IS NOT NULL AND acknowledged_at IS NULL`, grouped by job. **No inbox** — it surfaces on
the manager's board as a muted *"Waiting on &lt;name&gt;"* line ([[Event log]]).

> [!note] Reshaped 2026-07-28 by [[ADR-011 Acknowledgement as stored visibility]]
> The 2026-07-24 draft answered with a *holder* model (an unconfirmed pass leaves the sender holding
> the job); the reshape stores the receiver instead. The table above is the result — stamped at write
> time, so a row's meaning never shifts when catalog config is edited (the append-only fix). One
> behaviour to note:
> - **Acting on a job acknowledges it.** A door verb by whoever an open handoff is *for* stamps
>   `acknowledged_at` honestly (`JobActions.acknowledge_pending!`) — there is no confirm button, so
>   this is the only thing that ever fills it. Under `job.with_lock`, so a refused verb rolls it back.
> - The 2026-07-24 "blocker-resolve echo is tapped, never inferred" carve-out **retired** — with no
>   button it would be permanently unacknowledgeable, so it sweeps like everything else; its
>   verification lives in the `note` column.
>
> No inbox and no sender-side view — one board, read by the manager. The `.pending_ack` predicate is
> **deleted**: `receiver_id IS NULL` *is* "not a handoff", so no classifier is needed.

## State axis — before vs after Done
- **Before Done:** workshop roles edit per the matrix above.
- **At/after Done:** the job is **immutable** for everyone — answers and history frozen.
  Corrections open a **new** job ([[Design laws]] #8).

## Transition rules
- Allow-list, enforced in the service ([[Design laws]] #7). Forward by default; backward only where
  explicit (In-Progress → Assigned for reassignment / new fault) — **never** once Done.
- Every change writes a [[Event log|JobStageTransition]] — no silent changes.

## Acceptance (draft)
- [ ] A job moves through the stages; each transition permitted only for the right role.
- [ ] Assignment creates a `JobTechnician` + moves stage in one motion; the technician acknowledges (the `joined` event).
- [ ] A raised blocker checks the catalog's `raised_by_role`; resolving checks `cleared_by_role`.
- [ ] Illegal transitions rejected; a Done job rejects all edits.
- [ ] Every transition writes a `JobStageTransition`.

## Related
- [[Stage model]] · [[Intake]] · [[Blocker]] · [[Event log]] · [[Design laws]] ·
  [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-004 Multi-tenant foundation]] ·
  [[ADR-010 WorkshopStaff supersedes the edge split]] ·
  [[ADR-012 Intake-Job two-level aggregate]] · [[ADR-013 The door decomposed]]
