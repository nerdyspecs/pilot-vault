---
id: M1-F1
type: feature
module: M1
status: settled
updated: 2026-07-17
---
# M1-F1 — Status flow & role-gated transitions

## Goal
Move a job through its [[Stage model]] stages, each transition permitted only for the right
Employment role, **all through ONE DOOR** (`JobActions` — [[Design laws]] #7). Every
ownership handoff is **acknowledged** by the receiver ([[ADR-005 Acknowledged handoffs in V1]]).

## Settled 2026-07-12 (pre-Sprint-2 design session, builder decisions)
- **Job creation goes through the door too** — `JobActions` owns it. Creation is already a
  multi-record motion (Job + the `nil → registered` transition) and will grow (jobsheet
  values in Sprint 6; a possible parts list later). No `Job.create!` from controllers.
- **`in_progress → assigned` (the backward/reassignment move): service_advisor** (+
  workshop_manager/owner as always). It is a handoff, so it acknowledges like any other
  once Sprint 4 lands.
- **Assignment eligibility: active `technician` employment only; primary-only in v1.**
  The `primary` boolean column exists from day one but helpers stay dormant.
  *(2026-07-16 correction: the flag is **deferred entirely** — S2.6 ships without the column;
  when helpers arrive it lands as **`lead`**. See Settled 2026-07-16 below + [[Deferred design]].)*
- **No cancel from `done`.** Done is frozen, not active; its only exit is `delivered`.
  The two terminals are `delivered` (success) and `cancelled` (died before done).
- **Concurrency guards deferred** (stage-change race + `lock_version`) — see
  [[Deferred design]]; failure mode is loud (double transitions visible in the log),
  fix is one line in the one door. *(2026-07-16: the stage-change row lock was **woken**
  before Phase 3 was built — every verb wraps read-check-write in `job.with_lock`.
  `lock_version` stays deferred. See Settled 2026-07-16 (Phase 3) below.)*

## Settled 2026-07-16 (tracker-restructure sessions, builder rulings)
- **Crew tracker split — entity + event log** ([[Event log]]): `JobMechanic` (engagement) +
  `JobMechanicTransition` (`joined`/`left` events, each with author + ack pair). Assignment =
  one transaction, three rows (engagement + `joined` + stage transition). Removal writes a
  `left` event — compensating events, never deletion.
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
- **Open blockers stop `→ done`.** ⚠ Unresolved collision, decide at the Sprint 3 blocker
  deep-dive: seed blocker Hold For Payment, if used as "don't release until paid," would
  block `done` forever — redefine HFP as pre-work, or give the catalog per-blocker stage
  guards (the guard may belong on `→ delivered`). Recorded in [[Blocker]].
- **`primary` flag deferred entirely** (corrects 2026-07-12 above): S2.6 ships without it;
  all v1 technicians on a job are treated the same. When helpers arrive it lands as
  **`lead`** (`default: true` honestly backfills v1 engagements — every v1 mechanic was the
  lead); named `lead` not `primary` (reserved SQL keyword — quoting tax on raw-psql audit
  sweeps). Naming settled now so it isn't re-litigated ([[Deferred design]]).

## Settled 2026-07-16 (Phase 3 design session, builder rulings)
- **Named verbs, no generic `change_stage!`.** The door's public surface is **eight** bang
  methods — `register_job!`, `assign_mechanic!`, `remove_mechanic!`, `start_work!`,
  `mark_done!`, `deliver!`, `send_back!`, `cancel!` — each its own `def`/`end` block with a
  first-line stage guard. The allow-list **is** the verb set: no verb accepts `done` except
  `deliver!`, none accepts `delivered`/`cancelled` — the Done-freeze is structural, not a
  checked rule. Refusals raise `JobActions::Refused` (human message); every verb wraps
  read-check-write in `job.with_lock` (the row-lock deferral **woken** — [[Deferred design]]).
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
  axis. Per-tech workload is a query (`joins job_mechanics` merged with
  `JobMechanic.current`) feeding S5.2 — zero Phase 3 work.

## Permission matrix (v1)
| Role | Stage transitions (via ONE DOOR) | Blockers | Crew |
|---|---|---|---|
| technician | Assigned → In-Progress → Done — **only on jobs where they hold a current engagement** (crew gate, Settled 2026-07-16 Phase 3) | raises/clears per its `Blocker.raised_by_role`/`cleared_by_role` | — |
| service_advisor | create Registered; Registered → Assigned; Done → Delivered; Cancel | raises/clears per role fields | assigns/removes mechanic (responsibility rule below) |
| parts_advisor | — | raises/clears per role fields | — |
| workshop_manager | all | all (overrides any blocker's role fields) | all |
| owner | all | all (overrides any blocker's role fields) | all |

- **Stages:** Registered → Assigned → In-Progress → Done → Delivered, + **Cancelled** (see [[Stage model]]).
- **Assignment** = one motion, three rows *(2026-07-16)*: `JobMechanic` engagement + its
  `joined` event + the Registered → Assigned stage transition, one transaction.
- **Blockers** — who can raise/clear which blocker type is data on the [[Blocker]] catalog
  (`raised_by_role` / `cleared_by_role`), not hardcoded here. `workshop_manager`/`owner` always override.
- **Cancelled** reachable from any active stage (service_advisor / workshop_manager / owner).
- **Vehicle owner** is **read-only** — no ONE DOOR access at all ([[Design laws]] #8).

## Acknowledgement (the handshake)
The change takes effect immediately, but **ownership isn't transferred until acknowledged** — an
unacknowledged handoff stays visible as limbo. Three triggers:

| Trigger | Passed by | Acknowledged by |
|---|---|---|
| Mechanic joined/left | service_advisor | that mechanic — a **receipt**, never consent (ADR-005) |
| Stage change | technician | service_advisor (role — any holder clears it) |
| Blocker raised | raiser (per catalog) | resolver (per catalog) |

Inbox ("waiting on me") = a query across the event tables via the **handoff predicate**
(`.pending_ack` — unacked AND not-self-caused AND role-resolved; a bare
`acknowledged_at IS NULL` overcounts — [[Event log]]).

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
- [ ] Assignment creates a `JobMechanic` + moves stage in one motion; the mechanic acknowledges.
- [ ] A raised blocker checks the catalog's `raised_by_role`; resolving checks `cleared_by_role`.
- [ ] Illegal transitions rejected; a Done job rejects all edits.
- [ ] Every transition writes a `JobStageTransition`.

## Related
- [[Stage model]] · [[Blocker]] · [[Event log]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-004 Multi-tenant foundation]]
