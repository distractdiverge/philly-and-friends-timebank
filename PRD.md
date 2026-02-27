# Greater Philly Time Bank Exchange — Product Requirements (PRD v0.2)

**Last updated:** 2026-02-27  
**Region:** Greater Philadelphia (Philadelphia County, Delaware County, Montgomery County, Bucks County)  
**Product type:** Mobile-first PWA (hybrid “app-like” web) + thin desktop web client  
**Model:** Time banking / bartering marketplace using **hours as currency** (time credits)

---

## 0) Summary

Build a single, simple, reliable, **free** time-banking marketplace for Greater Philly that feels like:

- **Craigslist for time credits** (offers/requests marketplace)
- **Venmo-like ledger** (transparent, auditable credit transfers)
- **Safe messaging** (with moderation and strict anti-hate enforcement)

A key differentiator: a **trust / rating system that resists abuse** and supports community self-vetting.

---

## 1) Goals & Non-Goals

### 1.1 Goals
- Enable neighbors to exchange services using **hours** (e.g., “5 hours for a basic website”, “2 hours for help filing taxes”).
- Make it frictionless on mobile and accessible on low-end devices.
- Create a trustworthy environment with **strong safety norms** and **anti-hate enforcement**.
- Provide a durable platform that is not dependent on a brittle third-party portal.

### 1.2 Non-Goals (MVP)
- Full identity verification / background checks (optional later).
- Built-in video calling (optional later).
- Complex escrow / disputes arbitration (basic dispute flows only in MVP).
- Supporting paid cash transactions (cash is out of scope; out-of-pocket costs may be disclosed).

---

## 2) Target Users

1. Individuals needing help (rides, errands, tutoring, tech help, meal prep, light home tasks).
2. Individuals offering skills (web design, bookkeeping, decluttering, pet sitting, language help).
3. Small community pods / mutual aid groups needing a shared system (phase 2).
4. Nonprofits (phase 2, if aligned).

---

## 3) Key Product Principles

- **Mobile-first, low-friction**: one-thumb navigation; fast posting.
- **Privacy by default**: coarse location, no exact address required.
- **Safety first**: clear rules; easy reporting; fast moderation.
- **Anti-hate / inclusive**: zero tolerance for hate or harassment toward protected groups.
- **Community trust**: ratings that are meaningful and hard to game.
- **Auditability**: append-only transaction ledger; moderation actions logged.
- **Portability**: data export for users and for community governance.

---

## 4) Hard Requirements (Must-Haves)

### R1 — Free Service
- No required membership fee to join or transact.
- Optional donations allowed (must not restrict core trading).

### R2 — Region Coverage
- Supports listings and matching across:
  - Philadelphia County
  - Delaware County (Delco)
  - Montgomery County (Montco)
  - Bucks County

### R3 — Hours as Currency
- 1 hour of service = 1 time credit (or equivalent unit).
- Listings expressed explicitly in hours (can allow decimals, e.g., 1.5).

### R4 — Marketplace (Craigslist for Time)
- Users can post:
  - **Offer**: what I can do, hours rate, availability, constraints
  - **Request**: what I need, max hours offered, timing
- Browse/search/filter by:
  - county / neighborhood / ZIP
  - category
  - remote vs in-person
  - availability window (“this week”, “weekends”, etc.)
  - hour range

### R5 — Venmo-like Ledger
- Each user has a balance and transaction history.
- Every completed exchange creates a ledger entry:
  - from_user, to_user, hours, timestamp, reference (agreement id)
- Ledger is append-only (no silent edits).

### R6 — Safe Messaging
- In-app messaging between matched users.
- Built-in safety controls:
  - block user
  - report user / message / listing
  - safety prompts for first-time meetups
- Rate-limits and spam controls.

### R7 — Zero Tolerance for Hate / Harassment
- Clear code of conduct; enforced consistently.
- Protected categories include (non-exhaustive): race, ethnicity, nationality, religion, disability, gender identity, sexual orientation, etc.
- Fast enforcement tools:
  - temporary lock
  - listing removal
  - account suspension/ban
- Moderation audit log required.

### R8 — Reliability Baseline
- Avoid dependencies on brittle third-party hosted portals.
- Minimal reliability requirements:
  - uptime monitoring + alerting
  - graceful degradation: browsing & ledger still work if messaging is degraded
  - status page (even minimal)

