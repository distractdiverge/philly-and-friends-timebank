# State Machines — Agreements, Ledger, Reviews (PRD Supplement v0.2)

**Last updated:** 2026-02-27
**Scope:** State machines only (no API endpoints, no screen-by-screen spec)  
**Applies to:** Greater Philly Time Bank Exchange (mobile-first PWA)

---

## 0) Shared Definitions

### Actors
- **Requester (R):** person requesting help
- **Provider (P):** person offering help
- **Moderator (M):** trusted admin/mod role (human or delegated community role)
- **System (S):** automated rules/cron jobs, abuse detection, timeouts

### Shared Concepts
- **Agreement:** structured commitment between R and P tied to one listing (offer/request) that results in a time-credit transfer.
- **Ledger Transaction:** append-only record of hours transferred, tied to a finalized agreement.
- **Review:** post-completion feedback, designed to be double-blind and anti-abuse.

### Global Invariants (apply everywhere)
1. **No silent deletion:** Agreements, Transactions, and Reviews are never hard-deleted; only soft-hidden (moderation) or superseded (reversal/correction).
2. **Append-only ledger:** Balances derive from transaction sums (or caches that reconcile to the append-only truth).
3. **One agreement → at most one settled transaction** (plus optional reversal/correction transactions).
4. **Reviews only for participants** and only after completion (or mod-resolved completion).

---

## 1) Agreement State Machine

### 1.1 Agreement States (canonical)
- **A0 DRAFT** — created but not yet sent/proposed (may exist only client-side; optional server state)
- **A1 PROPOSED** — proposal sent to counterparty; awaiting response
- **A2 NEGOTIATING** — active edits/counteroffers; not yet mutually accepted
- **A3 ACCEPTED** — both parties accepted current terms; work may begin
- **A4 SCHEDULED** — (optional) concrete time/place arranged; still accepted
- **A5 IN_PROGRESS** — work started (explicitly marked or inferred)
- **A6 COMPLETED_PENDING_CONFIRMATION** — one party marked complete; waiting on other party
- **A7 COMPLETED_CONFIRMED** — both parties confirmed completion + final hours
- **A8 DISPUTED** — dispute raised (hours/scope/safety/no-show/etc.)
- **A9 MOD_REVIEW** — moderator reviewing dispute
- **A10 RESOLVED** — dispute resolved; may result in settlement or cancellation
- **A11 CANCELED** — mutually canceled or canceled by system policy (pre-work)
- **A12 EXPIRED** — auto-expired due to inactivity/timeouts

> Notes:
> - **A4 SCHEDULED** is optional but useful for analytics and clarity.
> - **A10 RESOLVED** is distinct from **A7** because resolution may be partial, reversed, or set by moderator.

### 1.2 Agreement Events / Triggers
- `create_draft(R|P)`
- `send_proposal(R|P)`
- `counter_propose(R|P)`
- `accept(R|P)`
- `decline(R|P)`
- `cancel(R|P)`
- `schedule_set(R|P)`
- `start_work(R|P)`
- `mark_complete(R|P)`
- `confirm_complete(R|P)`
- `raise_dispute(R|P)`
- `mod_take_case(M)`
- `mod_decide(M)`
- `timeout(S)` (proposal, negotiation, completion confirmation, reviews)

