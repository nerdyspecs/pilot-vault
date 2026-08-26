---
type: context
updated: 2026-08-26 (Session 37 — the no-build-step rule is reversed; Bootstrap compiles from Sass)
---
# Tech Stack
Use alongside [[Builder identity]], [[Product overview]], [[User stories]] at the start of every Claude session.

---

## Backend
- Ruby on Rails (primary framework) — **vanilla MVC**: models / controllers / views / routes, no
  extra layers. The one exception is a **service object** for all job state changes (ONE DOOR,
  a plain PORO) — see [[Design laws]] #7.
- PostgreSQL — relational data, need joins and structured relationships
- Devise — authentication for internal workshop users
- Rails API mode aware — controllers should be structured to serve both HTML views and JSON responses cleanly

## Frontend — current
- Hotwire / Turbo + Stimulus for the internal workshop UI
- Rails views in **Slim** (`slim-rails` gem, chosen 2026-07-06; Devise's generated views stay
  ERB until restyled) — no separate frontend build step
- Owner-facing job status page — lightweight, mobile-friendly, no login required
- Visual system (palette, typography, UI laws): [[Design system]] — **Bootstrap 5.3.3** base
  (~~vendored CSS, no build step~~, Bootstrap JS not loaded) + a brand layer of CSS custom
  properties; chosen 2026-07-06 for expansion speed over the earlier no-framework stance
  - **⚠ 2026-08-26 (Session 37): "no build step" is REVERSED, not narrowed.** Bootstrap now
    compiles from Sass source — the `bootstrap` gem (pinned 5.3.3, the same version) plus
    `dartsass-rails`, building `app/assets/stylesheets/application.scss` into
    `app/assets/builds/application.css`, which is gitignored. `bin/setup` builds it and
    `assets:precompile` runs it on deploy, so no new deploy step is needed; `dartsass-rails`
    ships its own Dart Sass binary, so **no Node**. Why: theming through `--bs-*` custom
    properties only reaches what Bootstrap chose to expose, and six of the hardcoded values
    have no variable behind them — `.form-check-input:checked` among them, which we use.
    Setting Sass variables before the import reaches all of them, so the brand layer stops
    patching Bootstrap after the fact. Cost: seven gems (two direct, five transitive).

## Device posture (per role) — v1
Web app, standard login (Devise) on whatever device — phone browser or PC. No native app, no
special floor-device / PIN mechanism yet ("as is" for now — revisit later).
- **Technician** — phone-first. Screens must be **absolutely mobile-friendly**: large, well-spaced,
  straightforward controls. No small or tightly-packed buttons.
- **Service advisor / front office** — PC-primary. Responsive mobile comes later, not a v1 priority.
- **Workshop owner / manager** — PC-primary, but the mobile view must convey enough to check
  workshop **progress & health** on the go.
- **Vehicle owner** (external) — token link, mobile-first, no login (see [[ADR-001 Core stack]]).

## Frontend — future proofing
- All business logic lives in the backend — never in views or Stimulus controllers
- Every action that mutates data must be available as a JSON API endpoint, not just an HTML form
- This allows a React frontend or mobile app to connect to the same API later without rewriting the backend
- Do not couple logic to the view layer — if it can't be called from an API, it's in the wrong place

## Hosting & infrastructure
- Render or Heroku — PaaS, simple deploy, no server management
- PostgreSQL managed instance via the same platform
- Keep infrastructure boring — no Kubernetes, no Docker complexity for now

## Notifications
- **Staff (in-app):** settled — "waiting on whom" is a query over unacknowledged handoffs surfaced on the board, **no inbox and no separate channel** ([[ADR-005 Acknowledged handoffs in V1]], [[ADR-011 Acknowledgement as stored visibility]]).
- **Owner:** OPEN — how job-status updates reach the vehicle owner isn't decided (v1 = a manual copy-paste message; SMS / WhatsApp / email automation later). Decide before building the intake flow.

## Gems — guiding rule
- Default to what Rails provides before adding a gem
- Every gem must justify its presence — what does it do that Rails can't?
- No gem suggestions without a clear reason why the built-in is insufficient

## What I am not using
- No React or separate SPA for now — Hotwire handles the internal UI
- No GraphQL — REST is sufficient and simpler
- No background job complexity for v1 — keep it synchronous unless there is a clear need
- No Redis for now — revisit if real-time features require it

## Local dev credentials
Kept here rather than in the app repo's `CLAUDE.md` so no password enters the code history.
- **Dev DB throwaway login:** `preview-check@test.local` / `proofdrive123`
- Seeded personas (`db/seeds.rb`, all roles): password `seedpass123`
