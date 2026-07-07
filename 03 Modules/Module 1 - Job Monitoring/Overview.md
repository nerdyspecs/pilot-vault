---
type: module-overview
module: M1
updated: 2026-07-04
---
# Module 1 — Job Monitoring

## Purpose
Real-time visibility of every active job. Core question: **"Where is every job right now?"**

## Problem it solves
Job progress is unclear; front desk loses track; managers walk the floor; owners call for
updates. See [[Main problem list]].

## The model (concepts)
A job's situation is two axes plus a history:
- [[Job]] — the entity that ties them together
- [[Stage model]] — how far along (progress axis)
- [[Blocker]] — what's pausing it, and whose court (overlay axis)
- [[Event log]] — the timestamped trail that powers the time analytics
- [[Data model]] — customers, vehicles, jobsheets (who owns it / who we talk to)

## Roles
Five roles, on **Employment** (not on User): technician, service_advisor, parts_advisor,
workshop_manager, owner. workshop_manager/owner are explicit god-mode. Full matrix in
[[M1-F1 Status flow and transitions]].

## Features
- [[M1-F1 Status flow and transitions]] — role-gated stage transitions + acknowledged handoffs *(first build)*

See [[Roadmap]] for the full slice plan + which slices still need design.

## Scope
V1 = monitoring only. Parts → V2, technician → V3, pricing deferred. See [[ADR-002 V1 scope]].

## Open
- [[Open questions]]