### 1.3 Transition Table (core)
| From | Event | To | Guard / Notes |
|---|---|---|---|
| A0 DRAFT | send_proposal | A1 PROPOSED | Must include scope_summary + proposed_hours |
| A1 PROPOSED | counter_propose | A2 NEGOTIATING | Version bump of terms |
| A1 PROPOSED | accept | A3 ACCEPTED | Requires other party already accepted current version OR accept is second acceptance |
| A1 PROPOSED | decline | A11 CANCELED | Decline ends agreement |
| A2 NEGOTIATING | counter_propose | A2 NEGOTIATING | Continues negotiation (new version) |
| A2 NEGOTIATING | accept (both) | A3 ACCEPTED | Both accept same version |
| A3 ACCEPTED | schedule_set | A4 SCHEDULED | Optional |
| A3 ACCEPTED / A4 SCHEDULED | start_work | A5 IN_PROGRESS | Either party can mark, but abuse rules apply |
| A3/A4 | cancel | A11 CANCELED | Pre-work only; mutually agreed cancellation |
| A5 IN_PROGRESS | cancel | A8 DISPUTED | Post-work-start; cancel-after-start requires dispute/resolution path |
| A5 IN_PROGRESS | mark_complete | A6 COMPLETED_PENDING_CONFIRMATION | Captures `final_hours_proposed_by_marker` |
| A6 COMPLETED_PENDING_CONFIRMATION | confirm_complete | A7 COMPLETED_CONFIRMED | Both agree on final hours → settlement |
| A6 COMPLETED_PENDING_CONFIRMATION | raise_dispute | A8 DISPUTED | Any party may dispute hours/scope/safety |
| A3/A4/A5/A6 | raise_dispute | A8 DISPUTED | Dispute at any time after acceptance |
| A8 DISPUTED | mod_take_case | A9 MOD_REVIEW | Moderator assigned |
| A9 MOD_REVIEW | mod_decide (settle) | A10 RESOLVED | May produce settlement transaction |
| A9 MOD_REVIEW | mod_decide (cancel) | A10 RESOLVED | No settlement or partial settlement |
| A10 RESOLVED | (if settled) | (terminal) | Terminal in practice; see ledger |
| A1/A2/A3/A4 | timeout | A12 EXPIRED | Inactivity thresholds |
| A6 | timeout | A8 DISPUTED or A10 RESOLVED | Configurable: auto-dispute or auto-resolve w/ defaults |

### 1.4 Agreement Versioning
Agreements have **terms versions**:
- `terms_version = integer`
- Each `counter_propose` increments version.
- Acceptance records `(user_id, terms_version, timestamp)`.

**Rule:** Agreement cannot enter **ACCEPTED** unless both parties accepted the same `terms_version`.

### 1.5 Terminal States
- **A7 COMPLETED_CONFIRMED**
- **A10 RESOLVED**
- **A11 CANCELED**
- **A12 EXPIRED**

### 1.6 `mod_decide` Outcome Fields
The `mod_decide` event must carry a structured outcome payload, because downstream ledger and review eligibility depend on it:

| Field | Type | Values | Effect |
|---|---|---|---|
| `outcome` | enum | `settled` | Full or partial hours awarded; creates ledger transaction |
| `outcome` | enum | `no_service` | No hours transferred; reviews remain NOT_ELIGIBLE (unless mod enables) |
| `outcome` | enum | `canceled` | Agreement voided; no settlement; reviews NOT_ELIGIBLE |
| `hours` | float \| null | e.g. `2.5` | Final hours for settlement; required when `outcome = settled` |
| `mod_enable_reviews` | boolean | `true \| false` | Explicitly unlocks review eligibility for `no_service` cases (case-by-case) |
| `rationale` | string | free text | Written to ModerationAction audit log (required) |

---

## 2) Ledger State Machine (Transaction Lifecycle)

Ledger is append-only; state machine applies to **the lifecycle of a transaction record** associated with an agreement.

### 2.1 Transaction States
- **T0 NONE** — no transaction exists yet for the agreement
- **T1 PENDING_CREATION** — agreement completed confirmed (or mod settled); transaction about to be created
- **T2 POSTED** — transaction written to ledger (final)
- **T3 FLAGGED** — transaction flagged for review (fraud, dispute reopened, abuse)
- **T4 REVERSED** — transaction has been reversed by one or more reversal transactions
- **T5 ADJUSTED** — corrected by adjustment transaction(s) (not deletion)

> Ledger entries themselves should remain immutable; “states” can be tracked via references.

### 2.2 Events / Triggers
- `agreement_completed_confirmed(S)`
- `mod_settlement(S|M)`
- `post_transaction(S)`
- `flag_transaction(S|R|P|M)`
- `post_reversal(M)`
- `post_adjustment(M)`
- `close_flag(M)`

### 2.3 Transition Rules
| From | Event | To | Notes |
|---|---|---|---|
| T0 NONE | agreement_completed_confirmed | T1 PENDING_CREATION | Agreement A7 reached |
| T0 NONE | mod_settlement | T1 PENDING_CREATION | Agreement A10 settled |
| T1 PENDING_CREATION | post_transaction | T2 POSTED | Creates ledger entry with hours; operation must be idempotent (use job queue with at-least-once delivery to handle crashes in T1) |
| T2 POSTED | flag_transaction | T3 FLAGGED | Does not change balances yet |
| T3 FLAGGED | close_flag (no change) | T2 POSTED | Flag archived (not deleted); tx stands unchanged; transaction may be re-flagged from T2 |
| T3 FLAGGED | post_adjustment | T5 ADJUSTED | Adjustment tx references original |
| T3 FLAGGED | post_reversal | T4 REVERSED | Reversal tx references original |

