---
type: reference
updated: 2026-07-12
---
# Deferred design
Consciously parked — **revisit later**, not dropped. Each is additive (won't require rewriting v1).

- **Jobsheet per-answer snapshot (level-3)** — freeze `label`/`kind` onto each `JobSheetFieldValue`
  at fill time, so even a field *rename* can't change how an old sheet reads. v1 relies on
  lock-on-Done + add-only fields instead. Add if a compliance/dispute need appears.
- **`lock_version` (optimistic locking on Job)** — guards against silent lost-updates when two staff
  edit the same job's free-text at once. Deferred: audit trail covers stage; concurrent free-text
  edits are rare in a small trusted team. Add the column + a "stale, reload" handler if it bites.
- **Stage-change row lock in `JobActions` (2026-07-12)** — two people moving the same job's stage
  in the same second (e.g. SA cancels while tech marks Done) can both pass the allow-list check
  read on stale state. Deferred by the builder pre-Sprint-2: the append-only transition log makes
  the failure *loud* (two exits from the same stage, visible, reconstructable — never silent
  corruption), and most "concurrent editing" in real workflow is inserts into different tables,
  which can't collide. The fix is one line (`job.with_lock` around read-check-write) in the one
  door, so the cost of waiting doesn't compound. **Trigger to revisit:** a workshop reports a job
  that "changed twice at once", or the log shows two transitions out of the same stage.
- **Dark mode = steel-blue chrome (2026-07-05).** v1 ships light mode only (high-contrast, per
  device-posture decision). When dark mode comes, the dark theme's surfaces derive from the brand
  steel blue (~`#22456B` family) — chosen because the navy app-bar sample read as "dark mode" and
  was liked exactly for that. Keep status colors (gray/blue/red/amber/green) identical in both modes.

## Related
- [[M1-F1 Status flow and transitions]] · [[Blocker]] · [[Job]]
