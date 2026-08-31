---
type: concept
module: M1
updated: 2026-08-14 (re-anchored to Intake — ADR-012; token + customer_id moved off Job)
---
# Job visibility
Who can see (and touch) rows in the `jobs`/`intakes` tables, and which RLS key serves each of
them. Written down so nobody has to hold it in their head (Session 15 discussion).

> [!warning] Re-anchored 2026-08-14 by [[ADR-012 Intake-Job two-level aggregate]]
> This note was written when `jobs.customer_id` and `jobs.token` existed — both moved to
> **`intakes`**. The claimed-owner and token-holder read policies below now key on the
> **intake**, not a specific repair: a token holder or a claimed owner wants their **car's
> visit**, which may have several repairs under it, not one arbitrary `job_id`. The reasoning
> is otherwise unchanged; every mention of `jobs.customer_id` / `jobs.token` below reads as
> `intakes.customer_id` / `intakes.token`. This note is Sprint 7's design input — build against
> the corrected version, not the original.

**The reframe:** "only see jobs related to them" is not one rule — *related* means a
different relation per party (an employment edge, a claim on a customer record, possession
of a token link), and each relation gets its **own RLS policy**, on `jobs` for crew and on
`intakes` for everyone else. Postgres ORs permissive policies: a row shows if *any* key turns.
Everything here extends the pattern already shipped on the edge tables
([[ADR-007 Row-Level Security pulled into Sprint 1]]: `tenant_isolation` + `own_rows`).

## The six scenarios — the complete map
There is no seventh. Anyone touching `jobs`/`intakes` is one of these:

| # | Who's asking | GUC notes set | Policy that answers | Sees | When |
|---|---|---|---|---|---|
| 1 | **Crew** inside their workshop | `app.user_id` + `app.workshop_id` | `tenant_isolation` (on both tables) | whole board, read **+ write** | **Sprint 2** |
| 2 | Signed-in user, no workshop picked (landing) | `app.user_id` only | none | zero rows — correct; landing reads edges, not jobs | live |
| 3 | **Private owner**, logged in, claimed records | `app.user_id` only | `owner_read` (SELECT, on `intakes`) | intakes stamped with their claimed customer records, cross-workshop | v2 |
| 4 | **Company member** (fleet manager) | `app.user_id` only | `company_read` (SELECT, on `intakes`) | intakes on customers their company claims, cross-workshop | v2 |
| 5 | **Token holder**, unauthenticated | a token note (design TBD) | `token_read` (SELECT, on `intakes`) | exactly one intake — the visit, all its repairs | Sprint 7 |
| 6 | **Stranger / attacker / buggy code** | wrong or none | none | zero rows, silently | always |

**The walk-in is not a scenario** — it's a *state of the customer record* (unclaimed: no
`user_id`, no `company_id`; most customers, forever). Walk-ins never query. Their visits are
reached only through rows 1 and 5 — crew board + token link — the paper status quo, preserved.

## The policy stack (end-state sketch)
```sql
-- ═══ SPRINT 2 — the only one built now (jobs itself; intakes' own tenant_isolation ships
--     with S4.5.2, same shape) ═══
CREATE POLICY tenant_isolation ON jobs                        -- all commands
  USING (workshop_id = <app.workshop_id note>);

-- ═══ SPRINT 7 — the token status page (now on intakes, not jobs) ═══
CREATE POLICY token_read ON intakes FOR SELECT                -- (or SECURITY DEFINER lookup fn)
  USING (token = <app.intake_token note>);

-- ═══ v2 — claimed identities (now on intakes) ═══
CREATE POLICY owner_read ON intakes FOR SELECT
  USING (customer_id IN (SELECT id FROM customers
                         WHERE user_id = <app.user_id note>));

CREATE POLICY company_read ON intakes FOR SELECT
  USING (customer_id IN (SELECT c.id FROM customers c
                         JOIN company_employments ce ON ce.company_id = c.company_id
                         WHERE ce.user_id = <app.user_id note>));
```
*(The two **global** tables — `workshops`, `users` — carry no RLS at all: deliberate, and
load-bearing for sign-in and claim-verification. Reasoned in
[[ADR-007 Row-Level Security pulled into Sprint 1]]'s 2026-07-14 footnote.)*

Four keys, ORed. Only `tenant_isolation` opens writes (`FOR SELECT` policies are never
consulted for writes) — and even then, state changes go through a **door**, per level
([[Architecture laws]] #7, [[ADR-013 The door decomposed]]). Every read key resolves **from the
person inward**: start at the `app.user_id` note, walk the claim (`customers.user_id` direct,
or `customers.company_id` → membership), test the intake's frozen `customer_id` stamp. The
stamp is the *anchor*, never the identity — identity lives on User/Company; the claim column on
customers is the link.

**The builder's formulation (2026-07-15, gate-2 confirmation — canonical, re-anchored
2026-08-14):** *a person's visits are queried from the person inward through the frozen stamp;
the vehicle is supporting information derived from the intake, never the path.* Person → claim
→ `intakes.customer_id`; `intake.vehicle` rides along as display data, and `intake.jobs` rides
along beneath it. The privacy cut works both directions: after a sale, the old owner keeps
their own service history and never sees the new owner's visits — and vice versa. The
vehicle-path query (`vehicle.intakes`, all visits regardless of payer) remains the *crew's*
question, asked inside the workshop — same tables, different perspective, both true at once.

## Why this doesn't need re-litigating per sprint
1. **Fail-closed default.** Any party without a key sees zero rows — forgetting someone
   hides, never leaks. (Scenario 6 needs no policy; it's what no-key-turns looks like.)
2. **Additive, never rework.** Sprint 7 and v2 are each one `CREATE POLICY` migration. No
   existing policy changes, no Rails changes, no schema change — `token` and `customer_id`
   are stamped from birth (now on `intakes`, per ADR-012).
3. **One anchor.** Every future read key hangs off `intakes.customer_id` (frozen at intake —
   [[Data model]] §Resolved, the sold-vehicle decision, now applying to Intake) or
   `intakes.token`. A mutable anchor would let a vehicle sale silently re-aim a security policy.

## Open (deliberately unbuilt)
- **Sprint 7:** how the token note reaches Postgres (a `SET app.intake_token` before lookup vs
  a `SECURITY DEFINER` function) — flagged in [[Deferred decisions]].
- **v2 design pass:** the `USING` subqueries read other *policed* tables (policy-chasing-
  policy: SECURITY DEFINER helper vs denormalizing the claim), and the projection question —
  companies likely get a thin slice (stage/ETA/vehicle) across all their jobs, not `SELECT
  intakes` wholesale. See worklog Session 15 + [[Data model]] v2 notes.

## Related
- [[Data model]] · [[Job]] · [[Intake]] · [[ADR-007 Row-Level Security pulled into Sprint 1]] ·
  [[Architecture laws]] · [[ADR-004 Multi-tenant foundation]] · [[ADR-012 Intake-Job two-level aggregate]]