### 2.4 Ledger Invariants
1. **Exactly one primary settlement transaction** per agreement (T2), unless mod chooses split settlement (still recorded as a set referencing agreement).
2. **Reversals/adjustments are new transactions** that reference the original transaction id.
3. Balance calculation:
   - `balance(user) = Σ(incoming hours) - Σ(outgoing hours)` across **all** transactions including reversals/adjustments.

### 2.5 Negative Balance Policy

**Decision:** Negative balances are **allowed** with configurable soft limits.

#### Effective minimum balance resolution
The floor for a user's balance is resolved in priority order (most specific wins):

1. **User override** — `user.min_balance_override` (nullable float); set by admin/moderator on a per-user basis
2. **Group override** — per community group/pod (phase 2; not in MVP data model)
3. **System default** — `config.system_min_balance`; initial value **-1.0 hours**, adjustable up to **-2.0 hours** at launch; stored in server configuration (not per-row)

```
effective_min(user) = user.min_balance_override
                      ?? group.min_balance_override   (phase 2)
                      ?? config.system_min_balance
```

#### Enforcement points
Checked at **two gates**; both must pass:

| Gate | Event | Check | On failure |
|---|---|---|---|
| A3 accept | `accept` | `balance(requester) - proposed_hours >= effective_min(requester)` | Reject acceptance; notify both parties |
| T1 → T2 | `post_transaction` | Same check with `final_hours` | Block posting; open dispute or return to A6 |

The T1 → T2 check is the authoritative enforcement point. The A3 check is best-effort (hours may change during work).

#### Admin operations
- Admin/moderator may set or clear `user.min_balance_override` at any time.
- Every change to `min_balance_override` is logged as a **ModerationAction** (`action_type = balance_limit_change`) with rationale.
- "Soft" means the limit is enforced at transaction time — not a hard database constraint — so the value can be adjusted without schema changes.

---

## 3) Reviews State Machine (Double-Blind, Anti-Abuse)

Reviews are linked to an agreement and allowed only for participants.

### 3.1 Review States (per agreement, aggregate)
- **R0 NOT_ELIGIBLE** — agreement not completed/settled
- **R1 ELIGIBLE** — agreement is completed (A7) or settled (A10 with settlement)
- **R2 ONE_SUBMITTED** — one party submitted review; other pending
- **R3 BOTH_SUBMITTED_PENDING_REVEAL** — both submitted; waiting on reveal job
- **R4 REVEALED** — reviews revealed to both parties; aggregate signals updated
- **R5 EXPIRED_UNREVEALED** — window expired; reveal what exists, mark missing
- **R6 MOD_HIDDEN** — one or both reviews hidden from public due to abuse
- **R7 MOD_ACTIONED** — reviews triggered enforcement (warning/suspension) (audit logged)

### 3.2 Review Window
- Review submission window: **W_submit = 7 days** (configurable)
- If only one review submitted by end of window:
  - reveal that review, but **do not** show it as a “gotcha”; label as “one-sided / timed out”
  - apply reduced weight to public signals (optional)

### 3.3 Review Contents (structured signals)
Avoid weaponizable star ratings; use structured fields:

**Required:**
- `reliability`: {met_time_commitments, communicated_changes, completed_as_agreed} (boolean or 3-point scale)
- `quality`: {met_scope_expectations} (3-point scale)
- `respect_safety`: {felt_respected, felt_safe} (boolean/3-point)

**Optional:**
- free-text note (subject to filtering / moderation)

### 3.4 Anti-Abuse Constraints
- Only participants can submit.
- Must be tied to a completed/settled agreement.
- Double-blind: reveal only after both submit OR timeout.
- Rate limits: max reviews per day; prevent spam.
- Text filters: auto-flag slurs/doxxing/PII.
- Weighting: reviews from new/low-activity accounts have reduced impact.

