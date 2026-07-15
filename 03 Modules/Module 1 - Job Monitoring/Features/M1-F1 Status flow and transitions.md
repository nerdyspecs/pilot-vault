---
id: M1-F1
type: feature
module: M1
status: settled
updated: 2026-07-16
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
  fix is one line in the one door.

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
  new engagement/`joined`, one door call). **Invariant: a job that has started work always
  names ≥1 responsible mechanic.** Sick tech with no replacement → the board keeps showing
  them, truthfully, as the responsible party until a swap. Time-in-stage deliberately does
  not split "worked slowly" from "responsible tech absent" in v1 *(soft note, non-binding:
  a "no-manpower" blocker type is a possible Sprint-3 catalog candidate if the floor demands
  the distinction)*. ⚠ Sanity-check the "touched in_progress" edge (a job moved back to
  `assigned` via the backward move) when the Phase 3 allow-list is written.
- **Open blockers stop `→ done`.** ⚠ Unresolved collision, decide at the Sprint 3 blocker
  deep-dive: seed blocker Hold For Payment, if used as "don't release until paid," would
  block `done` forever — redefine HFP as pre-work, or give the catalog per-blocker stage
  guards (the guard may belong on `→ delivered`). Recorded in [[Blocker]].
- **`primary` flag deferred entirely** (corrects 2026-07-12 above): S2.6 ships without it;
  all v1 technicians on a job are treated the same. When helpers arrive it lands as
  **`lead`** (`default: true` honestly backfills v1 engagements — every v1 mechanic was the
  lead); named `lead` not `primary` (reserved SQL keyword — quoting tax on raw-psql audit
  sweeps). Naming settled now so it isn't re-litigated ([[Deferred design]]).

## Permission matrix (v1)
| Role | Stage transitions (via ONE DOOR) | Blockers | Crew |
|---|---|---|---|
| technician | Assigned → In-Progress → Done | raises/clears per its `Blocker.raised_by_role`/`cleared_by_role` | — |
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
- [ ] Assignment creates a primary `JobMechanic` + moves stage in one motion; the mechanic acknowledges.
- [ ] A raised blocker checks the catalog's `raised_by_role`; resolving checks `cleared_by_role`.
- [ ] Illegal transitions rejected; a Done job rejects all edits.
- [ ] Every transition writes a `JobStageTransition`.

## Related
- [[Stage model]] · [[Blocker]] · [[Event log]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]] · [[ADR-004 Multi-tenant foundation]]
