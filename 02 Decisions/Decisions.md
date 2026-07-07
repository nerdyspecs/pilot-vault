---
type: index
updated: 2026-07-07
---
# Decisions
Why the product is built the way it is. **One file per decision (ADR).** Once accepted, an
ADR is not edited — if you change your mind, write a *new* ADR that supersedes the old one,
so the reasoning history never disappears.

New structural choice → new ADR with the next number.

## Accepted
- [[ADR-001 Core stack]] — Rails monolith + Hotwire + Postgres, Devise auth, PaaS hosting (the founding technical decision)
- [[ADR-002 V1 scope]] — V1 = job monitoring only; parts → V2, technician → V3; pricing deferred
- [[ADR-003 Digitized jobsheet in V1]] — V1 also includes an owner-configurable jobsheet (the adoption wedge)
- [[ADR-004 Multi-tenant foundation]] — Workshop tenant, thin User, Employment edges, session re-verification
- [[ADR-005 Acknowledged handoffs in V1]] — every ownership handoff (stage / blocker / mechanic) is acknowledged; the ack lives on the event record
- [[ADR-006 Ownership separate from Employment]] — Ownership edge (governance) split from Employment (operations); signup creates the person, workshop creation is a post-signup act; access resolved through one door

See also [[Design laws]] (invariants) and [[Rejected alternatives]] (dead ends, do not re-propose).

## Open questions (not yet decided)
Not decisions yet. When one is settled, promote it to a numbered ADR.
- _(none currently)_

**Resolved:**
- ~~Exact job stages~~ → defined in [[Stage model]]. Kept deliberately adjustable (add/remove as we learn), so **not** frozen as an ADR.

*(Feature-level questions, like the owner notification channel, live with their module — see [[Open questions]] in Module 1 — not here.)*

## Later
- When this list gets long, replace the manual "Accepted" list with a Dataview query over `type: decision`.
