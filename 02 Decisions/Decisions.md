---
type: index
updated: 2026-08-14 (ADR-013 accepted — the door decomposed)
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
- [[ADR-005 Acknowledged handoffs in V1]] — every ownership handoff (stage / blocker / technician) is acknowledged; the ack lives on the event record
- [[ADR-006 Ownership separate from Employment]] — Ownership edge (governance) split from Employment (operations); signup creates the person, workshop creation is a post-signup act; access resolved through one door
- [[ADR-007 Row-Level Security pulled into Sprint 1]] — database-enforced tenant isolation (Postgres RLS) built alongside the tenant tables, not deferred; schema-per-tenant re-examined and rejected again
- [[ADR-008 Crew joining requires acceptance]] — joining a workshop is bilateral (invitee accepts); leaving is unilateral (termination needs no ack); the `Invitation` becomes a `pending → fired → accepted/declined` state machine; supersedes ADR-006's passive add-crew clause
- [[ADR-009 Account deletion is refusal-first]] — **carries the lifetime invariant: a workshop cannot exist without an Ownership.** Deletion refused for accounts with edges/history (derived from that invariant + append-only history); bare accounts delete freely; last owner can never delete while the workshop stands (escape routes parked); PDPA/anonymization deferred with a dated trigger
- [[ADR-010 WorkshopStaff supersedes the edge split]] — `WorkshopEmployment` + `WorkshopOwnership` collapse into one `WorkshopStaff` record (`owner` boolean, governance) + append-only `WorkshopStaffRole` rows (operations); actors AND holdings point at the tenant-local person via a composite `(actor_id, workshop_id)` FK (DB-enforced actor isolation); supersedes ADR-006 §1 and Data model's "actions → User"
- [[ADR-011 Acknowledgement as stored visibility]] — **extends ADR-005, does not supersede it**: the receiver is **stored at write time** (`receiver_id`), so "waiting on whom" is a plain, always-answerable query — **no inbox, no confirm button**, just a *"Waiting on &lt;name&gt;"* line on the manager's board, cleared implicitly by acting on the job. Restores ADR-005's original `to_user` and fixes an append-only bug; settles [[Product gaps]] #5 (the Sprint 4 gate). *(Reshaped 2026-07-28 — the 2026-07-24 holder model was studied and dropped.)*
- [[ADR-012 Intake-Job two-level aggregate]] — **extends ADR-011**: splits the overloaded `Job` into **Intake** (the car's visit) → **Job** (one repair, its own stage/crew/blockers). Intake stores its terminal position (`status` enum, door-written); **`ready` is the derived reading** — all jobs terminal with ≥1 done ([[Design laws]] #3). `deliver!` moves to the Intake; receivers stay **stored at write time** across both levels, and intake `deliver!` sweeps its jobs to close the done-notices. HFP moves to a **non-acknowledgeable** intake blocker (no direction = no ack pair); `blocks` is the single discriminator (`done`→JobBlocker, `delivered`→IntakeBlocker). Sequenced as **Sprint 4.5** (design pass + schema squash) before the S6 board — no prod data = cheapest migration now.
- [[ADR-013 The door decomposed]] — **extends ADR-012**, reverses its "single service object" ruling. A door owns state moves only: creation left for `CreateIntake`/`CreateJob` (a birth isn't a transition), authorization left for a new `Permissions` class (checked at the controller boundary, returning the `WorkshopStaff` to stamp), and the door split by level — `JobActions` (repair) + `IntakeActions` (visit), one shared `ActionRefused`. Standing invariant recorded: a door verb is reached only from a gated controller or another door.

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
