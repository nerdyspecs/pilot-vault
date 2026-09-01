---
type: reference
created: 2026-08-25
updated: 2026-09-01 (S5A.9 re-cut against UI rules 8–23 — still TBD; prior: S5A.3a; page head is a pattern)
---
# UI rollout — what each task builds

Working note for **[[Sprint plan]] Sprint 5A**. The sprint plan holds the tasks and the ticks;
this note says **what you actually build** for each one.

Written to be picked up cold. Every entry names files, components or style rules — not reasoning.
The reasoning lives in [[Design system]]; the rules you must satisfy are the seven in `CLAUDE.md`.

> [!important] One fact, one place
> **Status lives only in [[Sprint plan]]** — never add checkboxes here, or the two will disagree
> about what is built and neither will be trustworthy. **Spec lives only here** — the sprint plan
> names a task, it never re-describes it. **Rules live only in [[Design system]].**
> Read this note when you pick up a task; read the sprint plan to know which task.

**Order is by fan-in** — most-depended-upon first ([[Sprint plan]] §Conventions).

---

## The component set

These are the only components. A screen is a recomposition of these; anything new gets added here
first.

| Component | Partial | Used on |
|---|---|---|
| Page head | **pattern, not a partial** — see S5A.4 | every app screen |
| Panel | Bootstrap `.card` — *but the board uses none (rule 12: whitespace before boxes)* | forms, gate |
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

### S5A.3a — Sidebar contents
**Builds:** the inside of `layouts/_sidebar.html.slim` (the frame ships empty from S5A.3).
**Nav — one board for everyone, four items, in this order:**

| Item | Path | Shown to |
|---|---|---|
| **Board** | `workshop_path` | every active staff member |
| Customers | `customers_path` | `counter_staff?` |
| Crew | `workshop_staff_index_path` | `Current.owner? \|\| Current.workshop_manager?` |
| Blocker types | `blockers_path` | `Current.owner? \|\| Current.workshop_manager?` |

**Gating:** reuse the predicates `workshops/show` already uses — same helpers, same conditions.
Do not write a new permission query in the sidebar ([[Agent guide]]; `PermissionsHelper` delegates
to `Permissions` so the page can't offer what the controller refuses).
**Label:** the top item reads **Board**, never "Dashboard" ([[Design system]] §L1).
**Removes:** the three outline buttons in the header row of `workshops/show` — they become nav.
**Style:** 42px rows · selected = full-bleed solid `--action`, not a tint · account chip pinned to
the bottom.
**Carries the open call:** the icon source for the collapsed rail (UI law 6 — a rail of unlabelled
glyphs needs real icons, or the collapse gives up its labels for nothing).
**Done when:** every app screen is reachable from the sidebar · a technician sees one item, an
owner four · no `link_to "Customers"` remains in `workshops/show`.

### S5A.4 — Page head
**Builds: nothing.** *(Ruled 2026-08-27, builder: **not a partial.** It is written inline in each
page, for developer readability — a maintainer opening `workshops/show.html.slim` sees the whole
page in one file. It was briefly `layouts/_page_head`, then `shared/_page_head`, then inlined.)*

**So this block is the canonical source — copy it.** Duplicated markup with no single source is
how ten screens ended up with two different column widths; the source of truth is here instead of
in a file.

```slim
.d-flex.justify-content-between.align-items-end.pb-3
  div
    .eyebrow= <eyebrow>
    h1.display-6.mb-0 <title>
  div
    / optional action — omit the div entirely on a monitoring screen
```

**The two details that drift if hand-typed:** `align-items-end` is a **spec** — the action sits
level with the title's *baseline*, not its cap — and the title is `.display-6` at **weight 400**,
never bold.
**Done when:** at least one screen renders it, and this block matches that screen.

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

> [!warning] **A first cut is built and is TBD** *(2026-08-27, branch `s5a-sass`, uncommitted)*
> Spec below describes what exists so it can be reacted to. Reasoning and what is deliberately
> missing: [[Board]].

**Structure** *(re-cut 2026-09-01 against [[UI rules]] 8–23)*: `page head → .split.gap-4(main,
aside)`. **No `.card`** — rule 12. Sections are uniformly *head (`.fs-5` + right-side `.tabular`
count) → body*, so the metric strip sits **inside** the *In the shop* body rather than being a
headless section.
**Main:** four `.flex-fill.p-3` tiles in a `.border-top.border-bottom` strip, divided by
`.border-start`; then a real `table.table.table-hover` with `.ps-0`/`.pe-0` on the outer cells so
every left edge lines up (rule 8). Whole-row target = `tr.position-relative` + `a.stretched-link`,
no JS.
**Rail:** `.list-group.list-group-flush` of `intakes/_board_row` — the row is the **car**, so there
is no single stage: one `jobs/_stage_badge` per repair, plus `jobs/_waiting_pin`. Replaces
`jobs/_board_row`, now dead.
**Controller provides:** `@intakes` · `@ready` · `@waiting` · `@pending_acknowledgements` ·
`@job_counts` · `@crew_load`.
**CSS added by this task:** `.split` · `.text-quiet` / `.text-faint` (rule 13's ramp; Bootstrap
ships only two of the four) · `.tabular` (rule 18) · `--bs-table-hover-bg` and
`--bs-list-group-action-hover-bg` themed to `--hover` (rule 16). No spacing values, no raw hex.

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
