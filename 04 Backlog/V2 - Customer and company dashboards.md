---
type: reference
created: 2026-07-18
updated: 2026-08-19 (Session 35 — recovered from an orphaned worktree branch; drift banner added, body left as the faithful Session 23 record; prior: 2026-07-18 added claim-flow token-bridge + QR self-enrollment subsections)
---
# V2 — Customer & company dashboards
> **Designed in discussion 2026-07-18 (Session 23).** Parked for a future v2 development pass —
> **circle back here when v2 starts.** Nothing below is built or scheduled. This note is the
> *feature vision + relational design*; the schema seeds it depends on already live in
> [[Data model]] §v2 (Company, `customers.company_id`/`user_id`, household claims, global vehicle
> identity). Read the two together. **v1 changes nothing to enable any of this** — that was the
> whole point of the session; v2 fills claim slots v1 leaves nil.

> [!warning] Recovered 2026-08-19 — the body predates ADR-010/012/013 and the sprint renumber
> This note was committed on a worktree branch that never merged, so it sat outside the vault for a
> month while v1 moved on. **The design still holds** — its §"What v1 should do NOW" asked for
> *"Schema: nothing"*, and every foundation it names as banked is in fact built (canonical
> plate/phone, tokenized exposure, flat `Customer`, the frozen stamp). Read the body as written
> **2026-07-18**, with five corrections:
>
> 1. **`WorkshopEmployment`/`WorkshopOwnership` no longer exist.** [[ADR-010 WorkshopStaff supersedes the edge split]]
>    collapsed both into one `WorkshopStaff` row (`owner` boolean) + append-only `WorkshopStaffRole`.
>    So the §"v1 down-payment" claim that v1's edge *naming* mirrors a future
>    `CompanyEmployment`/`CompanyOwnership` is **dead** — and it leaves a real open question for the
>    v2 pass: does Company mirror the **collapsed** shape (`CompanyStaff` + roles) or keep the
>    two-edge split? [[Data model]] §v2 still names the two-edge version. **Decide at v2 kickoff.**
> 2. **The public token moved off Job.** [[ADR-012 Intake-Job two-level aggregate]] put
>    `has_secure_token` on **Intake** (one link per visit, not per repair). Everywhere the
>    token-bridge section says "job-status token", read **intake status token**.
> 3. **`job.customer` is now `intake.customer`.** The frozen stamp — the thing that makes dashboard
>    history trustworthy — moved to Intake with the same ADR. The reasoning is unchanged; only the
>    table moved.
> 4. **Sprint numbers are pre-renumber.** Session 32 swapped Sprint 5 ↔ 6. "Sprint 6 phone-first
>    dedup tree" and "Sprint 6 [[Intake flow]]" are now **S5.5**; "Sprint 7 token page" is now
>    **S5.7**-era public status work. Treat all sprint references here as approximate.
> 5. **The one down-payment it asked for is still undone:** canonicalize `vehicle.vin` the same way
>    as `registration_number` (upcase + strip). `Vehicle.canonicalize` exists for the plate only.
>    Still a valid one-liner whenever `Vehicle` is next touched.
>
> Untouched and still load-bearing: the **claim-flow critique** (bearer link proves possession, not
> identity), the **QR self-enrollment risks**, and the **strategic fence** in §"Fenced OUT" (daily
> driver↔truck tracking is a second product on a second tenant — do not build it into the workshop
> schema). Those are the reasons this note was worth recovering.

## What v2 is (and isn't)
- **Is:** login-facing **read** surfaces — a **personal dashboard** (a private customer sees their
  own vehicles + service history across every workshop that touched them) and a **company
  dashboard** (a fleet's **owner / fleet_manager** sees the whole fleet's status + history in one
  pane instead of ringing five workshops). Value-for-companies is the headline.
- **Isn't:** any new *operation* on the workshop side. v2 reads data v1 has been accruing since
  Sprint 2 (frozen job stamps + vehicles). Nothing captured at intake, nothing written by the
  dashboard. Keep v2 **read-only** — see the RLS seam below for why that boundary is load-bearing.

