---
type: experiment
created: 2026-07-07
updated: 2026-07-07
---
# UI experiments
Two standalone HTML mocks, both fully self-contained (open in any browser, no server, no
network). All state is in-memory — **reload = reset**. Role switching is a navbar dropdown,
no auth. Same model spine as the vault: [[Stage model]], [[Blocker]] overlay, acknowledged
handoffs ([[ADR-005 Acknowledged handoffs in V1]]), theme tokens from [[Visual theme]].

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
- [[Visual theme]] · [[Design laws]] · [[M1-F1 Status flow and transitions]] ·
  [[ADR-005 Acknowledged handoffs in V1]] · [[Positioning]]
