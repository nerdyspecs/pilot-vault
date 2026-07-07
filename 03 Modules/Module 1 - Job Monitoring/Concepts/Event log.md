---
type: concept
module: M1
updated: 2026-07-04
---
# Event log
The immutable, timestamped history that powers all the time math. **Append-only — never edited.**
Written **only via ONE DOOR** ([[Design laws]] #7) — nothing changes a job's state without also logging it.

## What's recorded
Three trackers, each append-only, each carrying acknowledgement ([[ADR-005 Acknowledged handoffs in V1]]):
- **Stage changes** — `JobStageTransition` (`from`, `to`, who, when).
- **Blocker raise/resolve** — `JobBlockerTransition` (blocker, action, note, who, when). See [[Blocker]].
- **Mechanic on the job** — `JobMechanic` (who's working it, added/removed, who, when).

There is **no single grand events table** — that would be event-sourcing, more than this needs.
A job's *timeline* = the three trackers, merged by timestamp at read time.

## What it powers
- **Time-in-stage** — gap between consecutive `JobStageTransition`s.
- **Time-blocked, by type** — each blocker's raise→resolve, attributed to its resolver.
- **Dropped-handoff / time-to-acknowledge** — anything acknowledged late or not at all
  (`acknowledged_at` null). See [[ADR-005 Acknowledged handoffs in V1]].
- The full audit answer to "what happened to this job, and when?"

## Rules
- Append-only; no silent or retroactive edits.
- Log the **initial entry into Registered** (a `nil → Registered` row at creation) so every
  stage has a clean entry timestamp.

## In Rails
- `JobStageTransition`: `workshop_id, job_id, from_stage, to_stage, created_by, created_at,
  acknowledged_at, acknowledged_by`.
- `JobBlockerTransition`: `workshop_id, job_id, blocker_id, action, note, created_by, created_at,
  acknowledged_at, acknowledged_by`.
- `JobMechanic`: `workshop_id, job_id, user_id, primary, assigned_by, created_at, removed_at,
  acknowledged_at, acknowledged_by`. Current crew = `removed_at IS NULL`.

The **"waiting on me" inbox** = one query across the three: `acknowledged_at IS NULL` for the
receiving user — a query, not a table ([[Design laws]] #3).

## Related
- [[Job]] · [[Stage model]] · [[Blocker]] · [[Design laws]] · [[ADR-005 Acknowledged handoffs in V1]]
