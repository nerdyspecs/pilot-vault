---
id: ADR-009
type: decision
status: accepted
date: 2026-07-14
supersedes:
superseded_by:
---
# ADR-009 — Account deletion is refusal-first (derived, not chosen)

From the Session 14 risk review (R1) and the Session 15 edge discussions. Devise's stock
`registerable` ships a live `DELETE /users`; nobody had decided what deletion *means*, so it
meant `dependent: :destroy` accidents: an ownerless workshop, erased employment history, and
an FK crash on any fired invitation. This ADR decides it.

## The two principles (stated here, deliberately)
1. **A workshop cannot exist without an Ownership — a lifetime invariant.** ADR-006 §2 stated
   the birth-time rule (workshop + Ownership created in one transaction); the builder
   strengthened it (Session 14) to hold for the workshop's whole life. Every future lifecycle
   feature — deletion, transfer, close, the v2 Company governance edge — must obey it.
   *(Placement ruled by the builder, Session 15: this is a design decision, so it lives here
   in the ADR — not in [[Design laws]].)*
2. **History is append-only.** Employment rows (who worked where, when), ownership records,
   and — from Sprint 2 on — job/transition history are records of *what happened*, often
   about people other than the account holder. [[Design laws]] #8 states this for jobs; the
   same principle covers the edges.

## Decision
**Deletion is refused for any account holding edges or history; bare accounts delete freely.**
- A user with **no** Ownership rows, **no** Employment rows (active *or* ended), and **no**
  non-pending Invitations may delete their account outright — destroying it erases nobody's
  record but their own.
- Any edge or history ⇒ **refusal with a reason**, not a cascade. The refusal is *derived*:
  deleting an owner would orphan the workshop (principle 1); deleting ex-crew would erase the
  workshop's employment history (principle 2). We are not choosing to be unhelpful — the
  invariants leave no other answer.
- Immediate corollary: **the last owner of a workshop can never delete their account while
  the workshop stands.** The escape routes (add-another-owner-then-leave vs
  delete-workshop-first; real vs soft delete vs disable) are functional design, parked —
  [[Deferred design]] (2026-07-14 entry). Pre-launch, "contact support" bridges.

## What this replaces in code
Devise's `registerable` destroy path (`DELETE /users` → `resource.destroy`) and the
`dependent: :destroy` on `User`'s edges must go behind a guard: bare → destroy; else →
refuse with the derived reason. (~10 lines; closes [[Risk ledger]] R1.)

## Explicitly deferred (on the record, not forgotten)
**PDPA / anonymization — the whole question — waits for v1 to be up** (builder ruling,
Session 15). No anonymization code, no processor-role design now: zero real users means zero
live obligation, and everything about it is additive (scrub columns, no schema change), so
nothing v1 builds can contradict it. Parked with a dated trigger in [[Deferred design]]:
**anonymization must exist before the first real workshop's data enters the system** — the
moment Knot becomes a *processor* of someone else's customers. The two insights worth not
re-deriving live in that entry (data-user vs processor split; anonymization is two features —
Knot-side for users, workshop-side for customers).

## Why refusal-first (alternatives rejected)
- **Cascade (`dependent: :destroy` as shipped)** — violates both principles at once; also
  factually broken today (invitations FK has no cascade → 500).
- **Soft-delete users (flag, keep row)** — solves nothing the guard doesn't, adds a shadow
  state every query must remember, and quietly *becomes* anonymization done badly.
- **Anonymize now** — right end-state, wrong time; see the deferral above.

## Consequences
- R1 closes with a small guard, not a feature build.
- `has_many :ownerships/:employments, dependent: :destroy` on `User` becomes wrong-by-decision
  (the guard makes it unreachable for non-bare users, but bare users still need their
  no-history rows cleaned — keep `:destroy` for those; the guard is the gate).
- The refusal message is user-facing product copy ("You own Ah Seng Auto — a workshop cannot
  be left without an owner") — honest, and it teaches the model.

## Related
- [[ADR-006 Ownership separate from Employment]] (§2 birth-time rule this strengthens) ·
  [[ADR-008 Crew joining requires acceptance]] (the invitation FK that crashes stock destroy) ·
  [[Risk ledger]] R1 · [[Deferred design]] (trapped-owner escape routes; PDPA trigger) ·
  [[Design laws]] #8

**2026-07-14 (S2.0b implementation) — invitations don't block deletion.** The Decision
above listed "no non-pending Invitations" as part of bare-ness; refined during
implementation review: an invitation is an **offer**, not a commitment — the commitment is
the Employment it may create, and that's what's actually checked (`employments.none?`).
Decisive case: [[ADR-008 Crew joining requires acceptance]]'s mistyped-email stranger must
never have their account held hostage by someone else's typo. `User#bare?` now reads
`ownerships.none? && employments.none?` only. A matched (fired or declined) invitation is
instead **released** on destroy — back to `user: nil, status: :pending`, via new
`Invitation#release!` — so the admin's record survives and the fact stays true ("no account
for this email yet"). An accepted invitation can never be in play here: accepting created an
Employment, which already blocks. Built + live-verified, commit `0e135da`.
