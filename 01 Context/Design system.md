---
type: context
created: 2026-07-06
updated: 2026-08-25 (Session 36 — renamed from 'Visual theme' and made the SINGLE SOURCE OF TRUTH for design: absorbs the component inventory, navigation/IA, posture, journey and build order from [[Screen map]]; type scale, elevation and the border/ink ramps corrected against the live Bay prototype; RE-LOCKED on the Bay system's visual language: Helvetica/Arial, ink #101010, warm-neutral canvas, square corners + visible 1px borders, status chips re-derived as border/fill/text, new interaction + accessibility rules; Knot's brand and action colours retained; prior: 2026-07-28 waiting-pin ageing bands — ADR-011)
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
> 2. **`app/assets/stylesheets/application.css`** — the tokens as CSS custom properties,
>    implementing it.
> 3. **`/design-preview`** — renders every component from those tokens, so drift becomes
>    *visible* instead of a doc-versus-code guess.
>
> Layer 3 is the one that makes the other two honest. It is [[Sprint plan]] **S5.5a**.

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
| Brand | `--brand` | `#22456B` | steel blue — wordmark, heading accents. Scarce on purpose. **Kept — identity.** |
| Action | `--action` | `#2D5E94` | buttons, links, focus — "you can act here". **Kept — identity.** |
| Action hover | `--action-hover` | `#24507F` | hover/pressed |
| Page | `--page` | ~~`#F5F6F8`~~ **`#F4F4F2`** | app canvas — Bay's canvas value |
| Surface | `--surface` | `#FFFFFF` | cards, app bar |
| Border | `--border` | ~~`#E2E6EB`~~ **`#D8D8D4`** | **container edges** — card/panel outlines. Structure, not a whisper |
| Border inner | `--border-inner` | **`#E7E7E3`** | **rules *inside* a container** — row dividers, section splits. Lighter, so a dense list doesn't read as a grid of boxes |
| Toolbar surface | `--surface-sunk` | **`#F7F7F5`** | list toolbars and filter bars — a third tone between white and canvas |
| Text | `--text` | ~~`#1C2B3A`~~ **`#101010`** | body — Bay's ink |
| Text muted | `--text-muted` | ~~`#5C6670`~~ **`#55554F`** | default muted — row body copy |
| Text quiet | `--text-quiet` | **`#6D6D68`** | sub-navigation, counts, secondary labels |
| Text faint | `--text-faint` | **`#777773`** | micro-type only (10–11px): timestamps, "waiting" ages |

**Corrected 2026-08-25 against the live prototype**
*(see [[Bay system reference — external comparison]] §Verified)*: the first pass of this restyle
recorded **one** border token and **one** muted ink. Both are wrong — the system uses a **ramp**. Container edges and inner rules are
deliberately different weights (that is what stops a dense table reading as heavy), and muted text
steps down with size rather than being one grey. A separate sunken tone carries toolbars.

**Changed 2026-08-25:** ~~Neutrals are not true grays — each carries a faint blue undertone so the
whole app reads as one family.~~ Neutrals are now **warm-achromatic**, in Bay's idiom: the family is
held together by *contrast and geometry* rather than by a shared undertone, and the only chroma in
the chrome is the brand/action blue. This is a real trade — the old rule bought a soft, unified
wash; the new one buys crispness and makes the status chips read louder against their surroundings,
which is what the status vocabulary is for.

**`--page` is not white**, though Bay's operational canvas is. Bay's own review surface uses
`#F4F4F2` and Knot takes that value, because the 2026-07-06 lock rejected a pure-white canvas for
glare and nothing about the restyle changes that reasoning. This is a deliberate, reasoned decline
of one line of Bay's spec.

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

### The scale *(measured from the live prototype, 2026-08-25 — the written reference omitted all of this)*

Display type is the loudest part of the language, and it is **not** "big and bold". It is
**regular weight, set tighter than its own size, and negatively tracked** — the tracking gets
tighter as the size grows.

