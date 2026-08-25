---
type: reference
created: 2026-08-25
updated: 2026-08-25
---
# UI rollout — what each task builds

Working note for **[[Sprint plan]] Sprint 5A**. The sprint plan holds the tasks and the ticks;
this note says **what you actually build** for each one.

Written to be picked up cold. Every entry names files, components or style rules — not reasoning.
The reasoning lives in [[Design system]]; the rules you must satisfy are the seven in `CLAUDE.md`.

**Order is by fan-in** — most-depended-upon first ([[Sprint plan]] §Conventions).

---

## The component set

These are the only components. A screen is a recomposition of these; anything new gets added here
first.

| Component | Partial | Used on |
|---|---|---|
| Page head | `layouts/_page_head` | every app screen |
| Panel | `.panel` (CSS) | every app screen |
| List row | `shared/_list_row` | board, customers, visit, repair |
| Metric strip | `shared/_metric_strip` | board, home |
| Status chip | `jobs/_stage_badge` | board, visit, repair |
| Blocker item | `shared/_blocker_item` | visit, repair |
| Waiting pin | `jobs/_waiting_pin` | board |
| Side-nav item | `layouts/_sidebar` | shell |
| Eyebrow | `.eyebrow` (CSS) | everywhere |
| Inverted panel | `.panel-invert` (CSS) | board |

---

## 1 · Tokens

### S5A.1 — The brand layer
**Builds:** `app/assets/stylesheets/application.css`, rewritten.
**Styles:** colour tokens · `--bs-border-radius: 0` · `--bs-body-font-size: .875rem` ·
`--font-sans: Helvetica, Arial` · letter-spacing rules · the six component dimensions.
**Also fixes (same commit):** `.auth-badge`, `.wordmark`, `.hero-badge`, `.text-brand` — all read
the retired `--brand`.
**Source:** copy the `:root` block from `08 Experiments/knot-board-desktop.html`.
**Done when:** no `--brand` in the file · no raw hex in a component · **no spacing values at all**.

### S5A.2 — Flash messages
**Builds:** flash markup in `layouts/application.html.slim` and `layouts/auth.html.slim`, plus a
`.flash` style.
**Replaces:** `.alert-info` and `.alert-warning` (Bootstrap colours wearing reserved status hues).
**Done when:** no Bootstrap alert-colour class in any template.

---

## 2 · The shell

### S5A.3 — The frame
**Builds:** `layouts/application.html.slim` (rewritten as the grid only) ·
`layouts/_sidebar.html.slim` · `layouts/_topbar.html.slim`.
**Sidebar:** fixed, full viewport height, `14rem`, nav items + pinned account chip.
**Topbar:** `4.5rem`, **inside the content column** — not above the shell.
**Not `_navbar`:** in Bootstrap "navbar" means a horizontal top bar; we have both, so the name
misleads.
**Also fixes:** the `col-12 col-md-8 col-lg-6` on every screen (UI law 9 — content uses the canvas).
**Done when:** one page renders with sidebar + topbar + full-width content, and no `col-lg-6` remains.

### S5A.4 — Page head
**Builds:** `layouts/_page_head.html.slim`.
**Takes:** eyebrow, title, optional primary action. Bottom-aligned.
**Done when:** at least one screen renders through it.

---

## 3 · Rulings

### S5A.5 — Settle UI law 3
**Builds:** nothing. A decision recorded in [[Design system]] §UI laws.
**Question:** one solid primary per **screen**, or per **local area**?
**Why now:** it changes the layout of `jobs/show` (four primaries) and `home/index` (three), so it
must land before those screens are re-cut.

---

## 4 · Verification surface

### S5A.6 — Component inventory + `/design-preview`
**Builds:** `DesignPreviewController` · `design_preview/show.html.slim` · route `/design-preview`.
**Merges:** `intakes/_blocker_item` + `jobs/_blocker_item` → `shared/_blocker_item`.
**Done when:** every component in the table above renders on the page, in every state.

---

## 5 · The gate *(no shell dependency)*

### S5A.7 — Sign in + sign up
**Pages:** `devise/sessions/new` · `devise/registrations/new`.
**Archetype:** the gate. **Layout:** `layouts/auth.html.slim` (unchanged).
**Restyle, not new** — both exist, styled to the old theme.

### S5A.8 — Forgot + reset password
**Pages:** `devise/passwords/new` · `devise/passwords/edit`. Same archetype.

---

## 6 · Screens into the shell

Each is a **re-cut**, not a rebuild: same controller, same data, new layout and components.

### S5A.9 — The board
**Page:** `workshops/show`.
**Components:** page head · metric strip · panel · list row · status chip · waiting pin.
**Reference:** `08 Experiments/knot-board-desktop.html`.

### S5A.10 — The visit and the repair
**Pages:** `intakes/show` · `jobs/show`.
**Components:** page head · panel · list row · status chip · blocker item.
**Also:** applies S5A.5's ruling to `jobs/show`.

### S5A.11 — Customers
**Pages:** `customers/index` · `customers/show` · `customers/new` · `customers/edit`.
**Components:** page head · panel · list row · search toolbar · Bootstrap form fields.
**Rule:** "New customer" reachable only through an empty search result.

### S5A.12 — Crew and blocker types
**Pages:** `workshop_staff/index` · `blockers/index` · `invitations/new`.
**Components:** page head · panel · list row · Bootstrap form fields.

### S5A.13 — Home
**Pages:** `home/index` signed-in (workshop picker, invitation cards) · signed-out marketing page.
**Also:** applies S5A.5's ruling to `home/index`.
**Marketing page is last** and may be dropped from this phase.

---

## 7 · Tests

### S5A.14 — Tests
**Builds:** `test/application_system_test_case.rb` + `test/system/` — **if** the Capybara layer is
un-suspended. **Open call, not ruled.** Capybara and selenium-webdriver are still in the `Gemfile`,
so it costs one harness file and no new dependency.

---

## Definition of done — every screen

The seven rules from `CLAUDE.md`. Tick all seven or the screen is not done.

| # | Check |
|---|---|
| 1 | Bootstrap component used where Bootstrap ships one — no hand-rolled input, button, checkbox, select, table |
| 2 | Spacing only from Bootstrap's ladder, in the template — none in `application.css` |
| 3 | Type only from the six steps · display weight 400 · tracking in `em` |
| 4 | Colour only from tokens — no raw hex |
| 5 | Square · 1px borders · no shadow on inline structure |
| 6 | Exactly one solid primary action |
| 7 | Sits on a recorded archetype — the app page, or the gate |

## Related
[[Design system]] · [[Sprint plan]] · [[Screen map]] · [[UI experiments]] ·
[[Bay system reference — external comparison]]
