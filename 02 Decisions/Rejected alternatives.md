---
type: reference
updated: 2026-07-17
---
# Rejected alternatives
Dead ends already walked. **Do not re-propose** — resurfacing needs new evidence, not a re-run.
(ADRs record what we chose; this records what we ruled out and why it stays out.)

| Rejected | Why |
|---|---|
| STI user subclasses / universal `accounts` table with `account_type` | Roles live on edges ([[Design laws]] #1); User stays a thin login |
| Polymorphic vehicle ownership; nullable `user_id`/`company_id` on vehicles | Vehicles always belong to a Customer; owners *claim* customer records in v2 |
| Everything-is-a-company | Forces fake companies onto private owners |
| UserVehicles / CompanyVehicles split tables | Two tables for one concept |
| EmploymentRole join table | Parked to v3 — only if per-workshop custom roles become real demand |
| Dual-writing job copies to the owner side | Owner visibility is a query, not a second copy ([[Design laws]] #3) |
| Schema-per-tenant | Operational complexity a solo builder can't afford |
| RLS-first tenancy | App-level scoping remains primary; RLS shipped in Sprint 1 as the enforced backstop ([[ADR-007 Row-Level Security pulled into Sprint 1]]) — RLS-as-the-*only*-scoping stays rejected *(reason updated 2026-07-17; the old wording cited ADR-004's "additive hardening" clause, superseded by ADR-007)* |

## Related
- [[Decisions]] · [[Design laws]]
