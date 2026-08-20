---
id: ADR-008
type: decision
status: accepted
date: 2026-07-12
supersedes: parts of ADR-006 (the passive add-crew clause / S1.11 "no invitee-facing UI")
superseded_by:
---
# ADR-008 — Crew joining requires acceptance (join bilateral, termination unilateral)

From the 2026-07-12 pre-Sprint-2 session. Revises how a person *joins* a workshop's crew: the
S1.11 flow added an active `Employment` the instant an admin entered a matching email — the crew
member was drafted with no say. This ADR replaces that with a **consent step**, and names the
asymmetry that makes it coherent. Does not touch [[ADR-006 Ownership separate from Employment]]'s
core (Ownership vs Employment split, thin User, one access door) — only its onboarding clause.

## Decision
1. **Joining a workshop is bilateral — the invitee must accept.**
   An admin adding a crew member no longer creates the `Employment` directly. It creates an
   **`Invitation`** (the name is honest again — it now genuinely invites); the active Employment
   is created only when the invitee **accepts**. A mistyped email that happens to match a real,
   unrelated account can never silently become someone's crew — they simply never accept.

2. **Leaving is unilateral — termination needs no acknowledgement.**
   `Employment#end!` stays as-is: the employer removes a crew member without the member's
   consent or ack. This is the deliberate **asymmetry** — you consent to *enter* an employment
   relationship, you don't get a veto on being *removed* from one. It mirrors real employment
   and keeps the [[ADR-005 Acknowledged handoffs in V1]] philosophy applied where it belongs
   (consent to bind, no consent to unbind).

3. **The `Invitation` is a small state machine** (status enum):
   - `pending` — no account exists for that email yet.
   - `fired` — an account exists (carries `user_id`); awaiting the invitee's accept.
   - `accepted` — invitee said yes; the active `Employment` was created; terminal.
   - `declined` — invitee said no; kept visible to the admin (not silently dropped); terminal.

4. **Discovery is pull, not push (no email in V1).** A user with a `fired` invitation sees it
   on their **personal landing page** ("You've been invited to {workshop} as {role} —
   Accept / Decline"), slotting into the existing edge-count landing routing (ADR-006 §4). No
   email infrastructure, no tokens.

5. **`pending → fired` is admin-triggered, not automatic.** When the invitee finally signs up,
   the admin re-fires from their crew page ("Invite again") — which now finds the account and
   flips the row to `fired`. Auto-firing at signup was **rejected**: a signup-time lookup for
   "pending invitations matching my email" is a cross-tenant, email-keyed query that Postgres
   RLS ([[ADR-007 Row-Level Security pulled into Sprint 1]]) is built to block — it would need a
   privileged bypass. Manual re-fire runs inside the admin's own workshop context, where
   `tenant_isolation` already permits the read. Cheaper and RLS-aligned.

6. **RLS: the invitee sees their own `fired` invitation via an own-rows policy.** Before
   accepting, the invitee has no edge in that workshop, so `tenant_isolation` alone hides the
   row (the same chicken-and-egg solved for edges in S1.12-pre). Add a second permissive policy
   on `invitations`: `own_rows FOR SELECT USING (user_id = app.user_id)`. `fired` rows carry
   `user_id`; `pending` rows (no account) need no invitee visibility.

## Why
- **Consent + mis-add safety.** Being added to a workshop means someone can assign you jobs and
  see you as their crew. That should require your yes, and the accept step makes a typo'd email
  harmless.
- **Philosophical coherence.** Knot's promise is that things don't happen to you unseen and
  unacknowledged; silently drafting someone contradicted that. The join/leave asymmetry resolves
  it without over-applying ack to removal.
- **Future onboarding room.** The accept step is now a *place* that can grow (NDA, rulebook
  acknowledgement, a company brief) rather than an instantaneous edge insert. V1 keeps it a bare
  Accept / Decline; the seam is left for V2.

## Consequences
- `Invitation` gains a `status` enum and a nullable `user_id` (set when `fired`); an `own_rows`
  RLS policy is added to the table.
- New invitee-facing surface — the **first** in the app: a landing-page accept/decline card.
  This is a deliberate departure from S1.11's "no invitee-facing UI"; **[[Positioning]] updated
  the same day** to distinguish the *adoption* veto (still passive — crew leverage is
  disengagement, ease beats persuasion) from *membership*, which is now consensual. The
  operational handshake (ADR-005) and this membership accept are **different handshakes** — keep
  them distinct in copy.
- Sprint 1's S1.11 is reopened/extended (invitation state machine + own-rows policy + accept
  screen) — resequenced in the [[Sprint plan]]. `Employment#end!` unchanged.
- The manual re-fire chore from S1.11 remains (deliberately — see Decision 5).

## Related
- [[ADR-006 Ownership separate from Employment]] · [[ADR-005 Acknowledged handoffs in V1]] ·
  [[ADR-007 Row-Level Security pulled into Sprint 1]] · [[Positioning]] · [[Sprint plan]]

---
**Footnote 2026-07-17 (Session 21) — edge models renamed, decision unchanged.** `Employment` →
`WorkshopEmployment`, `Ownership` → `WorkshopOwnership` (tables/FKs follow): organisation-prefixed
naming, mirroring the v2 fleet edges (`CompanyEmployment` / `CompanyOwnership`-style, one
reusable organisation─membership─governance pattern, never one table). Every clause of this
ADR reads with the new names; nothing else changes.

**Footnote 2026-08-19 (Session 35) — flagged for a future reopen; decision unchanged for v1.**
Both renamed edges above were themselves superseded by [[ADR-010 WorkshopStaff supersedes the edge split]]
(one `WorkshopStaff` row + append-only `WorkshopStaffRole`); this ADR's *joining* ruling is
untouched by that. Separately, Session 23 raised **QR self-enrollment** — an owner displays a
"register as driver / fleet manager" QR, and scanning auto-joins with a role — which would bypass
this ADR's bilateral invite-then-accept requirement. Recorded, not adopted: a QR is a bearer secret
with no identity proof, and auto-assigning a role on scan is exactly what Decision 1 refuses. The
full proposal and its three recorded risks live in [[V2 - Customer and company dashboards]]
§QR self-enrollment. **Reopen only at v2 kickoff, and only for Company crew** — workshop crew
joining stays bilateral.
