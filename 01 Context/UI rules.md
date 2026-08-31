---
type: context
created: 2026-08-31
updated: 2026-08-31 (Session 39 — created; extracted from [[Design system]] and the repo's CLAUDE.md design rules)
---
# UI rules — the card

> **Working summary, held open *during* UI work.** [[Design system]] is authoritative; if this
> card disagrees with it, **the card is wrong — fix the card.** The reasoning behind every rule
> lives there; nothing here is new. The seven rules were written (2026-08-25) after a screen was
> built that broke five of them at once. They are checkable — check them.

1. **Bootstrap is the default answer.** If Bootstrap ships it — `.form-control`, `.form-label`,
   `.form-check-input`, `.form-select`, `.btn`, `.table`, focus rings — **use it and theme through
   its own variables. Never hand-roll an equivalent.** A hand-built input looks fine and silently
   forks the system.
2. **Spacing is Bootstrap's ladder only:** `0 · .25 · .5 · 1 · 1.5 · 3rem`. **12px and 32px do not
   exist.** Spacing lives in utility classes in the templates, never in `application.scss` — a
   padding or gap value written in our CSS is the smell.
3. **Type is six steps:** `.display-6` · `.fs-3` · `.fs-5` · `.fs-6` · body `.875rem` ·
   support `.75rem`. **Display weight is 400, never bold.** Tracking in `em`, never `px`.
4. **Colour comes from tokens.** No raw hex in a component, ever. Status colours are **sacred** —
   red/amber/green/blue-badge only ever mean job state.
5. **Square. 1px borders.** `$border-radius` and its five variants are `0`. **Shadows only on
   things that float** (drawer, tray, popover) — never on inline structure, and never as a
   hand-rolled focus ring.
6. **One solid primary action per screen.** An outlined button is a *style*, not a second primary.
7. **Every screen is a recorded layout archetype** — the app page, or the gate. A screen fitting
   neither is a **new archetype, recorded in [[Design system]] before it is built.**

**The component set is closed.** A screen is a recomposition of the components in
[[UI rollout]] §The component set — that table is their home. Anything new gets added there
*first*, then built.

**Scope: desktop only, for now** *(2026-08-25, builder)* — thumb-first is dormant, not deleted.

## While working
- Keep `bin/rails dartsass:watch` running; spacing goes in the template's utility classes,
  values go in tokens, never loose in the Sass.
- Per-screen reasoning → the screen's note ([[Screen decisions]]). The working loop →
  [[Workflows]] §UI.
- `/design-preview` ([[Sprint plan]] S5.5a) is the page that will render every component from the
  real tokens — **not built yet**; until then, this card and the suite are the check.