### R9 — Rating / Review System Resistant to Abuse
- Ratings must:
  - reduce malicious review attacks
  - avoid retaliation spirals
  - be meaningful for community trust

(See Section 9 for full rating design.)

---

## 5) User Flows (MVP)

### 5.1 Onboarding
1. Create account (email or phone).
2. Choose primary county + optional neighborhoods/ZIPs.
3. Create profile:
   - display name
   - pronouns (optional)
   - skills / interests (optional)
   - availability
   - remote/in-person preference
4. Agree to code of conduct and safety rules.

### 5.2 Post Offer
- Title + category
- Description with scope boundaries
- Hours requested (rate)
- Remote/in-person + approximate location
- Availability windows
- “What’s included / excluded”
- Optional out-of-pocket costs disclosure (materials/gas)

### 5.3 Post Request
- Title + category
- Description + constraints
- Max hours offered
- Timing need (ASAP / flexible / date range)
- Remote/in-person + approximate location

### 5.4 Match & Negotiate
- User messages from listing.
- Propose an agreement:
  - scope summary
  - estimated hours
  - meeting plan (if in-person)
- Accept agreement.

### 5.5 Complete & Confirm
- Mark complete.
- Both sides confirm:
  - final hours
  - optional private feedback flag (safety issues)
- Ledger entry created.

### 5.6 Dispute / Safety Escalation (Basic MVP)
- If one side disputes:
  - freeze the specific transaction pending review
  - collect structured report
  - moderator decision with audit log

---

## 6) Categories (Initial)
- Tech help (websites, troubleshooting)
- Admin & paperwork (tax help, forms)
- Tutoring & learning
- Household help (light tasks)
- Transportation (rides—note safety policy)
- Pet care
- Food / cooking
- Creative (design, music, art)
- Community support / companionship (non-sexual)

---

## 7) Policies & Rules (Marketplace)

### 7.1 Scope & Hours Clarity
- Listings must include:
  - scope
  - hour amount/rate
  - inclusions / exclusions
- Encourage bounded offers:
  - “One session (up to 90 minutes)”
  - “One-page landing page (no ecommerce)”

### 7.2 Out-of-pocket Costs
- Allowed only as disclosed reimbursement (materials, transit).
- Must be explicitly stated in listing and agreement.
- No hidden fees.

### 7.3 Prohibited / Restricted (MVP baseline)
- Illegal activities
- Controlled substances facilitation
- Hate/harassment content
- Medical diagnosis or treatment services (basic peer support okay)
- Anything high-risk without clear safety framing (admin may restrict categories)

---

## 8) Data Model (MVP minimum)

### 8.1 Entities
- **User**
  - id, contact, status, created_at
- **Profile**
  - user_id, display_name, pronouns(optional), bio, county, neighborhoods/ZIPs, remote_pref
- **Listing**
  - id, type(offer/request), user_id, title, category, description, hours_rate_or_max, location_scope, availability, status, created_at
- **Conversation**
  - id, participant_ids[], created_at, status
- **Message**
  - id, conversation_id, sender_id, body, created_at, moderation_flags
- **Agreement**
  - id, listing_id, requester_id, provider_id, scope_summary, proposed_hours, final_hours, status, created_at, completed_at
- **Transaction (Ledger)**
  - id, from_user_id, to_user_id, hours, agreement_id, created_at
- **Report**
  - id, reporter_id, target_type, target_id, reason, details, created_at, status
- **ModerationAction**
  - id, moderator_id/system, action_type, target_type, target_id, rationale, created_at

### 8.2 Ledger Invariants
- No deletion of transactions; only reversal transactions with references.
- Balance is derived from sum of transactions (or cached with reconciliation).

---

## 9) Trust, Ratings, and Anti-Abuse Review System

### 9.1 Goals
- Help users assess reliability and fit.
- Resist:
  - brigading (mass negative reviews)
  - retaliation reviews (“you rated me low so I rate you low”)
  - sockpuppet / sybil attacks
  - targeted harassment via reviews

### 9.2 MVP Design (Recommended)
**A) Only participants can review**
- Reviews only allowed after a completed agreement confirmed by both parties
  - (or by moderator decision in a dispute)