## The relational foundation it reads (recap — the v1 triangle)
```
Customer (tenant-scoped, kind: person|company; flat)
  ├─ user_id     → nil in v1   (personal claim slot)
  └─ company_id  → nil in v1   (company claim slot)
Vehicle  belongs_to :customer   ← MUTABLE present owner (moves on sale)
Job      belongs_to :customer   ← FROZEN at registration: who paid THEN (never moves)
```
The frozen `job.customer` is what makes dashboard history **trustworthy** — a sold car can't
retroactively rewrite who a past job was for ([[Data model]] §Resolved). `vehicle.customer` answers
"who owns it now." Two pointers, deliberately divergent; both dashboards lean on the split.

**The pivot:** a tenant-scoped Customer row is an **anchor a global identity can later claim.**
The claim FK points **up** (customer → global User/Company), never down. v2 adds identities
*above* v1 and queries downward; v1 never learns they exist.

## The additive v2 deltas (no v1 rewrite, no backfill)
- `customers.user_id` (nullable) — personal claim; `customers.company_id` (nullable) — company
  claim; **check constraint: never both set.**
- **`Company`** — a **global** entity (not tenant-scoped, like `User`), with a governance mirror of
  the workshop pattern: **`CompanyOwnership`** (governance) + **`CompanyEmployment`**
  (owner / fleet_manager / driver). *Same pattern as Workshop, never the same table —* Workshop is
  the tenant/RLS boundary, Company is a customer whose fleet crosses boundaries ([[Data model]] §v2,
  Session 14).
- The **"shadow" fact:** real-world Acme serviced at 3 workshops = **3 `kind: company` Customer
  rows**, each `company_id →` the one global Company. The dashboard is what unifies them.

## The two dashboards are the SAME query, inverted
Both start from a global identity, gather its claimed customer rows **across workshops**, then read
vehicles (current) + jobs (frozen):
```ruby
# Personal — private customer logs in
claimed     = Customer.where(user_id: me.id)       # cross-tenant read
my_vehicles = Vehicle.where(customer: claimed)     # owned now
my_history  = Job.where(customer: claimed)         # PAID FOR (frozen): includes a car I sold,
                                                   # excludes a bought car's pre-me jobs — correct
                                                   # person-inward visibility ([[Job visibility]])

# Company — fleet_manager / owner logs in
company_customers = Customer.where(company_id: acme.id)   # cross-tenant
fleet   = Vehicle.where(customer: company_customers)      # current fleet, all shops
history = Job.where(customer: company_customers)          # frozen fleet history
```
**They differ only in the claim column (`user_id` vs `company_id`).** One dashboard read path serves
both, parameterized by claim type. **This is the concrete payoff of the flat Customer + `kind`
toggle** — polymorphic person/company tables would have forced two dashboards over two schemas. The
trap we refused (Session 23) bought this.

## Contact resolution — "who do we notify about this job"
`job.contact_person` is a **def, never a column** — a **live, most-specific-wins chain**, anchored on
the frozen `job.customer`:
```ruby
def contact_person
  vehicle.active_contact_override(on: Date.today)  # 1. temporary override (see caveat)
    || vehicle.standing_contact                    # 2. per-vehicle standing contact
    || customer.default_contact                    # 3. account default (private: self ·
                                                   #    company: owner-delegated fleet_manager)
    || customer                                    # 4. floor: the owner
end
```
- **Config is company-side governance:** the owner delegates a `CompanyEmployment` (fleet_manager)
  as the contact (`companies.contact_employment_id`); the fleet_manager may set per-vehicle
  contacts. Resolved **live** — call whoever is the contact *now*, not whoever we spoke to in March.
- **Frozen vs live, the distinction v1 taught us:** the **customer stamp must freeze** (billing /
  ownership history — the sold-car argument). **Contact must NOT** — it's operational routing, so
  live-off-current-config is *more* correct. Don't copy the freeze law onto contact (I did, once, and
  it was wrong).
- **v1 is layer 4 for everyone** — no Company org, no delegation, so `default_contact` = self, and a
  company card shows its flat `contact_person` string + office phone. v2 adds the outer layers that
  fall through to that same floor. Ship v1 flat; it's the base link.
