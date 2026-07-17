---
id: ADR-005
type: decision
status: accepted
date: 2026-07-04
supersedes:
superseded_by:
---
# ADR-005 — Acknowledged handoffs in v1

Builds on [[ADR-004 Multi-tenant foundation]] and the [[M1-F1 Status flow and transitions]] flow.

## Decision
Every ownership-transferring event — a **stage change**, a **blocker raised/resolved**, a
**mechanic added to a job** — is recorded as an event the receiving party must **acknowledge**.
The change takes effect **immediately**; but ownership is **not considered transferred** until
acknowledged. Acknowledgement is tracked with `acknowledged_at` / `acknowledged_by` **columns on
the event record itself** — there is **no separate handoff or notification table**.

## Why
- The #1 floor pain is miscommunication — *"I'm only alerted when the customer complains."*
  This needs a mechanism where a handoff that **isn't picked up stays visible**.
- **Notify + acknowledge, not blocking:** the state machine is never held hostage waiting for a
  confirmation. Instead, an unacknowledged handoff (`created_at` old, `acknowledged_at` null)
  surfaces as **limbo** — that is the alert.
- This **reverses** the Session-2 decision to backlog the handshake; it is now a v1 KEY feature.
- A dedicated handoff table would **double-record** events already logged (a stage change would
  write a transition row *and* a handoff row — they can drift). The ack belongs **on** the event.
  The "waiting on me" inbox is a **query**, per [[Design laws]] (dashboards/notifications are
  queries, not tables).

## Model
Ack columns `acknowledged_at`, `acknowledged_by` on all three trackers:

| Event | Tracker | Acknowledged by |
| --- | --- | --- |
| Stage change | `JobStageTransition` | service advisor |
| Blocker raised / resolved | `JobBlockerTransition` | the resolver (raise) / raiser (resolve) |
| Mechanic added | `JobMechanic` | that mechanic (accepts the job) |

- Inbox = one query across the three: `to_user = me AND acknowledged_at IS NULL`.
- Time-to-acknowledge (`acknowledged_at − created_at`) is a free accountability metric.
- All of this still flows through the **ONE DOOR** service.

## Scope boundary
- **Non-blocking** — ack is accountability, not a gate on the transition.
- **No handoff/notification table**; **no delivery channel** beyond in-app. The owner-facing
  channel stays deferred (copy-paste message for now) — see [[Open questions]].

## Consequences
- Each event record carries two nullable ack columns.
- Reporting gains time-to-acknowledge / dropped-handoff metrics for free.
- Removes the "handshake backlogged" note from [[Deferred design]].

---
**Footnote 2026-07-16 (structure refined, decision unchanged).** The trackers were
restructured into **entity + event log** ([[Event log]], worklog Session 17): crew became
`JobMechanic` (engagement) + `JobMechanicTransition` (`joined`/`left` events), and Sprint 3's
blockers become catalog + `JobBlocker` items + events. The ack pair now rides the **event
rows** — so the Model table's "Mechanic added | `JobMechanic` | that mechanic" reads as
"mechanic joined/left | `JobMechanicTransition` | the party who didn't act". This
*strengthens* the decision: a leave now gets its own receipt (one ack pair can't shake hands
twice), and "the ack belongs **on** the event" holds more literally than before. Two
clarifications from the same sessions: acknowledgement is a **receipt, never consent** (the
change stands regardless — "accepts the job" in the table is loose wording); and the inbox
sketch `to_user = me AND acknowledged_at IS NULL` is **illustrative** — no `to_user` column
exists, receivers are derived at read time, and the real query is Sprint 4's `.pending_ack`
handoff predicate (a bare NULL check overcounts; NULL also means "never was a handoff").

## Related
- [[ADR-004 Multi-tenant foundation]] · [[M1-F1 Status flow and transitions]] · [[Event log]] · [[Blocker]] · [[Data model]] · [[Design laws]]

**Footnote 2026-07-17 (Session 21) — names only, decision unchanged.** The crew tracker was
restructured to "Design B" and renamed mechanic→technician: the 2026-07-16 footnote's
`JobMechanic`/`JobMechanicTransition` are now `JobTechnician` (present-tense membership,
deleted on remove) / `JobTechnicianTransition` (self-contained history). This *tightens* this
ADR again: the ack pair lives exclusively on event rows, and event rows are now the sole
permanent crew record. Edge models also renamed (`WorkshopEmployment`/`WorkshopOwnership`).
See [[Event log]].
