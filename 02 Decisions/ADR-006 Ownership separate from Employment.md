---
id: ADR-006
type: decision
status: accepted
date: 2026-07-07
supersedes: parts of ADR-004 (role enum, bootstrap)
superseded_by:
---
# ADR-006 — User↔Workshop linking: Ownership edge + onboarding split

From the 2026-07-07 session (worked out interactively while styling auth pages — the sign-up
copy exposed the modeling question). Revises two clauses of
[[ADR-004 Multi-tenant foundation]]; the tenancy foundation itself (Workshop tenant, thin User,
per-request re-verification) is unchanged.

## Decision
1. **Ownership is its own edge, not an Employment role.**
   `Ownership` (`user_id`, `workshop_id`, unique per pair) = **governance**: the workshop
   belongs to this person — billing/subscription, transfer, delete. It has no operational role
   and no `ended_at` lifecycle.
   `Employment` = **operations only**: `role` enum shrinks to
   `technician / service_advisor / parts_advisor / workshop_manager` (**`owner` removed**).
   - The wrenching towkay (owner who also works the floor — common in target market) holds
     **both edges**: an Ownership *and* an Employment with a real operational role. No
     special-casing "owner can also take jobs" anywhere.
   - Multiple owners (partners) = multiple Ownership rows.
   - Naming: **not** `CompanyOwner` — "Company" is reserved for the v2 fleet-customer entity
     (ADR-004's `CompanyEmployment`).

2. **Onboarding splits: signup creates the person, not the workshop.**
   - Signup = thin `User` only ("Create your account"). Same page for founders and crew —
     a mechanic joining an existing shop no longer accidentally founds one.
   - "Create your workshop" is a **post-signup act**: `Workshop` + founder `Ownership` in
     **one transaction**. A workshop can therefore never exist without a user.
   - **Workshop is v1's only module.** A bare User is the hallway; edges decide which rooms
     exist (customer and company modules arrive in v2 as new edges, per ADR-004).

3. **Access = one door.** The per-request check (re-verified every request, per ADR-004) now
   has two sources: **active Employment OR Ownership**. This resolution lives in exactly one
   method that every controller/query path calls — never sprinkled as separate checks, or a
   screen will eventually forget one. Same spirit as ONE DOOR for job state
   ([[Architecture laws]] #7).

4. **Landing routes by edge count** (not a personal dashboard by default):
   `0` edges → personal home, primary action "Create your workshop" / "ask your boss to add
   you" · `1` workshop → straight to its dashboard (the daily case — crew ease beats
   ceremony, [[Positioning]]) · `>1` → context picker (unchanged from ADR-004). Menu always
   allows stepping back to the personal home. UI assumes ≤1 workshop per user in v1; the
   model already tolerates more.

## Why
- Governance and operations have different lifecycles, permissions, and truths on screen;
  merging them forced "owner" special-cases into job assignment, handoffs, and the board.
- Signup-creates-workshop made the second user's path (crew) incoherent.
- Rejected counter-argument, knowingly: the industry default (Slack/GitHub: owner = member
  with role) keeps access checks single-table. We pay the two-source check for honest
  modeling of the working owner; the single access door contains the risk.

## Consequences
- New model + migration: `Ownership`. Employment enum loses `owner`.
- Sprint 1 tasks revised (S1.2/S1.3/S1.6/S1.7) + new tasks: Ownership model, minimal
  add-crew flow, edge-count landing.
- Sign-up page copy: "Create your account" (fixed in app same day); "Create your workshop"
  becomes an in-app action.
- What can an Ownership *do* in-app in v1 beyond governance? Read-everything oversight is
  implied by device posture; exact permission surface to be settled when the board is built.

## Superseded in part
- **2026-07-12 — [[ADR-008 Crew joining requires acceptance]].** Decision 2's onboarding split
  stands, but the add-crew mechanic it implied (S1.11: an active `Employment` created the instant
  a matching email is entered, no invitee say) is replaced. Joining is now bilateral — the
  invitee accepts — while termination stays unilateral. The rest of this ADR is untouched.
- **2026-07-13 — §4 refinement (same ADR-008 consequence):** the 1-edge auto-route to the
  dashboard now also requires the user to have **no fired invitations** — a pending
  Accept/Decline card must be seen on the personal home, not skipped past. Built in S1.15c.

## Related
- [[ADR-004 Multi-tenant foundation]] · [[ADR-008 Crew joining requires acceptance]] · [[Architecture laws]] · [[Positioning]] · [[Sprint plan]]

---
**Footnote 2026-07-17 (Session 21) — edge models renamed, decision unchanged.** `Employment` →
`WorkshopEmployment`, `Ownership` → `WorkshopOwnership` (tables/FKs follow): organisation-prefixed
naming, mirroring the v2 fleet edges (`CompanyEmployment` / `CompanyOwnership`-style, one
reusable organisation─membership─governance pattern, never one table). Every clause of this
ADR reads with the new names; nothing else changes.

---
**Footnote 2026-07-21 — §1 superseded by [[ADR-010 WorkshopStaff supersedes the edge split]].**
The two-edge split (§1) is replaced: `WorkshopEmployment` + `WorkshopOwnership` collapse into
**one `WorkshopStaff`** record (an `owner` boolean discriminator) + append-only
`WorkshopStaffRole` rows. Reason: a global-`User` actor FK can't be tenant-checked at the DB;
pointing actors/holdings at the tenant-local person + a composite `(actor_id, workshop_id)` FK
makes cross-tenant actors a foreign-key violation — and the merge dissolves the polymorphic-actor
problem that forced actions onto `User` in the first place (Data model §Resolved, also
footnoted). **§2 (onboarding split), §3 (one-door access), §4 (landing-by-edge-count) all
stand**, now reading against `WorkshopStaff`; "owner not in the role enum" stands too — the
distinction moved onto the boolean, not back into a role. The v2 note holds with the shape
adjusted: one `CompanyStaff`/`CompanyStaffRole` pair, not two company edges.