- **⚠ Two design cautions for build time:**
  1. **Drop the time-bounded override (layer 1) unless proven needed.** "My wife for 3 days" is
     effective-dating (valid_from/until, "active now" queries, expiry) — heavy machinery for an edge.
     "Set the contact, change it back" covers ~95% with the human doing the revert. Add periods only
     if scheduled handoffs turn out common.
  2. **Per-vehicle contact must be scoped to (owner × vehicle), not the bare vehicle** — else a sold
     car leaks its contact config to the new owner. A row keyed to owner+vehicle expresses this; a
     column on Vehicle can't. (General rule that keeps Vehicle lean: **relationships/config/temporal
     = their own tables, never columns on the thing** — already house style for trackers.)

## The two genuinely hard seams (additive, but NOT "just a column")
1. **Cross-tenant read lens.** The dashboards read across workshop RLS walls. Add a permissive
   `FOR SELECT` policy on customers/vehicles/jobs keyed to an `app.user_id` / `app.company_id` GUC,
   **beside** `tenant_isolation`, never replacing it (Postgres ORs permissive policies — precedent:
   S1.12-pre's `own_rows`). Additive **in principle**, but: a join inside the policy, real perf to
   watch, and a bug = **cross-tenant data leak**. **Keep v2 read-only** — `FOR SELECT` across tenants
   is tractable; the day a company *writes* across walls it gets much harder. This is the hard 20%;
   budget real design + proof time.
2. **Cross-workshop identity stitching.** v1 deliberately doesn't link records across workshops (the
   isolation working) — so "which customer rows *are* Acme / *are* me?" has no automatic answer. v2
   builds the reconciliation: a claim flow, or matching on **canonical phone / VIN**. Plus a **second
   access door** mirroring Sprint 1's (CompanyOwnership OR active CompanyEmployment → company context,
   re-verified per request; owner/fleet_manager read the fleet, `driver` scope is a v2-design call).
   *The wall that makes v1 safe is the wall v2 drills doors through — the correct, unavoidable cost of
   real tenancy, paid at the v2 seam, not in a v1 rewrite.*

### The claim flow — token bridge (builder's suggestion, Session 23)
**Preserved deliberately, flaws and all — this is the road we walked; don't re-derive or repeat the
mistakes at v2 build time.**

**What the builder suggested (the essence):** reuse the **Sprint 7 job-status token as the v2 claim
voucher.** In v1 a workshop fires a token link to the customer, valid while the job is "alive" (not
delivered), to view status. Since the link is issued against a specific job → vehicle → **customer**,
it *carries the customer identity*. So in v2, fire the same kind of link; when the receiver clicks:
(1) check if they have an account/app — if not, prompt to create one (that's where the record gets
linked); (2) branch on the customer's kind — **person** → link directly under them; **company** →
check whether the user has a company created / is part of one / their role; if they have **neither**,
just show the normal job-status update and ask them to pass the link to the company owner / contact
the workshop. "Where the two points touch" = the token is what lets us confidently say *receiver =
this customer = this new user*. Builder's own flagged risk: **the role of the receiver.**

**What's right about it (keep):** the token-as-voucher instinct is claim-mechanism #1 (workshop
vouches — the known side carries `customer_id`). And the safe default for an unrecognized company
receiver — **show read-only status, escalate to the owner / workshop** — is exactly correct.

**Why parts are bad (the two holes the suggestion glosses):**
1. **A bearer link proves *possession*, not *identity*.** A capability URL = whoever holds it has it.
   Fine for read-only status (low sensitivity, shared deliberately). **Not** fine for *claiming an
   identity*: the status link for Lim's car gets forwarded (WhatsApp, to the driver who dropped it
   off) → clicker creates an account → claims **Lim's** record → sees Lim's cross-workshop history.
   Account-takeover via forwarded link. A "password only the user knows" doesn't close it — the
   password is set by *whoever clicks first*, so it secures the account but doesn't prove the claimer
   *is* Lim at claim time. **Fix: separate the sensitivities.** Status page stays bearer; the **claim
   step escalates to phone-possession proof — an OTP to `customer.phone` on file.** The forwarded-to
   driver holds the link but can't receive Lim's OTP → link grants *view*, never *claim*. (Mandatory
   canonical phone in v1 exists precisely to be this channel.)
