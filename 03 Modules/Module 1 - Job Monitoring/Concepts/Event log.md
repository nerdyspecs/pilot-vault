---
type: concept
module: M1
updated: 2026-07-17 (Design B crew restructure)
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

**⚠ Superseded in part 2026-07-17 (Session 21, "Design B" — crew only).** The "entities are
written once" claim narrowed for crew: the crew entity is now a **present-tense membership
table** (`job_technicians` — a row means "on this job's crew right now"; created on assign,
**deleted** on remove), and its event table (`job_technician_transitions`) became
**self-contained** — each event carries `job_id` + `workshop_employment_id` directly, so
history never depends on a membership row surviving. What makes the deletion safe is this
note's own insight: **the events carry the complete history** — "who was ever on this job,
when" is answered by the log; "who is on it now" by the table. **Append-only binds the event
log, always; membership is a read model.** The ack pair remains the only designed mutation
on any *event* row. Stage and blocker trackers are untouched: `Job` is never deleted, and
Sprint 3's `JobBlocker` items stay written-once (an item's lifecycle lives in its events).

## What's recorded
Three axes:
- **Stage changes** — `Job` (the entity) + `JobStageTransition` (`from`, `to`, who, when).
- **Crew** — `JobTechnician` (membership: "Ah Boy is on WVK 3721 *right now*") +
  `JobTechnicianTransition` (`joined` / `left` events — the full history, self-contained).
  *(Split from one table 2026-07-16: a leave needs its own receipt — one ack pair can't
  shake hands twice — and the old `removed_at` was a timeline event hiding in a column
  mutation, the same smell as editing history. Restructured to Design B + renamed
  mechanic→technician 2026-07-17 — see the supersession note above. A returning technician's
  stints are counted from the event pairs, not membership rows.)*
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
- **The event log is append-only**; no silent or retroactive edits. Corrections and
  retractions are **compensating events** (a `left` row, a backward stage move) — never
  deletes. *(2026-07-17: "never deletes" binds the event tables; the crew membership table
  is a present-tense read model whose rows may be deleted — the events keep the history.)*
- Log the **initial entry into Registered** (a `nil → Registered` row at creation) so every
  stage has a clean entry timestamp.
- A crew membership asserts **responsibility, not real-time presence** — see
  [[M1-F1 Status flow and transitions]] (the removal-legality rule).

## In Rails
- `JobStageTransition`: `workshop_id, job_id, from_stage` (null only on the birth row),
  `to_stage, created_by, created_at, acknowledged_at, acknowledged_by`.
- `JobTechnician` (membership): `workshop_id, job_id, workshop_employment_id, created_at` —
  unique `(job_id, workshop_employment_id)`; **no ack columns; no `lead` flag in v1**
  (deferred with helpers, [[Deferred design]]). **Current crew = the table itself** — rows
  are created on assign and deleted on remove (Design B, 2026-07-17; replaced the
  no-`left`-event `.current` subquery). Points at the **stint** (`workshop_employment_id`,
  not `user_id` — the actor/holder split, [[Data model]] §Resolved); actor columns
  (`created_by`/`acknowledged_by`) stay User everywhere — owners act but hold no employment.
- `JobTechnicianTransition` (self-contained history): `workshop_id, job_id,
  workshop_employment_id, action (joined | left), created_by, created_at, acknowledged_at,
  acknowledged_by` — relates to membership only by the composite key, never an FK, so
  history survives the membership row's deletion. Read via `Job#timeline` or the employment.
- `JobBlocker` + `JobBlockerTransition` — Sprint 3; column detail in [[Blocker]].

> [!note] Updated 2026-07-21 by [[ADR-010 WorkshopStaff supersedes the edge split]]
> The bullets above read post-Design-B; ADR-010 renames the actor/holder targets. **`WorkshopStaff`
> replaces `WorkshopEmployment`** as the holder (`job_technicians.workshop_staff_id`,
> `job_technician_transitions.workshop_staff_id`), and — reversing the "actor columns stay User"
> line above — **`created_by`/`acknowledged_by` are now `WorkshopStaff` too** (the tenant-local
> person), each with a composite FK `(actor_id, workshop_id) → workshop_staff` so a cross-tenant
> actor is a DB foreign-key violation, not just an app check. Membership still carries no ack
> columns; the self-contained-history shape is unchanged. Operational roles live on append-only
> `WorkshopStaffRole` rows; `owner` is a governance boolean on `WorkshopStaff`, not a role.

The **"waiting on me" inbox** = one query across the event tables via **the handoff predicate**
(`.pending_ack`, a Sprint 4 design-pass item): unacknowledged AND not-self-caused AND
role-resolved to me. A bare `acknowledged_at IS NULL` **overcounts** — NULL has two meanings
(dropped handoff vs. never-was-a-handoff), and only the predicate tells them apart; it lives
in ONE shared scope so every inbox, manager board, and limbo flag agrees. Orphaned debts (a
role that resolves to nobody) render as "pending, unheld role" — never vanish. A query, not a
table ([[Design laws]] #3).

## Related
- [[Job]] · [[Stage model]] · [[Blocker]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]]
