# PPI Wallet — Epic Breakdown v1

Engineering Handoff Document  |  Paytm Payment Services Limited  |  March 2026

This document converts the PPSL PPI Wallet PRD into engineering-ready epics and user stories for import into JIRA or Notion. Each epic maps to one or more PRD product blocks. Stories follow the 'As a... I want... So that...' format with acceptance criteria, dependencies, and open questions.

## Epic Summary

EPIC-08 — MCP & AI Agent Layer [IMPLEMENTED]: MCP-01 through MCP-07 (MCP Server, Data Layer, Render Backend, Transaction Sync, AI Summarizer, KYC Expiry Intelligence, Agentic Chat)

MCP-08: KYC Expiry Alert System

As an admin, I want an automated system that detects users with KYC expiring in 7 days, drafts personalised outreach messages using Claude Haiku, and displays alerts in the dashboard so that the compliance team can proactively manage KYC renewals.

Acceptance: kyc-alert-service.js with 4-step pipeline (fetch, generate, simulate-send, summarise), two API endpoints (preview + run), KycAlertPanel React component with status bar and alert table, node-cron scheduler for daily 9 AM IST runs, graceful error handling per user, Claude Haiku for cost efficiency.

| Epic ID | Priority | Epic Name | PRD Blocks | Stories | Key Dependency |
| --- | --- | --- | --- | --- | --- |
| EPIC-01 | P1 | KYC & User Identity | Block 03 | 5 stories | UIDAI KUA registration (regulatory) |
| EPIC-02 | P0 | Wallet Issuance & Lifecycle | Block 01 (Ledger), Block 02 (PPI Engine), Block 12 (Lifecycle) | 4 stories | Block 03 (KYC events) |
| EPIC-03 | P1 | Load / Unload Flows (Add Money & Withdraw) | Block 04 (Money Movement) | 4 stories | Block 01 (Ledger debit-first pattern) |
| EPIC-04 | P0 | UPI Linking & VPA Management | Block 11 (UPI Integration) | 4 stories | NPCI TPAP membership (critical |
| EPIC-05 | P1 | Merchant Acceptance & Payments | Block 10 (Merchant Ecosystem) | 3 stories | PPSL PA-PG merchant network (reuse) |
| EPIC-06 | P1 | Settlement & Reconciliation | Block 08 (Recon), Block 05 (Nodal), Block 09 (Reporting) | 3 stories | Block 01 (Ledger) |
| EPIC-07 | P1 | Dispute Management | Block 14 (Dispute Management) | 3 stories | Block 04 (Transactions) |

# EPIC-01 — KYC & User Identity

| Epic ID | EPIC-01 |
| --- | --- |
| Priority | P1 |
| PRD Blocks | Block 03 |
| Description | End-to-end customer identity verification and KYC tier management for the PPSL PPI wallet. Covers Min-KYC onboarding via mobile + self-declaration, Full-KYC via Aadhaar OTP/biometric/Video KYC, PAN verification, CKYC registry integration, and ongoing KYC refresh. The KYC tier determines wallet balance caps and feature access. |
| Dependencies | UIDAI KUA registration (regulatory), NSDL/Karza PAN API, Video KYC vendor contract, Block 01 (Ledger), Block 02 (PPI Type Engine), Block 07 (Limits Engine) |
| Open Questions | UIDAI registration for PPSL entity (separate from Paytm TPAP), CKYC upload API contract with CERSAI, Video KYC vendor selection |

**User Stories**

## KYC-01: Initiate Min-KYC Wallet

| User Story | As a new user downloading the Paytm app, I want to create a minimum KYC wallet using only my mobile number and name, so that I can start making payments immediately without a lengthy verification process. |
| --- | --- |
| Acceptance Criteria | Mobile OTP verification completes in < 30 seconds Self-declaration form captures name and unique ID (mobile = unique identifier) Wallet created in ACTIVE state with Min-KYC tier Balance cap set to Rs 10,000; monthly load limit set to Rs 10,000 Wallet validity set to 12 months from creation date P2P transfers blocked for Min-KYC wallets |
| Dependencies | Block 01 (Wallet Ledger), Block 07 (Limits Engine) |
| Open Questions | Confirm whether mobile-only onboarding satisfies RBI PPI-MD Clause 9.2 unique ID requirement Define behavior when user already has a Paytm account — link or create new wallet? |

## KYC-02: Aadhaar OTP eKYC Verification

| User Story | As a Min-KYC wallet holder, I want to upgrade my wallet via Aadhaar OTP verification, so that I can get a higher wallet balance limit without visiting a branch. |
| --- | --- |
| Acceptance Criteria | Aadhaar OTP sent via UIDAI/KUA integration within 10 seconds OTP verified within 3 attempts before lockout On success: wallet tier updated to Full-KYC, balance cap raised to Rs 2,00,000 KYC document (masked Aadhaar) encrypted and stored per PMLA requirements KYC status event published: kyc.tier.upgraded CKYC registry lookup performed before Video KYC path; CKYC upload triggered post-KYC |
| Dependencies | UIDAI KUA agreement required (regulatory pre-condition), Block 02 (PPI Type Engine) |
| Open Questions | Confirm PPSL's UIDAI KUA registration status — separate from Paytm's existing TPAP What is the fallback if UIDAI OTP service is down? Offer Video KYC directly? |

## KYC-03: PAN Verification

| User Story | As a user completing Full-KYC, I want to verify my PAN as part of the KYC process, so that PPSL can comply with PMLA identity requirements. |
| --- | --- |
| Acceptance Criteria | PAN verified against ITD/NSDL API within 5 seconds Name on PAN matched against Aadhaar name (fuzzy match, 80% threshold) PAN stored masked in all logs (last 4 visible only) Duplicate PAN check: one PAN should not be linked to more than one Full-KYC wallet On PAN mismatch: prompt user to re-enter or initiate manual review |
| Dependencies | NSDL/Karza PAN API contract |
| Open Questions | How to handle NRI customers without Indian PAN? Define manual review SLA for PAN mismatches — who reviews? |

## KYC-04: Video KYC Session

| User Story | As a user who cannot complete Aadhaar OTP KYC, I want to complete KYC via a live video session with an agent, so that I can still access Full-KYC wallet features. |
| --- | --- |
| Acceptance Criteria | Video KYC vendor (IDfy or equivalent) integration handles session initiation Session recorded and stored for 5 years per RBI requirement Agent can capture live photo + ID document during session On successful review (within 24 hours): wallet upgraded to Full-KYC tier Async workflow: wallet stays in KYC_PENDING state during review Rejection reason communicated to user with retry option |
| Dependencies | Video KYC vendor contract, UIDAI Video KYC circular compliance |
| Open Questions | Which Video KYC vendor is selected? IDfy, Signzy, or bank-provided? What is the SLA for agent review — 24 hours acceptable during launch? |

## KYC-05: KYC Expiry & Re-verification

# LOAD-GUARD — Wallet Load Guard

# Wallet Load Guard — RBI PPI Limit Validation

# The Wallet Load Guard is a pre-transaction validation layer that enforces RBI PPI limits before any wallet load (add money) transaction is processed.

# Three RBI Rules Validated:

# 1. BALANCE_CAP — Maximum wallet balance of ₹1,00,000

# 2. MONTHLY_LOAD — Maximum ₹2,00,000 total loads per calendar month

# 3. MIN_KYC_CAP — Maximum ₹10,000 balance for Minimum KYC users

# Architecture:

# - Backend: mcp/services/wallet-load-guard.js — validation + Claude Haiku messages

# - API: POST /api/wallet/validate-load, GET /api/wallet/load-guard-log

# - UI: Add Money page with live helper text, block alerts, "Add max instead" button

# - Admin: Load Guard Log panel on Dashboard (last 10 blocked attempts)

| User Story | As a Min-KYC wallet holder approaching 12-month expiry, I want to receive timely notifications to upgrade my KYC, so that I don't lose access to my wallet balance. |
| --- | --- |
| Acceptance Criteria | 30-day, 7-day, and 1-day pre-expiry push + SMS notifications triggered automatically On expiry: wallet moves to EXPIRED state, all transactions blocked User can initiate KYC upgrade from expired state — balance preserved for 60 days If no KYC upgrade within 60 days post-expiry: wallet closed, balance returned to source |
| Dependencies | Block 12 (Wallet Lifecycle), Block 13 (Notifications) |
| Open Questions | Confirm 60-day grace period is acceptable per RBI guidance Define balance return path when wallet is closed |

# EPIC-02 — Wallet Issuance & Lifecycle

| Epic ID | EPIC-02 |
| --- | --- |
| Priority | P0 |
| PRD Blocks | Block 01 (Ledger), Block 02 (PPI Engine), Block 12 (Lifecycle) |
| Description | Creation, state management, and closure of PPI wallets for PPSL. The wallet is the core financial entity — every money movement flows through it. The ledger is the single source of truth. This epic covers wallet creation, state machine transitions (CREATED → ACTIVE → SUSPENDED → CLOSED), dormancy management, and KYC upgrade lifecycle. |
| Dependencies | Block 03 (KYC events), Block 04 (money movement), Block 06 (risk), Block 07 (limits), Block 09 (compliance reporting), Block 13 (notifications), Block 15 (admin panel) |
| Open Questions | Wallet ID format for RBI reporting, handling of users with multiple wallet types, dormancy period confirmation with legal |

**User Stories**

## WAL-01: Create Wallet on KYC Completion

| User Story | As a new PPSL user who has completed minimum KYC, I want to have a wallet automatically created and activated, so that I can immediately start using it for payments. |
| --- | --- |
| Acceptance Criteria | Wallet record created in wallet_accounts table with unique wallet_id (UUID) Initial state: ACTIVE KYC tier: MIN_KYC or FULL_KYC based on onboarding path Zero-balance ledger entry created (CREDIT 0, for initialisation) wallet.account.created Kafka event published within 200ms Idempotent: same user_id + kyc_tier cannot create duplicate wallets |
| Dependencies | Block 03 (KYC completion event triggers wallet creation) |
| Open Questions | Can a user have both a Min-KYC and Full-KYC wallet simultaneously, or must they upgrade? What wallet_id format is required — UUID, or numeric for RBI reporting? |

## WAL-02: Wallet Suspension

| User Story | As a compliance operations agent, I want to suspend a wallet flagged for suspicious activity, so that we can prevent further transactions while investigating. |
| --- | --- |
| Acceptance Criteria | POST /wallet/{id}/suspend requires ops agent role + reason code + reason text Suspension is immediate — all in-flight transactions rejected after suspension timestamp Customer notified via SMS + push within 2 minutes Suspension logged in audit trail with actor, timestamp, reason Pending settlements from suspended wallet are held, not cancelled 4-eye approval required for suspensions above Rs 50,000 active balance |
| Dependencies | Block 15 (Admin Panel), Block 06 (Risk Engine) |
| Open Questions | What is the SLA for investigating a suspended wallet? Who has authority to reactivate — same ops agent or senior? |

## WAL-03: Wallet Closure

| User Story | As a customer, I want to close my wallet and receive any remaining balance, so that I can exit the product cleanly. |
| --- | --- |
| Acceptance Criteria | Customer-initiated closure via settings flow; or ops-initiated via admin panel Balance > Rs 0: refunded to linked bank account via NEFT/IMPS within T+1 All pending transactions must complete or be cancelled before closure Wallet state moves to CLOSED — no further credits accepted Closure reason and timestamp recorded in audit trail 30-day cooling period: wallet_id cannot be reused |
| Dependencies | Block 04 (wallet-to-bank transfer), Block 13 (notification on closure) |
| Open Questions | What if the user has no linked bank account? Allow wallet-to-wallet transfer? Regulatory question: how long must closed wallet data be retained? |

## WAL-04: Dormancy Detection & Management

| User Story | As a compliance manager, I want wallets with no transactions for 12 months to be automatically flagged and handled, so that we comply with RBI dormancy requirements for PPIs. |
| --- | --- |
| Acceptance Criteria | Nightly batch job flags wallets with last_transaction_at > 12 months as DORMANT Dormant wallets: load and P2P blocked; spend at merchant allowed SMS + email notification sent at 11-month mark as pre-dormancy warning Re-activation: any transaction re-activates; dormancy flag cleared Wallets DORMANT > 24 months: balance transferred to suspense account, wallet CLOSED |
| Dependencies | Block 09 (Compliance Reporting), Block 13 (Notifications) |
| Open Questions | Confirm 12-month dormancy threshold with legal — is it calendar months or rolling? |

# EPIC-03 — Load / Unload Flows (Add Money & Withdraw)

| Epic ID | EPIC-03 |
| --- | --- |
| Priority | P1 |
| PRD Blocks | Block 04 (Money Movement) |
| Description | All inbound (load) and outbound (unload) money movement for the wallet. Load flows: UPI collect, UPI AutoPay, cards, net banking. Unload flows: wallet-to-bank transfer (NEFT/IMPS), bill payment (BBPS), merchant spend (which drives settlement). Refund handling is also covered here. |
| Dependencies | Block 01 (Ledger debit-first pattern), Block 05 (Nodal Bank API), Block 07 (Limits), Block 11 (UPI), Block 13 (Notifications), NPCI TPAP license, existing PPSL PG |
| Open Questions | UPI handle format for nodal, withdrawal fee structure, refund path for closed wallets |

**User Stories**

## LOAD-01: Add Money via UPI Collect

| User Story | As a wallet holder, I want to add money to my wallet by approving a UPI collect request, so that I can use UPI as the most convenient way to top up. |
| --- | --- |
| Acceptance Criteria | UPI collect request initiated from nodal account UPI handle Push notification sent to user's UPI app within 5 seconds On user approval: nodal receives credit, wallet credited within 30 seconds Idempotent: same UPI transaction ref cannot credit wallet twice If UPI payment fails or times out: no wallet credit; clear error surfaced to user Limits check (Block 07) performed before accepting collect request |
| Dependencies | Block 05 (Nodal inward credit webhook), Block 11 (UPI integration), NPCI TPAP license |
| Open Questions | What is the UPI handle format for PPSL nodal account? How to handle UPI timeout scenarios — auto-retry or manual resolution? |

## LOAD-02: Add Money via Debit Card / Netbanking

| User Story | As a wallet holder, I want to add money to my wallet using my debit card or netbanking, so that I have flexibility in how I fund my wallet. |
| --- | --- |
| Acceptance Criteria | Integrates with existing PPSL PA-PG card/NB infrastructure PG success callback triggers nodal credit + wallet credit atomically Wallet credited only after nodal credit confirmed (not on PG success alone) Maximum single transaction: Rs 10,000 for Min-KYC; Rs 50,000 for Full-KYC Failed PG transactions: user notified immediately; no wallet debit |
| Dependencies | Block 05 (Nodal), existing PPSL PG integration |
| Open Questions | Should we impose a per-load fee? If yes, at what threshold? |

## LOAD-03: Wallet-to-Bank Withdrawal

| User Story | As a Full-KYC wallet holder, I want to withdraw money from my wallet to my bank account, so that I can access my wallet balance as cash when needed. |
| --- | --- |
| Acceptance Criteria | Only available for Full-KYC wallets (Min-KYC: blocked) User selects linked bank account (pre-verified via penny drop) Withdrawal initiates IMPS (instant) or NEFT (4-hour batches) Wallet debited FIRST (hold), bank transfer initiated second (debit-first pattern) On bank transfer failure: wallet debit reversed, user notified Daily withdrawal limit: Rs 25,000 for Full-KYC (configurable via admin) |
| Dependencies | Block 05 (Nodal outward debit API), Block 07 (Limits Engine) |
| Open Questions | What is the fee structure for withdrawal — free or per-transaction? Minimum balance requirement before withdrawal — Rs 1 or Rs 0? |

## LOAD-04: Refunds to Wallet

| User Story | As a customer who has a refund pending from a merchant, I want the refund to be credited back to my wallet automatically, so that I receive my money back without manual intervention. |
| --- | --- |
| Acceptance Criteria | Refund initiated by merchant via /refund/initiate API Refund credited to source wallet (not to bank account, unless wallet is closed) Idempotency: same refund_ref cannot credit wallet twice Partial refunds supported Refund credited within T+1 of merchant initiation User notified via SMS + push on refund credit |
| Dependencies | Block 04, Block 13 (Notifications) |
| Open Questions | What is the refund path if wallet is closed between purchase and refund? |

# EPIC-04 — UPI Linking & VPA Management

| Epic ID | EPIC-04 |
| --- | --- |
| Priority | P0 |
| PRD Blocks | Block 11 (UPI Integration) |
| Description | UPI Virtual Payment Address (VPA) creation, management, and interoperability for Full-KYC PPSL wallets. This includes wallet-as-UPI-funding-source, third-party UPI app linkage (mandatory per Dec 2024 RBI amendment), UPI AutoPay mandates, and UPI dispute management. |
| Dependencies | NPCI TPAP membership (critical, 6-12 month lead time), NPCI UPI certification, NPCI inter-PSP technical certification, Block 03 (KYC), Block 07 (Limits), Block 01 (Ledger) |
| Open Questions | PPSL UPI handle registration with NPCI, third-party app certification priority, PPI wallet UPI interchange commercial model |

**User Stories**

## UPI-01: VPA Creation for Full-KYC Wallet

| User Story | As a Full-KYC wallet holder, I want to have a UPI Virtual Payment Address automatically assigned to my wallet, so that I can receive and send money via UPI using my wallet balance. |
| --- | --- |
| Acceptance Criteria | VPA format: {mobile}@ppsl or {mobile}@paytm (confirm NPCI handle) VPA uniqueness enforced at NPCI level VPA creation triggered automatically on Full-KYC completion VPA linked to wallet_id in vpa_registrations table wallet.vpa.created event published Min-KYC wallets: no VPA assigned |
| Dependencies | NPCI TPAP membership (critical pre-condition), Block 03 (KYC completion event) |
| Open Questions | CRITICAL: What is PPSL's UPI handle (e.g., @ppsl)? NPCI registration required. Can the user choose a custom VPA? |

## UPI-02: Use Wallet Balance for UPI Payment

| User Story | As a Full-KYC wallet holder, I want to select my PPSL wallet as the funding source when making a UPI payment, so that I can pay merchants via UPI without needing a bank account linked. |
| --- | --- |
| Acceptance Criteria | Wallet VPA appears as a payment option in PPSL's UPI payment flow Balance check performed in real-time before payment initiation Limits check (wallet monthly spend) enforced before debit On payment success: wallet debited, merchant UPI VPA credited via NPCI On payment failure: wallet debit reversed within 2 minutes |
| Dependencies | Block 07 (Limits), Block 01 (Ledger), NPCI UPI rails |
| Open Questions | PPI wallet UPI interchange: PPSL will incur interchange on wallet-funded UPI transactions above Rs 2,000 to merchants. Confirm commercial model. |

## UPI-03: Third-Party UPI App Linkage (Dec 2024 RBI Mandate)

| User Story | As a Full-KYC PPSL wallet holder, I want to link my PPSL wallet as a funding source in Google Pay, PhonePe, or BHIM, so that I can use my wallet balance in any UPI app I prefer. |
| --- | --- |
| Acceptance Criteria | PPSL wallet appears in UPI app's 'Add payment method' / 'Link new account' flow User authenticates via PPSL app redirect + wallet PIN Wallet balance displayed in third-party UPI app in near-real-time (60-second refresh) Payment debits from wallet processed via NPCI inter-PSP UPI protocol User can unlink at any time from either PPSL app or third-party app NPCI technical certification for inter-PSP linking required before launch |
| Dependencies | NPCI TPAP certification, NPCI inter-PSP UPI technical compliance |
| Open Questions | CRITICAL: This is mandated by Dec 2024 RBI amendment — must be on launch roadmap Which third-party apps to certify first? Google Pay, PhonePe, BHIM priority order? |

## UPI-04: UPI AutoPay Mandate Setup

| User Story | As a Full-KYC wallet holder, I want to set up a recurring UPI AutoPay mandate funded from my wallet, so that I can automate recurring payments (subscriptions, utility bills) from my wallet. |
| --- | --- |
| Acceptance Criteria | User can create mandate with: biller VPA, amount, frequency (monthly/weekly), start/end date Mandate registered with NPCI via UPI AutoPay protocol On due date: wallet auto-debited and biller credited if balance sufficient If balance insufficient: user notified 24 hours before due date User can pause, modify, or cancel mandate at any time |
| Dependencies | Block 04 (money movement), NPCI AutoPay certification |
| Open Questions | Wallet-funded mandates have different NPCI rules than bank-account mandates. Confirm applicable limits. |

# EPIC-05 — Merchant Acceptance & Payments

| Epic ID | EPIC-05 |
| --- | --- |
| Priority | P1 |
| PRD Blocks | Block 10 (Merchant Ecosystem) |
| Description | Enabling merchants to accept PPSL wallet payments — online and offline. Covers QR code payments, payment links, in-app SDK integration, and the merchant settlement track for wallet-funded transactions. Leverages PPSL's existing PA-PG merchant network wherever possible. |
| Dependencies | PPSL PA-PG merchant network (reuse), Block 05 (Nodal), Block 07 (Limits), Block 08 (Recon), Block 11 (UPI) |
| Open Questions | MDR structure for wallet payments, wallet-funded settlement track timing |

**User Stories**

## MERCH-01: Merchant QR Payment

| User Story | As a Full-KYC wallet holder at a physical merchant, I want to scan the merchant's Bharat QR or UPI QR and pay from my wallet, so that I can use my wallet for in-store purchases without cash. |
| --- | --- |
| Acceptance Criteria | QR scan decodes merchant VPA or Bharat QR payload Wallet balance check + limits check before presenting payment confirmation screen Payment completes in < 4 seconds end-to-end Merchant receives credit to their settlement account per MDR/settlement terms Wallet debit and merchant credit are atomic — no partial state P2P: receipt is generated with merchant name, amount, timestamp |
| Dependencies | Block 11 (UPI), Block 07 (Limits), existing PPSL merchant QR infrastructure |
| Open Questions | MDR for wallet payments to merchants — is there a differential MDR vs UPI? |

## MERCH-02: Online Merchant Payment via Wallet

| User Story | As a customer checking out on a PPSL-integrated merchant website, I want to select 'Paytm Wallet' as my payment method, so that I can complete purchases using my wallet balance. |
| --- | --- |
| Acceptance Criteria | Wallet payment option surfaced in PPSL PG checkout SDK User authenticates via wallet PIN (no OTP for amounts <= Rs 2,000; OTP for amounts > Rs 2,000) Transaction completes via PPSL PA-PG acquiring infrastructure Wallet debit recorded in ledger before merchant credit initiated Refund path: merchant-initiated refund credits wallet within T+1 |
| Dependencies | Existing PPSL PA-PG SDK, Block 06 (Risk: OTP step-up logic) |
| Open Questions | Should wallet-funded online payments have a different risk scoring profile? |

## MERCH-03: Merchant Settlement for Wallet-Funded Transactions

| User Story | As a merchant accepting wallet payments, I want to receive settlement for wallet-funded transactions on the agreed settlement cycle, so that I can manage my cash flow predictably. |
| --- | --- |
| Acceptance Criteria | Wallet-funded transactions settled in a separate settlement batch (distinct from card/UPI settlements) Settlement initiated from PPSL nodal account to merchant bank account Settlement timing: T+1 by default; same-day available for premium merchants Settlement report available in merchant dashboard Failed settlements flagged in recon engine (Block 08) and retried within 24 hours |
| Dependencies | Block 05 (Nodal outward), Block 08 (Reconciliation) |
| Open Questions | Are wallet-funded merchant settlements governed by PPSL's existing PA settlement SLAs? |

# EPIC-06 — Settlement & Reconciliation

| Epic ID | EPIC-06 |
| --- | --- |
| Priority | P1 |
| PRD Blocks | Block 08 (Recon), Block 05 (Nodal), Block 09 (Reporting) |
| Description | End-of-day and intraday reconciliation of PPSL's internal wallet ledger against the nodal bank statement. Covers float integrity, merchant settlement reconciliation, exception management, and RBI-mandated compliance reporting. |
| Dependencies | Block 01 (Ledger), Block 05 (Nodal Bank API/SFTP), Block 15 (Admin Panel), RBI DPSS report format |
| Open Questions | RBI-prescribed PPI report format confirmation, holiday handling for recon, break escalation matrix |

**User Stories**

## RECON-01: Daily Nodal Bank Reconciliation

| User Story | As a finance operations analyst, I want to run the daily reconciliation job comparing our internal ledger to the nodal bank statement, so that I can identify and resolve any discrepancies before they become compliance issues. |
| --- | --- |
| Acceptance Criteria | Automated EOD recon job runs at 23:30 IST daily Bank statement ingested via SFTP or bank API (MT940 or bank-specific format) Every internal ledger entry matched against a corresponding bank statement line Match criteria: amount, direction, reference_id, date window (same day +/- 1 business day for late credits) Unmatched items flagged as RECON_BREAK and assigned to ops queue Recon report auto-generated and stored for 5 years Float assertion: sum(all active wallet balances) <= nodal bank balance (breach = P0 alert) |
| Dependencies | Block 05 (Nodal bank API/SFTP), Block 01 (Ledger) |
| Open Questions | What is the reconciliation report format required for RBI inspection? How to handle public holidays when bank statements arrive late? |

## RECON-02: Reconciliation Break Resolution

| User Story | As a finance operations agent, I want to view, investigate, and resolve reconciliation breaks from the ops dashboard, so that discrepancies are resolved within SLA and the ledger remains accurate. |
| --- | --- |
| Acceptance Criteria | Recon breaks displayed in ops dashboard with: break_type, amount, date, affected_wallet_id, suggested_resolution Break types: MISSING_CREDIT, DUPLICATE_CREDIT, AMOUNT_MISMATCH, DATE_MISMATCH Agent can mark break as resolved with resolution notes SLA tracking: systemic breaks <= 24 hours; manual breaks <= 72 hours Unresolved breaks > 72 hours auto-escalate to Finance Manager All resolution actions audit-logged |
| Dependencies | Block 15 (Admin / Ops Panel), Block 01 (Ledger — reversal entries) |
| Open Questions | What is the escalation path for breaks > Rs 1 lakh? |

## RECON-03: RBI Periodic Compliance Report Generation

| User Story | As a compliance manager, I want to generate the monthly RBI PPI report from the system automatically, so that we can submit accurate data to RBI DPSS on time without manual data extraction. |
| --- | --- |
| Acceptance Criteria | Monthly report covers: total wallets issued, active wallets, total load (by tier), total spend, P2P volume, merchant payment volume, outstanding balance Report generated from ledger data in RBI-prescribed format Auto-generated by 3rd of following month Manual review and sign-off by Compliance Manager before submission Report stored for 5 years per RBI direction Quarterly report also supported (RBI may require quarterly submission) |
| Dependencies | Block 01 (Ledger), Block 03 (KYC tier data), RBI DPSS prescribed format |
| Open Questions | Obtain RBI-prescribed PPI report format from DPSS before building this module Does PPSL need to submit reports to both RBI and NPCI? |

# EPIC-07 — Dispute Management

| Epic ID | EPIC-07 |
| --- | --- |
| Priority | P1 |
| PRD Blocks | Block 14 (Dispute Management) |
| Description | End-to-end customer dispute management for wallet transactions. Covers customer-raised disputes, auto-routing to correct teams, SLA tracking, UPI chargeback integration, and RBI Ombudsman escalation handling. |
| Dependencies | Block 04 (Transactions), Block 06 (Risk), Block 11 (UPI), Block 13 (Notifications), Block 15 (Admin Panel), Block 09 (Compliance Reporting) |
| Open Questions | Dispute reason code taxonomy, UPI chargeback NPCI certification, auto-close-in-favour-of-customer legal sign-off |

**User Stories**

## DISP-01: Customer Raises a Transaction Dispute

| User Story | As a wallet holder, I want to raise a dispute for a transaction I don't recognise, so that I can get a refund if the transaction was unauthorised or failed. |
| --- | --- |
| Acceptance Criteria | Dispute raise available from transaction history in app — tap 'Dispute this transaction' Dispute form captures: reason code (from predefined list), description, expected resolution Dispute auto-assigned to correct team based on reason code (see DISP-02) Customer receives acknowledgement with case_id and expected resolution date (T+30 days per RBI) Dispute status visible in app: OPEN, UNDER_REVIEW, RESOLVED, ESCALATED |
| Dependencies | Block 04 (transaction reference), Block 13 (Notifications) |
| Open Questions | Dispute reason code taxonomy needs to be defined (align with UPI dispute categories + wallet-specific) |

## DISP-02: Dispute Auto-Routing

| User Story | As a dispute management system, I want to automatically route disputes to the correct handling team based on transaction type, so that disputes reach the right resolver without manual triage. |
| --- | --- |
| Acceptance Criteria | Failed/pending transaction disputes → Settlement Team Unauthorised transaction disputes → Risk/Fraud Team Merchant refund disputes → Merchant Relations Team UPI chargebacks → UPI Dispute Team (integrates with NPCI chargeback flow) Routing rules configurable by admin without deployment If routing fails: dispute goes to default ops queue with high priority |
| Dependencies | Block 15 (Admin Panel — routing rules config), Block 06 (Risk), Block 11 (UPI) |
| Open Questions | UPI chargeback integration with NPCI requires NPCI certification — confirm timeline |

## DISP-03: SLA Tracking & Auto-Escalation

# EPIC-08 — MCP & AI Agent Layer [IMPLEMENTED]

Status: Implemented (April 2026) — 35 MCP tools, Render backend, agentic chat

## MCP-01: MCP Server with 35 Tools

As an AI agent, I want to query wallet data via 35 MCP tools so that I can answer natural language questions about users, transactions, balances, compliance, and analytics.

Acceptance: 35 tools registered in wallet-mcp-server.js with Zod validation, accessible via Claude Desktop stdio transport, covering 9 categories (User, Admin, Transactions, Compliance, Disputes, Notifications, KYC, Analytics, KYC Expiry).

## MCP-02: Mock Data Layer (200 Users)

As the system, I need a data layer with 200 users and 500+ transactions so that the demo environment has realistic, consistent data across all interfaces.

Acceptance: 200 users via seeded PRNG (seed=100) matching admin dashboard generators.ts, 500+ transactions (seed=200), 41 exported functions, BigInt paise arithmetic, RBI limits enforced.

## MCP-03: Render Backend API

As the deployed frontends, I need a backend API serving MCP data so that GitHub Pages apps can display real wallet data without a full Fastify+PostgreSQL backend.

Acceptance: 13 REST endpoints on Render (ppi-wallet-api.onrender.com), CORS for GitHub Pages, health check, user/transaction/KYC endpoints, wallet balance/ledger/status, transaction sync, AI chat.

## MCP-04: Transaction Sync (Wallet → MCP)

As the AI agent, I want wallet transactions to be synced to the MCP data layer so that my responses reflect the latest balance and transaction history.

Acceptance: sync.ts POSTs to Render /api/wallet/transact on every credit/debit, Render backend executes against mock-data.js in memory, supports ADD_MONEY, MERCHANT_PAY, P2P_TRANSFER, BILL_PAY.

## MCP-05: AI Transaction Summarizer

As an admin, I want to select transactions and get an AI-generated summary so that I can quickly understand patterns, anomalies, and compliance issues.

Acceptance: POST /api/summarise-transactions endpoint, admin role system prompt, returns natural language summary highlighting patterns, anomalies, failed transaction analysis.

## MCP-06: KYC Expiry Intelligence (3 Tools)

As a compliance officer, I want to filter users by KYC expiration date, query expiry data with flexible filters, and generate compliance reports so that I can ensure regulatory adherence.

Acceptance: query_kyc_expiry (urgency bands: expired/critical/warning/upcoming + sort_by, include_expired) (date range, state, balance filters with aggregations), generate_kyc_renewal_report (executive summary, financial impact, RBI recommendations).

## MCP-07: Agentic Chat Handler

As the system, I need a chat handler that routes natural language queries through Claude with tool_use capability so that both users and admins can interact conversationally with wallet data.

Acceptance: chat-handler.js with 35 TOOLS array, executeTool switch dispatcher, user and admin roles, multi-turn tool invocation, accessible via /api/chat endpoint.

| User Story | As a dispute operations manager, I want disputes approaching the 30-day RBI deadline to be automatically escalated, so that we never miss the regulatory SLA. |
| --- | --- |
| Acceptance Criteria | Every dispute has a calculated SLA_deadline = raised_at + 30 calendar days At day 25: auto-escalation email to team lead + app notification to customer At day 30: auto-escalation to Compliance Manager if unresolved; flagged in compliance dashboard At day 30 + 1: case auto-closed in favour of customer with wallet credit if no resolution SLA report exported monthly for RBI compliance review |
| Dependencies | Block 09 (Compliance Reporting) |
| Open Questions | Confirm 30-day SLA is current RBI requirement for PPI dispute resolution |
