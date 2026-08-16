---
type: concept
module: M1
updated: 2026-07-15
---
# Intake flow — the SA's decision tree
Designed 2026-07-15 (worklog Session 16) from real-world cases; **build target: Phase 4 /
Sprint 5 intake UI**. Schema-complete as of Phase 1 — nothing here needs columns beyond the
spine. The compiled shape: **two lookup keys** (registration number → vehicles, phone →
people), **one script question** ("whose file should this be under?"), and the **two-branch
mismatch confirm** ([[Deferred design]]). Everything else is find-or-create.

Cast for the cases: **Lim** (file-holder), **the wife** (household contact), **Eddie**
(bought Lim's car), **Speedy** (company fleet).

## Entry
SA keys the **registration number** (canonicalized: whitespace-collapsed, upcased) → lookup
in *this workshop's* book only (tenant walls; other workshops' books don't exist here).

## 1 — Vehicle EXISTS → customer card auto-fills (Lim). Verify the counter.
Script: ask **"can I have your phone number to check?"** — inbound, compare silently.
Never read the file's number aloud (leaks Lim's phone to whoever holds his keys — R10's
cousin).

- **1a. Phone matches** → same person / household contact. Register; stamp = Lim.
  *Happy path, ~90% of visits, zero friction, no warning ever shown.*
- **1b. Phone differs** → someone else at the counter. Next question: *"who should this
  visit be billed to?"* Four forks:
  - **1b-i "Bill my husband as usual"** → file untouched, stamp = Lim. The bringer is
    hands, not a payer. *(Accepted v1 gap: no `brought_in_by` record — a future
    contact-person feature, changes nothing about stamps.)*
  - **1b-ii "I'm paying this one"** (borrower / third party) → **[Just this visit]**:
    phone-lookup *them* first (may already be a customer) → find-or-create their card →
    stamp = them; vehicle stays under Lim. *The payer≠file case the deferred validation
    makes legal.*
  - **1b-iii "I bought this car"** (Eddie) → **[Vehicle changed hands]**: phone-lookup
    Eddie → find-or-create his card → `vehicle.update!(customer: eddie)` + stamp = Eddie,
    one transaction. Lim's old jobs stay Lim's forever (frozen stamps = the de facto
    ownership timeline).
  - **1b-iv "That's my old number"** → **edit the card's phone** — never a new card. The
    sneakiest dedup trap: miss it and Lim becomes two people.
- **1c. Vehicle has an ACTIVE job** (R5 index) → there is no "register" — the lookup
  surfaces *"in-house: job #101, in_progress"*; registration lookup doubles as job lookup.
  A `done`-not-collected job doesn't block: the follow-up job is law #8's correction flow.

## 2 — Vehicle NOT in the book. Careful: **new truck ≠ new customer.**
- **2a. Phone lookup hits an existing card** → attach the new vehicle to it ("your second
  car goes under your file"). *Reverse-wife trap: skip this lookup and one person becomes
  two duplicate cards — self-inflicted split history.*
- **2b. Phone unknown → genuinely new.** The script question earns its keep: *"whose file
  should this be under — yours, or someone else's?"*
  - **2b-i "Mine"** → create card (name + phone, `kind: person`) → vehicle → stamp.
  - **2b-ii "My husband's / boss's"** → take *that* person's phone → **look up again**
    (he may exist from his other car → 2a) → else create *his* card.
  - **2b-iii "Company van"** (Speedy's driver) → `kind: company`, company name, **office
    phone — never the driver's**. Company assets file under the company, always.

## Cross-cutting edges (known, accepted v1)
- **Typo'd registration number** on first visit → a later correct entry makes a second
  vehicle row; manual fix. Canonicalization kills spacing/case variants; the per-workshop
  unique index blocks same-spelling dupes.
- **Shared household phone** → the wife "matches" Lim's card. Fine: phone identifies the
  *household contact*, and filing by responsible-contact is correct routing ([[Data model]]:
  a routing problem, not a registry).
- **Same truck in another workshop's book** → invisible, irrelevant, by design (tenant
  walls; JPJ is the ownership registry, Knot never is).
- Households that *correctly* end as two cards (wife's own car under her own file) are not
  errors — v2's **household/shared claims** ([[Data model]] v2) reunites the views if ever
  needed.

## Principles this flow encodes
1. **Phone is the person-key the way registration number is the vehicle-key** — both
   canonicalized at storage, both looked up *before* any create. **And phone is mandatory**
   (builder ruling 2026-07-15): no card without a phone — intake always collects one, which
   is also what makes the 2a/2b lookups reliable.
2. **File under the responsible contact, once, consistently** — not necessarily the legal
   owner.
3. **Mismatches are decisions, not errors** — every payer≠file case reaches a human
   question with two honest answers ([[Deferred design]] two-branch confirm).

## Related
- [[Deferred design]] (two-branch confirm — the mismatch half of this flow) ·
  [[Data model]] · [[Job visibility]] · [[Risk ledger]] (R5, R10)
