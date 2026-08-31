---
id: ADR-004
type: decision
status: accepted
date: 2026-07-03
supersedes:
superseded_by:
---
# ADR-004 — Multi-tenant foundation: Workshop tenant, thin User, Employment edges

From the 2026-07-03 foundation session. Resolves the open "User model + tenancy" question
([[worklog]] Session 1). Revises the "two user populations" sketch in [[Data model]].

> **Note (2026-07-04):** the "v1 is single-actor (no handshake)" consequence below is
> **superseded** — acknowledged handoffs are now a v1 feature ([[ADR-005 Acknowledged handoffs in V1]]).
> The tenancy / User / Employment decision itself is unchanged.

> **Note (2026-07-07):** two clauses **superseded** by
> [[ADR-006 Ownership separate from Employment]]: (1) `owner` is no longer an Employment
> role — ownership is its own `Ownership` edge (governance vs operations); (2) the bootstrap
> changed — signup creates the `User` only, and "create workshop" is a post-signup act that
> creates `Workshop` + founder `Ownership` in one transaction. Tenancy, thin User, and
> per-request re-verification stand (the access check now resolves Employment OR Ownership,
> through one method).

> **Note (2026-07-08):** the "RLS later as additive hardening" clause is **superseded** by
> [[ADR-007 Row-Level Security pulled into Sprint 1]] — RLS is now built alongside the
> tenant tables in Sprint 1, not deferred. Shared-DB tenancy and schema-per-tenant's
> rejection both stand (reinforced, in fact).

## Decision
- **Workshop** is the tenant. One shared DB, `workshop_id` on every tenant-scoped table.
  App-level scoping via `Current.workshop` (ActiveSupport::CurrentAttributes). RLS later as
  additive hardening. Schema-per-tenant rejected.
- **User** — global, thin Devise login. No role, no org FK.
- **Employment** — the User↔Workshop edge carrying the role.
  `role` enum: `technician / service_advisor / parts_advisor / workshop_manager / owner`.
  `ended_at` (null = active). Partial unique index: one ACTIVE employment per (user, workshop).
- **Auth flow:** one login → contexts discovered from active Employments → switcher only if >1.
  Session stores context but is **re-verified against edges every request** — never trust the
  session alone. Ending an employment kills access next request. Set `Current` explicitly in
  jobs/console/rake (no request cycle there).
- **Bootstrap:** first signup creates user + workshop + owner-employment in one transaction.

## Why
- Solo-buildable multi-tenancy: a column + a query rule, no schema machinery.
- Roles-on-edges keeps User future-proof — v2 customer/company contexts are *new edges*,
  never a mutation of User.
- Session re-verification makes staff offboarding instant.

## Consequences
- Every tenant model carries `workshop_id`; every query goes through `Current.workshop` ([[Architecture laws]] #2).
- Jobs are double-stamped (`workshop_id` + `vehicle_id`) — whose board + whose history.
- **v1 is single-actor** (no handshake — see [[Deferred decisions]]); all state changes flow through ONE DOOR.
- v2 adds `CompanyEmployment` (roles: `owner / fleet_manager / driver`) as a parallel edge.
- RLS gotchas for later: transaction-local `set_config`, non-owner DB role / `FORCE ROW LEVEL SECURITY`.

## Rejected on the way
See [[Rejected alternatives]] — STI users, universal accounts table, schema-per-tenant,
RLS-first, and more.

## Related
- [[ADR-001 Core stack]] · [[Architecture laws]] · [[Rejected alternatives]] · [[Data model]]

---
**Footnote 2026-07-17 (Session 21) — edge models renamed, decision unchanged.** `Employment` →
`WorkshopEmployment`, `Ownership` → `WorkshopOwnership` (tables/FKs follow): organisation-prefixed
naming, mirroring the v2 fleet edges (`CompanyEmployment` / `CompanyOwnership`-style, one
reusable organisation─membership─governance pattern, never one table). Every clause of this
ADR reads with the new names; nothing else changes.