2. **A bearer link must never confer *governance*.** Clicking must not, by itself, make someone a
   company's owner or file the company's vehicle under a *personal* account. Company governance is
   established deliberately (like v1's "create workshop" → WorkshopOwnership), never by a click.

**Model correction:** you claim the **customer** (`customers.user_id = user.id`), not the vehicle —
the vehicle(s) + frozen history follow the customer. Claiming the customer is the atom.

**The three rules that dissolve the branch tree (the safe reframe):**
- **Bearer link = view only.**
- **Claim = proven identity** (OTP to the on-file phone), never bare link possession.
- **Governance = owner-gated, never link-conferred.** So the company tree becomes: owner (proven) →
  may claim this workshop's company shadow row; member (fleet_manager/driver) → view only unless the
  owner delegated linking; neither → read-only status + escalate. **The link never elevates a role** —
  which is the direct answer to the builder's named risk.

**Scope flag:** the builder floated "pages that **require your actions**." Sprint 7's token page is
**read-only by design** (exit: owner "cannot write anything"). A customer-facing *action* surface is a
separate, bigger design with its own auth stakes — almost certainly v2. Do **not** let it ride in on
the v1 status page.

### QR self-enrollment for company crew (builder's suggestion, Session 23)
**What the builder suggested:** give a company owner role-specific QR codes ("register as driver",
"register as fleet manager") — an employee scans the one for their role and is **automatically
registered under the company with that role assigned.** Goal: make standing up a company's crew
easy (esp. driver-heavy fleets with high turnover — typing N invite emails one at a time doesn't
scale). Builder also floated the same mechanism **at the workshop level**, and confirmed intent
(2026-07-18): **eventually reopen [[ADR-008 Crew joining requires acceptance]]** for workshop crew
too, not just apply QR to Company (which has no existing crew-join model to conflict with). See
the footnote added to ADR-008 pointing here.

**What's right about it:** bulk/self-serve onboarding is a real gap the admin-targeted invite flow
doesn't solve well at volume; a poster a new hire scans is a legitimately better UX than an admin
typing each email. Worth building — just not as a bare bearer grant.

**Why the naive version is bad — same bearer-vs-governance rule as the claim-token flow above:**
1. **A QR code is a bearer secret with zero identity proof.** Photographed/forwarded/leaked (an
   ex-employee's old screenshot, a WhatsApp forward) and anyone who ever saw it can self-enroll.
   Weaker than even the customer claim flow, which at minimum escalates to phone-OTP — a bare QR
   scan proves nothing about who the person is.
2. **Auto-assigning the role ON SCAN is a bearer link conferring power** — the exact failure mode
   flagged in the claim-flow section, just via QR instead of a URL. "Register as fleet manager" is
   a high-blast-radius grant (manages the whole fleet, sets per-vehicle contacts — see the contact
   chain above) handed to whoever holds an image.
3. **No per-person revocation.** The existing admin-invite flow is inherently per-person and
   auditable (end one employment, only that person loses access). A shared static QR has no
   "per-person" — the owner's only lever is nuking the whole code and reprinting, punishing anyone
   still mid-onboarding.

**The fix — same shape as the claim-flow fix: separate convenient distribution from proof + grant.**
- QR encodes a **short-lived, count-limited or single-use enrollment token**, not a permanent
  secret — minted (and revocable/rotatable anytime) by **CompanyOwnership only**. Scanning starts a
  flow; it doesn't complete one.
- **Identity is still captured at scan-time** (phone/account creation) so there's an audit trail of
  who joined, via which code, when — mirrors the claim flow's proof step, lighter bar (joining as
  crew isn't claiming someone else's history).
- **Tier by role sensitivity, don't treat all QR-grantable roles alike.** Low-blast-radius roles
  (driver) are reasonable to self-serve via a vetted, rotating code. High-blast-radius roles
  (fleet_manager) should stay **admin-targeted** (the existing invite-and-accept shape) even at v2 —
  don't let one mechanism cover both; that's where this gets risky.

**Workshop level — explicitly NOT a casual extension, a planned ADR-008 supersession.** ADR-008 is
a **deliberate, reasoned, settled decision** — it replaced an earlier permissive flow specifically
because unvetted, unconsented access was the risk (mis-add safety: a typo'd email can never
silently become someone's crew; philosophical coherence: "things don't happen to you unseen and
unacknowledged"). A workshop QR that lets *anyone who scans* become crew reverses the **admin
targets a specific named person** model into **self-service, no targeting** — a different trust
model, not an addition to the existing one. Vault convention: ADRs are **never edited**, only
superseded with a dated footnote or a new ADR — so this is not a backlog-note-and-move-on item.
**Builder's stated intent (2026-07-18): reopen ADR-008 for workshop crew when this is designed
properly** — write a new ADR (or a dated supersession) at that time; don't retrofit QR into the
existing crew flow informally. When designing it, the fix pattern above (rotating/single-use token,
identity capture at scan, owner-gated minting, role-tiering) is the starting point — plus explicitly
re-deciding whether ADR-008's consent/mis-add-safety reasoning is preserved, weakened, or
deliberately traded away, and saying so in the new ADR's "Why."

## What v1 should do NOW for v2 (almost nothing — by design)
- **Schema: nothing.** The v2-helpful decisions are already banked for their *own* reasons: canonical
  phone/plate/email (= v2's matching keys), tokenized public exposure (= no UUID PKs needed), the
  `WorkshopEmployment`/`WorkshopOwnership` naming mirroring future `CompanyEmployment`/
  `CompanyOwnership`, the "Company" model name reserved, frozen stamp + flat Customer.
- **One cheap hygiene down-payment:** canonicalize `vehicle.vin` the same way as
  `registration_number` (upcase + strip) when Vehicle is next touched — VIN is v2's strongest global
  vehicle key. Low priority, one line, consistent with v1's own rules (not speculative v2-building).
- **The real investment is data quality, not schema: intake dedup.** Identity stitching's difficulty
  scales with how clean the records are. The name/phone search in Sprint 2.5 is the down-payment; the
  full phone-first dedup tree (Sprint 6, [[Intake flow]]) is a **v2 enabler**, not just intake
  ergonomics. Duplicate cards = fragmented dashboard history = a spoiled "pleasant surprise."

## Fenced OUT of Knot entirely (not v2 — a different product)
**Daily driver↔truck responsibility tracking** (drivers scan a plate at shift start/end to "own" the
truck that day). Real and valuable — but it's **fleet-operations software**, serving the *company*
managing its own fleet, on a **second tenant** (Company-as-tenant), which the vault deliberately
refused. It needs a new write surface + new operational users + a second tenancy root — a strategic
pivot, not an additive column. **Do NOT build it into the workshop schema.** The light half that *does*
fit Knot — *which driver dropped this truck at the workshop this visit* — is the long-deferred
`brought_in_by` ([[Intake flow]] §1b-i), captured at intake, optional, probably never needed.
**The test to keep:** does the **workshop** (the tenant) benefit, or the **company**? Workshop → it's
Knot. Company → different product wearing Knot's clothes.

## Triggers to revisit this note
- v2 kickoff (customer/company dashboards) — this note + [[Data model]] §v2 are the design start.
- Before then: hold the intake-dedup line (Sprint 2.5 search box; Sprint 6 phone-first tree).
- If a company ever needs to **write** from a dashboard → re-open the read-only RLS boundary decision.

## Related
- [[Data model]] (§v2 schema seeds, §Resolved frozen stamp) · [[Intake flow]] · [[Job visibility]] ·
  [[Deferred decisions]] · [[ADR-004 Multi-tenant foundation]] · [[ADR-006 Ownership separate from Employment]] ·
  [[ADR-008 Crew joining requires acceptance]] (§QR self-enrollment — flagged for reopening)
