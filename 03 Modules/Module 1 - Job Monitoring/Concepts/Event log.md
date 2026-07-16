---
type: concept
module: M1
updated: 2026-07-16
---
# Event log
The immutable, timestamped history that powers all the time math. **Append-only — never edited.**
Written **only via ONE DOOR** ([[Design laws]] #7) — nothing changes a job's state without also logging it.

## The tracker pattern: entity + event log *(restructured 2026-07-16, Session 17)*
Every tracker is **an entity plus its append-only event table**, mirroring the original
`Job` + `JobStageTransition` idiom. Entities are written once and **never carry ack columns**;
every event row has **one author** (`created_by`), **one derived receiver**, and **one dormant
ack pair** (`acknowledged_at` / `acknowledged_by` — lit in Sprint 4,
[[ADR-005 Acknowledged handoffs in V1]]). Post-split, the ack pair is the **only designed
mutation** on any tracker row — everything else is write-once, which is what makes
append-only a discipline the schema helps hold rather than fights.

## What's recorded
Three axes:
- **Stage changes** — `Job` (the entity) + `JobStageTransition` (`from`, `to`, who, when).
- **Crew** — `JobMechanic` (the engagement: "Ah Boy on WVK 3721"; the same person can have
  several over a job's life — count engagements, not events) + `JobMechanicTransition`
  (`joined` / `left` events). *(Split from one table 2026-07-16: a leave needs its own
  receipt — one ack pair can't shake hands twice — and the old `removed_at` was a timeline
  event hiding in a column mutation, the same smell as editing history.)*
- **Blocker raise/resolve** — catalog + `JobBlocker` items + `JobBlockerTransition` events.
  Sprint 3; see [[Blocker]] (items are independent — two subcons on one car — each with its
  own raise/resolve cycle).

There is **no single grand events table** — that would be event-sourcing, more than this needs.
A job's *timeline* = the event tables, merged by timestamp at read time.

**In code this read is `Job#timeline`** *(named 2026-07-12 — builder's call: "event log" is
programmer vocabulary, fine for this doc, wrong for code a workshop person reads; `history`
rejected because Sprint 6's vehicle history would overload the word)*. Arrives in layers:
Sprint 2 merges stage + crew events (ack columns dormant), Sprint 3 splices in blocker events,
Sprint 4 lights up acknowledgement — the method and view never change shape, they gain rows.

## Acknowledgement — the framings (settled 2026-07-15/16)
- **The ack pair belongs to a direction, not a table.** All event rows carry the same dormant
  pair; whether a given row is a *handoff* (NULL = a debt someone owes) or *pure history*
  (NULL forever, by design) is derived per row: creator vs. receiver.
- **Receivers are derived, never stored.** Stage transitions → the `service_advisor` **role**,
  always (floor → counter); crew events → **the party who didn't act** (SA assigns → the tech
  acks; a tech self-acting (v2) → the SA role acks — one comparison, no columns); blocker
  raises → the catalog's `cleared_by_role`; blocker resolves → the item's `created_by` (the
  echo back — in the subcon flow the resolve-ack doubles as a verification receipt). Derived
  receivers survive resignations and role swaps; a stored target would dangle.
- **Ack-owed is a role-level test.** A creator holding the receiving role incurs no debt —
  Siti's own `registered → assigned` is silent; same-role blockers (Hold For Payment,
  SA-raised SA-cleared) generate zero ack traffic by design.
- **The assignment handshake rides the `joined` crew event.** No transition landing on
  `assigned` ever has an ack end in v1. Stage transitions keep their pair anyway, because an
  unacked `in_progress → done` — finished car the counter hasn't noticed — is the product's
  founding pain expressed as a single NULL.

## What it powers
- **Time-in-stage** — gap between consecutive `JobStageTransition`s.
- **Time-blocked, by type** — each blocker item's raise→resolve, attributed to its resolver.
- **Dropped-handoff / time-to-acknowledge** — anything acknowledged late or not at all
  (`acknowledged_at` null on a *handoff* row). See [[ADR-005 Acknowledged handoffs in V1]].
- The full audit answer to "what happened to this job, and when?"

## Rules
- Append-only; no silent or retroactive edits. Corrections and retractions are **compensating
  events** (a `left` row, a backward stage move) — never deletes.
- Log the **initial entry into Registered** (a `nil → Registered` row at creation) so every
  stage has a clean entry timestamp.
- An engagement asserts **responsibility, not real-time presence** — see
  [[M1-F1 Status flow and transitions]] (the removal-legality rule).

## In Rails
- `JobStageTransition`: `workshop_id, job_id, from_stage` (null only on the birth row),
  `to_stage, created_by, created_at, acknowledged_at, acknowledged_by`.
- `JobMechanic` (engagement): `workshop_id, job_id, employment_id, created_at` — **no ack
  columns; no `lead` flag in v1** (deferred with helpers, [[Deferred design]]). Current crew =
  engagements with no `left` event — a query, not a column. *(2026-07-16, Session 19:
  `user_id` → `employment_id` — an engagement is held by a stint; actor columns
  (`created_by`/`acknowledged_by`) stay User, owners act but hold no employment. The
  actor/holder split — [[Data model]] §Resolved.)*
- `JobMechanicTransition`: `workshop_id, job_mechanic_id, action (joined | left), created_by,
  created_at, acknowledged_at, acknowledged_by`.
- `JobBlocker` + `JobBlockerTransition` — Sprint 3; column detail in [[Blocker]].

The **"waiting on me" inbox** = one query across the event tables via **the handoff predicate**
(`.pending_ack`, a Sprint 4 design-pass item): unacknowledged AND not-self-caused AND
role-resolved to me. A bare `acknowledged_at IS NULL` **overcounts** — NULL has two meanings
(dropped handoff vs. never-was-a-handoff), and only the predicate tells them apart; it lives
in ONE shared scope so every inbox, manager board, and limbo flag agrees. Orphaned debts (a
role that resolves to nobody) render as "pending, unheld role" — never vanish. A query, not a
table ([[Design laws]] #3).

## Related
- [[Job]] · [[Stage model]] · [[Blocker]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]]
