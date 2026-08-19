---
type: reference
module: M1
updated: 2026-08-19 (Session 34 — F6 jobsheet storage decided, ADR-015; prior: Session 33 — F6 jobsheet re-pointed at ADR-014, fixed/product-defined; prior: 2026-08-17 Session 32 — F1 board regroup deferred, F6 jobsheet marked not-built, ADR-012 citation)
---
# Features overview

The **capability-cluster** view of Module 1 — the layer that sits *between* the three marketing
pillars and the build slices in [[Roadmap]]. A feature is "a thing the product does"; a slice is
"a thing we build in order"; a screen is "a place you land". They don't line up 1:1, so this note
names the features explicitly (the vault tracked slices + concepts, but only ever named one
feature, `M1-F1`).

Screens per feature are the ones inventoried in [[Screen map]].

## The features

| Feature | What it does | Slice(s) | Serving screens | Status |
| --- | --- | --- | --- | --- |
| **F1 · Live board** | "Where is every car right now" — active work, aging, done-awaiting-delivery, needs-your-attention | 3 + 5.5 | Dashboard, Intake board, All-intakes | built (repair-grouped); **regroup by car deferred (S6)** — [[ADR-012 Intake-Job two-level aggregate]] |
| **F2 · Intake→repair lifecycle & timeline** | The two-level engine (open visit, add repairs, floor moves, deliver/cancel) **and** the per-visit timeline that records it | 1 + 2 + 5.5 | Intake show, Job show, **Intake timeline**, Open visit | engine built; **create-path UI is the S5 hole**; timeline built |
| **F3 · Blockers & holds** | Overlay axis: raise/resolve/note; job blocks *done*, intake blocks *delivery*; role-gated + catalog | 4 | Blocker show, Workshop setup, forms on Intake/Job | built |
| **F4 · Acknowledged handoffs** | Nothing stalls silently — "waiting on «name»", pending acks surfaced on the board | 5 | *(cross-cuts board + Intake/Job)* | built (aging colour deferred, S6.7) |
| **F5 · Crew, roles & access** | Bilateral invite, roles, gating, tenancy | 2 | Workshop setup, Staff show/create, Landing | built |
| **F6 · Customers, vehicles & jobsheet** | The records intakes attach to, plus the fixed inspection form | 6 | Customers, Customer show, Vehicle show, *(jobsheet)* | customer/vehicle built; **jobsheet designed, not built** — fixed/product-defined ([[ADR-014 Jobsheet is a fixed product-defined inspection]]), storage decided ([[ADR-015 Jobsheet answers are rows against a frozen question set]]), no field-admin (dropped) |
| **F7 · Customer status page** | Public "how's my car?" token link on the visit | 7 | Status page | needs design; not built |
| **F8 · Reporting (aggregate)** | Workshop-wide numbers — sums/averages across visits | 8 | *(feeds Dashboard health)* | needs design; **which reports still TBD** |

## Per-visit time analytics ≠ reporting

A deliberate split, easy to get wrong:

- **Per-visit** (this car's story) — **time-in-stage**, **blocked-by-department** — is a *drill-down
  on the record*. It lives on **Intake show / Job show / the Intake timeline**, under **F2**. This is
  what [[ADR-002 V1 scope]] means by "time-in-stage / time-blocked tracking", and it's V1.
- **Aggregate** (the whole workshop) — sums and averages across many visits — is **F8 Reporting**,
  and *which* reports exist is still undefined.

Both read from the same source: the **[[Event log]]** (the stored timestamps). Per-record it renders
as a **timeline**; aggregated it becomes **reports**. The Event log is shared infrastructure under
F1 (aging), F4 (acks), F2 (timeline), and F8 (reports) — not a feature or a screen itself.

## The marketing collapse

The home page folds these eight into **three stories** — the rest is the machinery that makes them
true:

- **One live board** → F1
- **Handoffs that hold** → F3 + F4
- **Owners stop calling** → F7

## The V1 fence (what "V1 = monitoring only" actually means)

[[ADR-002 V1 scope]] is a scope *fence*, not a line through these features. The test for "is it V1?":
> does it help answer **"where is every job right now?"** (staff) or **"what's happening to my car?"**
> (owner)?

By that test, **F1–F7 are all V1** — including the owner status page, which ADR-002 lists explicitly.
The only fuzzy edge is **F8 aggregate reporting** (V1 tracks the per-record time math, but *reports*
across visits aren't specced).

What's **outside** the fence isn't a slice of these features — it's whole other domains, deferred to
later modules:

- **Parts / warehouse → V2.** V1 models "waiting on parts" only as a *blocker reason*, no parts entity.
- **Technician skill / training → V3.**
- **Money** (pricing, quotes, invoicing) → deferred; "awaiting customer approval" is a blocker type,
  and any future invoicing integrates AutoCount / SQL Account rather than building our own.

So: **one fence around F1–F7** (= V1), F8 as the one undefined edge, and parts/tech/money sitting
entirely outside.

## Open
- **F8** — decide *which* aggregate reports earn their place (candidates: time-in-stage distribution,
  time-blocked-by-department, active/stuck/unassigned counts, cars in-vs-out, awaiting-collection).
  Per-blocker two-bucket attribution (owner time vs raiser time) is the math to build on; the
  *display* is what needs design.
- **Intake timeline** — decide whether it's its own page or a section on Intake show.

## Related
[[Roadmap]] · [[Screen map]] · [[Event log]] · [[M1-F1 Status flow and transitions]] ·
[[Intake]] · [[Job]] · [[Blocker]] · [[Stage model]] · [[Job visibility]] ·
[[ADR-002 V1 scope]] · [[ADR-012 Intake-Job two-level aggregate]]
