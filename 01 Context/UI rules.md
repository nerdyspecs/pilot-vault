---
type: context
created: 2026-08-31
updated: 2026-08-31 (Session 39 — forks ruled as 22–23: narrow red exemption on form controls, no icons; prior: rules 8–21 captured: composition, ink jobs, behaviour, full canvas, form anatomy — argued against three standalone mocks the same day; prior: created, extracted from [[Design system]] and the repo's CLAUDE.md design rules)
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

## Composition *(8–12 captured 2026-08-31, Session 39 — demonstrated in the mocks; discussion in [[worklog]] S39)*

8. **One grid.** Page head, section heads, panels and table text share the same left edges. An
   element whose left edge aligns with nothing is wrong.
9. **One rhythm per level, off the ladder.** Sections separate by `1.5rem`, panels pad `1rem`,
   component internals `.5/.25rem` — the same value for the same job everywhere on the page.
10. **One section anatomy.** Every section is *head (title + optional right-side note or action)
    → body*. A screen never contains two differently-shaped sections.
11. **One focal point.** One display-size element, one solid primary — the weight budget is
    spent once per screen. *(Extends rule 6 from buttons to everything.)*
12. **Whitespace before boxes.** Separate with space first, a hairline second, a bordered panel
    last. Never border-in-border-in-border.

## Ink and colour *(13–15, same capture)*

13. **The grey ramp has fixed jobs.** `--text-muted` = row body copy · `--text-quiet` = counts
    and secondary labels · `--text-faint` = micro-type (eyebrows, timestamps, ages) ·
    `--placeholder` = inputs only, never copy. A grey chosen by eye instead of by job is the smell.
14. **Indigo means interactive — and only interactive.** `--action` colour ⇔ clicking it does
    something (or it's a whole-row target, rule 16). No decorative indigo.
15. **One dark block per screen, and it must inform.** The inverted panel carries the single
    most consequential computed statement, never mood copy. No urgent statement → no dark block.

## Behaviour *(16–19, same capture)*

16. **Every interactive element has its states.** Hover is `--hover` grey and nothing else;
    focus is the visible ring, never a shadow; rows that open something are whole-row targets
    with a hover affordance. State is never conveyed by colour alone.
17. **Empty is a state, not an absence.** A section with no data renders a quiet one-line
    message; it never disappears. An empty board is the first thing every new workshop sees.
18. **Numbers align right, in tabular figures.** Durations, counts, money — always. Ragged
    numerals kill glanceability, and glanceability is the product.
19. **Motion is feedback only.** ~150ms, state changes only, `prefers-reduced-motion` respected.
    No decorative animation — the one reserved exception is the acknowledge knot-tie, if it
    ever ships.

## Layout and forms *(20–21, same capture)*

20. **Full canvas.** Desktop content uses the available width; density (columns, a rail) absorbs
    wide screens, margins never do. A centred content column on an app screen is a bug. The
    **gate archetype is the exception** — it centres by design: `--hover` ground, 400px box,
    the 52px K badge, display-type greeting (anatomy lives in `application.scss` §auth).
21. **Form anatomy.** Label `.75rem` bold above the field, `.25rem` label gap, hint text in
    `--text-quiet` below, controls at `--control` height. Square checkboxes; radios keep their
    circle — the one deliberate radius in the system.

## The forks, ruled *(2026-08-31, Session 39 — delegated to the agent by the builder, "do what you think is best")*

22. **Form-validation red is a narrow exemption to the sacred palette.** On **form controls
    only** — invalid border + error line in the danger family — because users' red-means-error
    instinct is stronger than any internal vocabulary, and fighting it costs failed intakes.
    The exemption is *narrow*: chips, rows, panels and everything outside a form control keep
    red = blockers, exclusively. Bootstrap's `is-invalid` mechanics are the implementation
    (rule 1), themed to `--st-danger-*`.
23. **No icons — text labels only.** Now a rule, not a pending question. An icon set is a
    dependency by another name, every icon is a vocabulary the crew must learn, and the system's
    density comes from type doing the work. Revisit only if a real screen produces a label that
    cannot fit — that trigger, not taste, reopens this.

## While working
- Keep `bin/rails dartsass:watch` running; spacing goes in the template's utility classes,
  values go in tokens, never loose in the Sass.
- Per-screen reasoning → the screen's note ([[Screen decisions]]). The working loop →
  [[Workflows]] §UI.
- `/design-preview` ([[Sprint plan]] S5.5a) is the page that will render every component from the
  real tokens — **not built yet**; until then, this card and the suite are the check.
