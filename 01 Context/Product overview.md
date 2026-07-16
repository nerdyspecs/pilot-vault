---
type: context
updated: 2026-07-17
---
# Product Overview
Use alongside [[Builder identity]] at the start of every Claude session.

---

## What this product is
A job monitoring app for vehicle workshops. It gives the workshop team a single place to track every job in progress, and gives vehicle owners real-time visibility into what is happening to their car — without them having to call.

## The problem it solves

### For workshops
Most workshops currently manage job flow through Notion, whiteboards, or manual handoffs between staff. There is no single source of truth — the manager, front desk, mechanics, and warehouse are all working from different information. Jobs stall silently because no one knows who is waiting on whom.

### For vehicle owners
Owners are kept completely in the dark. They don't know what stage their car is at, when it will be ready, or whether anything unexpected has come up. Their only option is to call — which wastes time for both sides and still doesn't build trust.

## What the app must do well
- Let the workshop track every job through its stages, visible to all staff roles
- Let staff update job status easily — it must be simple enough that mechanics actually use it
- Give vehicle owners a live view of their job without needing to log in to a complex portal
- Enable basic communication between workshop and owner (status updates, ready for pickup)

## What this product is NOT
- Not a pricing or invoicing tool — cost tracking is out of scope for now
- Not a full warehouse management system — parts ordering is noted but not the core
- Not a CRM — customer history and marketing are not the focus *(app scope — marketing the
  product itself lives in [[Brand overview]])*
- Not a complex enterprise product — must stay simple enough for a small workshop to adopt without training

## Stage
- Early — mid-Sprint 2: tenancy spine live (Sprint 1), job engine door (`JobActions`) built; job UI + remaining slices ahead *(status refreshed 2026-07-17)*
- Stack: Ruby on Rails