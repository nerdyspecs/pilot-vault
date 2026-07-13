---
type: open-questions
module: M1
updated: 2026-07-13
---
# Module 1 — Open questions
Feature-level questions to resolve during feature design. These are **not** architecture
decisions — they're details of how a feature behaves.

- **Owner notification channel** — v1: generate a **copy-paste message** the service advisor sends
  manually (whichever channel is free/available). WhatsApp Business API vs email automation is
  **deferred — circle back later**; decide during the **intake feature** design.
- **Single vs multiple assignees** — ✅ resolved: `JobMechanic` supports multiple (one `primary` +
  optional helpers). **Circle back** — the primary/helper distinction itself is still informal
  and may need its own permissions later.
- **One active job per vehicle** — leaning yes: partial unique index on
  `jobs(vehicle_id) WHERE stage NOT IN ('delivered', 'cancelled')`. Note: a job sitting at
  **Done-but-not-Delivered** still counts as active, so "reopen = new job" only works once the
  first job is Delivered.
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
