---
id: ADR-007
type: decision
status: accepted
date: 2026-07-08
supersedes: one clause of ADR-004
superseded_by:
---
# ADR-007 — Row-Level Security pulled into Sprint 1 (not deferred hardening)

From the 2026-07-08 session, while explaining how the S1.5 partial unique index works under
the hood. The builder's real question was tenant isolation strength — wanting schema-per-tenant
or database-per-tenant, on the assumption that physical separation is what "commercial-grade"
isolation requires. Revises one clause of [[ADR-004 Multi-tenant foundation]]; the shared-table
tenancy model itself is unchanged and, if anything, reinforced.

## Decision
- **Row-Level Security (RLS) is built in Sprint 1**, not deferred to a later "hardening pass."
  Every tenant-scoped table gets a Postgres policy restricting reads/writes to the current
  session's `workshop_id` — enforced by the database itself, not just by app-level query
  discipline.
- **Schema-per-tenant and database-per-tenant remain rejected** — re-examined against the
  builder's own stated requirement (Job/Vehicle history must query *across* workshops) and
  rejected harder: physical separation makes the cross-tenant queries the product actually
  needs (a vehicle's full service history) *worse*, not just operationally expensive.
- **Mechanism:** the access door (`ApplicationController#set_current_context`, [[ADR-006
  Ownership separate from Employment]] §3) sets a transaction-local Postgres session variable
  (`set_config('app.workshop_id', ..., true)`) every request, from the same `Current.workshop`
  it already resolves. Policies filter on that variable. Requires a non-table-owning app DB
  role (Rails' default connection owns the tables it creates, which **bypasses RLS by
  default**) — or `FORCE ROW LEVEL SECURITY` as a fallback if role separation is deferred.

## Why
- The builder's underlying want was a real, defensible property: "workshop A can never see
  workshop B's data," not as an app-code promise but as a database-enforced guarantee —
  the kind that satisfies "prove tenant isolation" in a security review.
- RLS delivers that guarantee **on the already-locked shared-table model** — it does not
  require schema-per-tenant. Shared DB + `tenant_id` + RLS is the mainstream commercial
  pattern for multi-tenant SaaS on Postgres precisely because it gets database-enforced
  isolation without the migration/backup/connection-pooling multiplication of
  schema-per-tenant ([[Rejected alternatives]]).
- **Cheapest moment to add it is now:** only two tenant-scoped tables exist (`ownerships`,
  `employments`). Retrofitting RLS after Sprint 2's Job/Vehicle/Customer tables land means
  auditing and policy-writing across a much larger surface, under time pressure, instead of
  as a template repeated per new table from the start.
- Defense in depth, not a replacement: app-level scoping (`Current.workshop`,
  [[Design laws]] #2, the S1.11 scoping convention) still applies. RLS is the backstop for
  the query someone eventually forgets to filter — it does not excuse writing bare queries.

## Consequences
- New Sprint 1 tasks (inserted as S1.8–S1.9, existing S1.8–S1.12 renumbered — none had
  commits yet, safe to shift): Postgres role/RLS-enable, then policies + `set_config` wiring
  into the access door.
- Test coverage (folded into the renumbered S1.14) must include: a query with **no**
  `WHERE workshop_id` still can't see another tenant's rows — proving the backstop actually
  backs something up, not just re-testing the app-level filter.
- Ops note for later: connection pooling means `set_config(..., true)` (transaction-local)
  is mandatory, not `is_local := false` — otherwise tenant context could leak across pooled
  connections between requests. Already flagged as a gotcha in ADR-004's consequences.
- v1 timeline: a small addition now (~2 tasks) vs. an open-ended retrofit later.

## Related
- [[ADR-004 Multi-tenant foundation]] · [[ADR-006 Ownership separate from Employment]] ·
  [[Rejected alternatives]] · [[Design laws]] · [[Sprint plan]]