### 3.5 Transitions
| From | Event | To | Notes |
|---|---|---|---|
| R0 NOT_ELIGIBLE | agreement_completed_confirmed OR mod_settlement (outcome=settled) | R1 ELIGIBLE | Eligibility begins |
| R0 NOT_ELIGIBLE | mod_enable_reviews (M) | R1 ELIGIBLE | Mod-authorized for A10 "no_service" or special cases only |
| R1 ELIGIBLE | submit_review (R or P) | R2 ONE_SUBMITTED | Stores review privately |
| R1 ELIGIBLE | timeout (S) | R5 EXPIRED_UNREVEALED | Window expired; no reviews submitted; nothing to reveal |
| R2 ONE_SUBMITTED | submit_review (other party) | R3 BOTH_SUBMITTED_PENDING_REVEAL | Both present |
| R3 BOTH_SUBMITTED_PENDING_REVEAL | reveal_job (S) | R4 REVEALED | Reveal to both; update aggregates |
| R2 ONE_SUBMITTED | timeout (S) | R5 EXPIRED_UNREVEALED | Reveal what exists; mark missing |
| R4 REVEALED | flag_review (S|R|P|M) | R6 MOD_HIDDEN | Participants may flag; moderator acts |
| R6 MOD_HIDDEN | mod_action (M) | R7 MOD_ACTIONED | Enforcement may occur |

### 3.6 Public Reputation Aggregation (signals over raw reviews)
Public profile should emphasize:
- `completed_exchanges_count`
- `no_show_rate` (derived from cancellations/disputes patterns; see below)
- `reliability_band`: Excellent/Good/Mixed (computed)
- `respect_safety_indicator`: e.g., “No recent safety flags” vs “Under review”

**Avoid** prominently displaying raw negative text to reduce harassment vectors.

### 3.7 Deriving No-Show / Cancellation Signals
Define system events (non-review) that feed reliability:
- If agreement reaches **A12 EXPIRED** repeatedly after acceptance → impacts reliability
- If user cancels frequently after scheduling → impacts reliability
- If disputes result in mod finding “no-show” → impacts reliability

This reduces reliance on subjective review text.

---

## 4) Cross-Machine Coupling Rules

### 4.1 Agreement → Ledger
- Only these agreement states may create ledger transactions:
  - **A7 COMPLETED_CONFIRMED**
  - **A10 RESOLVED (settled)**
- Ledger transaction references `agreement_id` and final hours.

### 4.2 Agreement → Reviews
- Reviews become **ELIGIBLE** when:
  - **A7 COMPLETED_CONFIRMED**
  - **A10 RESOLVED** with settlement OR mod-authorized completion
- If **A11 CANCELED** or **A12 EXPIRED** before work:
  - Reviews remain **NOT_ELIGIBLE** (prevents weaponized reviews without service)
- If **A10 RESOLVED** with “no service delivered”:
  - Reviews optional and only mod-enabled (case-by-case)

### 4.3 Ledger ↔ Disputes
- If a dispute is raised **before** posting transaction:
  - hold settlement (stay T0/T1) until resolution
- If dispute raised **after** transaction posted:
  - flag transaction (T2 → T3) and proceed with reversal/adjustment if needed

### 4.4 Reviews ↔ Moderation
- Safety-related review signals (e.g., “felt unsafe”) may:
  - create an automatic report (S)
  - route to moderation queue
- Free-text containing hate/doxxing:
  - auto-hide pending review (R? → R6) + open report

---

## 5) Timeouts & Defaults (Configurable)

### Agreement timeouts
- Proposal timeout: 7 days → **EXPIRED**
- Negotiation inactivity: 14 days → **EXPIRED**
- Completion confirmation timeout: 7 days:
  - default path: auto-open **DISPUTED** to force review
  - alternative: auto-resolve using marker’s final hours (not recommended)
- Dispute SLA (A8 DISPUTED): 48 hours → moderator assignment alert if unclaimed (no auto-resolution; requires human action)
- Mod review SLA (A9 MOD_REVIEW): 7 days → escalation alert to admin (no auto-resolution without moderator decision)

### Review timeouts
- Review submit window: 7 days
- Double-blind reveal runs:
  - immediately when both submitted
  - daily job for timeouts

---

## 6) Abuse & Edge Cases (MVP handling)

### 6.1 One party disappears after work
- Agreement in **COMPLETED_PENDING_CONFIRMATION** hits timeout:
  - move to **DISPUTED**
  - moderator can settle based on evidence/messages

### 6.2 Malicious rating attempts
- Prevent pre-service reviews by requiring completed/settled agreement.
- Double-blind reduces retaliation.
- Weighting reduces impact of sockpuppet accounts.

### 6.3 Transaction correction without deletion
- If hours were wrong:
  - post **adjustment** transaction referencing original
- If fraud/abuse:
  - post **reversal** transaction referencing original
- Always record moderator rationale in audit log.

