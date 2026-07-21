---
id: ADR-010
type: decision
status: accepted
date: 2026-07-21
supersedes: parts of ADR-006 (the two-edge split, §1) + Data model §Resolved (actor/holder — "actions point at the person")
superseded_by:
---
# ADR-010 — WorkshopStaff + WorkshopStaffRole: one tenant-local person, owner as a governance boolean

From the 2026-07-21 build (designed across three chip-out sessions 2026-07-19/21). Collapses
the two organisation edges of [[ADR-006 Ownership separate from Employment]] into one
person-per-workshop record, and moves every tenant-local actor/holding column off the global
`User` onto that record. The tenancy foundation (Workshop tenant, thin User, per-request
re-verification, RLS) is unchanged — this changes *what a person's tie to a workshop is* and
*what an event row points at*.

## Decision

1. **One person-per-workshop record: `WorkshopStaff`.** Replaces `WorkshopEmployment` +
   `WorkshopOwnership`. Columns: `user_id, workshop_id, owner:boolean`. Unique
   `(user_id, workshop_id)` — a person is at most one row per workshop. It **is the read-model**:
   every "can this person do X" predicate (`owner?`, `technician?`, `service_advisor?`,
   `parts_advisor?`, `workshop_manager?`, `active?`) lives here, not scattered across
   controllers/views (same delegation-only discipline as `PermissionsHelper`).

2. **`owner` is a governance boolean that grants operational power (Path B).** Not a role.
   A pure owner has a `WorkshopStaff` row with `owner: true` and no roles; the door's owner
   branch stays (owner passes `counter_staff?`/`job_crew?` directly). The wrenching towkay
   holds `owner: true` **and** a `technician` role row — the same person, two truths on one
   record. Multiple owners = multiple rows with `owner: true`.

3. **Operational roles are `WorkshopStaffRole` rows** — a pure capability table, append-only.
   Columns: `workshop_staff_id, workshop_id, role:enum, ended_at`. One active row per
   `(staff, role)` (partial unique WHERE `ended_at IS NULL`); multi-role is a person holding
   several rows (the dual-hat technician + counter case). Role change = `end!` the old row +
   insert a new one, **never edit in place** — an immutable role per row makes the row a
   permanent, truthful stand-in for "capacity held at the time" (load-bearing once acks read
   it, Sprint 4). `owner` is **not** in the role enum (as ADR-006 already ruled — that
   distinction now lives on the boolean instead of across two tables).

4. **Every tenant-local actor and holding column points at `WorkshopStaff`, not `User`.**
   `created_by` / `acknowledged_by` on all event tables (`JobStageTransition`,
   `JobTechnicianTransition`, and Sprint 3's blocker events); the crew holding
   (`job_technicians.workshop_staff_id`). Each carries a **composite FK
   `(actor_id, workshop_id) → workshop_staff(id, workshop_id)`** — so "this actor belongs to
   this tenant" is enforced by the database, not just application code. (`WorkshopStaff` grows
   a `UNIQUE(id, workshop_id)` so the composite FK has a target.)

## Why

- **The composite FK is the whole point.** A global `User` actor FK can be tenant-checked only
  in Ruby — a bug or a forged id could stamp an event with an actor from another workshop and
  the database wouldn't notice. Pointing the actor at the tenant-local `WorkshopStaff` and
  adding `(actor_id, workshop_id)` makes cross-tenant actors a foreign-key violation. This is
  the guarantee ADR-007's RLS gives *reads*; ADR-010 gives it to *actor integrity*.
- **The two-edge split forced a polymorphic actor.** Under ADR-006, actions had to point at
  `User` because an owner held no `Employment` — pointing at `Employment` was unrecordable for
  owners. Merging the edges dissolves that: an owner is a `WorkshopStaff` like anyone, so the
  actor is a single, non-polymorphic FK. (This **reverses** Data model §Resolved's 2026-07-16
  ruling that "actor columns stay `User`" and its rejection of auto-created owner employments
  as fictional records — see the footnote there. The calculus changed because there is now one
  real, honest table, not a shim: a working owner's role row is a genuine capability, and a
  pure owner simply holds no role.)
- **Governance vs operations still matters** — it's just cheaper to model as a discriminator
  (`owner`) within one table than as two tables that every access path had to union.

## Consequences

- New models `WorkshopStaff` + `WorkshopStaffRole`; `WorkshopEmployment` / `WorkshopOwnership`
  deleted. Because there was **no production data**, this shipped as a **schema squash + reseed**
  (migration history rewritten to a clean 6-file set born in final shape), not a data migration —
  the drift check (`git diff structure.sql` on unchanged tables → byte-identical) is the safety net.
- `Current` carries `workshop` + `workshop_staff` (predicates delegated). The access door
  (`User#access_for`) returns the `WorkshopStaff` iff `active?`. The door's guards return the
  authorizing `WorkshopStaff` to stamp; `acting_user:` params stay (guards authorize the person).
- New RLS test class: the composite FK rejecting a cross-tenant actor is provable and tested.
- The **create-workshop and invite-accept flows** must seat a person as `WorkshopStaff` first,
  then attach roles. A **solo working owner** gets a governance row at signup and must add
  themselves an operational role on the crew page before taking a job on the floor — the crew UI
  exposes self-service role add for exactly this.

### Accepted trade-offs
- **No ownership-transfer history in v1.** Governance rows aren't append-only the way roles are
  (a pure governance row is never an actor/holding target, so it's hard-deletable) — who owned
  when isn't preserved. Fine while workshop lifecycle/transfer is deferred (v2).
- **≥1 owner per workshop** is app-level only (no remove-last-owner path — the ADR-009 invariant).
- Multi-role is on for v1; permission is fixed per role. A configurable ability matrix is v2 (a
  policy layer over roles feeding the same ONE DOOR — not a parallel authority).

## Superseded / revised
- **ADR-006 §1 (two-edge split)** — superseded: one discriminated table replaces
  `Employment` + `Ownership`. ADR-006 §2 (onboarding split), §3 (one-door access), §4
  (landing-by-edge-count) **stand**, now reading against `WorkshopStaff`. The "owner removed
  from the role enum" clause stands — the distinction moved to the boolean, not back into a role.
- **Data model §Resolved, the actor/holder split** — the "holdings → stint" half is preserved
  (holdings now point at `WorkshopStaff`, the tenant-local person); the "actions → person
  (`User`)" half is reversed (actions point at `WorkshopStaff` too). Dated footnote there.
- **v2 note:** the reusable pattern is `OrganisationAffiliation` (one table per org type, `owner`
  boolean + role rows). Company gets a single `CompanyStaff`/`CompanyStaffRole` pair when it
  arrives — additive, no migration to the workshop side. Named as a concept only; a shared Ruby
  concern is extracted on second use, not now.

## Related
- [[ADR-004 Multi-tenant foundation]] · [[ADR-006 Ownership separate from Employment]] ·
  [[ADR-007 Row-Level Security pulled into Sprint 1]] · [[ADR-009 Account deletion is refusal-first]] ·
  [[Data model]] · [[Event log]] · [[Sprint plan]]
