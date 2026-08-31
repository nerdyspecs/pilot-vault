---
type: reference
created: 2026-08-25
updated: 2026-08-25 (Session 36 — verified against the live prototype at bay-qwyg.onrender.com; two restyle errors corrected, the omissions recorded in [[Design system]])
---
# Bay system reference — external comparison

**What this is.** On 2026-08-25 (Session 36) the builder brought in the *Bay system reference* — a
concise product/system/visual reference for **Bay**, a job-monitoring system for vehicle workshops.
It is a **parallel design of the same product**: same domain, same roles, contemporaneous decision
dates (2026-08-04, 2026-08-09). This note records what came from it, what did not, and why.

**Status: NOT binding.** Bay's document is an external source. Knot's own hierarchy is unchanged —
ADRs, then the detailed notes, then this. Where Bay conflicts with a recorded Knot decision, **Knot
wins**, and the conflict is recorded below rather than quietly resolved.

**Why it was worth reading at all.** Two independent designs of the same product agreeing on
something is evidence; disagreeing is a question worth asking. Several of Bay's rules turned out to
name things Knot had decided in practice but never written down, and one of them exposed a law Knot
was breaking app-wide.

---

## The finding it produced about Knot's own code

Bay's *"desktop content uses the available canvas width… not constrained to a centred fixed-width
page"* prompted a check of ours. **Every workshop screen is `col-12 col-md-8 col-lg-6`** — a
half-width centred column on a PC: the board, the visit, the repair, customers, crew, blockers.

