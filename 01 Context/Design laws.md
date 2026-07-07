---
type: context
updated: 2026-07-04
---
# Design laws
Invariants — never violate. Adopted from the 2026-07-03 foundation session.
ADRs record *choices*; these are the *rules* every choice must obey.

1. **Roles live on EDGES, never on User.** User = thin login. No STI, no `account_type`, no subclasses.
2. **Tenancy is a property of the QUERY, not the table.** One shared DB, `workshop_id` column.
   Never query a tenant table bare — always through `Current.workshop`. `Job.all` unscoped = the leak.
3. **Dashboards are queries, not tables.** New viewpoint = new scope, never new model.
4. **Vehicles always belong to a Customer.** Users/Companies never own vehicles directly —
   in v2 they *claim* customer records.
5. **Every v2 feature must be ADDITIVE.** No migration may rewrite or move v1 rows.
6. **Jobs never move between tenants.** New visit = new Job row.
7. **ONE DOOR.** All job state changes (stage, blockers, assignment) go through one service object.
   No scattered updates.
8. **A Done job is immutable.** History is append-only — you never edit a finished job; corrections
   open a **new** job. Vehicle owners are read-only everywhere.

## Related
- [[Decisions]] · [[Rejected alternatives]] · [[ADR-004 Multi-tenant foundation]] · [[ADR-005 Acknowledged handoffs in V1]]
