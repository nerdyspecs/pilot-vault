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
- **Mechanism:** the access door (`ApplicationController#set_current_context`,
  [[ADR-006 Ownership separate from Employment]] §3) sets a transaction-local Postgres session variable
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
  [[Architecture laws]] #2, the S1.11 scoping convention) still applies. RLS is the backstop for
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
  [[Rejected alternatives]] · [[Architecture laws]] · [[Sprint plan]]

---

## Footnotes (dated — ADRs are never edited, only annotated)

**2026-07-08 (S1.9 implementation) — session-local `SET`, not transaction-local.** The ops
note above turned out backwards in practice: Rails does **not** wrap a request in one
transaction (each bare statement autocommits), so transaction-local `set_config(..., true)`
evaporates before the next query *within the same request*. Built instead: session-local
`SET app.workshop_id`, wiped **reset-first at the top of every request** in
`set_current_context` — the pool-leak risk the note worried about is closed by the reset, not
by transaction locality. Transaction-local `SET LOCAL` remains the right tool *inside* an
explicit transaction (used by `Workshop.create_with_founder!`, S1.10).

**2026-07-12 (S1.12 pre-work) — second permissive policy `own_rows`.** Landing routing
(ADR-006 §4) must list a user's own edges *before* any workshop is selected — with only
`tenant_isolation`, no `app.workshop_id` means zero rows and every user looks edgeless
(chicken-and-egg). Added `own_rows FOR SELECT` on `ownerships`/`employments`:
`user_id = current_setting('app.user_id')` (a second GUC set at sign-in). Postgres ORs
permissive policies — visible if in-workshop-context OR about-you. Slack-shaped: you can
always list your own workspaces, you only see inside the one you've opened. `FOR SELECT`
only: reads about yourself, never writes — writes still require workshop context. Rejected
alternative: un-policing the edge tables as "access metadata, not tenant data" — simpler, but
walks back this ADR a day after verification and leaves Sprint 2 without a live worked
example.

**2026-07-12 (pre-Sprint-2) — method rename note.** The transaction-local `SET LOCAL` example
cited above (footnote 2026-07-08) lived in a method then named `Workshop.create_with_founder!`;
it was renamed **`Workshop.create_with_owner!`** — "founder" named a status the schema never
tracks (all owners are equal Ownership rows), so it aligned to the ADR-006 governance term
*owner*. RLS mechanics unchanged.

**2026-07-14 (Session 15 audit) — scope note: why `workshops` and `users` carry no RLS.**
The audit found this ADR records what gets policed but never why the two global tables don't
— closing that gap here. Neither has a `workshop_id` to police on: they are what tenancy is
*made of* (the tenant and the global person), not things inside it. And both must be readable
with **zero GUC notes set**, non-negotiably: Devise sign-in finds the `users` row by email
*before* authentication (before `app.user_id` can exist), and claim-verification reads the
candidate `workshops` row *before* the tenant note is confirmable (`set_current_context` —
policing it would deadlock the very check that legitimizes the note). Their guard is the app
layer, which is both **necessary** (above) and **sufficient**: no user-reachable path renders
either table bare — the session workshop claim is access-door-verified before anything shows,
and `users` is thin by law (ADR-006), so its worst leak is "this email exists". That leak
does exist in miniature at the app layer (invite flow's two flash messages form an
account-enumeration oracle — [[Risk ledger]] R10, accepted for v1). If workshops ever needed
DB-level hiding, the shape is an edge-based `own_rows`-species policy
(`id IN (my ownerships/employments)`) — but v1 has no surface that lists workshops bare, so
it buys nothing today. See [[Job visibility]] for the per-party map on the policed side.

**2026-07-17 (Session 21) — edge models renamed.** `Employment` → `WorkshopEmployment`,
`Ownership` → `WorkshopOwnership` (organisation-prefixed, mirrors the v2 fleet edges).
The policies this ADR describes ride `rename_table` untouched — Postgres keeps
policies/FKs bound through renames; only index names are refreshed in the same migration
for tidiness. Table names in the examples above read as the old names — the record of what
was built; the live names are `workshop_employments` / `workshop_ownerships`.

**2026-07-24 (Session 26) — `workshop_staff_roles` gains a read-only `own_rows`.** The tenant-spine
squash (ADR-010) noted own_rows on the roles table would be "dead weight — Gate 2 never needs to read
a role before a workshop is chosen." **Landing routing falsified that.** `User#accessible_workshops`
must exclude a dormant husk (every role ended), which means evaluating `WorkshopStaff#active?` —
which reads `workshop_staff_roles` — *before* any workshop is selected, when `app.workshop_id` is
blank and `tenant_isolation` alone hides every role row (collapsing `active?` to `owner?`-only and
locking out live non-owner staff). Added a **`FOR SELECT own_rows`** policy: `workshop_staff_id IN
(SELECT id FROM workshop_staff WHERE user_id = NULLIF(current_setting('app.user_id', true),
'')::bigint)` — reached through `workshop_staff` because the roles table has no `user_id` of its own
(and `workshop_staff`'s own own_rows narrows that subquery to the caller; no recursion). Read-only:
see your own role-periods pre-workshop, never write. **Safe against leaks** because every app read of
`workshop_staff_roles` is already association-keyed or `.for_current_workshop`, so the app-level
WHERE always intersects the (now wider) RLS result back down to the current tenant — own_rows can't
pollute a tenant-scoped role query. App commit `a283a27`; the model half made `accessible_workshops`
bottom out in the same `active?` the access door uses, so the two can't drift again.
