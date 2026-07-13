---
type: index
updated: 2026-07-14
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
- [[ADR-007 Row-Level Security pulled into Sprint 1]] — database-enforced tenant isolation (Postgres RLS) built alongside the tenant tables, not deferred; schema-per-tenant re-examined and rejected again
- [[ADR-008 Crew joining requires acceptance]] — joining a workshop is bilateral (invitee accepts); leaving is unilateral (termination needs no ack); the `Invitation` becomes a `pending → fired → accepted/declined` state machine; supersedes ADR-006's passive add-crew clause
- [[ADR-009 Account deletion is refusal-first]] — **carries the lifetime invariant: a workshop cannot exist without an Ownership.** Deletion refused for accounts with edges/history (derived from that invariant + append-only history); bare accounts delete freely; last owner can never delete while the workshop stands (escape routes parked); PDPA/anonymization deferred with a dated trigger

See also [[Design laws]] (invariants) and [[Rejected alternatives]] (dead ends, do not re-propose).

## Open questions (not yet decided)
Not decisions yet. When one is settled, promote it to a numbered ADR.
- *(none currently)*

**Resolved:**
- ~~Account deletion semantics~~ → promoted to [[ADR-009 Account deletion is refusal-first]]
  (2026-07-14; all three open edges ruled — escape routes and PDPA parked in [[Deferred design]]).
- ~~Exact job stages~~ → defined in [[Stage model]]. Kept deliberately adjustable (add/remove as we learn), so **not** frozen as an ADR.

*(Feature-level questions, like the owner notification channel, live with their module — see [[Open questions]] in Module 1 — not here.)*

## Later
- When this list gets long, replace the manual "Accepted" list with a Dataview query over `type: decision`.
