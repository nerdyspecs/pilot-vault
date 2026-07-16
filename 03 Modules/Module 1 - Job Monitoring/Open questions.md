---
type: open-questions
module: M1
updated: 2026-07-17
---
# Module 1 — Open questions
Feature-level questions to resolve during feature design. These are **not** architecture
decisions — they're details of how a feature behaves.

- **Owner notification channel** — v1: generate a **copy-paste message** the service advisor sends
  manually (whichever channel is free/available). WhatsApp Business API vs email automation is
  **deferred — circle back later**; decide during the **intake feature** design.
- **Single vs multiple assignees** — ✅ resolved *(re-ruled 2026-07-16, supersedes the earlier
  "one `primary` + optional helpers" answer)*: v1 is **single mechanic per job, no flag at all** —
  S2.6 shipped `JobMechanic` without the column. When helpers arrive the flag lands as **`lead`**
  (not `primary` — naming settled to avoid re-litigation). See [[Deferred design]] (crew entry) +
  [[M1-F1 Status flow and transitions]] Settled 2026-07-16.
- **One active job per vehicle** — ✅ resolved 2026-07-15 ([[Risk ledger]] R5, commit `2c5ca91`),
  **the other way from the old leaning**: active = per-visit — the shipped partial unique index is
  `jobs(vehicle_id) WHERE stage IN (0,1,2)` (registered/assigned/in_progress), so a **Done or
  Delivered job does NOT block a new job**. A follow-up job after Done is legal (proven by a
  passing test) and [[Intake flow]] step 1c depends on it. *(Stale leaning corrected 2026-07-17 —
  the old text described `WHERE stage NOT IN (delivered, cancelled)`, the opposite of what shipped.)*
- **Full attribute audit trail (e.g. paper_trail gem)** — leaning **later**; `JobStageTransition` + the jobsheet cover v1.
- **Blocker taxonomy** — ✅ resolved: no fixed taxonomy. `Blocker` is a **workshop-owned catalog**
  (`label`, `raised_by_role`, `cleared_by_role`); seed **"Hold For Payment"**. The workshop defines
  its own list rather than us guessing one.
- **Bootstrap onboarding** — ✅ superseded twice since this was written: signup creates the
  *person only*; "create workshop" is a post-signup act creating `Workshop` + `Ownership`
  ([[ADR-006 Ownership separate from Employment]]); crew joins via a fired invitation the
  invitee must accept ([[ADR-008 Crew joining requires acceptance]]). *(Stale text fixed
  2026-07-13 — it predated ADR-006 and was missed in both revision sweeps.)*

Broader **product-design** gaps (ETA, aging jobs, partial-adoption, floor access, photos, etc.) —
parked in [[Product gaps]] for a future review pass.
