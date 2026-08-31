---
type: context
created: 2026-07-06
updated: 2026-08-31 (Session 39 — [[UI rules]] card created from this note, pointer added; prior: Session 38 — L1 fork ruled: one board for everyone, viewpoints are scopes; the Board's content is safe for every role; named the Board; front door moved off it, page pattern narrowed)
---
# Design system

> [!important] This note is the **single source of truth for everything design.**
> Renamed from *Visual theme* on 2026-08-25 (Session 36), when it absorbed the component
> inventory, navigation and IA, posture, the journey and the build order from [[Screen map]].
> **[[Screen map]] now says what screens *exist*; this note says how everything should look and
> behave, and what to build next.**
>
> A document alone drifts, so the truth is **three layers that must agree**:
> 1. **This note** — authoritative. Values here win on any conflict.
> 2. **`app/assets/stylesheets/application.scss`** — the tokens as CSS custom properties,
>    implementing it.
> 3. **`/design-preview`** — renders every component from those tokens, so drift becomes
>    *visible* instead of a doc-versus-code guess.
>
> Layer 3 is the one that makes the other two honest. It is [[Sprint plan]] **S5.5a**.
>
> **The working card is [[UI rules]]** *(2026-08-31, Session 39)* — the seven rules with their
> values, held open during UI work so this 900-line book doesn't have to be. This note stays
> authoritative: if the card disagrees, the card is wrong.
>
> **Scope: desktop only, for now** *(2026-08-25, builder — "for simplicity and sanity, everything on
> monitor screens first")*. This does **not** reverse the three-posture commitment; it confirms the
> desk-first sequencing already recorded in L2. UI law 5 (thumb-first) is **dormant, not deleted**,
> and the fluid-type note under §The scale is the first thing to re-check when the phone posture
> returns.
>
> **Bootstrap 5.3.3 is the base, and the default answer.** Where Bootstrap has a value, we take it;
> the brand layer holds only what Bootstrap cannot express — colour, negative tracking, the six
> component dimensions, and five components.

~~Locked 2026-07-06 (worked out interactively — samples compared, choices deliberate).~~
**Re-locked 2026-08-25 (Session 36)** on the visual language of the **Bay system reference**, so the
two read as one house style — see [[Bay system reference — external comparison]] for provenance and
the full adopt/reject triage. The 2026-07-06 lock's *structure* survives intact (one theme for both
clients, status colours as reserved words, tokens in one CSS file); what changed is the **surface**:
typeface, neutrals, geometry, and the shape of a status chip.

**Knot's identity is NOT part of the adoption** — the wordmark, the K tile, and brand steel blue
`#22456B` stay ours, and Bay's brand asset and the name "Bay blue" are deliberately not taken.
Bay's accent `#2727D9` was offered and declined; `--action` stays Knot's `#2D5E94`.

One theme for **both** clients (workshop staff + vehicle owner) — one predictable feel, no split identity.
Mood: **industrial & confident** · light · high-contrast. *(The mood is unchanged — Bay's
square-cornered, hairline-bordered, low-chroma system expresses it more literally than Bootstrap's
rounded defaults ever did.)*

## Color roles
Colors are named by **job, not rank** (no "primary/secondary" — a name should tell you what to
reach for). There is deliberately **no secondary hue**: a second decorative color would compete
with the status vocabulary. A "secondary button" is a *style* (outlined, quiet), not a color.

| Role | Token | Value | Job |
|---|---|---|---|
| Action | `--action` | **`#2727D9`** | primary buttons, selected nav, links, icons, live bullets |
| Action hover | `--action-hover` | **`#1F1FB0`** | hover/pressed |
| Page **and** surface | `--page` / `--surface` | **`#FFFFFF`** | canvas and panels are **both white** — hairlines do all the separating |
| Hover | `--hover` | **`#F4F4F2`** | the *only* job the grey has: row, tile and nav hover |
| Border | `--border` | **`#D8D8D4`** | container edges **and** grid dividers (metric strip) |
| Rule | `--rule` | **`#E5E5E1`** | dividers between rows *inside* a panel |
| Ink | `--ink` | **`#101010`** | body text, inverted panels, avatars |
| Muted | `--text-muted` | **`#55554F`** | row body copy |
| Quiet | `--text-quiet` | **`#6D6D68`** | sub-nav, counts, secondary labels |
| Faint | `--text-faint` | **`#777773`** | micro-type: eyebrows, timestamps, ages |
| Placeholder | `--placeholder` | **`#A9A9A4`** | input placeholder text only — never body copy |

> [!warning] `--brand` `#22456B` is **retired** *(2026-08-25, builder)*
> Knot's steel blue was kept in the first pass of the restyle, and the accent stayed at `#2D5E94`
> with Bay's `#2727D9` explicitly declined. **Both of those are reversed.** The builder leaned the
> design the rest of the way toward Bay, and a steel-blue wordmark beside an indigo primary button
> reads as broken — so the mark takes `--action` too. The wordmark, the K tile and the square
> avatar are Knot's; the *hue* is now shared. There is no separate brand colour.

**Two border weights, deliberately — not three.** The reference implementation uses three
(`#D8D8D4`, `#E8E8E5`, `#E5E5E1`); the last two are visually indistinguishable, so Knot standardises
on two. The distinction that *does* earn its keep: a metric strip's dividers use the **full**
`--border` so the strip reads as a grid, while a list's row dividers use `--rule` so a dense list
reads as a list. Same hairline logic, opposite jobs.

**The muted ink is a ramp, not one grey** — it steps down as type gets smaller. Recorded because
the first pass of this restyle had a single muted value, which is what made dense rows look flat.

**Changed 2026-08-25:** ~~Neutrals are not true grays — each carries a faint blue undertone so the
whole app reads as one family.~~ Neutrals are now **warm-achromatic**, in Bay's idiom: the family is
held together by *contrast and geometry* rather than by a shared undertone, and the only chroma in
the chrome is the brand/action blue. This is a real trade — the old rule bought a soft, unified
wash; the new one buys crispness and makes the status chips read louder against their surroundings,
which is what the status vocabulary is for.

> [!danger] **The canvas is white. An earlier note here said otherwise and was wrong.**
> The first pass recorded `--page: #F4F4F2` as "a deliberate, reasoned decline" of Bay's
> operational-canvas-is-white line, on the 2026-07-06 glare argument. **Checked against the
> shipped CSS on 2026-08-25 and that was a mistake:** the reference sets
> `.bay-app-shell { background: #f4f4f2 }` and then *overrides it* to `#fff` — the operational
> canvas has always been white, exactly as its own written spec said.
> The grey is not unused: it is the **hover** tone. Retaining it as the canvas gave it a job it
> never had and left hover with none.

Chrome style: **all-light** — white app bar and canvas, blue only in the mark and in actions.

Chrome style: **all-light** ("Option A") — white app bar, blue only in wordmark + actions.

## Status colors (SACRED — semantic, never decoration)
The product's real color language; a manager reads color before text. **Identical in every view** —
workshop, owner page, future dark mode.

**Re-derived 2026-08-25 (Session 36), builder ruling.** Two things forced it: the surface is now
pure white with `#101010` ink (the old pairs were mixed against `#F5F6F8` / `#1C2B3A`), and Bay's
geometry gives every chip a **visible 1px border**. That second point changes the *structure* of a
badge, not just its values:

> ~~Badge = pale bg + dark text of the same family.~~
> **Badge = 1px border + pale fill + dark text, all of the same family** — square-cornered.
> With the border carrying the edge, the fill can lighten; the border becomes the hue's mid-strength
> anchor and does the work the heavier fill used to do.

**The hue families do not move.** That is what keeps the reserved words reserved — red is still red,
amber still amber. Only the values are re-derived.

| Status | border | fill | text |
|---|---|---|---|
| Registered / neutral | `#D8D6CE` | `#F6F5F1` | `#3A3A37` |
| In progress / info | `#A9C7E6` | `#EDF4FB` | `#0A3A6B` |
| Waiting / aging | `#E2C68C` | `#FBF3E4` | `#5A3105` |
| Blocked / danger | `#E8B4B4` | `#FCEFEF` | `#6E1A1A` |
| Done / success | `#C4DA9F` | `#F1F7E8` | `#21450A` |

> [!important] These values are a **first derivation, not yet sample-compared.**
> The 2026-07-06 palette was locked by *comparing rendered samples* — "choices deliberate". These
> five have been derived on reasoning and contrast maths (every text-on-fill pair clears 4.5:1
> comfortably) but have **not** been looked at side by side on a real screen. Give them the same
> treatment before calling them final; adjust values freely, but keep the hue families and the
> three-part structure.

**The old blue-on-blue quirk is largely resolved by the geometry.** "In progress" pale blue and the
solid `--action` blue used to be told apart by colour alone; now a badge is square with a hairline
border and a button is a solid fill, so **shape carries the distinction** and colour no longer has
to. Still worth watching, but it is no longer the fragile part.

**Colour is never the only carrier.** Adopted from Bay: a status must be readable from its **text**
with colour and icon removed. Knot already does this in practice — the badge prints the humanised
stage — but it was never written down as a rule. It is one now.

### The waiting pin — ageing *(spec 2026-07-24, [[ADR-011 Acknowledgement as stored visibility]]; deferred to S6.7)*
A job with an unacknowledged handoff shows a muted *"Waiting on &lt;name&gt;"* line on the board.
**Sprint 4 ships it un-coloured** (neutral text only); the ageing colour below is **S6.7**, a styling
pass after real use. Uses the existing palette — **no new colour is introduced**:

| Age of the unacknowledged handoff | Colour |
|---|---|
| under 1h | **Registered / neutral** — age printed in plain text, nothing shouts (UI law #1) |
| 1h – 1 day | **Waiting / aging** (amber) — already this row's exact meaning |
| over 1 day | **Blocked / danger** (red) |

- **Red is clarified, not overloaded.** Red's reserved word is **"this job is stuck — act now"**,
  which has *two* causes: an open blocker, and an unclaimed handoff. **The chip's own text says
  which** — so red still means one thing to the eye (UI law #2 holds).
- **Colour the chip, never the whole row** — a red row is indistinguishable from a blocked job.
- **Green is deliberately not the "fresh" band**: green means Done/success, and would say
  "finished" about a job nobody has started.
- Boundaries are one workshop-wide constant and a **tunable styling call** — moving them touches
  no model.

## Typography
- ~~**System stack** (locked): `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica
  Neue", Arial, sans-serif` — 0KB, can't fail, native per device.~~
  **Changed 2026-08-25 (Session 36):** `--font-sans: Helvetica, Arial, sans-serif` — Bay's typeface.
  Still 0KB and still cannot fail, but it **gives up "native per device"** on purpose: the neutral
  grotesque is a large part of what makes the two systems look like one house, and letting each OS
  pick its own face is exactly what a shared style cannot afford. Expect a visible change on macOS
  (SF Pro → Helvetica).
- Mono for plates & job numbers: `ui-monospace, SFMono-Regular, Menlo, Consolas, monospace`.
  **Unchanged** — Bay says nothing about mono, and the plate treatment is Knot's own.

### The scale — **Bootstrap's, not ours** *(ruled 2026-08-25)*

The scale was measured off the reference first (44 / 28 / 20 / 15 / 13 / 12 / 11 / 10 px, on a 1.6
ratio) and then **compared side by side against Bootstrap's own ladder in a working mock**. Bootstrap
won on the builder's call. Three of the six sizes were already identical; the rest moved by 1–4px.

| Role | Bootstrap | rem / px | Weight | Tracking | Leading |
|---|---|---|---|---|---|
| Page title | `.display-6` | 2.5 / 40 | 400 | `-0.05em` | 1.0 |
| Metric number | `.fs-3` | 1.75 / 28 | 400 | `-0.05em` | 1.0 |
| Panel title | `.fs-5` | 1.25 / 20 | 400 | `-0.03em` | 1.2 |
| Section heading | `.fs-6` | 1 / 16 | 600 | 0 | 1.3 |
| Body | `--bs-body-font-size` | 0.875 / 14 | 400 | 0 | 1.45 |
| Row title | — | 0.875 / 14 | **700** | 0 | 1.3 |
| Support · eyebrow · chip | `.small` | 0.75 / 12 | 400 / **700** | 0 / **+0.08em** | 1.4 |

**Six steps, five of them Bootstrap classes.** The only override is body size, and Bootstrap 5.3.3
exposes it as a variable (`body { font-size: var(--bs-body-font-size) }`), so it is a one-line
change with no build step.

Three rules matter more than the numbers:

1. **Display type is weight 400, never bold.** Size and tracking carry the emphasis. A bold 40px
   heading is the fastest way to stop looking like this system.
2. **Tracking is always `em`, never `px`.** Negative above 20px, zero on body, positive on eyebrows.
   This is what replaced values like the reference's `0.825px`.
3. **Leading drops below 1.0 nowhere.** The reference sets 0.93–0.98 on display sizes; Bootstrap's
   ladder plus `line-height: 1` is close enough and is one less number to carry.

> [!note] Fluid type, and why desktop-only makes it safe
> `.display-6` and `.fs-1`–`.fs-4` are **fluid** below the `xl` breakpoint — Bootstrap's RFS makes
> them `calc(rem + vw)` and only snaps to the exact rem at ≥1200px. Knot is **desktop-only for now**
> (below), so they resolve exactly. **Revisit this deliberately when the phone posture returns**
> rather than discovering it then.

### The eyebrow
A recurring component, not a one-off: **small uppercase letterspaced muted label** sitting directly
above a display heading (`DASHBOARD · SERVICE ADVISOR`, `NEEDS ACTION NOW`). Written in sentence
case in the markup and uppercased in CSS. It is how a screen says where you are without spending
a heading on it.
- ~~Escape hatch if it ever "looks funky": self-hosted Inter (~100KB, closest free match to SF
  Pro), one-line swap via `--font-sans`.~~ **Moot** — the escape hatch existed to approximate SF
  Pro, which is no longer the target. `--font-sans` remains the one-line swap point if the face is
  ever revisited.

## Geometry *(new 2026-08-25 — Knot had no shape rule at all)*
Until now Knot inherited **Bootstrap's rounded defaults by accident**, not by decision. Adopted
from Bay, and the most visible half of the restyle:

- **Square corners.** `--bs-border-radius` and friends go to `0`. No rounded cards, buttons, inputs,
  or badges.
- **1px borders, meant to be seen.** Structure comes from hairlines, which is why `--border`
  darkened to `#D8D8D4`. A border is not a whisper any more.
- **Elevation is reserved for what floats.** *(Corrected 2026-08-25 against the live prototype —
  the first pass recorded a blanket "no shadows", which is wrong.)* Inline structure — cards,
  panels, tiles, rows — has **no shadow at all**; separation is a border. But anything that
  **floats above the page** carries one, and they are specified:

  | Layer | Shadow |
  |---|---|
  | Side drawer | `12px 0 32px rgba(16,16,16,.16)` |
  | Action tray / popover | `0 12px 32px rgba(16,16,16,.18)` |
  | Floating control (mobile menu toggle) | `0 4px 12px rgba(16,16,16,.08)` |
  | Scrim behind a drawer | `rgba(16,16,16,.28)` |

  The rule in one line: **a shadow means "this is above the page", never "this is important".**
- **No round pills or circles as a default control style.** Avatars, counters, and tags are squares
  unless there is a specific reason otherwise.

This obeys law 1 rather than straining it: geometry is doing the work decoration used to, and none
of it draws the eye the way status does.

## Sizing and spacing — Bootstrap's ladder *(ruled 2026-08-25)*

**Every spacing decision comes from Bootstrap's spacer scale.** Nothing is ours to maintain, and
no value is an orphan like `15.5px`.

`0 · .25rem · .5rem · 1rem · 1.5rem · 3rem`  →  **0 · 4 · 8 · 16 · 24 · 48**

| Need | Bootstrap | px |
|---|---|---|
| Page padding | `p-5` | 48 |
| Section gap · panel head | `gap-3` · `p-3` | 16 |
| Row padding | `py-2 px-3` | 8 × 16 |
| Sidebar block rhythm | `mb-4` | 24 |
| Icon ↔ label · chip padding-x | `gap-2` · `px-2` | 8 |
| Nav gap | `gap-1` | 4 |

Two things this buys beyond tidiness: the values can live as **utility classes in the Slim
templates** rather than in our CSS at all, and row padding-x now equals panel-head padding-x, so
a row's text aligns with its panel's title. (In the reference those differ by 4px.)

**The ladder is coarse** — it jumps 24 → 48 with nothing at 32. This template never needs 32. If a
screen does, that is one component value, not a reason to fork the scale.

### The six dimensions Bootstrap has no utility for

Component sizes, not spacing. They would live in CSS under any scheme; all six are clean 8px
multiples.

| Token | rem | px |
|---|---|---|
| `--sidebar` | 14 | 224 |
| `--topbar` | 4.5 | 72 |
| `--rail` | 20 | 320 |
| `--control` (button, nav item) | 2.5 | 40 |
| `--row-min` | 5 | 80 |
| `--tile-min` | 4 | 64 |

**Six numbers for the whole system.**

## Layout archetypes

**Every screen is one of the archetypes below.** There are two. A screen that fits neither is not a
styling problem — it is a **new archetype, and it gets recorded here before it gets built.** That
rule exists because the sign-in screen was designed first and defined second, and came out
disconnected from everything else.

### Archetype 1 — the app page

Everything behind the sign-in wall. Four things, always in this order.

1. **Frame** — fixed sidebar `--sidebar`, full viewport height, 1px right edge. The content column
   is grid column 2, and **the topbar lives inside it**, not above the whole shell.
2. **Topbar** — `--topbar` tall, sticky, eyebrow + context on the left, avatar right.
3. **Page head** — eyebrow → page title, one primary action on the right, **bottom-aligned**
   (the button sits level with the title's baseline, not its cap), `p-3` below.
4. **Section stack** — sections separated by `gap-3`. Only two section types exist:
   - **Metric strip** — *n* equal columns, borders collapsed, `--border` between.
   - **Split** — `minmax(0,1fr)` + `--rail`, `gap-3`.

**Panel anatomy:** head (`p-3`, `--rule` below) + rows (`--row-min`, `py-2 px-3`, `--rule` between,
`--hover` on hover).

Any screen is *frame → head → n sections → panels*. If a screen needs a fifth thing, that is a
design decision, not a layout one.

### Archetype 2 — the gate

The screens **outside** the wall: sign in, sign up, forgot password, reset password. No sidebar, no
topbar, no page head — so archetype 1 does not apply and its own layout (`layouts/auth.html.slim`)
is correct. What makes it *on-system* rather than a one-off:

1. **Canvas** — `--hover` grey, not white. The gate is outside the product, and the grey is what
   lets a white card lift off the page without a shadow. This is the same token doing a second,
   documented job; it is **not** a second grey.
2. **One centred column**, `max-width: 25rem`, vertically centred, page padding `p-5`.
3. **Head** — the square mark, then **eyebrow → title** exactly as archetype 1's page head, and one
   supporting line at body size in `--text-quiet`. Left-aligned. The eyebrow is not optional here.
4. **One card** — `--border`, square, `p-4`, holding the form and both actions.
5. **The action pair** — one solid primary (the screen's own verb), then a `--rule` divider, then
   the **cross-link as an outlined full-width button** (Sign in ⇄ Create an account). Law 3 holds:
   the outlined button is a *style*, not a second primary.
6. **Footnote** below the card in `--text-faint`: the owner-facing line that keeps vehicle owners
   out of staff signup.

The gate never does workshop creation, invitation acceptance, or role selection. **Its job ends at
authentication**; everything after is `home#index`.

## Bootstrap is the default answer *(rule, not preference)*

**If Bootstrap ships the component, we use Bootstrap's and theme it through `--bs-*` variables.
We never hand-roll an equivalent.** This is the whole point of the single-source-of-truth ruling,
and it is the rule most easily broken without noticing — a hand-built input looks fine and quietly
forks the system.

| Need | Use | Theme via |
|---|---|---|
| Text / email / password field | `.form-control` | `--bs-border-radius`, `--bs-border-color`, `--bs-body-font-size` |
| Label | `.form-label` | our type tokens |
| Checkbox / radio | `.form-check-input` | `--bs-form-check-bg`, `--bs-border-color` |
| Select | `.form-select` | as above |
| Button, solid | `.btn.btn-primary` | `--bs-btn-bg`, `--bs-btn-border-color`, `--bs-btn-hover-bg` |
| Button, outlined | `.btn.btn-outline-secondary` | `--bs-btn-color`, `--bs-btn-border-color` |
| Table | `.table` | `--bs-table-*` |
| Focus ring | Bootstrap's | `--bs-focus-ring-color/width/opacity` — **never a hand-rolled `box-shadow`** |
| Validation message | `.invalid-feedback` | our type tokens |

**The brand layer holds only what Bootstrap cannot express:** colour tokens, `--bs-border-radius: 0`,
`--bs-body-font-size`, `--font-sans`, negative tracking, the six component dimensions, the eyebrow,
and the five components Bootstrap has no equivalent for — panel, list row, metric tile, status chip,
side-nav item.

**Spacing never appears in our CSS.** It comes from Bootstrap utility classes in the Slim templates.
If a padding or gap value is written in `application.scss`, that is the smell.

## Components

The named inventory. **A component is named here *before* a screen uses it twice** — UI law 7 has
no enforceable meaning until the pieces have names, and `_blocker_item` getting built twice is the
evidence. `/design-preview` renders each of these once, in every state.

### The page pattern
Every screen, without exception: **eyebrow → display heading → one primary action, top right.**
On mobile the primary action becomes a **full-width bar** below the heading instead.

> [!note] **Narrowed 2026-08-27 — "one" is a ceiling, not a quota.**
> A screen whose job is to *monitor* may carry **none**. The Board is the first: its front door
> moved to the vehicle (L1), leaving the head with an eyebrow and a title and nothing else. This
> **narrows** the rule, it does not reverse it — no screen may still carry two. Worth saying
> because a deliberately empty top-right corner otherwise reads as something failing to load, the
> same way a non-counter role's blank slot does. See [[Board]].

### Structure

| Component | Spec |
|---|---|
| **Card / panel** | White, `1px --border`, square, **no shadow**. Row dividers inside use `--rule` |
| **Inverted panel** | `#101010` background, white text, `1px --border`, 18px padding. For context//"what this screen is" copy — used sparingly, it is the loudest thing on a page |
| **Metric strip** | Tiles **share their 1px rules** (`--border`) rather than sitting in a gapped grid. Each: `--tile-min` tall, number at `.fs-3`/400, label beside it, arrow right |
| **List row** | CSS grid, `--row-min`, `py-2 px-3`, divider `--rule`, `--hover` on hover. Title 0.875rem/**700**, subline 0.75rem `--text-quiet`, then columns, then a right arrow. **One row component** — the attention queue and every list reuse it |
| **List toolbar** | A panel head carrying search left (with icon) and the record count right, 0.75rem `--text-quiet` |
| **Status chip** | 1px border + pale fill + dark text of one hue family, square, `0.75rem`, `px-2`. See §Status colors |
| **Square bullet** | 7×7, no radius. `--action` when the row is live, `#B9B9B4` when it is not |
| **Avatar** | 32×32 **square**, `#101010` fill, white initials. Never a circle |

### Navigation *(desktop)*

| Part | Spec |
|---|---|
| Sidebar | White, **fixed**, full viewport height, `--sidebar` wide, `1px --border` right edge, collapsible to an icon rail. Brand + collapse toggle top, workshop name under it. **The topbar is not above it** — it sits in the content column |
| Group label | Eyebrow style, above each cluster of items |
| Nav item | `--control` tall, body size, `#4E4E49`, `gap-2` to its icon |
| Nav item, selected | **Full-bleed solid `--action`, white text.** Not a tint, not a left bar |
| Sub-nav item | 0.75rem/600, `--text-quiet`, indented **with a vertical rule**. Exists **only** under a parent — never as a standalone indent |
| Sidebar footer | **Account chip pinned to the bottom** — square avatar + name/role + chevron |

### Navigation *(mobile)*

| Part | Spec |
|---|---|
| Menu toggle | **40×40 bordered square**, fixed, with the floating-control shadow — not a bare icon |
| Drawer | White panel + scrim, mirrors the desktop hierarchy including sub-nav |
| Action control | **52×52 square**, `--action`, fixed 16px from the bottom-right, persists on scroll |
| Action tray | Opens above the control: white, `1px #101010`, role-specific **quick actions only — never a duplicate navigation tree** |

> [!warning] Adopting the mobile chrome costs a dependency decision
> Knot loads **no Bootstrap JS**. A collapsing sidebar, a drawer and a tray all need Stimulus or a
> CSS-only pattern, and an icon rail needs an icon source. **One coupled call**, still open — see
> [[Bay system reference — external comparison]].

### Not to be copied
The reference implementation's list rows are `div`s in a grid, not a `<table>`. For a dense board
that is an accessibility regression — **use a real table** and let the grid behaviour come from CSS.
Its attention rows also **drop the owner and waiting columns on mobile**, which discards two of the
four questions the component exists to answer.

## UI laws
Invariants for every screen, in the spirit of [[Architecture laws]].

1. **Chrome whispers, status shouts.** The interface stays neutral and quiet; if something draws
   the eye it must be information (status, aging, pending ack), never decoration. Master rule.
2. **Status colors are reserved words.** Red/amber/green/blue-badge only ever mean job state.
   No red delete buttons, no green save toasts, no amber accents. If red can mean two things,
   the board stops being scannable. Neutral/action colors cover those cases. *(2026-07-24,
   [[ADR-011 Acknowledgement as stored visibility]]: red's word is **"stuck — act now"**, and it has two
   causes — a blocker, and a handoff nobody has claimed for over a day. That is a **clarification,
   not a second meaning**: the eye still reads one word, and the chip's text names the cause.)*
3. **One primary action per screen.** Exactly one solid `--action` button — the most likely next
   step for that role there. Everything else outlined or plain. The screen chooses, not the user.
4. **Pass the glance test.** Every screen answers its core question from arm's length (phone) or
   across the room (PC) *before any text is read*. Visual twin of "dashboards are queries."
5. **Thumb-first for anything a technician touches.** Tap targets ~~≥44px~~ **≥48px on mobile,
   ~40px on desktop** *(2026-08-25, adopted from Bay — Knot previously specified only the phone
   number and had no desktop target at all)*, primary actions in bottom-half reach, no hover-only
   interactions, works one-handed. Tech screens never go dense. Mobile surfaces also need
   **safe-area bottom spacing** and must never obscure the active task.
6. **Words over icons.** Buttons say what they do ("Raise blocker", not a glyph). Icons accompany
   labels, never replace them. From [[Product overview]]: adoptable without training.
7. **Same component, every screen.** Badges, job cards, plate numbers look identical everywhere —
   board, job page, owner view. New view = recompose existing pieces, never restyle. UI twin of
   "new viewpoint = new scope, never new model."
8. **Empty states and errors tell the truth.** What would be here, why it isn't, what to do next.
   No dead ends, no "something went wrong."
9. **PC buys density, not size.** Bigger screens show *more* (tables, columns, filters), never a
   stretched phone layout. Job board on PC = dense table; same data on phone = stacked cards
   (same components, law 7). Long-form text still caps at ~70ch line length.
10. **Keyboards are first-class on PC.** Intake and forms flow on tab/Enter alone — logical tab
    order, autofocus first field, no click-only widgets. Desk twin of law 5.

> [!question] **Open call — law 3 at PC density** *(raised 2026-08-25, not ruled)*
> Bay states the same rule as **"one visually primary action per *local area*"**, not per screen.
> That difference is exactly the problem the law-3 audit faces: `jobs/show` carries four solid
> primaries because it has four genuinely distinct regions (technician · floor moves · blockers ·
> timeline), and `home/index` three. Either law 3 gains a density qualifier, or those screens
> restructure so each really does have one next step. Decide it *with* the audit, not before.

## Interaction and accessibility *(new 2026-08-25 — adopted from Bay)*
Knot practised most of this and had written down none of it.

- **Visible keyboard focus is never removed.** Pairs with law 10 — a flow that works on tab/Enter
  is useless if you cannot see where you are.
- **Semantic labels**, and **text that explains a status without relying on colour or icon alone**
  (also stated under Status colors — it matters in both places).
- **Motion is brief and optional.** v1 has essentially none, and whatever arrives must honour
  `prefers-reduced-motion`. *(This narrows the "animation rules — extract later" deferral below:
  the accessibility floor is decided now even though the vocabulary isn't.)*
- Control sizes per law 5 above.

**Posture note:** each role's device decides which laws bite hardest — technician = phone
(law 5 hard), SA/manager = PC (laws 9–10 hard), owner = phone read-only (laws 4 + 7, zero
actions). One system, three postures. Matches the device-posture decision (Session 3, [[Tech stack]]).

## The design plan

Written 2026-08-25 (Session 36) with the builder, from a clean-slate question: **"suppose we
removed everything — what is the plan to recreate the UI screens?"**

### Why a plan, and not twelve screen redesigns

Answering that question screen by screen misses what is actually wrong. Every problem that
survives a per-screen pass sits **above** the screen level. Five of them, all verified against
code at `2c5816f`:

1. **There is no navigation model.** `app/views/layouts/application.html.slim` is a wordmark, the
   workshop name, the signed-in email, and Sign out. Every screen then invents its own way back —
   *"Back"*, *"Back to crew list"*, *"← This car's visit"*, *"Back to {name}"*. `/customers` is
   reachable from exactly one place: the board. This is the layer everything else is downstream
   of, and the one thing that cannot be retrofitted cheaply.
2. **Three postures were designed; one has screens.** [[Design system]] commits to SA/manager on
   PC, technician on phone, owner read-only on phone. Every built view is the desk layout.
   `app/views/jobs/show.html.slim` **is the technician's screen** — Start work, Mark done — and it
   is a 106-line desk page. UI law 5 (thumb-first, ≥44px, no hover-only) has never been exercised
   by anything in the app.
3. **UI law 3 is broken exactly where it is load-bearing.** One primary action per screen:
   `jobs/show` carries **four** solid `btn-primary`, `home/index` carries **three**. That is not
   styling drift — it means those screens never decided what their one next step is.
4. **UI law 2 is broken in the layout, therefore on every page.** Flash messages render as
   `.alert-info` (Bootstrap cyan) and `.alert-warning` (Bootstrap yellow). Neither is remapped in
   `app/assets/stylesheets/application.scss`, and both wear hues the status vocabulary has reserved.
   The status *badges* are implemented correctly (`.stage-badge`, the sacred palette, read from one
   partial) — so the single violation sits at the one place that touches all 27 templates.
5. **UI law 7 is broken concretely.** `app/views/intakes/_blocker_item.html.slim` and
   `app/views/jobs/_blocker_item.html.slim` are twins; the diff is six lines and all of it is
   naming. Two components that must look identical forever, maintained separately — out of only
   **four** shared partials in the whole app.

6. **UI law 9 is broken app-wide.** *(Found 2026-08-25 while comparing against the Bay system
   reference — see [[Bay system reference — external comparison]].)* **Every workshop screen is
   `col-12 col-md-8 col-lg-6`** — a half-width centred column on a PC: the board, the visit, the
   repair, customers, crew, blocker types. Law 9 says *"PC buys density, not size… bigger screens
   show more, never a stretched phone layout"* — and a centred half-column is exactly a stretched
   phone layout. This is arguably the **largest** of the six, because it caps what the board can
   ever show, and it is the same decision as the shell: page width and sidebar-vs-centred are one
   call, not two.

None of these is an intake problem. Designing the intake screens first would have invented a
header and a component set that the other eleven screens then had to match.

### The rulings this plan is built on *(2026-08-25, builder)*

- **The clean slate is design-level, not code-level.** Derive the screen set as if nothing
  existed, then diff against what is built. Nothing is deleted. The **rebuild-or-keep verdict per
  screen is deliberately not recorded in this session** — it is taken screen by screen as each is
  reached in L3.
- **Scope is every surface**, including the signed-out marketing page — but **ordered
  bread-and-butter first**: account → workshop → crew → setup → customers → vehicles → intake →
  blockers. That ordering was arrived at independently by the builder and it matches
  [[Screen flow]]'s recorded Setup → Records → Daily loop spine, which is corroboration rather
  than a new structure.
- **Desk-first.** Floor and owner postures are planned later, with the trap named (see L2).
- **The plan lives here**, restructured into this note. The risk — mixing intent with the half
  that gets re-synced from `bin/rails routes` — was raised and accepted; the two-part split above
  and the ritual at the bottom are how it is contained.
- **Vehicles get an ordinary create screen, and intake ships §1a happy-path only** (see L3 step 6
  and step 7 for what that costs and what it must still handle).

### The order, and the reason for it

One rule drives the whole sequence: **do the things that change every screen before drawing any
screen.** Done in the other order, the screens get drawn twice.

That is the design-side statement of a general sequencing rule the builder named on 2026-08-25 —
**sequence by fan-in**, then cost of late change, then verification leverage. The full statement,
and the mistake that forced it to be written down, are in [[Sprint plan]] §Conventions. The pattern
is called **outside-in** or **shell-first**; it is the deliberate opposite of Atomic Design's
bottom-up order, which suits a published component library but not an application.

**First, the four cross-cutting fixes.** None of these is a screen; all four change every screen.

| | What | Why it comes before any screen |
|---|---|---|
| **X1** | Component inventory — write it down, **build the `/design-preview` route**, and merge the `_blocker_item` twins | UI law 7 has no enforceable meaning until the components are *named*, and a review route makes it **checkable rather than asserted** (adopted from Bay; it would have caught the twins on day one). Every screen below is supposed to be a recomposition of these pieces |
| **X2** | **The visual re-lock** — apply [[Design system]]'s 2026-08-25 restyle: Helvetica/Arial, warm-neutral tokens, square corners + visible 1px borders, the re-derived three-part status chip, and flashes moved off the reserved status hues | A **token-level** change in the brand layer that reaches all 27 templates; only the badge and the flash need markup. Doing it after the screens means restyling them twice — and it folds in the law-2 violation (flashes wearing `.alert-info`/`.alert-warning`) |
| **X3** | The shell — one header, one back affordance, reachability, **and page width (law 9)** | Every screen's header is downstream of this, and it is the **only** item genuinely expensive to retrofit. Draw screens first and each hard-codes its own answer — which is exactly how four wordings for "back" and twelve `col-lg-6` columns already happened |
| **X4** | Law-3 audit: `jobs/show` (4 primaries), `home/index` (3) — **and settle "per screen" vs "per local area"** | Deciding a screen's one next step *changes its layout*, so it precedes any redesign of those two. Bay states the rule per *local area*, which is exactly the tension `jobs/show`'s four regions create ([[Design system]] UI laws, open call) |

**Then the spine, in the order a real workshop lives through it.** That ordering is not
arbitrary and not merely "setup before use" — it is the only order in which **the product can be
walked end to end after every single step**. Each step leaves the app usable one notch further
than before, so every step is independently demonstrable and independently abandonable.

*(Step numbers below are L3's, so the two sections can be read against each other.)*

- **Steps 1–3 · Sign up → create workshop → invite crew** — all built. They get the new shell (X3)
  and a redesign verdict each.
- **Step 4 · Blocker catalog** — built, and it needs **no setup screen at all**:
  `Workshop.create_with_owner!` already seeds four default types at workshop birth. Its screen is
  *maintenance*, so it belongs late in the journey rather than in the setup run where it looks like
  it should sit.
- **Step 5 · Customers** — built. Earns the **search-first rule** (L3), because add-vehicle is about
  to make duplicate customers cheap to create.
- **Step 6 · Add vehicle — NEW.** The first genuinely new screen, and the smaller of the two:
  ordinary CRUD under a customer.
- **Step 7 · Intake, happy path — NEW.** The second new screen, and the last hole in the daily loop.
  It comes *after* add-vehicle by necessity: with vehicles created up front, intake's first cut is
  [[Intake flow]] §1a alone and the whole dedup tree can wait.
- **Steps 8–9 · Raise/clear blocker · deliver** — built; redesign verdicts when reached.
- **Step 10 · Marketing home** — last. Different audience, different job from the app shell, and
  nothing else depends on it.

**Why only two new screens.** The clean-slate framing invites rebuilding twelve pages. The honest
read of the code is that most of them are *right* and merely unstyled or unshelled — what is
actually missing is a front door (`intakes#new`) and a vehicle door, and what is actually broken is
the four cross-cutting items above. That is the whole gap. Anything beyond it is a redesign verdict
taken per screen, deliberately deferred by ruling.

---

**The layers behind that order.**

L0–L3 are the reasoning the order above rests on: the components a screen is made of, the shell it
sits in, the posture it is drawn for, and the journey it belongs to. Read them when a step's
"why" is not obvious from the table; skip them when it is.

### L0 — Components: where Knot stands today

The **target spec** is §Components above. This is the audit against it — what exists in the code
right now and what has to happen to each piece. `/design-preview` ([[Sprint plan]] S5.5a) is where
the two get compared.

| Piece | Where | Verdict |
|---|---|---|
| Stage badge | `jobs/_stage_badge` + `.stage-badge` CSS | **Correct** — one source, sacred palette, the model for the rest |
| Waiting pin | `jobs/_waiting_pin` | **Correct** — deliberately uncoloured, reasoning in the partial |
| Board row | `jobs/_board_row` | Built; will be re-cut when the board regroups by Intake |
| Blocker item | `jobs/` **and** `intakes/` | **Merge** — one component, the raise/clear vocabulary passed in |
| Page shell / header | layout only | **Undefined** — L1's job |
| Flash | layout | **Recolour** — off the status palette (finding 4) |
| Empty state | ad hoc per screen | **Undefined** — law 8 requires a shape, not a sentence |
| Inline note form | inside both blocker items | Extract when the twins merge |
| Actor + timestamp line | inside both blocker items | Recurs in the job timeline too — name it |
| Plate | `.font-monospace` inline | Name it; law 7 says a plate looks identical everywhere |
| Action bar | ad hoc `.d-grid` / `.d-flex` | **Undefined** — this is where law 3 gets enforced or lost |
| Form field + inline error | Devise partial + hand-rolled | Two vocabularies today; pick one |

### L1 — Navigation and information architecture

The layer nothing else can be designed without. What must be settled:

- **What the shell is** for a signed-in staff member — and whether the answer differs by posture
  (it should not; law 7 says recompose, not restyle).
- **Where "back" comes from.** Today it is a per-screen invention, which is why four different
  wordings exist for the same affordance.
- **How a screen is reached without going through the board.** Customers, Crew, and Blocker types
  are currently a row of outline buttons on `workshops#show` and exist nowhere else.
- ~~**Where the front door lives.** `app/views/workshops/show.html.slim` already carries a reserved
  slot — *"Restore a button here once intakes#new exists."* Every sibling control in that row is
  `btn-outline-secondary`, so a single solid `--action` button there **satisfies** law 3 rather
  than straining it, and law 1 argues against a persistent nav item (the header is chrome, and
  chrome whispers).~~
  > **REVERSED 2026-08-27 (builder).** The front door leaves the board: **check-in starts from the
  > vehicle.** The board becomes a monitoring surface with no create path. The reasoning is not
  > cosmetic — starting from a found vehicle makes the **search-first rule enforced rather than
  > advisory**, which is exactly the trap named under L3 step 6 (a standalone add-vehicle path
  > makes duplicate customers cheap). The reserved slot in `workshops/show` is now dead and should
  > go when the screen is re-cut. Full context: [[Board]].
- **Page width.** Finding 6: twelve screens are `col-lg-6`. Whatever the shell becomes, content
  uses the available canvas and responds to it — law 9 is not optional.

#### Adopted into L1 from the Bay system reference *(2026-08-25)*

- **The IA rule that answers "how is anything reached?":** *keep only durable work areas in primary
  navigation; detail pages, create flows, and item-specific actions are reached **contextually**.*
  That is a usable rule, not a preference, and it resolves the Customers / Crew / Blocker-types
  question directly — they are durable work areas, so they belong in nav; "New customer",
  "Add vehicle" and every job action do not.
- **The nav pattern:** a vertical sidebar on desktop, collapsible to an icon rail, with content
  responding to the collapse; on mobile a drawer that mirrors the *same* role-scoped hierarchy, and
  a role-specific quick-action tray that is **not** a duplicate navigation tree.
  **Cost to weigh before adopting it wholesale:** Knot loads **no Bootstrap JS**, so collapse and
  drawer need Stimulus or a CSS-only pattern, and an icon rail needs an icon source (the
  `lucide-rails` question, deliberately deferred to this same decision — see
  [[Bay system reference — external comparison]]).
- **The attention-item test**, sharper than law 4's glance test. Any attention surface must make
  four things immediately clear: **what is blocked or overdue · who owns the next action · how long
  it has been waiting · the fastest safe action the viewer can take.** Knot's waiting pin and
  ageing bands already answer the third from the *stalled state* rather than creation date, which
  is the same conclusion [[ADR-011 Acknowledgement as stored visibility]] reached independently.
- **Ruled 2026-08-27: one board for everyone. A viewpoint is a *scope* on it, never a screen of
  its own.** Bay gives every role its own dashboard with a role-scoped attention queue and
  destinations; Knot does not fork the landing surface. [[Architecture laws]] #3 ("dashboards are queries,
  not tables — new viewpoint = new scope, never a new model") makes role-scoped views *cheap*, so
  cost was never the argument on either side. Three things decided it:
    - **Roles here are plural and simultaneous.** `WorkshopStaff#titles` returns a *list* and leaves
      precedence to the caller; the topbar already renders `Owner · Technician`. A dashboard per
      role needs a function role → screen, so it would force us to invent a precedence order the
      schema deliberately refuses to hold — and the towkay who owns the shop and works the floor is
      the normal case in a small workshop, not an edge case.
    - **The laws already say "scope, not screen".** Law #3 licenses a new *scope*; UI law 7 says a
      new viewpoint is a recomposition of the same components, never a second screen set. A
      per-role dashboard is the one reading of #3 that UI law 7 forbids.
    - **Knot's nav is already role-scoped — by gating, not by forking.** `workshops#show` hides
      Customers behind counter staff, and Crew and Blocker types behind owner-or-manager. So the
      reference's role matrix is **adopted, not rejected**: it lands as (a) gated nav items under
      the durable-work-areas rule, (b) the screen's one primary action chosen for the viewer's role
      — which UI law 3 already licenses, "the most likely next step *for that role* there" — and
      (c) scopes on the board's content. Only the extra *screens* are rejected.

  **What this does not settle, and must never be read as settling.** Under one shared board a
  technician still has to find themselves in a shop-wide list. That is a real glance-test failure
  (UI law 4) and a real miss on the attention-item test's second question — *who owns the next
  action*. It is **deferred with the floor posture (L2), by ruling and not by finding.** When that
  posture is drawn the fix is a **mine-first scope on this board** at a different density; a
  technician dashboard would reopen this ruling, not implement it.

  **Consequence for the landing page: none.** `HomeController#index` auto-routes a single-workshop
  user with no pending invitation straight to the board, and that stands — under this ruling there
  is nowhere else it could land.

#### The board's content rule — **safe for everyone, or not on the board** *(builder, 2026-08-27)*

Everything the board shows is safe for **every** role to see, so **nothing on it is hidden per
role**. Role-segregated information on this screen is to be avoided, not managed. Four parts, all
checkable:

- **Data is uniform; actions are gated.** This rule governs *information*, never *permission*. Who
  may move a job, raise a blocker or deliver a car is still `Permissions`' call, and the view still
  hides a control the controller would refuse (`PermissionsHelper` delegates so the two cannot
  drift). The board's **one primary action stays role-varying** under UI law 3 — that is an action,
  not content, so it does not contradict this.
- **What is not safe for everyone is not board content.** It does not become a per-role panel on
  the board; it becomes **its own screen**, reached through the gated nav. This is what stops the
  rule eroding one exception at a time — the first genuinely role-sensitive information in the
  product is Sprint 8's attribution and health reporting, and it is already planned as separate
  surfaces rather than as blocks on this one.
- **A scope may reorder, never conceal.** The technician's deferred mine-first scope (L2) sorts or
  defaults the same rows; it must never remove a row the viewer is not allowed to see, **because
  there is no such row**, and the whole board stays reachable to the same person.
- **True in code today.** `workshops#show` loads `@active_jobs`, `@done_jobs` and
  `@pending_acknowledgements` scoped to `Current.workshop` alone — no role filter anywhere. The
  only role-conditional markup on the page is the three outline buttons, and S5A.3a moves those
  into the sidebar. After that task the board has **zero** role-conditional content, which is the
  state this rule holds it at.

**The cost, accepted rather than hidden:** this caps what the board may ever carry. The day pricing,
margin, or per-person throughput becomes daily-loop information, it cannot go here — it needs a
screen of its own. That is the rule working, not the rule failing.

**Worth naming, and not a role problem:** the waiting pin renders `event.receiver.user.email`,
because `User` carries no name. It is the only personal datum on the board and it is shop-internal,
so it passes — but it is an *identity* shown where everything else is *work*, and it is where a
staff-name field would first pay off.

#### The screen's name — **Board** *(ruled 2026-08-27)*

One screen carried four names: *Dashboard* ([[Screen flow]]), *the board* ([[Screen map]]),
`/workshop`, and `workshops#show`. It is **the Board** — the nav label, the page title, and the
word every note uses from here.

- **"Dashboard" is retired.** It is the abstract word, nobody at a workshop says it, and it now
  names the very thing ruled against above. Keeping it would leave the rejected design sitting in
  the nav label.
- **"Board" is the concrete domain word.** The whiteboard is the artefact Knot replaces
  ([[Product overview]]), and the code already speaks it (`jobs/_board_row`,
  `08 Experiments/knot-board-desktop.html`). It also survives the ADR-012 regroup — a board of cars
  is still a board.
- **The URL and the controller do not change.** `/workshop` is the tenant root and `workshops#show`
  is honest REST. What is settled here is a **UI-surface** name — nav label, page title, page head
  — not a routing one; renaming the route would churn `workshop_path` across nine call sites to buy
  nothing a user ever sees.

#### The role matrix *(measured from the live prototype, 2026-08-25)*

The reference implementation's nav is **a shared spine plus role-specific additions, and exactly one
role-specific primary action.** Recorded because it is the concrete form of the "durable work areas
only" rule, and because Knot's equivalent question — one board for everyone vs a dashboard per role
— ~~is still open (below)~~ *(2026-08-27: **ruled above** — one board. The matrix is adopted as
gating plus a role-varying primary action, not as four screens.)*

| Role | Nav beyond the shared spine | Primary action |
|---|---|---|
| Service advisor | Customers · Vehicles | Check in vehicle |
| Technician | Blockers | View my jobs |
| Parts advisor | Parts requests · Blockers | View requests |
| Workshop manager | Customers · Vehicles · Team · Workshop setup | Check in vehicle |

Shared by every role: **Dashboard**, and **Jobs** with an *Active jobs* / *Job history* sub-nav.
Note what is **absent** from every nav: customer detail, vehicle detail, job detail, and every
create flow. Those are reached contextually — that is the rule doing its work.

> [!note] One divergence Knot should decide rather than absorb
> On the reference's customer list, **"Add customer" is the top-right primary next to the search** —
> the opposite of the search-first rule recorded for Knot ([[Sprint plan]] S5.5e), which exists
> because a standalone add-vehicle path makes duplicate customers cheap. Knot's reason still holds;
> this is noted so the difference is deliberate.

### L2 — Posture

**Desk-first is the ruling.** The floor and owner postures are planned after the bread-and-butter
spine walks end to end.

The trap to keep named while that is true: **`jobs/show` is the technician's screen wearing a desk
layout.** Start work and Mark done are floor actions on a 106-line PC page. Desk-first is a
sequencing choice, not a finding that the floor posture is unnecessary — and law 7 means the fix,
when it comes, is a recomposition of L0 components at a different density, never a second screen
set.

### L3 — The journey, step by step

The bread-and-butter spine. Each step names what the design pass must settle; the rebuild-or-keep
verdict is taken when the step is reached.

| # | Step | Screen | Today | What the design pass settles |
|---|---|---|---|---|
| 1 | Sign up | Devise 🔒 | built, unstyled | Whether the theme has grown enough to restyle Devise, or it stays exempt |
| 2 | Create workshop | `workshops#new` | built | The shell it lands in (L1) |
| 3 | Invite crew · accept | `invitations#new` · `home#index` card | built | The invite card as an L0 component; the accept/decline pair under law 3 |
| 4 | Blocker catalog | `blockers#index` | built | **No setup screen is needed** — `Workshop.create_with_owner!` seeds four default types at workshop birth, in the same transaction as the owner. This screen is **maintenance, not setup**, and belongs late in the journey, not in the setup run |
| 5 | Customers | `customers#index` · `#new` · `#show` · `#edit` | built | **Search-first rule:** `Customer.search` already matches name *or* canonicalized phone, so "New customer" must be reachable only *through* an empty search result, never as a peer button beside the search. See the dedup note below |
| 6 | **Add vehicle** | — | **nothing built** — no controller, no route, no screen | A new screen. Ruled 2026-08-25: a vehicle is created here, as ordinary CRUD under a customer |
| 7 | Intake | `intakes#new` **not built** | endpoint exists, no UI | **Happy path only** — see below |
| 8 | Raise · clear blocker | on `jobs#show` / `intakes#show` | built | The merged blocker-item component (L0); the note form; law 3 on a screen that currently has four primaries |
| 9 | Deliver · cancel | `intakes#show` | built | The visit header; `ready?` gating |
| 10 | Marketing home | `home#index` signed-out | built | Last. Different audience and different job from the app shell |

#### The dedup consequence of step 6 *(named, not hidden)*

[[Intake flow]] §2a exists to stop one person becoming two cards: the plate-first tree makes the
phone lookup **unavoidable** before a vehicle is created. A standalone add-vehicle screen hanging
off a customer page bypasses that tree, because the customer is already fixed on arrival. The trap
does not disappear — it moves to the counter and becomes the *primary* path: an SA who cannot find
the customer creates a new one and hangs the car off it. That is §2a's reverse-wife trap, promoted
from edge case.

The guard is a screen rule, not new logic, and it is why step 5 carries the search-first rule.

#### What "happy path only" means for step 7 *(and what it does not excuse)*

Intake's first cut is [[Intake flow]] **§1a** — plate keyed, vehicle found, visit opened. The
§1b mismatch forks and the §2a/§2b first-visit dedup tree are deferred, because steps 5 and 6 now
create customers and vehicles ahead of the visit.

Three things this does **not** excuse:

- **§1c stays in.** `index_intakes_one_open_per_vehicle` is a partial unique index
  (`… ON intakes (vehicle_id) WHERE (status = 0)`). Keying a plate that is already in house
  raises `RecordNotUnique` — a **500, not a branch**. The lookup must report the visit already in
  house and link to it, exactly as §1c says.
- **The not-found outcome is a routed dead end, not a refusal.** "That car isn't on file" must say
  what to do next and take the SA there (step 6). UI law 8.
- **The silent-compare constraint survives for whenever §1b lands.** [[Intake flow]] §1 forbids
  showing or reading the file's phone. Recorded here so it is not re-derived: comparison is
  **server-side on submit** — no client-side compare, no hidden field, no `data-` attribute. The
  instinct to make it feel instant with JS would leak the file-holder's phone to view-source.
  *(The file-holder's **name** may be shown before the check — builder ruling 2026-08-25: a
  narrower exception, since the SA says it aloud at the counter anyway, and without it they cannot
  tell they have hit the wrong file. The phone is different in kind: it is contactable.)*

Because §1b is deferred, `IntakesController#create` stays narrow — it widens by `complaint:` only,
not by `customer:`.

### Named as open

- **Floor and owner postures** — deferred by ruling, not by finding (L2).
- **The §1b mismatch tree and §2a/§2b dedup forks** — deferred with step 7. [[Intake flow]] stays
  the behaviour spec; [[Deferred decisions]] holds the payer-confirm design and its four-fork
  correction.
- **Rebuild-or-keep per screen** — by ruling, taken at each L3 step, not now.
- **Add a repair to an open intake**, the **jobsheet fill screen**, and the **owner token page** —
  still unbuilt, still outside this spine.
- **Devise restyle** — step 1's open question.
- **No vehicle-type column exists** (`vehicles` has only `vin`), so `inspection_type` can never be
  inferred from a plate. Whenever the intake screen carries the type picker, it is a real question
  to a human.
- ~~**One board vs a dashboard per role**~~ — **ruled 2026-08-27**, L1 above: one board, viewpoints
  are scopes on it. What stays open is the **technician's mine-first scope**, deferred with the
  floor posture.
- **The sidebar, and what it drags in** — ~~Stimulus (or a CSS-only pattern) for collapse and
  drawer~~, plus an icon source for the rail. *(2026-08-27: the Stimulus half is **settled by
  build** — `sidebar_controller.js` ships the collapse. What remains is the **icon source**, and it
  is inseparable from the collapse: UI law 6 says words over icons, so a collapsed rail with no
  icons is unreadable. Belongs to S5A.3a.)*
- **UI law 3: "per screen" or "per local area"** — settle it with X4, not before
  ([[Design system]] UI laws).
- **The re-derived status chips have not been sample-compared.** [[Design system]] flags this; the
  `/design-preview` route from X1 is where it gets done.

---

## Deferred
- **Dark mode** — surfaces will derive from brand steel blue (the navy-chrome sample read as a
  good dark theme and was liked for it). See [[Deferred decisions]].
- Devise views stay unstyled until the theme grows.
- Spacing/grid system — extract later from real screens, like an abstraction from real code.
- Animation rules — v1 has essentially none; Turbo defaults are fine.

## Implementation
All tokens are CSS custom properties in `app/assets/stylesheets/application.scss` — the single
source of visual truth. ~~Plain CSS, no framework~~ **Changed 2026-07-06:** looking at where the
project is heading, the base is **Bootstrap 5.3.3** (~~vendored `bootstrap.min.css`, no build
step~~, no CDN) with `application.scss` as a **brand layer** on top — it maps Bootstrap's CSS variables
(`--bs-body-bg`, `--bs-btn-*`, links) to the tokens above and holds branding-only classes
(`.wordmark`, `.text-action`, `.hero*`, `.prop-num`). Rules: theme through Bootstrap variables,
never redefine a Bootstrap rule wholesale; new screens compose Bootstrap utilities + brand classes, no
inline styles. Token values in this note still win if anything differs.

> [!warning] **2026-08-26 (Session 37): the vendoring and the no-build-step rule are REVERSED.**
> Bootstrap compiles from Sass source — the `bootstrap` gem (pinned 5.3.3) plus `dartsass-rails`.
> `application.css` is now `application.scss`, and the compiled output lives in
> `app/assets/builds/application.css`, gitignored. **This narrows nothing above except the
> delivery mechanism** — every token value in this note still stands.
>
> What it changes for anyone writing styles: **there are now two places to theme, and the Sass one
> is preferred.** Sass variables set *before* the `@import` (`$primary`, `$border-radius`,
> `$font-size-base`, `$display-font-weight`) regenerate every rule Bootstrap derives from them;
> `--bs-*` custom properties after it are for what Sass cannot reach. Six hardcoded values had no
> variable behind them at all — `.form-check-input:checked` among them — and now derive correctly
> from `$primary` with no rule of ours.
>
> Two consequences recorded rather than discovered later. **`$enable-rounded: false` is not
> sufficient** — `_root.scss` emits `--bs-border-radius*` regardless of that flag and
> `.form-control` reads the property directly, so all six radius values must be set. And
> **Bootstrap derives button hover/active as percentage shades of `$primary`**, landing on colours
> this note does not contain; the builder ruled the system colour wins, so `.btn-primary` overrides
> those four states back to `--action-hover`.

**What the brand layer holds after the 2026-08-25 ruling** — and nothing else: the colour tokens ·
`--bs-border-radius: 0` (squares buttons, cards, inputs, badges in one line) ·
`--bs-body-font-size: .875rem` · `--font-sans` · the six component dimensions · negative tracking
(Bootstrap has no letter-spacing utilities) · the eyebrow · five components (panel, list row, metric
tile, status chip, nav item). **Every spacing value comes from Bootstrap utilities in the templates**,
so no padding or gap number lives in our CSS at all.

~~**Token additions from the 2026-08-25 correction**~~ (on top of the re-lock below): `--border-inner`,
`--surface-sunk`, and the muted ink ramp `--text-muted` / `--text-quiet` / `--text-faint` — plus the
three floating-layer shadows. The type scale is `--font-sans` sizes with proportional negative
tracking; nothing here needs a build step.

**What the 2026-08-25 re-lock costs in `application.scss`** (all through Bootstrap variables, still
never forking the vendored file): `--font-sans` → Helvetica/Arial · the neutral tokens re-pointed
(`--page`, `--border`, `--text`, `--text-muted`) · `--bs-border-radius` family → `0` · card and
button borders made visible at 1px · `.stage-badge` re-cut from two-part to **border + fill + text**
· flashes moved off `.alert-info`/`.alert-warning` onto neutral tokens (they were wearing reserved
status hues — see §The design plan). It is a **token-level restyle, not a rewrite**: no
template needs new markup except the badge and the flash. Bootstrap **JS is not
loaded** — add via importmap only when a component (dropdown/modal) demands it.

## Related
- [[Architecture laws]] · [[Tech stack]] · [[Product overview]] · [[Deferred decisions]] ·
  [[Bay system reference — external comparison]] (provenance for the 2026-08-25 re-lock, and the
  live-prototype verification behind the type scale, ramps and component specs) ·
  [[Screen map]] (what screens *exist* — the code reflection this note plans against) ·
  [[Sprint plan]] S5.5a–i (the build order this note's plan half feeds)
