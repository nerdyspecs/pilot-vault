---
id: ADR-001
type: decision
status: accepted
date: 2026-06-30
superseded_by:
---
# ADR-001 — Core stack: Rails monolith + Hotwire + Postgres

This is the founding technical decision — stack, auth, and hosting, chosen together at the
start for the same reasons. It is not edited. Any later change gets its own ADR that
references and supersedes the relevant part of this one.

## Decision
Build the product as a single Rails monolith. Use Hotwire (Turbo / Stimulus) for the UI,
while keeping every data mutation available as a JSON endpoint. Store data in PostgreSQL.
Authenticate staff with Devise; vehicle owners get a no-login token link (`/jobs/:token`).
Host on a PaaS (Render or Heroku) with managed PostgreSQL.

*(For the plain "what we use" list, see [[Tech stack]]. This note is the **why**.)*

## Why / the tradeoffs we accepted
- **Monolith, not microservices** — solo builder, early stage. The bottleneck is one
  person's time to build, debug, and deploy, not scale. Microservices would add
  coordination cost with no payoff yet.
- **Hotwire, not a separate React SPA** — no separate frontend build pipeline means
  shipping faster. Tradeoff accepted: rich client-side interactions are harder than in
  React. We mitigate that by keeping all mutations available as JSON, so a React or mobile
  client can be added later **without rewriting the backend**.
- **PostgreSQL, not SQLite or Mongo** — the data is relational (jobs → vehicles → owners,
  jobs touch multiple staff roles). We need joins and foreign keys. Tradeoff: marginally
  more setup than SQLite; accepted.
- **Devise, not hand-rolled auth** — staff need real accounts; Devise is the standard,
  well-maintained Rails solution, so rolling our own adds risk for no gain. Owners get a
  unique token link instead of an account — they shouldn't have to sign up to see status.
- **PaaS (Render / Heroku), not self-managed servers** — solo builder, no DevOps time; PaaS
  handles servers, SSL, and deploys. Tradeoff: less control, higher per-unit cost at scale;
  accepted for now. *This is the most likely of these to change later — if it does, it gets
  its own superseding ADR.*

## Consequences (what this commits us to)
- One Rails app, one repo.
- Controllers respond to both HTML and JSON.
- Business logic lives in models and service objects — never in views or Stimulus.
- Use ActiveRecord associations fully. No document store.
- Devise covers internal users only; owner access is unauthenticated but scoped to one job
  via its token.
- No Docker, no Kubernetes, no Nginx config. Deploy is a git push.

## Related
- [[Tech stack]] — the "what"
- [[Decisions]] — index of all decisions
