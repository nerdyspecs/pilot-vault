---
type: experiment
created: 2026-07-07
updated: 2026-08-25 (Session 36 — added knot-board-desktop.html, the reference implementation of [[Design system]] with the Bay-vs-Bootstrap scale switch)
---
# UI experiments
Two standalone HTML mocks, both fully self-contained (open in any browser, no server, no
network). All state is in-memory — **reload = reset**. Role switching is a navbar dropdown,
no auth. Same model spine as the vault: [[Stage model]], [[Blocker]] overlay, acknowledged
handoffs ([[ADR-005 Acknowledged handoffs in V1]]), theme tokens from [[Design system]].

**These are look/feel/flow references, not code to port.** The JS is throwaway vanilla —
the Rails app re-implements the logic properly (real ONE DOOR service object, Turbo).
What transfers: the layouts, the components, the CSS token values, and the interaction
patterns. UI only — the "photos" and "voice notes" are dummies; nothing uploads.

## 1 · `knot-v1-ui-flow.html` — the faithful V1 template
The V1 scope exactly as designed, all five views: service advisor (board / keyboard-first
intake / inbox), technician (phone, thumb-first), parts advisor (blocker queue), workshop
manager (dashboard + limbo), vehicle owner (token link, read-only). Full flow works
end-to-end: intake → assign → accept → start → block/resolve → done → ack → deliver.
Reusable component inventory: status badge, job card, mono plate, inbox item, timeline,
owner progress steps.

## 2 · `knot-design-vision.html` — the "full authority" imagination pass
Same spine, eight design swings (session 2026-07-07). We will circle back to these —
none are committed scope:

1. **Wall board (TV mode)** — zero-interaction live board; beats the whiteboard at its own
   game, defeats the crew's passive veto ([[Positioning]]).
2. **Heat aging** — rows/cards accumulate amber as time-in-stage passes its norm; you feel
   the stall before reading it.
3. **One-card technician flow** — the current job IS the screen; one giant button, swipe
   for the next job; the list is gone.
4. **Photo + voice evidence on events** — no typing with greasy hands; evidence flows to
   the owner page ("here's your worn pad"). The single biggest owner-trust upgrade.
5. **Owner story page** — updates with photos; at Delivered it freezes into a permanent
   service record (seed of v2 vehicle history).
6. **"Loose ends" + the knot tie** — limbo renamed; acknowledging plays the app's ONE
   animation (a knot pulling tight). Brand-as-interface ([[Positioning]] name story).
7. **Plate-scan intake** — camera mock; returning vehicles autofill. Intake in seconds.
8. **Exceptions-only manager view** — "Needs you" list + "N jobs moving normally" calm
   state; handoff health shown **by role, never by person** (no surveillance leaderboard).

## 3 · `knot-board-desktop.html` — the reference implementation of the design system
*(2026-08-25, Session 36)*

**Unlike experiments 1 and 2, this one is not exploratory — it is the design system made real**, and
it is what [[Design system]] was written from. The board (`workshops#show`) at desktop width, built
only from settled tokens: Bootstrap's spacing ladder and type scale, the re-derived status chips, the
white canvas, square geometry, the fixed full-height sidebar with the topbar inside the content
column, and the page-head bottom-alignment.

**It carries a scale switch** (bottom-right checkbox, pure CSS via `:has()` — no JavaScript, so it
survives sandboxed previews). Unticked renders the values measured off the reference implementation;
ticked renders Bootstrap's ladder. Same markup, only the token block swaps — the two sets are
verified to hold identical token names. **This is the A/B the builder ruled on**: Bootstrap won, and
that comparison is reproducible by opening the file.

Its `:root` block is deliberately written as a preview of what `application.css` should hold, so
[[Sprint plan]] S5.5b is largely a transcription rather than a fresh authoring job.

**What it deliberately does not show:** the amber Waiting/ageing band (deferred to S6.7 — the pin
ships muted per [[ADR-011 Acknowledgement as stored visibility]]), and any phone layout (desktop-only
by ruling). Its icons are hand-vendored inline SVG, not a gem — the `lucide-rails` question stays
open.

## Judgment calls embedded (not settled by the vault — candidates for [[Open questions]])
- **Deliver is gated on active blockers** (Hold for payment blocks Delivered) — gives the
  payment hold teeth, but bends "acks never block" in spirit.
- **Hold for payment can be raised on a Done job** — rubs against "Done freezes the job"
  ([[Design laws]] #8); payment holds naturally happen *at* Done.
- **Starting work implicitly accepts the assignment**; handoffs on closed jobs drop out of
  inbox/loose-ends.
- **Vision only: owner Approve button** — resolving a `cust_approval` blocker from the
  token page. Challenges law #8 (owners read-only); argued as "resolving a blocker
  addressed to them", not editing the job. Needs an ADR if ever adopted.

## Related
- [[Design system]] · [[Design laws]] · [[M1-F1 Status flow and transitions]] ·
  [[ADR-005 Acknowledged handoffs in V1]] · [[Positioning]]
