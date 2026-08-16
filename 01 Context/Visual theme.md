---
type: context
created: 2026-07-06
updated: 2026-07-28 (waiting-pin ageing bands — ADR-011; ships muted in S4, colour deferred to S6.7; no new colours)
---
# Visual theme
Locked 2026-07-06 (worked out interactively — samples compared, choices deliberate).
One theme for **both** clients (workshop staff + vehicle owner) — one predictable feel, no split identity.
Mood: **industrial & confident** · light · high-contrast.

## Color roles
Colors are named by **job, not rank** (no "primary/secondary" — a name should tell you what to
reach for). There is deliberately **no secondary hue**: a second decorative color would compete
with the status vocabulary. A "secondary button" is a *style* (outlined, quiet), not a color.

| Role | Token | Value | Job |
|---|---|---|---|
| Brand | `--brand` | `#22456B` | steel blue — wordmark, heading accents. Scarce on purpose. |
| Action | `--action` | `#2D5E94` | buttons, links, focus — "you can act here" |
| Action hover | `--action-hover` | `#24507F` | hover/pressed |
| Page | `--page` | `#F5F6F8` | app background (cool gray, not pure white — glare) |
| Surface | `--surface` | `#FFFFFF` | cards, app bar |
| Border | `--border` | `#E2E6EB` | hairlines |
| Text | `--text` | `#1C2B3A` | body (dark slate) |
| Text muted | `--text-muted` | `#5C6670` | timestamps, hints |

Neutrals are not true grays — each carries a faint blue undertone so the whole app reads as one
family. Chrome style: **all-light** ("Option A") — white app bar, blue only in wordmark + actions.

## Status colors (SACRED — semantic, never decoration)
The product's real color language; a manager reads color before text. Badge = pale bg + dark text
of the same family. **Identical in every view** — workshop, owner page, future dark mode.

| Status | bg | text |
|---|---|---|
| Registered / neutral | `#F1EFE8` | `#444441` |
| In progress / info | `#E6F1FB` | `#0C447C` |
| Waiting / aging | `#FAEEDA` | `#633806` |
| Blocked / danger | `#FCEBEB` | `#791F1F` |
| Done / success | `#EAF3DE` | `#27500A` |

Known quirk to watch: "In progress" blue vs Action blue coexist (pale badge vs solid button) —
keep that contrast.

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
- **System stack** (locked): `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica
  Neue", Arial, sans-serif` — 0KB, can't fail, native per device.
- Mono for plates & job numbers: `ui-monospace, SFMono-Regular, Menlo, Consolas, monospace`.
- Escape hatch if it ever "looks funky": self-hosted Inter (~100KB, closest free match to SF Pro),
  one-line swap via `--font-sans`. Considered and consciously deferred — not forgotten.

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
5. **Thumb-first for anything a technician touches.** Tap targets ≥44px, primary actions in
   bottom-half reach, no hover-only interactions, works one-handed. Tech screens never go dense.
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

**Posture note:** each role's device decides which laws bite hardest — technician = phone
(law 5 hard), SA/manager = PC (laws 9–10 hard), owner = phone read-only (laws 4 + 7, zero
actions). One system, three postures. Matches the device-posture decision (Session 3, [[Tech stack]]).

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
inline styles. Token values in this note still win if anything differs. Bootstrap **JS is not
loaded** — add via importmap only when a component (dropdown/modal) demands it.

## Related
- [[Design laws]] · [[Tech stack]] · [[Product overview]] · [[Deferred design]]
