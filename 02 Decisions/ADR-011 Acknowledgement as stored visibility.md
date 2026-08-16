---
id: ADR-011
type: decision
status: accepted
date: 2026-07-28
extends: ADR-005 (acknowledgement — this decides what the system stores and shows when a handoff ISN'T acknowledged)
supersedes:
superseded_by:
---
# ADR-011 — Acknowledgement as stored visibility

> [!note] Sprint numbering — this ADR predates the Sprint 5 ↔ 6 swap *(2026-08-17)*
> Where the body says **Sprint 5 / S5.7** (the live board, the ageing-pin colour) read **Sprint 6 /
> S6.7**; where it says **Sprint 6** (the intake vertical) read **Sprint 5**. The two were renumbered
> so the intake/jobsheet vertical runs first. The body keeps its original numbers — ADRs aren't
> edited. See [[Sprint plan]].

Settles **[[Product gaps]] #5 — "the ack model assumes full adoption"**, the explicit gate on
Sprint 4. Arrived at over two sessions: a 2026-07-24 chip-out that taught the acknowledgement
model and first answered the gap with a *holder* framing, and a 2026-07-28 study of the proposed
`handoff` module that reshaped the answer into the simpler, truer one recorded here. The holder
framing and why it was dropped are kept below under Rejected alternatives — the reasoning is worth
the shelf space; the mechanism is not.

**Extends [[ADR-005 Acknowledged handoffs in V1]]; does not supersede it.** ADR-005 decided *that*
every ownership handoff is acknowledged and *where* the ack lives (on the event row). It never said
what the system should store or show when nobody acknowledges — and that silence is the gap. This
ADR answers exactly that.

## Decision

Two rulings, and the feature falls out of them.

### 1. The receiver is stored at write time, not derived at read time
Every event row already carries `acknowledged_at`. Add a companion **`receiver_id`** (the dormant
`acknowledged_by_id` column, renamed — it was never populated). The door **stamps who the event is
for at the moment it is written**, and it never changes afterward. Two columns then encode three
states with no computation:

| `receiver_id` | `acknowledged_at` | Meaning |
|---|---|---|
| NULL | — | **not a handoff** — plain history (a birth row, a raised blocker) |
| set | NULL | **an open handoff** — stamped for someone who hasn't picked it up |
| set | set | **picked up** — the receiver (or their own later action) acknowledged it |

That first NULL check *is* "is this a handoff?" — so the derived classifier ADR-005's Session-16
footnote handed to Sprint 4 (`.pending_ack`, plus a per-row role comparison) **is not needed and
is deleted**. `WorkshopStaff#pending_acknowledgements` becomes three plain `where`s.

**This is a restoration.** ADR-005's original inbox sketch keyed on a stored `to_user`
(`to_user = me AND acknowledged_at IS NULL`); the 2026-07-16 footnote removed that column and
handed the problem to a read-time predicate. Storing `receiver_id` puts the column back — and the
predicate that existed only because it was removed goes with it.

### 2. The feature is visibility, not an inbox
There is **no inbox, no routing, no notification, no confirm button** in v1. The stored receiver
exists so the **board can answer "waiting on whom"** — a job carries a muted *"Waiting on
&lt;name&gt;"* line, and the manager, reading the board, walks over. That is the whole system.
Acknowledgement is **implicit**: acting on a job through any door verb clears what you owe on it, so
a workshop that never adopts anything on the floor still reads correctly.

### Why store it — the append-only bug this fixes
Deriving the receiver at read time (from the blocker's `cleared_by_role`, say) has a latent bug:
**editing a catalog type's `cleared_by_role`** — shipped in Sprint 3 (`8a27e1d`) — would silently
re-point the receiver of every already-open handoff of that type. A log whose rows change meaning
when a config row is edited **is not append-only**. Storing intent at write time makes the claim
true: a row means what it meant when it was written, forever.

### Implicit acknowledgement — one sweep, no button
Because there is no confirm button, the only thing that ever fills `acknowledged_at` is
`JobActions.acknowledge_pending!(job, by:)` — it stamps every unacknowledged row on the job whose
`receiver` is the person now acting. It runs as `transition!`'s first line and is called explicitly
by the three blocker verbs (which don't reach `transition!`). Everything runs inside `job.with_lock`,
so a refusal further down rolls the acknowledgement back with the verb — no acknowledgement without
the act. A verb can never acknowledge the handoff it is *itself* creating: the sweep runs before the
new row exists.

**Fully implicit — no carve-out.** The 2026-07-24 draft exempted the blocker-resolve echo ("tapped,
never inferred", because the resolve doubles as verification). With no button, exempting it would
leave it **permanently unacknowledgeable**. So it sweeps like everything else; the verification
content lives in the `note` column, which carries it better than a tap ever did.

## The receivers, verb by verb
Who each event is stamped for (`created_by` is always the actor):

| Verb | Event | Receiver |
|---|---|---|
| `register_job!` | birth row | **nobody** — a job's origin is not a handoff |
| `assign_technician!` | `joined` | the **technician** |
| `remove_technician!` | `left` | the **technician** |
| `mark_done!` | `in_progress → done` | **`job.registered_by`** — the SA who took intake |
| `send_back!` | `in_progress → assigned` | the job's **current technician** |
| `raise_blocker!` | `raised` | **nobody** (see Fork 1) |
| `resolve_blocker!` | `resolved` | the **raiser** (`job_blocker.raised_by`) |
| `deliver!` / `cancel!` | terminal | **nobody** |
| `start_work!` / `note_blocker!` | — | **nobody** |

**Self-caused passes owe nothing** (the existing role rule stands): if the actor already holds the
receiving side, the sweep clears it in the same breath, so an SA's own `registered → assigned` and
same-role blockers (Hold for payment) generate zero traffic.

## Three forks, settled 2026-07-28
The reframe that settles all three: **there is no message or inbox concept in v1 — it is all
visibility.** Events are wall posts on the board and the timeline; `receiver_id` is the pin that
lets the board answer "waiting on whom". The manager is the backstop who reads the board and walks
over.

1. **Blocker-raise receiver = nobody.** A raised blocker is already visible (Sprint 3's blocked
   view), and `cleared_by_role` already routes who may clear it. Stamping a person would only
   manufacture a to-do and an "ignored it" stat. **B2 directed blockers stay deferred** — the
   column is there to pin a person in one line if a real workshop ever asks.
2. **Terminal verbs are untouched; the board is scoped to what's live.** Stamping `acknowledged_at`
   on delivery would write that someone acknowledged when nobody did — **a lie in an append-only
   log**. Not needed anyway: a delivered/cancelled job leaves the board, so its open rows stay
   honestly NULL and simply never surface. The boundary falls out of "the board only shows live
   work", with no special-case. The old NULLs remain for a "everything we ever dropped" report
   someday — a feature, not a leak.
3. **`mark_done!` pins `registered_by` (the intake SA).** They hold the customer's complaint context
   and are who a manager falls back on. Weakest of the six (a done-notice for an off-shift SA sits
   on the board while the car also reads done) — but at 1–4 SAs on the counter the intake SA usually
   *is* who's there, and it's a one-line change to revisit if a 3–4-SA shop makes it bite. See Stated
   limits.

**Done is not terminal.** A finished car nobody has collected — the founding pain, *"the counter
hasn't noticed"* — carries the `mark_done!` pin. `Job.active` excludes `done`, so the board surfaces
done jobs in their own **"Done — awaiting delivery"** group; only delivered/cancelled leave entirely.

## Colour — deferred, not decided here
The pin ships **muted, never coloured** in Sprint 4. Ageing colour (amber/red at a threshold) is
**S5.7**, on the chip and never the row (a red row is indistinguishable from a blocked job). The
2026-07-24 draft's colour-band table moved to [[Deferred design]] with the overnight-hours wart.

## Consequences
- **The derived classifier is deleted.** No `.pending_ack`, no per-row role comparison at read time.
  `WorkshopStaff#pending_acknowledgements` and `Job.pending_acknowledgements_by_job` are plain
  `where`s over the three event tables (`receiver_id = me/these-jobs AND acknowledged_at IS NULL`).
- **The concern shrinks to a 4-method `Acknowledgeable`** with zero per-model hooks: `has_receiver?`,
  `acknowledged?`, `awaiting_from?(staff)`, `acknowledge!`, plus a `scope :unacknowledged`. Included
  bare in all three transition models. The 2026-07-24 draft's per-model `handoff?` / `receiver` /
  `satisfied_by?` polymorphism is obviated by the stored column.
- **The sweep lives in `JobActions`, not the model.** "Which rows a verb clears" is a fact about the
  action, expressed where the action is.
- Time-to-acknowledge (`acknowledged_at − created_at`) stays derivable for free; no report in v1.
- **On-behalf-of acknowledgement is not recorded.** With one column, "whose item it was" beats
  "whose finger tapped it" — every consumer keys on the former. Re-add an `acknowledged_by_id`
  beside `receiver_id` if it ever bites; the loss is made cheap, not solved.

## Rejected alternatives
*(Do not re-propose without new information — see [[Rejected alternatives]].)*

- **The holder model (the 2026-07-24 draft's own answer — "every job is always held").** Its
  spine: read ADR-005 literally, so an unconfirmed pass leaves the *sender* holding the job, and the
  holder is a derived query, always computable — settling #5 because the sender is always an adopter.
  Elegant, and it did settle the gap. Dropped because it **over-built the answer**: there is no v1
  consumer for "who holds this" — the question is useless when the job is healthy, misleading when
  it's blocked (the holder isn't the blocker's counterpart), and blind when it's stalled. *"Who
  hasn't picked up yet"* — the stored receiver — is a truer, more useful question, and it settles #5
  the same way (the person who sees the stuck pass is the sender/manager, who is using the system).
  The holder framing also forced a receiver-derivation that carried the append-only bug above. The
  bomb metaphor (a job always in one pair of hands, defused at delivery) was a good teaching model;
  it is not a feature.
- **A stored acknowledgement-participation flag on `WorkshopStaff`.** Declaring someone a
  non-participant *excuses* them — the opposite of accountability. The technician with no phone shows
  up truthfully as "not picked up", needing no flag.
- **A per-workshop acknowledgement on/off switch.** All-or-nothing fails the constraint's word —
  ***partial*** use must provide value.
- **A threshold / snooze as *the* answer.** A knob only chooses *when* a permanent false alarm
  starts. Thresholds survive as S5.7 presentation on a model already correct.
- **A universal "Got it" button.** Would let someone signal "seen, not yet actionable" — a state the
  two-column schema can't represent (it measures "has acted"). Deferred as not-important-now, parked
  in [[Deferred design]].
- **Acknowledgements as their own INSERT-only event rows** (the purist append-only alternative, a DB
  trigger enforcing no UPDATE). Rejected: costs a `NOT EXISTS` subquery on every board read and
  doubles the rows, to protect a mutation that is already monotonic, two-column, and serialized under
  `job.with_lock`. The price — append-only stays a convention in comments rather than a Postgres
  invariant — is the one place this codebase doesn't push a rule to the database. Recorded as a
  considered rejection, not an omission.

## Stated limits
*(Recorded so they are not rediscovered as surprises.)*

- **`registered_by` is the weakest receiver (Fork 3).** "Who understands this job" is not always
  "who must act next"; with 3–4 SAs on rotation, done-notices pile by accident of intake. Fine at
  1–2 SAs — an empirical question about the target workshop, not a design one. Revisiting means
  carrying the done signal on the board differently, not a schema change.
- **A receiver who won't act on the job leaves an eternal open handoff.** Two cases, same shape: a
  **retired** receiver (staff are soft-retired, so the FK holds) and a **removed technician** — the
  `left` event pins the removed tech (kept uniform with `joined`, builder's call 2026-07-28 for
  simplicity), so a job removed-then-not-yet-reassigned briefly shows *"Waiting on \<removed tech\>"*
  until reassignment. Neither will ever be cleared by that person acting. Mostly masked, since the
  board's pin shows only the most-recent open handoff — any later `joined`/`done` hides it. Reads
  truthfully as "the last handoff nobody picked up"; the manager reassigns. Acceptable under "users
  interpret", but decided rather than discovered.
- **Counter-side receivers are a person, not a role.** `mark_done!` pins a specific SA
  (`registered_by`), so person-level attribution is truthful on both sides now — at the cost of Fork
  3's weakness above.
- **The receiver never reaches the customer.** The Sprint 7 token page stays **stage + ETA**;
  internal accountability is not owner-facing ([[Job visibility]]).

## Related
- [[ADR-005 Acknowledged handoffs in V1]] (extended by this) · [[Product gaps]] #5 ·
  [[Event log]] · [[M1-F1 Status flow and transitions]] · [[Blocker]] ·
  [[Design laws]] · [[Visual theme]] · [[ADR-002 V1 scope]] · [[Sprint plan]] · [[Deferred design]]
