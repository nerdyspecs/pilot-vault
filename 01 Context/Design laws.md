---
type: context
updated: 2026-08-03 (law #6 restated for the Intake/Job split — ADR-012)
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
6. **Jobs never move between tenants.** A visit is never reopened or re-tenanted — **new visit = new
   Intake** (with its own Jobs). *(2026-08-03, [[ADR-012 Intake-Job two-level aggregate]]: "new visit
   = new Job row" was the one-job-per-visit wording; the visit is now the **Intake**, a repair is a
   **Job** under it. A returning delivered car is a fresh Intake, never a mutated old one.)*
7. **ONE DOOR.** All job state changes (stage, blockers, assignment) go through one service object.
   No scattered updates.
8. **A Done job is immutable.** History is append-only — you never edit a finished job; corrections
   open a **new** job. Vehicle owners are read-only everywhere.
9. **Calculations live in the model layer.** Business logic and derived values belong on models
   (or POROs like `JobActions`), never in controllers or views — so they are unit-testable in
   isolation and consistent everywhere they're reused. Controllers stay thin; a calculation you
   can't call without a request is in the wrong place. *(Added 2026-07-08; underpins the
   model-unit-test-first strategy — see [[Sprint plan]] conventions.)*
   *(2026-07-13: reaffirmed — a separate action class is justified only when cross-model
   orchestration, shared permission rules, and a mandatory audit log co-occur (e.g.
   `JobActions`); single-aggregate commands stay on the model as bang methods. A `WorkshopActions`
   layer was considered and rejected — see worklog Session 13.)*

## Related
- [[Decisions]] · [[Rejected alternatives]] · [[ADR-004 Multi-tenant foundation]] · [[ADR-005 Acknowledged handoffs in V1]]
