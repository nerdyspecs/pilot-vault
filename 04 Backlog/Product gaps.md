---
type: reference
created: 2026-07-04
updated: 2026-07-28 (#5 decided — ADR-011 reshaped to stored receiver; verdict amended)
---
# Product gaps — V1 design review
> **Reviewed: 2026-07-04** (Session 3). Snapshot of the vault as of this date — re-check how much
> is still true before acting; some items may have been built or decided since.

Blindspots found in a **product-design** review (2026-07-04) of the vault against
[[Main problem list]] and [[User stories]]. This is *product*, not code (code deferrals live in
[[Deferred design]]). **Nothing here is decided yet** — this is a backlog to weed through later;
each item has a *suggested* disposition only. The sharpest test used: does V1 answer the pains we
already wrote down?

## A. Gaps vs our own pain doc

| # | Gap | Why it matters | Suggested |
|---|---|---|---|
| 1 | **No ETA / promised-ready date** | Owner page shows *stage*, never *when*. Directly the L2 pain "estimate lead time." Without it the owner still calls — the page's headline promise breaks. One field on Job; SA sets it, owner sees it, overdue-vs-promise becomes visible. | **Pull into V1** (touches model — 1 field) |
| 2 | **No aging / stalled-job signal** | Ack-limbo catches *unpicked-up* work, not work that was acknowledged then sat. Pain #2 / manager "learns too late" is mostly slowness, not dropped handoffs. A "In-Progress > N days" highlight on the live list closes it. | **Pull into V1** (threshold, ~trivial) |
| 3 | **Incoming / expected jobs invisible** | Flow starts at Registered = intake done. L1 "see truck arrival"; L3 "can't see jobs coming in." May be fine for walk-in shops, but it's in the pain doc twice and never consciously rejected. | Decide & likely **defer** (needs a conscious call) |
| 4 | **Language (BM / Chinese)** | Our own open note "Language?" never landed. If mechanic screens are English-only, "mechanics actually use it" is at risk on a Malaysian floor. | Decide (V1 posture / defer) |

## B. Real-world blindspots (not in the pain doc)

| # | Gap | Why it matters | Suggested |
|---|---|---|---|
| 5 | **Ack model assumes full adoption** | Our constraint says "partial use must still provide value." But ack is receiver-must-act: non-adopting techs → flooded inbox + screaming limbo → "system doesn't work." Need a graceful-degradation story (SA ack *on behalf of*, logged? snooze/threshold on limbo?). | ✅ **Decided 2026-07-28** — [[ADR-011 Acknowledgement as stored visibility]]: the receiver is **stored at write time**, so "waiting on whom" is a plain query that is always answerable regardless of who adopts. There is **no inbox** to flood — the answer surfaces on the manager's board, and the manager (an adopter by definition) reads it and walks over. Acknowledgement is **implicit**: acting on a job clears what you owe, so a counter-only shop generates zero confirmation traffic and still reads correctly. Ack-on-behalf, snooze, and the 2026-07-24 holder model were all **rejected** (see the ADR). |
| 6 | **Floor device & login reality** | Techs auth via Devise/accounts/email — many floor techs have no email, won't do passwords on a shared greasy phone. Shared tablet per bay? PIN fast-switch? This decides whether acknowledgement *physically happens* — i.e. whether the key feature works at all. | ✅ **Decided 2026-07-04** — standard web login on phone; technician screens absolutely mobile-friendly; SA/owner PC-primary + owner mobile health view; special floor-device/PIN revisited later → [[Tech stack]] |
| 7 | **No photos** | Jobsheet is checkbox\|text only. Real intake needs damage photos (liability at collection) + repair evidence. One place digital beats paper → strengthens the adoption wedge. "Photos attach to the job" is enough. | Decide (V1 or V2) |
| 8 | **Customer approval has no record** | Quotation deferred (fine), but the *approval moment* still happens (SA calls, customer says go). No trace → dispute risk ("I never approved this!"). Could be a disciplined blocker-resolve note. | Decide (small, V1-friendly) |
| 9 | **Vehicle history at intake** | Model supports it (`vehicle has_many :jobs`) but no screen shows it. "This car was here 2 months ago for the same complaint" = high value, zero model cost. | **Pull into V1** (a screen section) |

## C. Consciously fine to punt (no action needed)
- Pricing/invoicing outside the system (AutoCount) — explicit in [[ADR-002 V1 scope]].
- Owner two-way chat — token page + copy-paste message is a valid V1 posture.
- Appointment *scheduling* (vs #3's mere visibility) — genuinely V2+.
- Parts, skills, marketing, feedback loops — correctly fenced by [[ADR-002 V1 scope]].

## Verdict (as reviewed)
The core loop (intake → jobsheet → assign → track → blockers → deliver, with acknowledgement as
the differentiator) is a **sellable wedge** — a real product. But as specced it would likely fail
its first workshop on **#1 (no ETA)** and **#5 (partial-adoption)** — those decide *survival*
(**#6 floor access** since **decided**, see row); **#2 (aging)** decides whether the *manager* renews.

*(**Updated 2026-07-28:** **#5 is no longer an open survival risk** — settled by
[[ADR-011 Acknowledgement as stored visibility]], which makes "waiting on whom" a stored, always-answerable
query with no inbox to flood. **#1 (no ETA) remains the open survival item**, still parked at Sprint
task S5.6.)*

Suggested cut when we act on this: pull **#1, #2, #5** (and cheap **#9**) into V1; make explicit
*decisions* to defer **#3, #4, #7, #8** so they're parked on purpose, not by omission.

## Related
- [[Main problem list]] · [[User stories]] · [[Open questions]] · [[Deferred design]] · [[ADR-002 V1 scope]] · [[Roadmap]]