### 6.4 Post-confirmation dispute (post-A7)
`raise_dispute` is not available from A7 COMPLETED_CONFIRMED (terminal state). If a problem surfaces after confirmation (e.g., fraud discovered after the fact):
- Parties use the **Report** mechanism (not `raise_dispute`)
- Moderator reviews the report and may flag the ledger transaction (T2 → T3)
- If warranted, moderator posts a reversal or adjustment transaction (T3 → T4/T5)
- The Agreement remains in A7; all moderator actions are logged in ModerationAction audit log
- Reviews are not retroactively suppressed unless flagged via the standard R4 → R6 path

---

## 7) Listing State Machine

### 7.1 Listing States
- **L0 DRAFT** — created by author but not yet published
- **L1 ACTIVE** — visible on marketplace; can receive messages/proposals
- **L2 PAUSED** — temporarily hidden by author (not accepting new inquiries)
- **L3 FULFILLED** — author marked as completed/no longer needed
- **L4 REMOVED_BY_MOD** — hidden by moderator action (soft delete)
- **L5 EXPIRED** — auto-expired after inactivity (configurable; e.g., 90 days)

### 7.2 Listing Events
- `publish(R|P)` — author publishes draft
- `pause(R|P)` — author hides listing temporarily
- `reactivate(R|P)` — author re-publishes paused listing
- `mark_fulfilled(R|P)` — author closes listing as done
- `mod_remove(M)` — moderator removes for policy violation
- `timeout(S)` — system expires inactive listing

### 7.3 Transition Table
| From | Event | To | Notes |
|---|---|---|---|
| L0 DRAFT | publish | L1 ACTIVE | Listing visible on marketplace |
| L1 ACTIVE | pause | L2 PAUSED | Author hides; existing conversations unaffected |
| L2 PAUSED | reactivate | L1 ACTIVE | Restore visibility |
| L1 ACTIVE | mark_fulfilled | L3 FULFILLED | Author closes; terminal |
| L2 PAUSED | mark_fulfilled | L3 FULFILLED | Can fulfill while paused |
| L1 ACTIVE | mod_remove | L4 REMOVED_BY_MOD | Moderation action; audit logged |
| L2 PAUSED | mod_remove | L4 REMOVED_BY_MOD | Can be removed while paused |
| L1 ACTIVE | timeout | L5 EXPIRED | Inactivity threshold (configurable) |
| L2 PAUSED | timeout | L5 EXPIRED | Inactivity while paused |

### 7.4 Terminal States
- **L3 FULFILLED**
- **L4 REMOVED_BY_MOD**
- **L5 EXPIRED**

---

## 8) User Account State Machine

### 8.1 Account States
- **U0 PENDING** — registered but not yet verified (email/phone confirmation)
- **U1 ACTIVE** — verified and in good standing
- **U2 WARNED** — received a formal warning; can still transact
- **U3 TEMPORARILY_LOCKED** — short-term suspension; cannot post or message
- **U4 SUSPENDED** — longer-term suspension pending review
- **U5 BANNED** — permanently removed from platform

### 8.2 Account Events
- `verify(S)` — email/phone verification confirmed
- `issue_warning(M)` — moderator issues formal warning
- `temp_lock(M)` — moderator applies short-term lock
- `suspend(M)` — moderator suspends account
- `ban(M)` — moderator permanently bans account
- `restore(M)` — moderator lifts lock or suspension
- `appeal_granted(M)` — appeal overturns a moderation action

### 8.3 Transition Table
| From | Event | To | Notes |
|---|---|---|---|
| U0 PENDING | verify | U1 ACTIVE | Account fully activated |
| U1 ACTIVE | issue_warning | U2 WARNED | Warning logged; user notified with reason |
| U2 WARNED | issue_warning | U2 WARNED | Repeat warning; audit logged |
| U1/U2 ACTIVE/WARNED | temp_lock | U3 TEMPORARILY_LOCKED | Duration configurable (e.g., 24–72 hours) |
| U3 TEMPORARILY_LOCKED | restore | U1 ACTIVE | Lock lifted; user restored to active |
| U1/U2/U3 | suspend | U4 SUSPENDED | Pending further review |
| U4 SUSPENDED | restore | U1 ACTIVE | Suspension lifted after review |
| U4 SUSPENDED | ban | U5 BANNED | Escalation to permanent ban |
| U1/U2/U3/U4 | ban | U5 BANNED | Direct ban for severe violations |
| U3/U4 | appeal_granted | U1 ACTIVE | Appeal overturns action; audit logged |