| Role | Size | Line-height | Tracking | Weight |
|---|---|---|---|---|
| Hero (marketing / empty-state) | `4.48rem` ≈ 72px | **0.93** | `-0.065em` | 400 |
| Page title (h1) | `2.56rem` ≈ 41px | **0.98** | `-0.06em` | 400 |
| Card title / stat number | `1.75rem` = 28px | 1.0 | `-0.045em` … `-0.07em` | 400 |
| Section heading | 15px | normal | 0 | 600 |
| Body | 13px | normal | 0 | 400 |
| Row title | 13px | normal | 0 | **700** |
| Sub-navigation, counts | 11px | normal | 0 | 400–600 |
| Micro (ages, timestamps) | 10–11px | normal | 0 | 400 |

Sizes step on a **1.6 ratio**. Two rules that matter more than the numbers:

1. **Display weight is 400, never bold.** Size and tracking carry the emphasis. A bold 41px
   heading is the single fastest way to stop looking like this system.
2. **Negative tracking is proportional, not fixed.** Roughly `-0.045em` at 28px growing to
   `-0.065em` at 72px. Body text and anything under 15px is tracked normally.

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

## Components

The named inventory. **A component is named here *before* a screen uses it twice** — UI law 7 has
no enforceable meaning until the pieces have names, and `_blocker_item` getting built twice is the
evidence. `/design-preview` renders each of these once, in every state.

### The page pattern
Every screen, without exception: **eyebrow → display heading → one primary action, top right.**
On mobile the primary action becomes a **full-width bar** below the heading instead.

### Structure

