---
type: module-overview
module: M1
updated: 2026-08-14 (re-pointed at Intake as the tying entity — ADR-012)
---
# Module 1 — Job Monitoring

## Purpose
Real-time visibility of every active job. Core question: **"Where is every job right now?"**

## Problem it solves
Job progress is unclear; front desk loses track; managers walk the floor; owners call for
updates. See [[Main problem list]].

## The model (concepts)
A visit's situation is a car, its repairs, and a history:
- [[Intake]] — one car's visit; the entity that ties a car, its customer, and its repairs
  together *(2026-08-14: this used to be [[Job]] — ADR-012 lifted the visit out above the
  repair. [[Job]] is now one repair under an Intake, not the anchor.)*
- [[Job]] — one repair on that visit, its own stage/crew/blockers
- [[Stage model]] — how far along a repair is (progress axis)
- [[Blocker]] — what's pausing a repair or the whole visit, and whose court (overlay axis)
- [[Event log]] — the timestamped trail that powers the time analytics, per level
- [[Data model]] — customers, vehicles, jobsheets (who owns it / who we talk to)

## Roles
Four roles, on **WorkshopEmployment** (not on User): technician, service_advisor,
parts_advisor, workshop_manager. The **owner** is not a role — it's the separate
**WorkshopOwnership** governance edge (ADR-006; a working owner holds both).
workshop_manager/owner are explicit god-mode. Full matrix in
[[M1-F1 Status flow and transitions]]. *(2026-07-17: stale "five roles incl. owner" fixed —
predated ADR-006; edge names updated in the same pass.)*

## Features
- [[Features overview]] — the capability-cluster view (F1–F8), how features map to slices + screens, the V1 fence
- [[M1-F1 Status flow and transitions]] — role-gated stage transitions + acknowledged handoffs *(first build)*

See [[Roadmap]] for the full slice plan + which slices still need design.
See [[Screen map]] for the screen-by-screen UI inventory (built + intended surface, per-sprint drift check).
See [[Screen flow]] for the flows (task journeys Setup→Records→Daily loop, with screen · API · components · actions · triggers + build status).

## Scope
V1 = monitoring only. Parts → V2, technician → V3, pricing deferred. See [[ADR-002 V1 scope]].

## Open
- [[Open questions]]
