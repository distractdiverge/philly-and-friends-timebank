# Greater Philly Time Bank — Claude Code Context

## Project Overview

A free, mobile-first time-banking marketplace for Greater Philadelphia. Users exchange services using **hours as currency** (time credits). The product feel is:

- **Craigslist for time credits** — offers/requests marketplace
- **Venmo-like ledger** — transparent, auditable credit transfers
- **Safe messaging** — with moderation and strict anti-hate enforcement

Key differentiator: a trust/rating system that resists abuse (brigading, retaliation, sockpuppets).

## Repository Structure

This repo is **pre-development** (no implemented code yet). Source of truth:

- `PRD.md` — full product requirements (v0.2)
- `state-machines.md` — Agreement, Ledger, and Reviews state machine specs
- `CLAUDE.md` — this file

## Tech Stack

| Layer | Decision |
|---|---|
| Frontend | Next.js PWA (React-based, mobile-first) |
| Database | PostgreSQL |
| Realtime | WebSockets or managed realtime service (TBD) |
| Hosting | TBD — low-cost, with monitoring/runbooks (PRD §14) |

PWA requirements: installable ("Add to Home Screen"), offline draft saving, cached browse results, fast on cellular/older devices.

## Geographic Coverage

- Philadelphia County
- Delaware County (Delco)
- Montgomery County (Montco)
- Bucks County

## Core Data Models (PRD §8.1)

- **User** — id, contact, status, created_at
- **Profile** — user_id, display_name, pronouns, bio, county, neighborhoods/ZIPs, remote_pref
- **Listing** — id, type(offer/request), user_id, title, category, description, hours_rate_or_max, location_scope, availability, status, created_at
- **Conversation** — id, participant_ids[], created_at, status
- **Message** — id, conversation_id, sender_id, body, created_at, moderation_flags
- **Agreement** — id, listing_id, requester_id, provider_id, scope_summary, proposed_hours, final_hours, status, created_at, completed_at
- **Transaction (Ledger)** — id, from_user_id, to_user_id, hours, agreement_id, created_at
- **Report** — id, reporter_id, target_type, target_id, reason, details, created_at, status
- **ModerationAction** — id, moderator_id/system, action_type, target_type, target_id, rationale, created_at

## Critical Business Invariants

These must never be violated in code:

1. **Append-only ledger** — no hard deletes on Agreements, Transactions, or Reviews; use soft-hide (moderation) or reversal/adjustment transactions with references
2. **One agreement → at most one settled transaction** (plus optional reversal/correction transactions)
3. **Balance = Σ(incoming hours) − Σ(outgoing hours)** across all transactions including reversals/adjustments
4. **Reviews only for participants** and only after agreement completion (A7) or mod-resolved settlement (A10 with settlement)
5. **Double-blind reviews** — reveal only after both parties submit OR 7-day timeout expires
6. **Location is always coarse** — neighborhood/ZIP only; never store or expose exact addresses publicly
7. **All moderation actions must be audit-logged** with rationale (ModerationAction record required)
8. **No silent edits to terms** — agreement terms are versioned; acceptance records `(user_id, terms_version, timestamp)`; ACCEPTED state requires both parties on same version

## State Machines (see `state-machines.md` for full spec)

### Agreement (12 states + DRAFT)
`DRAFT → PROPOSED → NEGOTIATING → ACCEPTED → [SCHEDULED] → IN_PROGRESS → COMPLETED_PENDING_CONFIRMATION → COMPLETED_CONFIRMED`

Dispute path: any accepted state → `DISPUTED → MOD_REVIEW → RESOLVED`

Terminal states: `COMPLETED_CONFIRMED`, `RESOLVED`, `CANCELED`, `EXPIRED`

Only **A7 COMPLETED_CONFIRMED** and **A10 RESOLVED (settled)** may create ledger transactions.

### Ledger Transaction (6 states)
`NONE → PENDING_CREATION → POSTED → FLAGGED → [REVERSED | ADJUSTED]`

Ledger entries are immutable; corrections are new transactions referencing the original.

### Reviews (7 states)
`NOT_ELIGIBLE → ELIGIBLE → ONE_SUBMITTED → BOTH_SUBMITTED_PENDING_REVEAL → REVEALED`

Alternate paths: `ONE_SUBMITTED → EXPIRED_UNREVEALED` (timeout); `REVEALED → MOD_HIDDEN → MOD_ACTIONED`

### Cross-machine coupling
- If dispute raised before transaction posted: hold at T0/T1 until resolved
- If dispute raised after transaction posted: flag (T2 → T3), then reverse/adjust if needed
- Safety-flagged reviews auto-route to moderation queue
- Free-text with hate/doxxing: auto-hide pending review (→ R6) + open report

## Service Categories (9 initial)

1. Tech help (websites, troubleshooting)
2. Admin & paperwork (tax help, forms)
3. Tutoring & learning
4. Household help (light tasks)
5. Transportation (rides — note safety policy)
6. Pet care
7. Food / cooking
8. Creative (design, music, art)
9. Community support / companionship (non-sexual)

## Rating System Design

Structured signals instead of weaponizable star ratings:

- **Reliability** — met time commitments, communicated changes, completed as agreed
- **Quality** — met scope expectations
- **Respect & safety** — felt respected, felt safe

Public profile displays: completion count, no-show rate, reliability band (Excellent/Good/Mixed), safety indicator. Avoid displaying raw negative text prominently.

Anti-abuse: rate limits, new-account weight reduction, cluster anomaly detection, auto-flag for slurs/doxxing/PII.

## Open Decisions (PRD §16)

Track these as issues before implementation:

1. **Negative balance policy** — no negatives (Option A), credit line (Option B), or free negatives (Option C, not recommended)
2. **Identity verification** — how strict; out of scope for MVP but hook exists
3. **Tech stack finalization** — Next.js confirmed; realtime and hosting TBD
4. **Governance/moderator selection** — who are initial moderators, rotation model
5. **Category restrictions** — rides, childcare, home-entry tasks need explicit safety policy
6. **Accessibility + language priorities** — Greater Philly community needs

## MVP Deliverables Checklist (PRD §15)

- [ ] Onboarding + profiles + region selection
- [ ] Offers/requests posting + search/filter
- [ ] Messaging + block/report
- [ ] Agreements flow + completion confirmation
- [ ] Ledger + transaction history
- [ ] Rating system (double-blind, structured signals, weighted)
- [ ] Moderation tools + audit logs
- [ ] PWA install + mobile-first UI
- [ ] Basic monitoring + status page

## Key Product Principles

- **Mobile-first, low-friction** — one-thumb navigation; fast posting
- **Privacy by default** — coarse location; private contact details never shown publicly
- **Safety first** — zero tolerance for hate/harassment; fast moderation
- **Auditability** — append-only ledger; all mod actions logged
- **Free service** — no required membership fee; optional donations must not restrict core trading
