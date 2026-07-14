---
type: reference
updated: 2026-07-15
---
# Deferred design
Consciously parked — **revisit later**, not dropped. Each is additive (won't require rewriting v1).

- **Jobsheet per-answer snapshot (level-3)** — freeze `label`/`kind` onto each `JobSheetFieldValue`
  at fill time, so even a field *rename* can't change how an old sheet reads. v1 relies on
  lock-on-Done + add-only fields instead. Add if a compliance/dispute need appears.
- **`lock_version` (optimistic locking on Job)** — guards against silent lost-updates when two staff
  edit the same job's free-text at once. Deferred: audit trail covers stage; concurrent free-text
  edits are rare in a small trusted team. Add the column + a "stale, reload" handler if it bites.
- **Stage-change row lock in `JobActions` (2026-07-12)** — two people moving the same job's stage
  in the same second (e.g. SA cancels while tech marks Done) can both pass the allow-list check
  read on stale state. Deferred by the builder pre-Sprint-2: the append-only transition log makes
  the failure *loud* (two exits from the same stage, visible, reconstructable — never silent
  corruption), and most "concurrent editing" in real workflow is inserts into different tables,
  which can't collide. The fix is one line (`job.with_lock` around read-check-write) in the one
  door, so the cost of waiting doesn't compound. **Trigger to revisit:** a workshop reports a job
  that "changed twice at once", or the log shows two transitions out of the same stage.
- **Trapped last owner — the escape routes from account deletion (2026-07-14).** Since a
  workshop cannot exist without an Ownership (lifetime invariant, ADR-009), the **last owner
  of a workshop can never delete their account** while the workshop stands — that refusal is
  settled. What's parked is the *escape route*, a functional-design question: builder leaning
  = either **add another owner, then remove yourself** (needs an add-owner flow — none in v1)
  or **delete the workshop first, then the account** (needs workshop deletion — none in v1;
  and whether that's a real delete, soft delete, or disable is itself TBD). Pre-launch the
  trap has zero occupants; "contact support" bridges v1. Revisit when designing workshop
  lifecycle (close/transfer) — likely v2. See [[Risk ledger]] R1 / ADR-009.
- **PDPA / anonymization — the whole question (2026-07-14, builder ruling: after v1 is up).**
  Everything is additive (scrub columns, no schema change), so deferring costs nothing
  structurally. **Trigger: anonymization must exist before the first real workshop's data
  enters the system** — that's the moment Knot becomes a *processor* of someone else's
  customers, the heavier PDPA role. Two insights preserved so future-us doesn't re-derive:
  (1) **data user vs processor split** — for accounts (users/edges) Knot is the data user and
  requests land on us; for customers/vehicles/jobs the *workshop* is the data user and Knot
  processes on its behalf, so requests land on the workshop; (2) anonymization is therefore
  **two features**: Knot-side (scrub a User, keep the edge/history skeleton — the ADR-009
  path) and workshop-side (a crew-facing tool to scrub a Customer, keep the job history as
  business records — retention principle covers that). At trigger time also re-check the 2024
  PDPA amendments (breach notification, DPO, portability, processor liability) and add a
  privacy-notice line at signup. See [[ADR-009 Account deletion is refusal-first]].
- **Job↔vehicle customer-match validation (2026-07-15, builder: "a little too far — circle
  back").** Phase 1 planned a create-time rule: the job's stamped `customer_id` must equal
  `vehicle.customer_id` at registration (refuse "Lim's van billed to Speedy's file").
  Parked as **too rigid without a recovery path**: legitimate mismatches exist (borrowed car
  where the borrower pays, informal sale nobody filed, third party covering a repair), and a
  hard block at intake with the car on the lift is workflow poison. **The stamp itself
  stands** (gate 2, [[Data model]] §Resolved) — the door still *copies* the vehicle's
  customer by default; what's parked is only the law forbidding an explicit different
  choice. Circle back when the intake UI exists (Phase 4 / Sprint 6). **Recorded leaning
  (2026-07-15, designed with the builder): a two-branch SA-facing confirm, not a generic
  warning.** Happy path: registration number → vehicle → customer auto-fills (the door's
  default-copy), no warning ever. On mismatch: "vehicle filed under Lim, billing Tan" with
  two explicit choices — **[Just this visit]** (stamp only; file untouched — borrower/third-
  party payer) vs **[Vehicle changed hands]** (also `vehicle.update!(customer:)` in the same
  transaction — informal sale; future visits auto-fill right). Two buttons because the two
  stories need different writes; one generic "confirm" would leave stale files and train
  click-through. Mechanically: `register_job(customer: ...)` defaults to `vehicle.customer`;
  cue = ids differ. Schema untouched — Phase 1 already supports it. **Consequences audited
  2026-07-15:** (1) payer-with-no-vehicle became possible → Customer must restrict deletion
  on **jobs directly**, not just vehicles (patched into Phase 1); (2) no ownership-change
  log (accepted — the frozen stamps form a de facto timeline); (3) v2 nuance: a borrowed-car
  job is visible to the payer, not the vehicle's owner — person-inward being consistent;
  note it in the v2 design pass. See [[Job visibility]].
- **Token page × RLS — how the note reaches Postgres (2026-07-14).** The Sprint 7 status page
  is unauthenticated: no `app.user_id`, no `app.workshop_id` — under fail-closed RLS the page
  can't even look up the job by token (zero rows). Needs its own read key at build time: either
  a token-keyed `FOR SELECT` policy (`USING (token = current_setting('app.job_token', true))`,
  set before the lookup) or a `SECURITY DEFINER` lookup function. Purely additive — `token`
  is stamped at birth (S2.3). Decide at Sprint 7 kickoff. See [[Job visibility]].
- **Dark mode = steel-blue chrome (2026-07-05).** v1 ships light mode only (high-contrast, per
  device-posture decision). When dark mode comes, the dark theme's surfaces derive from the brand
  steel blue (~`#22456B` family) — chosen because the navy app-bar sample read as "dark mode" and
  was liked exactly for that. Keep status colors (gray/blue/red/amber/green) identical in both modes.

## Related
- [[M1-F1 Status flow and transitions]] · [[Blocker]] · [[Job]]