That is a stretched-phone layout, which **UI law 9 forbids** ("PC buys density, not size… never a
stretched phone layout"). Law 9 is violated app-wide, and it caps what the board can ever show.
Recorded as finding 6 in [[Design system]]; it is Knot's own finding, independent of whether
anything else here is adopted.

---

## Adopted

| From Bay | Where it landed | Note |
|---|---|---|
| Helvetica / Arial | [[Design system]] §Typography | Replaces the native system stack. Gives up "native per device" deliberately — a shared face is most of what makes two systems look like one house |
| Ink `#101010`, canvas `#F4F4F2`, visible 1px borders | [[Design system]] §Color roles | Neutrals moved from blue-undertoned to **warm-achromatic**. Bay's *"operational canvas is white"* is **declined** — the 2026-07-06 lock rejected pure white for glare, and the restyle doesn't change that reasoning |
| Square corners · 1px borders · restrained elevation · no pills | [[Design system]] §Geometry (**new section**) | Knot had **no shape rule at all** and was on Bootstrap's rounded defaults by accident |
| Desktop ~40px / mobile ≥48px controls | [[Design system]] UI law 5 | Knot specified only the phone number (≥44px) and had no desktop target |
| Colour-independence; visible focus; `prefers-reduced-motion`; mobile safe-area | [[Design system]] §Interaction and accessibility (**new section**) | Knot practised most of this and had written down none of it |
| A component-inventory **route** (`/design-preview`) | [[Design system]] §Components · [[Sprint plan]] S5.5a | The single best idea in the document: it makes UI law 7 *checkable* instead of asserted, and it would have caught the duplicated `_blocker_item` immediately |
| "Only durable work areas in primary navigation; detail, create flows and item actions reached contextually" | [[Design system]] L1 | Directly answers Knot's open question of how Customers / Crew / Blocker types get reached |
| Sidebar → icon rail; content responds to collapse; mobile drawer mirrors the same hierarchy | [[Design system]] L1 | A credible answer to L1 that also fixes the law-9 problem. **Carries a real cost** — Knot loads no Bootstrap JS, so collapse and drawer need Stimulus or a CSS-only pattern |
| The attention-item test: what is blocked · who owns the next action · how long has it waited · fastest safe action | [[Design system]] L1 | Sharper and more testable than law 4's glance test |

### Re-derived rather than copied

**The status palette.** Bay has **no status colour system** — its only colour semantics are
"blue = live, grey = historical" on a timeline. So there was nothing to copy, and the builder ruled
(2026-08-25) to **re-derive Knot's five against Bay's neutral system** rather than keep values
mixed for the old surface. The hue families did not move — that is what keeps the reserved words
reserved. What changed is structure: a badge is now **1px border + pale fill + dark text**,
square-cornered, because with a visible border the fill can lighten. Values and the
pending-sample-comparison caveat live in [[Design system]] §Status colors.

*No footnote is owed against [[ADR-011 Acknowledgement as stored visibility]]:* that ADR says
explicitly **"Colour — deferred, not decided here."** The ageing bands live in [[Design system]] and
[[Deferred decisions]], so re-deriving the palette moves values the ADR never fixed.

---

## Verified against the live prototype *(2026-08-25, `bay-qwyg.onrender.com`)*

The pasted document turned out to be a **thin summary of a much more specific system**. Walked the
login, the component inventory, all four role dashboards, a destination view, and the mobile
posture. What the prose omitted, measured off the running app, is now recorded in [[Design system]]
(§The scale, §Components, §Geometry, §Color roles).

**Two things the first pass of the restyle got wrong, both corrected:**

1. **"Restrained elevation" is not "no shadows".** Inline cards genuinely have none, but every
   *floating* layer carries a specified shadow — drawer `12px 0 32px rgba(16,16,16,.16)`, tray
   `0 12px 32px rgba(16,16,16,.18)`, floating control `0 4px 12px rgba(16,16,16,.08)`, scrim
   `rgba(16,16,16,.28)`. The rule is **elevation means "above the page", never "important"**.
2. **One `--border` and one muted ink are both wrong — each is a ramp.** Container edges `#D8D8D4`
   (independently derived, and confirmed exactly), inner rules `#E5E5E1`–`#E8E8E5`, toggle
   `#DEDEDB`; muted ink steps `#4E4E49` → `#55554F` → `#6D6D68` → `#777773` with decreasing size.
   Plus a third surface `#F7F7F5` for list toolbars.

**Omitted from the document entirely** — the type treatment (a 1.6 modular scale at **weight 400**
with **proportional negative tracking**, display line-height *below* 1.0), the eyebrow component,
the page pattern (eyebrow → display heading → one primary top-right), the whole nav spec (42px rows;
selected = **full-bleed solid accent**, not a tint; pinned account chip), the role matrix, the
inverted `#101010` context panel, square 7×7 bullets, square 32px avatars, border-collapsed stat
tiles, the list toolbar, the `bay-` class convention, and the responsive rules (primary action
becomes a full-width bar; 52×52 square floating control).

**Confirmed as documented:** square corners, white cards, 1px borders, ~40–42px desktop controls.

**Disproved: the canvas.** `#F4F4F2` is the *review-surface* value. The shipped CSS sets
`.bay-app-shell { background: #f4f4f2 }` and then **overrides it to `#fff`** — the operational canvas
is white, exactly as Bay's own written spec said. Knot's first pass recorded a "reasoned decline" of
that line and was wrong; corrected in [[Design system]]. The grey's real job is the **hover** tone
(`.bay-action-row:hover`, `.bay-signal-strip a:hover`).

**Bay has no status *colour* system — but it does have badge components.** *(Corrected 2026-08-25:
an earlier version of this line claimed no `badge`/`chip`/`pill`/`status` class existed anywhere.
That was based on dashboard markup alone and was too strong.)* The stylesheet carries `.status`,
`.status-muted`, `.status-active`, `.status-ready`, `.status-risk` and `.label-chip` — but they are
**monochrome and blue only** (grey fill, blue-tint fill, solid blue), flat, borderless, 10px. There
is no red/amber/green vocabulary and nothing semantic to copy, so **the substance holds**: the
builder's ruling to *re-derive* Knot's five was made on accurate information. What is worth noting
is the shape difference — Bay's badges are flat fills, Knot's are border + fill + text.

> [!warning] **`/design-preview` is an empty shell.** All nine sections (`typography`, `colour`,
> `buttons`, `fields`, `status`, `navigation`, `data`, `feedback`, `bootstrap`) return **empty
> Turbo frames**. The sidebar and the IA exist; there is no content behind them. The idea remains
> the best one taken from Bay — but it is **unproven there**, and Knot would be the first to
> actually populate it ([[Sprint plan]] S5.5a).

**Two patterns deliberately not copied**, both recorded in [[Design system]] §Not to be copied:
list rows are `div`s in a grid rather than a `<table>` (an accessibility regression for a dense
board), and attention rows **drop the owner and waiting columns on mobile**, discarding two of the
four questions the component exists to answer.

---

## Corroborated — nothing changed, but the agreement is worth knowing

- **"Use time in the current stalled state rather than job creation date alone."** This is exactly
  what [[ADR-011 Acknowledgement as stored visibility]] does. Two independent designs reaching it
  is a good sign the reasoning holds.
- **Timeline colour semantics — urgency in text, not a competing colour rule.** The twin of
  ADR-011's "colour the chip, never the row; the chip's own text says which". Independent
  confirmation, no change.
- **Intake under a minute** — the same budget [[User stories]] records.
- **"Surface attention and exceptions rather than require staff to search every job."** Knot's
  [[Architecture laws]] #3, "dashboards are queries, not tables", said the same thing structurally.

---

## Rejected, with reasons

| From Bay | Why not |
|---|---|
| **`Employment` as the authorisation edge** | **The loudest one.** Knot *retired* this concept — `WorkshopEmployment`/`WorkshopOwnership` were collapsed into `WorkshopStaff` on 2026-07-21 ([[ADR-010 WorkshopStaff supersedes the edge split]], Session 25). Importing Bay's role table would resurrect a dead vocabulary into notes that took a session to clean |
| Brand asset `bay.svg`; the name "Bay blue" | Another product's **badge**. Knot keeps its own wordmark, K tile and square avatar |
| ~~Bay blue `#2727D9`~~ **— adopted after all** | Declined in the first pass, then **reversed 2026-08-25** when the builder leaned the design the rest of the way toward Bay. `--action` is now `#2727D9` and `--brand #22456B` is **retired** — a steel-blue mark beside an indigo button reads as broken. See [[Design system]] §Color roles |
| Stack: Rails 8.1 · ERB · SQLite · Propshaft · Docker/Render · Netlify | Conflicts with Knot's pinned 8.0.5, Slim views, vendored Bootstrap with no build step, and PaaS deploy ([[Tech stack]], ADR-001) |
| `lucide-rails` | A third gem after devise and slim-rails. **Deferred, not refused** — the icon rail is the only real use, and the sidebar it belongs to is still an open call. Decide both together; inline-vendored SVGs are the no-dependency alternative |
| The `/` role chooser | Knot has real Devise auth and `Permissions`. Adopting a prototype's role selector would be a regression |
| Signed jobsheet snapshot; signature capture; amendments | Knot went a different way — [[ADR-014 Jobsheet is a fixed product-defined inspection]] and [[ADR-015 Jobsheet answers are rows against a frozen question set]] |
| Bay's entire **"Still Proposed"** list | Knot has already *decided* nearly all of it: job stages and transitions, blocker types with raise/clear roles and `blocks`, acknowledgement rules. Knot is ahead here; there is nothing to take |

---

## Vocabulary differences worth not blurring

Bay and Knot use several of the same words for slightly different things. Kept apart deliberately:

- **Intake.** Bay: "the workshop's digital jobsheet, can contain multiple Jobs." Knot: the *visit*,
  with `Jobsheet` a separate 1:1 record beside it ([[ADR-012 Intake-Job two-level aggregate]]).
  Close, but not the same object — do not merge the definitions.
- **Employment** vs **WorkshopStaff** — see Rejected above.
- **Foreman.** Bay folds it into `workshop_manager`. Knot's role enum is its own; no change implied.

## Related
[[Design system]] · [[Screen map]] · [[Architecture laws]] · [[Tech stack]] · [[Deferred decisions]] ·
[[ADR-010 WorkshopStaff supersedes the edge split]] · [[ADR-011 Acknowledgement as stored visibility]] ·
[[ADR-012 Intake-Job two-level aggregate]]
