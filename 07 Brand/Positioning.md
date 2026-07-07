---
type: context
created: 2026-07-06
updated: 2026-07-06
---
# Brand positioning
Locked 2026-07-06 (worked out interactively — options pressure-tested, choices deliberate).
The words-side twin of [[Visual theme]]. Every piece of marketing derives from this note.

## The name
**Knot.** Committed. The story: the system **ties all the playing parties together** — front
desk to foreman, foreman to mechanic, mechanic to warehouse, and the workshop to the
customer watching their own car. Every handoff is a tie, and **a good knot doesn't slip**.
(The UI-flow diagram's "handoff = the tie" is this story; the name and the product spine —
handshakes — say the same thing. The customer link is the last strand, not a bonus feature.)

**Known weakness — spoken as "not."** Silent K; word-of-mouth is our channel, and some of the
audience won't have the rope image to fall back on. Rules that follow:
- **The name never travels alone** in speech or cold contexts: always "Knot — no job goes
  unseen" or "Knot, the job board for workshops."
- Wordmark makes the **K** unmissable; the logo should *be* a knot, so the eye resolves what
  the ear can't.
- **No homophone puns** ("Knot a problem") — lean on the rope, not the pun. See
  [[Voice and tone]].
- Domain: `knot.com` is The Knot (weddings — different industry, low confusion risk); shop in
  `knotapp.com` / `useknot.com` territory. Unresolved.

*(Naming note: **Pilot** stays the internal working name — repo, databases, this vault. **Knot**
is the product/brand. Nothing internal needs renaming.)*

## Audience & market
- **Buyer:** small independent workshop **owner-boss** (Malaysia first). Decides alone;
  pragmatic — money, control, reputation.
- **Veto:** the crew — and the veto is **passive**: they don't refuse, they just stop updating.
  Ease beats persuasion.
- **Real competitor:** doing nothing. The whiteboard is free and "mostly works." Every claim
  must survive *"why pay to replace what's free?"* — so we never sell the replacement, we sell
  what the whiteboard can't do.

## Message hierarchy (the spear)
1. **Villain:** the whiteboard, the boss's memory, "I told him already."
2. **Promise:** *No job goes unseen.* Every job, every handoff, visible to the whole team —
   the shop stops living in one person's head.
3. **Crew insurance (ease first):** updating Knot is faster than walking to the board.
   Secondary: the handshake protects whoever did their part — *unvalidated, keep out of copy
   until tested.*
4. **Kicker:** your customers watch their own car's progress — **the one thing no whiteboard
   can ever do.** The calls stop. *(Build dependency: "the calls stop" needs the ETA field
   staying in V1 — [[Product gaps]] #1, Sprint task S6.6 — or the owner still calls to ask "when".)*

## Pricing posture (number deferred)
Flat **per workshop, never per user** — per-seat pricing punishes crew adoption, and crew
adoption is what survives the veto. Brand signals: one honest fee, get everyone on.

## Assumptions to validate with real workshops
- Do mechanics actually feel "proof of handoff" as protection, or is it boss-mandate + ease
  that drives use?
- Do target shops even use whiteboards, or is it memory + WhatsApp? (Villain imagery may need
  adjusting.)

Same bet as [[Product gaps]] #5 (the ack model assumes full crew adoption) — the marketing claim
and the product's graceful-degradation design (Sprint task S4.6) rise or fall together; one
workshop validation session settles both.

## Related
- [[Voice and tone]] · [[Product overview]] · [[Visual theme]] · [[Builder identity]]