| Component | Spec |
|---|---|
| **Card / panel** | White, `1px --border`, square, **no shadow**. Section rules inside use `--border-inner` |
| **Inverted panel** | `#101010` background, white text, `1px --border`, 18px padding. For context//"what this screen is" copy — used sparingly, it is the loudest thing on a page |
| **Stat tile row** | Tiles **share their 1px rules** rather than sitting in a gapped grid. Each: number at 28px/400 tracked `-0.07em`, label beside it, arrow right. Reflows 3-up → 2-up → 1-up, borders collapsing throughout |
| **List row** | CSS grid, ~76px, `12px 20px` padding, divider `--border-inner`. Title 13px/**700**, muted subline 11px `--text-faint`, then columns, then a right arrow. **One row component** — the attention queue and every list reuse it |
| **List toolbar** | `--surface-sunk` bar at the top of a list: search left (with icon), record count right, 11px `--text-quiet` |
| **Status badge** | 1px border + pale fill + dark text of one hue family, square. See §Status colors |
| **Square bullet** | 7×7, no radius. `--action` when the row is live, `#B9B9B4` when it is not |
| **Avatar** | 32×32 **square**, `#101010` fill, white initials. Never a circle |

### Navigation *(desktop)*

| Part | Spec |
|---|---|
| Sidebar | White, `1px --border` right edge, collapsible to an icon rail. Brand + collapse toggle top, workshop name under it |
| Group label | Eyebrow style, above each cluster of items |
| Nav item | **42px** tall, 13px, `#4E4E49`, padding `0 10px` |
| Nav item, selected | **Full-bleed solid `--action`, white text.** Not a tint, not a left bar |
| Sub-nav item | 33px, 11px/600, `--text-quiet`, indented |
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
Invariants for every screen, in the spirit of [[Design laws]].

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
   `app/assets/stylesheets/application.css`, and both wear hues the status vocabulary has reserved.
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
- **Where the front door lives.** `app/views/workshops/show.html.slim` already carries a reserved
  slot — *"Restore a button here once intakes#new exists."* Every sibling control in that row is
  `btn-outline-secondary`, so a single solid `--action` button there **satisfies** law 3 rather
  than straining it, and law 1 argues against a persistent nav item (the header is chrome, and
  chrome whispers).
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
- **Still open, and it is a real fork:** Bay gives every role its own dashboard with a role-scoped
  attention queue and destinations. Knot has **one board for everyone** with per-control gating.
  [[Design laws]] #3 ("dashboards are queries, not tables — new viewpoint = new scope, never a new
  model") makes role-scoped views *cheap*, so this is a genuine choice rather than a cost question.
  Not ruled.

#### The role matrix *(measured from the live prototype, 2026-08-25)*

The reference implementation's nav is **a shared spine plus role-specific additions, and exactly one
role-specific primary action.** Recorded because it is the concrete form of the "durable work areas
only" rule, and because Knot's equivalent question — one board for everyone vs a dashboard per role
— is still open (below).

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
  the behaviour spec; [[Deferred design]] holds the payer-confirm design and its four-fork
  correction.
- **Rebuild-or-keep per screen** — by ruling, taken at each L3 step, not now.
- **Add a repair to an open intake**, the **jobsheet fill screen**, and the **owner token page** —
  still unbuilt, still outside this spine.
- **Devise restyle** — step 1's open question.
- **No vehicle-type column exists** (`vehicles` has only `vin`), so `inspection_type` can never be
  inferred from a plate. Whenever the intake screen carries the type picker, it is a real question
  to a human.
- **One board vs a dashboard per role** — the L1 fork above. Design law #3 makes role-scoped views
  cheap, so this is a design choice, not a cost one.
- **The sidebar, and what it drags in** — Stimulus (or a CSS-only pattern) for collapse and drawer,
  since no Bootstrap JS is loaded, plus an icon source for the rail. One coupled decision, not three.
- **UI law 3: "per screen" or "per local area"** — settle it with X4, not before
  ([[Design system]] UI laws).
- **The re-derived status chips have not been sample-compared.** [[Design system]] flags this; the
  `/design-preview` route from X1 is where it gets done.

---

## Deferred
- **Dark mode** — surfaces will derive from brand steel blue (the navy-chrome sample read as a
  good dark theme and was liked for it). See [[Deferred design]].
- Devise views stay unstyled until the theme grows.
- Spacing/grid system — extract later from real screens, like an abstraction from real code.
- Animation rules — v1 has essentially none; Turbo defaults are fine.

## Implementation
All tokens are CSS custom properties in `app/assets/stylesheets/application.css` — the single
source of visual truth. ~~Plain CSS, no framework~~ **Changed 2026-07-06:** looking at where the
project is heading, the base is **Bootstrap 5.3.3** (vendored `bootstrap.min.css`, no build step,
no CDN) with `application.css` as a **brand layer** on top — it maps Bootstrap's CSS variables
(`--bs-body-bg`, `--bs-btn-*`, links) to the tokens above and holds branding-only classes
(`.wordmark`, `.text-brand`, `.hero*`, `.prop-num`). Rules: theme through Bootstrap variables,
never fork `bootstrap.min.css`; new screens compose Bootstrap utilities + brand classes, no
inline styles. Token values in this note still win if anything differs.

**Token additions from the 2026-08-25 correction** (on top of the re-lock below): `--border-inner`,
`--surface-sunk`, and the muted ink ramp `--text-muted` / `--text-quiet` / `--text-faint` — plus the
three floating-layer shadows. The type scale is `--font-sans` sizes with proportional negative
tracking; nothing here needs a build step.

**What the 2026-08-25 re-lock costs in `application.css`** (all through Bootstrap variables, still
never forking the vendored file): `--font-sans` → Helvetica/Arial · the neutral tokens re-pointed
(`--page`, `--border`, `--text`, `--text-muted`) · `--bs-border-radius` family → `0` · card and
button borders made visible at 1px · `.stage-badge` re-cut from two-part to **border + fill + text**
· flashes moved off `.alert-info`/`.alert-warning` onto neutral tokens (they were wearing reserved
status hues — see §The design plan). It is a **token-level restyle, not a rewrite**: no
template needs new markup except the badge and the flash. Bootstrap **JS is not
loaded** — add via importmap only when a component (dropdown/modal) demands it.

## Related
- [[Design laws]] · [[Tech stack]] · [[Product overview]] · [[Deferred design]] ·
  [[Bay system reference — external comparison]] (provenance for the 2026-08-25 re-lock, and the
  live-prototype verification behind the type scale, ramps and component specs) ·
  [[Screen map]] (what screens *exist* — the code reflection this note plans against) ·
  [[Sprint plan]] S5.5a–i (the build order this note's plan half feeds)
