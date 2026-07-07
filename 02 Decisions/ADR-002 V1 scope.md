---
id: ADR-002
type: decision
status: accepted
date: 2026-06-30
supersedes:
superseded_by:
---
# ADR-002 — V1 scope: job monitoring only

> **Extended by [[ADR-003 Digitized jobsheet in V1]]** — V1 also includes a configurable jobsheet.
>
> **Note (2026-07-04):** the stage list below is illustrative — canonical stages live in
> [[Stage model]] (Registered → Assigned → In-Progress → Done → Delivered, + Cancelled).
> "Parts → V2" means the **parts-list** specifically; the `parts_advisor` **role** and job
> **intake/complaints** are in v1. Handshakes, once deferred, are now v1 — see
> [[ADR-005 Acknowledged handoffs in V1]].

## Decision
V1 is exactly **Module 1 (Job Monitoring)**. Nothing else ships in V1.

The filter for every "should this be in V1?" question:
> Does it help answer **"where is every job right now?"** (staff) or **"what's happening to
> my car?"** (owner)? If not, it is V2+.

### In V1
- Job intake / registration
- Status flow: Registered → Assigned → In Progress → Completed → Delivered, role-gated
- **Blockers** as an overlay — a job can't progress *and* another role must act to clear it.
  Carries the responsible owner + timestamps so we can attribute time across the job's
  lifecycle (which department is the bottleneck), not just blame the mechanic.
- Time-in-stage / time-blocked tracking
- Live job list + filtering (status / technician / stage)
- Owner status page via token link (no login)

### Out of V1 — and where it goes
- **Parts / warehouse** → **V2.** V1 only models "waiting on parts" as a *blocker reason*,
  not a parts entity.
- **Technician skill tracking / training** → **V3.**
- **Pricing, quotation, job orders, invoicing** → deferred. "Awaiting customer approval" is
  handled as a **blocker type**, not a price field. When invoicing is eventually built,
  **integrate with existing accounting systems (AutoCount / SQL Account) rather than building
  our own — no tax handling.** (For now.)
- **CRM, marketing, cost/profit tracking** → not planned.

## Why
- Solo builder — the only way to ship is to do one thing well.
- The [[Main problem list]] backlog is large, but ~80% of it doesn't answer the core
  question; it's future modules, not V1.
- Keeping money out keeps the data model simple and avoids a financial-system rabbit hole.

## Consequences
- The Job model carries: status, current responsible owner, blocker overlay, timestamps.
  **No price fields.**
- Parts exist only as a blocker reason in V1 — no parts entity yet.
- Roadmap is sequenced: **V1 monitoring → V2 parts → V3 technician.**

## Related
- [[ADR-001 Core stack]]
- [[Decisions]]
- Absorbs the earlier "no pricing in v1" stance from the old `L4` log (to be retired).