> Every enforcement transition creates a **ModerationAction** record with actor, reason, and timestamp.

### 8.4 Terminal States
- **U5 BANNED**

---

## 9) Report State Machine

### 9.1 Report States
- **Rep0 OPEN** — submitted; not yet reviewed
- **Rep1 IN_REVIEW** — assigned to moderator; under active review
- **Rep2 RESOLVED_ACTIONED** — review complete; enforcement action taken
- **Rep3 RESOLVED_DISMISSED** — review complete; no action warranted
- **Rep4 ESCALATED** — complexity or sensitivity requires senior/admin review

### 9.2 Report Events
- `submit(R|P)` — user submits a report
- `mod_claim(M)` — moderator takes ownership of report
- `escalate(M)` — moderator escalates to admin/senior mod
- `resolve_action(M)` — moderator closes report with enforcement action
- `resolve_dismiss(M)` — moderator closes report with no action

### 9.3 Transition Table
| From | Event | To | Notes |
|---|---|---|---|
| — | submit | Rep0 OPEN | Creates Report record; appears in mod queue |
| Rep0 OPEN | mod_claim | Rep1 IN_REVIEW | Assigned to moderator |
| Rep0 OPEN | timeout (SLA breach) | Rep4 ESCALATED | Auto-escalate if unclaimed beyond SLA |
| Rep1 IN_REVIEW | escalate | Rep4 ESCALATED | Manual escalation |
| Rep1 IN_REVIEW | resolve_action | Rep2 RESOLVED_ACTIONED | Enforcement taken; ModerationAction created |
| Rep1 IN_REVIEW | resolve_dismiss | Rep3 RESOLVED_DISMISSED | No action; rationale logged |
| Rep4 ESCALATED | mod_claim | Rep1 IN_REVIEW | Admin/senior mod claims it |
| Rep4 ESCALATED | resolve_action | Rep2 RESOLVED_ACTIONED | Senior mod resolves with action |
| Rep4 ESCALATED | resolve_dismiss | Rep3 RESOLVED_DISMISSED | Senior mod dismisses |

### 9.4 Terminal States
- **Rep2 RESOLVED_ACTIONED**
- **Rep3 RESOLVED_DISMISSED**

---

## 10) Recommended Minimal Diagram (Mermaid)

```mermaid
stateDiagram-v2
  state "Agreement" as A {
    [*] --> PROPOSED: send_proposal
    PROPOSED --> NEGOTIATING: counter_propose
    NEGOTIATING --> ACCEPTED: accept (both)
    PROPOSED --> ACCEPTED: accept (both)
    ACCEPTED --> IN_PROGRESS: start_work
    IN_PROGRESS --> COMPLETED_PENDING: mark_complete
    COMPLETED_PENDING --> COMPLETED_CONFIRMED: confirm_complete
    COMPLETED_PENDING --> DISPUTED: raise_dispute / timeout
    ACCEPTED --> DISPUTED: raise_dispute
    IN_PROGRESS --> DISPUTED: cancel / raise_dispute
    DISPUTED --> MOD_REVIEW: mod_take_case
    MOD_REVIEW --> RESOLVED: mod_decide
    PROPOSED --> CANCELED: decline/cancel
    ACCEPTED --> CANCELED: cancel
    NEGOTIATING --> EXPIRED: timeout
    ACCEPTED --> EXPIRED: timeout
  }

  state "Ledger Tx" as T {
    [*] --> NONE
    NONE --> PENDING: agreement_completed_confirmed / mod_settlement
    PENDING --> POSTED: post_transaction
    POSTED --> FLAGGED: flag_transaction
    FLAGGED --> POSTED: close_flag
    FLAGGED --> ADJUSTED: post_adjustment
    FLAGGED --> REVERSED: post_reversal
  }

  state "Reviews" as R {
    [*] --> NOT_ELIGIBLE
    NOT_ELIGIBLE --> ELIGIBLE: agreement_completed_confirmed / mod_enable
    ELIGIBLE --> ONE_SUBMITTED: submit_review
    ELIGIBLE --> EXPIRED_REVEAL: timeout
    ONE_SUBMITTED --> BOTH_SUBMITTED: submit_review (other)
    BOTH_SUBMITTED --> REVEALED: reveal_job
    ONE_SUBMITTED --> EXPIRED_REVEAL: timeout
    REVEALED --> MOD_HIDDEN: flag_review
    MOD_HIDDEN --> MOD_ACTIONED: mod_action
  }