**B) Double-blind reviews**
- Both users submit feedback privately.
- Ratings are revealed only after:
  - both have submitted, or
  - a timeout window expires (e.g., 7 days)
This reduces retaliation pressure.

**C) “Signal over stars”**
Instead of a single 1–5 star score that’s easy to weaponize, use structured signals:
- **Reliability** (showed up / communicated / completed)
- **Quality** (met scope expectations)
- **Respect & safety** (felt safe / respectful)
- Optional free-text, but:
  - filtered for slurs / hate
  - shown only if it passes moderation checks or is summarized

**D) Weighting & anti-sybil**
- New accounts’ ratings have limited impact.
- Weight reviews by reviewer reputation signals:
  - account age
  - number of completed exchanges
  - history of valid reports vs spam reports

**E) Public display focuses on aggregate**
Show:
- completion count
- cancellation/no-show rate
- “reliability” band (e.g., Excellent / Good / Mixed)
- “community safety” indicator (e.g., “No recent safety reports”)
Avoid showing raw negative comments prominently.

**F) Abuse detection**
- Rate-limit reviews.
- Flag outliers (e.g., sudden cluster of negative reviews).
- Auto-hide reviews containing personal info (doxxing risk).
- Easy “appeal review” flow routed to moderators.

### 9.3 Phase 2 Enhancements
- Reputation graph: trust networks / vouching.
- Optional verified org endorsements.
- Location-based community moderators.

---

## 10) Moderation & Enforcement

### 10.1 Code of Conduct
- Clear “zero tolerance” policy for hate and harassment.
- Clear boundaries re: sexual services (not allowed).
- Clear consent expectations and meet-up safety guidance.

### 10.2 Moderator Tooling (MVP)
- Review queue for reports
- Actions:
  - remove listing
  - warn user
  - message deletion (if required)
  - temporary lock
  - ban
- Every action creates an audit entry (ModerationAction)

### 10.3 Transparency
- Provide users:
  - reason for enforcement (at a high level)
  - appeal mechanism

---

## 11) UX Requirements (Mobile-first)

- PWA installable (“Add to Home Screen”)
- Fast load on cellular and older devices
- One-thumb navigation:
  - Home
  - Search
  - Post
  - Messages
  - Ledger
  - Profile
- Accessibility:
  - readable type sizes
  - keyboard navigation support
  - screen reader labels
- Offline resilience:
  - draft saved locally for listings
  - cached browse results (best effort)

---

## 12) Security & Privacy

- Default to coarse location (neighborhood/ZIP), not exact address.
- Private contact details never shown publicly.
- Minimize stored sensitive data.
- Abuse prevention:
  - spam throttling
  - login protections
  - content filtering
- Data export:
  - user can export their own profile, listings, messages (where appropriate), ledger

---

## 13) Metrics (Success & Safety)

### 13.1 Success
- Activation: % who post an offer or request within 24 hours
- Match rate: % listings leading to agreement
- Completion rate: completed agreements / accepted agreements
- Monthly time velocity: credits exchanged per active user

### 13.2 Safety
- reports per 1,000 exchanges
- repeat offender rate
- median moderation response time
- no-show rate trends

---

## 14) Technical Notes (Implementation Direction, Non-binding)

- Frontend: PWA (React/Next.js, SvelteKit, etc.)
- Realtime messaging: WebSockets or managed realtime
- Ledger: append-only transaction table with reconciliation
- Hosting: low-cost, documented runbooks; monitoring and alerts

---

## 15) MVP Deliverables Checklist

- [ ] Onboarding + profiles + region selection
- [ ] Offers/requests posting + search/filter
- [ ] Messaging + block/report
- [ ] Agreements flow + completion confirmation
- [ ] Ledger + transaction history
- [ ] Rating system (double-blind, structured signals, weighted)
- [ ] Moderation tools + audit logs
- [ ] PWA install + mobile-first UI
- [ ] Basic monitoring + status page

---

## 16) Open Questions (Track in Issues)

1. Should we allow negative balances (credit lines) or require non-negative?
2. Should “out-of-pocket costs” be tracked as a separate field in agreements?
3. How strict should category restrictions be for rides / childcare / home entry tasks?
4. Governance model: who are initial moderators? How are they rotated/selected?
5. Accessibility + language support priorities for Greater Philly communities.