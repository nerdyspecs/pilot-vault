---
type: reference
updated: 2026-08-14 (R5 register_job! reference corrected — ADR-013)
---
# Risk ledger
Engineering risks and known sharp edges, numbered. Promoted out of worklog narrative
(Session 15) — R-numbers were becoming load-bearing ("decide R5 at Phase 1 kickoff") while
living only inside Session 14's story. Worklog sessions **reference** entries here; this
table is the lookup. New finds get the next number; nothing is ever renumbered.

Severity: **high** = wrong data / broken invariant possible · **med** = ugly failure
(unrescued 500, confusing state) · **low** = cosmetic / info-leak of non-tenant data.

| ID | What | Severity | Status | Trigger to revisit |
|----|------|----------|--------|--------------------|
| R1 | **Account deletion undefined** — Devise `registerable` ships a live `DELETE /users`; `dependent: :destroy` would orphan workshops (breaks the lifetime Ownership invariant), erase employment history (breaks append-only), and FK-crash on any fired invitation | high | **fixed `0e135da`** — [[ADR-009 Account deletion is refusal-first]]; `User#bare?` gates a prepended `before_destroy`; matched invitations release to pending rather than block | — |
| R2 | **Invitation state machine unguarded in the model** — `accept!`/`decline!` carry their own RLS key (`SET LOCAL`), so row security can't catch misuse | med | **fixed `bd7e079`** — guards live inside the bypass; principle: *any method that bypasses RLS must enforce its own preconditions* | — |
| R3 | **GUC writes are interpolated SQL** — all sources are integers today (`.to_i` in controller; model ids), but the pattern invites a future string | low | accepted for v1 | harden (quote/`set_config`) before any non-integer GUC (e.g. Sprint 7's `app.job_token`) |
| R4 | **Double-accept race** — two concurrent `accept!` both pass the `fired?` guard; the DB's active-employment unique index holds (no corruption) but the loser gets an unrescued 500 | med | accepted for v1 | a real user reports it, or when a global error-page pass happens |
| R5 | **"One active job per vehicle" undecided** — allow or refuse a second open job on the same vehicle; if refuse, a partial unique index on `jobs(vehicle_id) WHERE stage IN (0,1,2)` | high (schema-shaping) | **fixed `2c5ca91`** — ruled *refuse* at Phase 1 kickoff (a Job is per-visit by definition); `index_jobs_one_active_per_vehicle` live, partial unique on `jobs(vehicle_id) WHERE stage IN (0,1,2)`; follow-up-after-Done proven legal in tests. **⚠ MOVED + INVERTED 2026-08-03 by [[ADR-012 Intake-Job two-level aggregate]]** — the guard now lives on `intakes` as `index_intakes_one_open_per_vehicle` (partial unique, `WHERE status = 0`); the *ruling* stands (a vehicle can't be in two **visits** at once) but what it refuses flipped: a second **job** per vehicle is now legal and expected (parallel repairs — the split's whole point), a second **open intake** is the violation. Don't go looking for the old `jobs` index — it's gone. **⚠ 2026-08-14, [[ADR-013 The door decomposed]]: `register_job!` (mentioned above as "opens-or-finds") no longer exists.** The guard itself is unaffected — it's a DB constraint, not app code — but the *resolution* moved: `CreateIntake` opens a visit, `CreateJob` adds a repair to a given intake, and the caller decides which to call. Nothing "opens-or-finds" in production anymore. | — |
| R6 | **Invitation email display drift** — invitation stores the email as typed at invite time; a renamed account shows the stale address on the crew page | low | accepted for v1 | cosmetic pass |
| R7 | **Duplicate fired invitations for one person** — uniqueness is `(workshop, email)`, not `(workshop, user)`: user changes email → admin re-invites under the new one → invitee sees two Accept cards for the same workshop; accepting the second → unrescued 500 (R4's family) | med | open (backlog) | likely fix: partial unique index `(workshop_id, user_id) WHERE status = fired` — decide deliberately, with R4's rescue |
| R8 | **Role-swap under an open Accept card** — admin re-fires with a different role while the invitee's card is on screen; invitee accepts a role they never saw | low | accepted for v1 | complaints, or an invitation-details page |
| R9 | **Accept vs concurrent destroy** — admin removes the invitation as the invitee clicks Accept → `RecordNotFound` 404 | low | accepted for v1 | error-page pass (with R4) |
| R10 | **Account-enumeration oracle in the invite flow** — the two flash messages ("awaiting their acceptance" vs "no account yet") let any workshop manager probe whether an email has a Knot account. Manager-gated; `users` is thin (ADR-006) so the leak is existence only. Devise stock flows have cousins (password-reset messaging) | low | accepted for v1 | pre-launch security pass; align wording with whatever Devise enumeration hardening is chosen |

## Related
- [[Decisions]] · [[Deferred design]] (parked *design*, vs parked *risk* here) · worklog Sessions 14–15 (origin stories for R1–R10)
