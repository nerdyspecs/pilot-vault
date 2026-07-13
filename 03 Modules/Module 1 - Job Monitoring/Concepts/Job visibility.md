---
type: concept
module: M1
updated: 2026-07-14
---
# Job visibility
Who can see (and touch) rows in the `jobs` table, and which RLS key serves each of them.
Written down so nobody has to hold it in their head (Session 15 discussion).

**The reframe:** "only see jobs related to them" is not one rule — *related* means a
different relation per party (an employment edge, a claim on a customer record, possession
of a token link), and each relation gets its **own RLS policy** on the jobs table. Postgres
ORs permissive policies: a row shows if *any* key turns. Everything here extends the pattern
already shipped on the edge tables ([[ADR-007 Row-Level Security pulled into Sprint 1]]:
`tenant_isolation` + `own_rows`).

## The six scenarios — the complete map
There is no seventh. Anyone touching `jobs` is one of these:

| # | Who's asking | GUC notes set | Policy that answers | Sees | When |
|---|---|---|---|---|---|
| 1 | **Crew** inside their workshop | `app.user_id` + `app.workshop_id` | `tenant_isolation` | whole board, read **+ write** | **Sprint 2** |
| 2 | Signed-in user, no workshop picked (landing) | `app.user_id` only | none | zero jobs — correct; landing reads edges, not jobs | live |
| 3 | **Private owner**, logged in, claimed records | `app.user_id` only | `owner_read` (SELECT) | jobs stamped with their claimed customer records, cross-workshop | v2 |
| 4 | **Company member** (fleet manager) | `app.user_id` only | `company_read` (SELECT) | jobs on customers their company claims, cross-workshop | v2 |
| 5 | **Token holder**, unauthenticated | a token note (design TBD) | `token_read` (SELECT) | exactly one job, thin slice | Sprint 7 |
| 6 | **Stranger / attacker / buggy code** | wrong or none | none | zero rows, silently | always |

**The walk-in is not a scenario** — it's a *state of the customer record* (unclaimed: no
`user_id`, no `company_id`; most customers, forever). Walk-ins never query. Their jobs are
reached only through rows 1 and 5 — crew board + token link — the paper status quo, preserved.

## The policy stack (end-state sketch)
```sql
-- ═══ SPRINT 2 — the only one built now ═══
CREATE POLICY tenant_isolation ON jobs                        -- all commands
  USING (workshop_id = <app.workshop_id note>);

-- ═══ SPRINT 7 — the token status page ═══
CREATE POLICY token_read ON jobs FOR SELECT                   -- (or SECURITY DEFINER lookup fn)
  USING (token = <app.job_token note>);

-- ═══ v2 — claimed identities ═══
CREATE POLICY owner_read ON jobs FOR SELECT
  USING (customer_id IN (SELECT id FROM customers
                         WHERE user_id = <app.user_id note>));

CREATE POLICY company_read ON jobs FOR SELECT
  USING (customer_id IN (SELECT c.id FROM customers c
                         JOIN company_employments ce ON ce.company_id = c.company_id
                         WHERE ce.user_id = <app.user_id note>));
```
*(The two **global** tables — `workshops`, `users` — carry no RLS at all: deliberate, and
load-bearing for sign-in and claim-verification. Reasoned in
[[ADR-007 Row-Level Security pulled into Sprint 1]]'s 2026-07-14 footnote.)*

Four keys, ORed. Only `tenant_isolation` opens writes (`FOR SELECT` policies are never
consulted for writes) — and even then, state changes go through `JobActions` on top
([[Design laws]] #7). Every read key resolves **from the person inward**: start at the
`app.user_id` note, walk the claim (`customers.user_id` direct, or `customers.company_id`
→ membership), test the job's frozen `customer_id` stamp. The stamp is the *anchor*, never
the identity — identity lives on User/Company; the claim column on customers is the link.

## Why this doesn't need re-litigating per sprint
1. **Fail-closed default.** Any party without a key sees zero rows — forgetting someone
   hides, never leaks. (Scenario 6 needs no policy; it's what no-key-turns looks like.)
2. **Additive, never rework.** Sprint 7 and v2 are each one `CREATE POLICY` migration. No
   existing policy changes, no Rails changes, no schema change — `token` and `customer_id`
   are stamped from birth.
3. **One anchor.** Every future read key hangs off `jobs.customer_id` (frozen at
   registration — [[Data model]] §Resolved, the sold-vehicle decision) or `jobs.token`.
   A mutable anchor would let a vehicle sale silently re-aim a security policy.

## Open (deliberately unbuilt)
- **Sprint 7:** how the token note reaches Postgres (a `SET app.job_token` before lookup vs
  a `SECURITY DEFINER` function) — flagged in [[Deferred design]].
- **v2 design pass:** the `USING` subqueries read other *policed* tables (policy-chasing-
  policy: SECURITY DEFINER helper vs denormalizing the claim), and the projection question —
  companies likely get a thin slice (stage/ETA/vehicle), not `SELECT jobs` wholesale. See
  worklog Session 15 + [[Data model]] v2 notes.

## Related
- [[Data model]] · [[Job]] · [[ADR-007 Row-Level Security pulled into Sprint 1]] · [[Design laws]] · [[ADR-004 Multi-tenant foundation]]
