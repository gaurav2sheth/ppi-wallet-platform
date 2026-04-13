**PPSL PPI Wallet**

Product Requirements & Build Blueprint

Paytm Payment Services Limited (PPSL)

Confidential — Internal Use Only

| Version | 1.0 — Initial Release |
| --- | --- |
| Status | For Internal Review |
| Classification | Confidential |

# 1. Executive Summary

This document defines the complete product requirements for building a Prepaid Payment Instrument (PPI) wallet under Paytm Payment Services Limited (PPSL). The objective is to build a standalone, RBI-compliant PPI wallet that will be linked into the Paytm consumer app as a new product surface.

This document covers 18 product blocks (5 newly added vs. the initial brief), the escrow partner bank evaluation framework, and a detailed sprint-by-sprint development plan structured for parallel execution by a high-velocity team using AI-assisted coding (Claude / Cursor).

| PPI License | PPSL must hold a valid RBI PPI license before any development begins |
| --- | --- |
| Total Blocks | 18 product blocks (13 original + 5 newly identified: UPI, Wallet Lifecycle, Notifications, Disputes, Admin Panel) |
| Build Strategy | BUILD (8 blocks) \| REUSE from Paytm (7 blocks) \| BUY/Integrate (3 blocks) |
| P0 Blockers | 5 hard blockers: PPI License, Nodal/Escrow bank tie-up, Wallet Ledger, Limits Engine, UPI/NPCI membership |
| Key Principle | Ledger is the single source of truth. Every money movement must produce a ledger entry before any external call is made. |

# 2. Build Block Summary

The table below summarises all 18 product blocks, their build/reuse/buy decision, and priority. Blocks marked ★ NEW were identified as missing from the original brief.

| # | Block Name | Decision | Priority |
| --- | --- | --- | --- |
| 01 | Core Wallet Architecture (Ledger-first design) | BUILD | P0 |
| 02 | RBI Compliance Configuration & PPI Type Engine | BUILD | P0 |
| 03 | User Identity & KYC Stack | REUSE | P1 |
| 04 | Money Movement Flows | REUSE | P1 |
| 05 | Banking & Escrow / Nodal Layer | BUILD | P0 |
| 06 | Risk, Fraud & Security Engine | REUSE | P1 |
| 07 | PPI Limits & Rules Engine | BUILD | P0 |
| 08 | Reconciliation & Settlement Engine | REUSE | P1 |
| 09 | Reporting & Compliance Module | BUILD | P1 |
| 10 | Merchant Ecosystem Integration | REUSE | P1 |
| 11 | UPI Integration & VPA Management (NEW BLOCK) ★ | BUILD | P0 |
| 12 | Wallet Lifecycle Management (NEW BLOCK) ★ | BUILD | P1 |
| 13 | Notification & Communication Engine (NEW BLOCK) ★ | REUSE | P1 |
| 14 | Customer Support & Dispute Management (NEW BLOCK) ★ | BUILD | P1 |
| 15 | Admin, Ops & Configuration Panel (NEW BLOCK) ★ | BUILD | P1 |
| 16 | AI / Differentiation Layer | BUY | P1 |
| 17 | Frontend Experiences | REUSE | P1 |
| 18 | Platform & Infrastructure | BUY | P1 |

| Section 2: Product Blocks — Detailed Requirements |
| --- |

| Block 01: Core Wallet Architecture (Ledger-first design) |  |
| --- | --- |
| Decision | BUILD — Net New \| P0 CRITICAL |
| What to Build | Double-entry Wallet Ledger, Balance Service (real-time), Transaction Orchestrator, Idempotency Layer, Event Sourcing backbone |
| PPSL Context | PPSL has no ledger today. TPAP/PCL owns the current wallet ledger. Must be built fresh under PPSL legal entity with its own general ledger (GL) entries for regulatory ring-fencing. |
| API Surface | POST /wallet/credit \| POST /wallet/debit \| GET /wallet/balance \| GET /wallet/ledger \| POST /wallet/hold \| POST /wallet/release |
| Data Models | wallet_accounts, ledger_entries (debit/credit/balance), transaction_intents, idempotency_keys |
| Technical Notes | Event-sourced ledger recommended (append-only). Balance is derived state. All mutations go through orchestrator. Idempotency key = {client_id}_{txn_ref}_{amount}_{ts_bucket}. |
| Compliance | RBI PPI Directions Clause 8 — master record of all e-money issuance. Audit log retention: 5 years. |

| Block 02: RBI Compliance Configuration & PPI Type Engine |  |
| --- | --- |
| Decision | BUILD — Net New \| P0 CRITICAL |
| What to Build | PPI type registry (Min-KYC / Full-KYC / Gift / Corporate), KYC-tiered limit enforcement, interoperability flag management, compliance rule versioning |
| PPSL Context | Rules for PA/PG limits cannot be shared. PPI limits are balance-level (not just per-transaction) and differ by wallet type. The engine must support hot-reload of limits without deployment (RBI changes rules periodically). |
| API Surface | GET /compliance/wallet-types \| GET /compliance/limits/{kyc_tier} \| POST /compliance/limits/override (admin) \| GET /compliance/interoperability-status/{wallet_id} |
| Data Models | ppi_types, compliance_rules, kyc_tiers, limit_override_log |
| Technical Notes | Store limit configs in DB with versioning + effective-from date. Never hardcode limits in application code. Admin override requires 4-eye approval in the audit trail. |
| Compliance | RBI PPI Master Directions 2021, Clause 9 (limits) and Clause 10 (interoperability). Full-KYC wallets must be interoperable via UPI from Day 1 of Full-KYC upgrade. |

| Block 03: User Identity & KYC Stack |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | User Profile Service, KYC Orchestrator (Aadhaar OTP / Aadhaar biometric / PAN / Video KYC), Risk Tiering Engine, KYC status state machine |
| PPSL Context | Paytm's Aadhaar + PAN KYC infrastructure can be reused. PPSL must be separately registered as a KUA (KYC User Agency) with UIDAI, or use an existing licensed intermediary. Video KYC needs a separate UIDAI tie-up for PPSL entity. |
| API Surface | POST /kyc/initiate \| POST /kyc/aadhaar/otp-verify \| POST /kyc/pan-verify \| POST /kyc/video-kyc/session \| GET /kyc/status/{user_id} \| GET /kyc/tier/{user_id} |
| Data Models | user_profiles, kyc_submissions, kyc_documents (encrypted), kyc_audit_trail, risk_tier_assignments |
| Technical Notes | KYC documents must be encrypted at rest (AES-256). PAN must be masked in logs. CKYC (Central KYC) check must be run before Video KYC. Retain KYC artefacts for 10 years per PMLA. |
| Compliance | RBI PPI Directions Clause 13-17 (customer due diligence). UIDAI circular on Aadhaar eKYC. PMLA Section 12 (record retention). |

| Block 04: Money Movement Flows |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | Add Money (UPI collect, UPI AutoPay, Cards, Netbanking), Wallet Spend (merchant QR, payment links, in-app pay), P2P transfer (wallet-to-wallet), Wallet-to-Bank withdrawal, Bill payment (BBPS), Refunds engine |
| PPSL Context | Paytm UPI / Card rails exist. However, wallet-specific flows need new orchestration: (a) P2P wallet-to-wallet has separate RBI limits from UPI P2P; (b) Wallet-to-bank withdrawal requires nodal debit flow; (c) Refunds must credit back to wallet, not source instrument. |
| API Surface | POST /money/add \| POST /money/spend \| POST /money/transfer-p2p \| POST /money/withdraw \| POST /money/bill-pay \| POST /refund/initiate \| GET /refund/status/{ref} |
| Data Models | money_movement_requests, payment_legs, refund_requests, refund_legs |
| Technical Notes | Every money movement must produce a ledger entry before external call is made (debit-first pattern). Refund idempotency is critical — same refund ref must never debit wallet twice. BBPS payments must use existing BBPS BBPOU status. |
| Compliance | RBI PPI Directions Clause 20-22 (permitted transactions). P2P monthly limit = ₹10,000 for Min-KYC, ₹1 lakh for Full-KYC. |

| Block 05: Banking & Escrow / Nodal Layer |  |
| --- | --- |
| Decision | BUILD — Net New \| P0 CRITICAL |
| What to Build | Escrow account API integration with partner bank, Nodal account management, Daily balance reconciliation (wallet system vs. bank), Float management, Bank statement ingestion, Real-time balance feeds |
| PPSL Context | Critical blocker: post-PPBL restructuring, PPSL cannot rely on Paytm Payments Bank for escrow. A new escrow/nodal tie-up with an RBI-scheduled commercial bank (e.g. YES Bank, Axis, ICICI) must be executed as a business priority before any tech build starts. |
| API Surface | POST /nodal/debit \| POST /nodal/credit \| GET /nodal/balance \| GET /nodal/statement/{date} \| POST /nodal/recon/trigger \| GET /nodal/recon/status/{date} |
| Data Models | nodal_accounts, nodal_transactions, recon_runs, recon_breaks, float_ledger |
| Technical Notes | Bank statement ingestion via SFTP (daily) or bank API (intraday). Float = sum(all active wallet balances) must always be <= nodal balance. Tolerance threshold alerts at 95% utilisation. |
| Compliance | RBI PPI Directions Clause 7 — all outstanding e-money must be held in escrow with a scheduled commercial bank. Daily recon is mandatory. |

| Block 06: Risk, Fraud & Security Engine |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | Real-time transaction risk scoring, Velocity checks (per wallet / per device / per BIN), Device fingerprinting, Behavioural analytics, OTP / 2FA authentication, Account takeover (ATO) detection |
| PPSL Context | PPSL / Paytm has MARS (fraud risk engine) and device intelligence. Extend existing velocity rules with wallet-specific patterns (e.g. rapid load + immediate P2P out, mule wallet rings). Wallet fraud patterns differ from card-present fraud. |
| API Surface | POST /risk/score \| POST /risk/challenge \| POST /risk/block-wallet \| GET /risk/events/{wallet_id} \| POST /auth/otp/send \| POST /auth/otp/verify |
| Data Models | risk_scores, risk_events, velocity_counters, blocked_entities, otp_requests |
| Technical Notes | Risk score must be synchronous in transaction path (p99 < 150ms). Async enrichment for post-transaction analysis. Velocity counters stored in Redis with TTL. Add wallet-specific ML features: load-to-spend ratio, new-payee velocity, night-time transaction rate. |
| Compliance | RBI PPI Directions Clause 24 (security) and Clause 27 (fraud monitoring). CERT-In guidelines for OTP delivery. |

| Block 07: PPI Limits & Rules Engine |  |
| --- | --- |
| Decision | BUILD — Net New \| P0 CRITICAL |
| What to Build | Balance cap enforcement (₹10,000 Min-KYC / ₹2,00,000 Full-KYC), Monthly load limit tracking, Monthly spend limit tracking, Monthly P2P limit tracking, Per-transaction limit checks, Admin override with audit trail |
| PPSL Context | PA limits engine tracks per-transaction limits for merchants. PPI limits are fundamentally different — they track cumulative monthly rolling balances across load/spend/P2P/withdrawal dimensions simultaneously. This is a net-new build. |
| API Surface | POST /limits/check (sync, pre-transaction) \| POST /limits/commit (post-auth) \| POST /limits/rollback \| GET /limits/usage/{wallet_id} \| PUT /limits/config (admin) |
| Data Models | limit_configs, limit_usage (rolling monthly), limit_check_log, limit_overrides |
| Technical Notes | Limits check must be atomic with ledger debit (use distributed lock or CRDT counter). Rolling month window = calendar month. On KYC upgrade from Min-KYC to Full-KYC, limits reset to new tier immediately. Never use eventually consistent counters for limit checks — use strong consistency. |
| Compliance | RBI PPI Directions Clause 9 — minimum KYC PPI up to ₹10,000, full-KYC up to ₹2,00,000. Non-compliance = mandatory PPI suspension. |

| Block 08: Reconciliation & Settlement Engine |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | Internal ledger vs. nodal bank reconciliation, PG-level reconciliation (UPI/card inflows), Merchant settlement (wallet-funded), Exception identification and resolution workflow, Auto-recon dashboard |
| PPSL Context | Gaurav's team owns the Settlement product at PPSL. Core recon engine can be adapted. Wallet recon needs nodal bank file reconciliation (daily bank statement vs. internal float ledger), which is distinct from PA MID-based merchant settlement. |
| API Surface | POST /recon/run/{date} \| GET /recon/report/{date} \| GET /recon/breaks/{date} \| POST /recon/break/resolve/{id} \| GET /settlement/merchant/{mid} |
| Data Models | recon_jobs, recon_matches, recon_breaks, settlement_batches, settlement_legs |
| Technical Notes | Recon runs at EOD (automated) and on-demand (triggered). Break resolution SLA = 24 hours for systemic breaks, 72 hours for manual. Recon report must be stored for 5 years per RBI direction. |
| Compliance | RBI PPI Directions Clause 7 — nodal account reconciliation mandatory. Float > outstanding wallets = violation. |

| Block 09: Reporting & Compliance Module |  |
| --- | --- |
| Decision | BUILD — Net New |
| What to Build | RBI periodic reports (monthly / quarterly PPI data), Suspicious Transaction Reports (STR) via FIU-IND portal, Cash Transaction Reports (CTR), AML/CFT dashboards, Compliance calendar and task management |
| PPSL Context | RBI mandates specific PPI reports distinct from PA/PG reports. PPSL must file directly with FIU-IND as a Reporting Entity under PMLA. STR filing is mandatory within 7 days of suspicion. This is a net-new compliance build. |
| API Surface | POST /reports/rbi/generate \| GET /reports/rbi/{period} \| POST /reports/str/file \| GET /reports/str/status/{id} \| GET /reports/aml/dashboard |
| Data Models | rbi_report_runs, str_filings, ctr_filings, compliance_calendar, alert_assignments |
| Technical Notes | RBI report formats are prescribed in PPI Directions Annex. FIU-IND uses FINNET 2.0 portal for STR filing. Automate data extraction from ledger; manual review before filing. Alert: any wallet with load > ₹50,000 in a month must be flagged for enhanced due diligence. |
| Compliance | PMLA 2002, Section 12 and 12A. RBI PPI Circular on periodic reporting. FIU-IND guidelines on STR/CTR filing. |

| Block 10: Merchant Ecosystem Integration |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | Wallet-acceptance flag on existing merchant profiles, QR code issuance (wallet-linked), Payment API for in-store and online wallet pay, Wallet MDR configuration, Merchant settlement from wallet transactions |
| PPSL Context | PPSL already has merchant onboarding and QR infrastructure for PA business. Key additions: (a) wallet acceptance toggle per merchant, (b) separate MDR pricing for wallet-funded transactions, (c) wallet-specific settlement track (nodal debit for merchant credit). |
| API Surface | PUT /merchant/{mid}/wallet-enable \| GET /merchant/{mid}/wallet-config \| POST /qr/issue/{mid} \| POST /payment/wallet/initiate \| GET /settlement/wallet/merchant/{mid} |
| Data Models | merchant_wallet_configs, wallet_payment_requests, wallet_settlement_batches |
| Technical Notes | Wallet MDR is often 0% for specific categories (as per RBI direction on MDR for PPIs). Ensure MDR config is separate from UPI MDR. QR codes must be interoperable (Bharat QR / UPI QR) for Full-KYC wallets. |
| Compliance | RBI PPI Directions Clause 21 — PPIs must be accepted at all merchant locations where QR/UPI is accepted (for interoperable Full-KYC wallets). |

| Block 11: UPI Integration & VPA Management (NEW BLOCK)  ★ NEW |  |
| --- | --- |
| Decision | BUILD — Net New \| P0 CRITICAL |
| What to Build | VPA (Virtual Payment Address) issuance under PPSL UPI handle (e.g. @paytmpay), UPI collect / pay flows linked to wallet balance, NPCI UPI certification for PPSL as TPAP or PSP, UPI AutoPay (mandate) for recurring wallet loads, UPI balance check (encrypted) |
| PPSL Context | This is a CRITICAL missing block. Full-KYC wallets must be UPI-interoperable per RBI mandate. PPSL currently doesn't have its own UPI handle / TPAP license. Options: (a) apply for PPSL TPAP license from NPCI, or (b) partner with an existing PSP bank to sponsor PPSL as a Third-Party App. This is a multi-month regulatory process and the longest lead-time item after nodal bank. |
| API Surface | POST /upi/vpa/create \| GET /upi/vpa/{wallet_id} \| POST /upi/collect \| POST /upi/pay \| POST /upi/mandate/create \| GET /upi/mandate/status/{id} \| POST /upi/mandate/revoke |
| Data Models | vpa_registry, upi_transactions, upi_mandates, upi_dispute_log |
| Technical Notes | VPA format: {mobile_last4}@paytmpay or {handle}@paytmpay. UPI certification requires NPCI audit (functional + security). UPI AutoPay is the primary mechanism for recurring wallet load. Ensure UPI dispute management is wired into the wallet refund flow. |
| Compliance | NPCI UPI Procedural Guidelines. RBI PPI Directions Clause 10 — Full-KYC PPIs must be interoperable. NPCI circular on TPAP eligibility. |

| Block 12: Wallet Lifecycle Management (NEW BLOCK)  ★ NEW |  |
| --- | --- |
| Decision | BUILD — Net New |
| What to Build | Wallet creation (with KYC linkage), KYC upgrade flow (Min-KYC to Full-KYC), Wallet suspension (auto on risk trigger / manual by ops), Wallet closure (regulatory and user-initiated), Dormancy management, Balance expiry handling (for gift wallets), Wallet merge / split |
| PPSL Context | A wallet is not just an account — it goes through a lifecycle. The document has no block for this. Ops teams regularly deal with suspended / dormant / closing wallets. Dormant wallets with unclaimed balances must be handled per RBI unclaimed funds direction. |
| API Surface | POST /wallet/create \| POST /wallet/upgrade-kyc \| POST /wallet/suspend \| POST /wallet/reactivate \| POST /wallet/close \| GET /wallet/status/{id} \| GET /wallet/lifecycle-history/{id} |
| Data Models | wallet_lifecycle_events, wallet_status_log, dormancy_registry, unclaimed_balance_log |
| Technical Notes | Wallet status state machine: CREATED → ACTIVE → KYC_PENDING → SUSPENDED → CLOSED. On closure, remaining balance must be returned to last linked bank account (if verified). Dormancy = no transaction for 1 year. Unclaimed balances > 3 years — process per RBI unclaimed deposits guidelines. |
| Compliance | RBI PPI Directions Clause 18 (closure) and Clause 19 (dormancy). RBI circular on unclaimed deposits / outstanding balances. |

| Block 13: Notification & Communication Engine (NEW BLOCK)  ★ NEW |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | Transaction confirmation SMS/push, OTP delivery (SMS / WhatsApp), Limit breach alerts, KYC expiry reminders, Suspicious activity alerts to customer, Regulatory communication (RBI-mandated alerts), Email statements |
| PPSL Context | Paytm's CPaaS / ConnectPlus (which Gaurav's team owns) is the natural reuse candidate here. Wallet notifications are a distinct use-case category with RBI-mandated delivery requirements (e.g. every transaction must trigger an alert). This block was entirely missing from the original document. |
| API Surface | POST /notify/transaction \| POST /notify/otp \| POST /notify/alert \| POST /notify/statement \| GET /notify/preferences/{user_id} \| PUT /notify/preferences/{user_id} |
| Data Models | notification_requests, notification_delivery_log, notification_preferences, template_registry |
| Technical Notes | Transaction alerts are mandatory per RBI (not optional). Delivery SLA = < 30 seconds for OTP, < 2 minutes for transaction alerts. Use CPaaS for SMS/WhatsApp. Push notifications via FCM/APNs. All notification content must be pre-approved (no dynamic PII in message body beyond masked account/amount). |
| Compliance | RBI PPI Directions Clause 24 — transaction alerts mandatory. TRAI DLT registration for transactional SMS. DPDP Act 2023 — customer consent for marketing communications. |

| Block 14: Customer Support & Dispute Management (NEW BLOCK)  ★ NEW |  |
| --- | --- |
| Decision | BUILD — Net New |
| What to Build | Failed transaction resolution workflow, Chargeback initiation and tracking, P2P dispute (wrong transfer), Escalation matrix (L1/L2/L3), Ops dashboard for support agents, SLA tracking, Regulatory grievance redressal (as per RBI) |
| PPSL Context | RBI mandates a grievance redressal mechanism for all PPI issuers. Response within 30 days, escalation to RBI Ombudsman if unresolved. This block was missing from the original document and is a regulatory mandatory — not optional. |
| API Surface | POST /dispute/raise \| GET /dispute/status/{id} \| POST /dispute/resolve \| POST /dispute/escalate \| GET /dispute/list/{wallet_id} \| GET /ops/dispute/queue |
| Data Models | disputes, dispute_events, dispute_assignments, sla_log, rbi_ombudsman_cases |
| Technical Notes | Each dispute must have an auto-generated ticket ID. SLA clock starts on submission. Failed transaction disputes must check nodal bank credit status before resolving. Integration with CPaaS for automated status updates to customer. |
| Compliance | RBI PPI Directions Clause 26 — grievance redressal mandatory. RBI Integrated Ombudsman Scheme 2021. Response time SLA = 30 days. |

| Block 15: Admin, Ops & Configuration Panel (NEW BLOCK)  ★ NEW |  |
| --- | --- |
| Decision | BUILD — Net New |
| What to Build | Wallet search and view (ops agents), Manual wallet suspension / reactivation, KYC override with reason logging, Limit override with 4-eye approval, Compliance task management, System-wide kill switch (circuit breaker), Feature flag management, Role-based access control (RBAC) |
| PPSL Context | An admin panel was mentioned in the original document under 'Frontend' but it needs to be a standalone product block with its own API surface and RBAC. Ops team will use this daily. This is always underestimated in wallet builds. |
| API Surface | GET /admin/wallet/search \| GET /admin/wallet/{id} \| POST /admin/wallet/suspend \| POST /admin/kyc/override \| POST /admin/limits/override \| POST /admin/killswitch \| GET /admin/audit-log |
| Data Models | admin_users, admin_roles, admin_permissions, admin_actions_log, feature_flags, killswitch_registry |
| Technical Notes | Every admin action must be logged with actor, timestamp, reason, and before/after state. No bulk mutation without approval flow. RBAC: Ops Agent (view + suspend), Compliance Officer (KYC override), Finance (limit override), System Admin (killswitch). MFA mandatory for all admin users. |
| Compliance | RBI PPI Directions Clause 25 — internal controls. CERT-In guidelines on privileged access management. |

| Block 16: AI / Differentiation Layer |  |
| --- | --- |
| Decision | BUY / Integrate |
| What to Build | Smart spend insights (category-wise analytics), ML-based fraud detection (wallet-specific models), Conversational wallet (GenAI, Claude/GPT APIs), Personalised offers engine, Spend forecasting |
| PPSL Context | Phase 4 play — do not build this until Phases 1-3 are live. Paytm AI team can build spend insights. Conversational wallet via Anthropic Claude API. Fraud ML available partially via MARS — extend with wallet-specific features. |
| API Surface | GET /ai/insights/{wallet_id} \| POST /ai/chat \| GET /ai/offers/{wallet_id} |
| Data Models | spend_categories, insight_events, offer_assignments, chat_sessions |
| Technical Notes | Phase 4 only. Build only after 6 months of wallet transaction data. Conversational wallet: integrate Claude claude-sonnet-4-6 via Anthropic API. Fraud ML: add wallet-specific features (load velocity, P2P out-ratio, new payee rate) to MARS feature store. |
| Compliance | DPDP Act 2023 — user consent required before using transaction data for insights/offers. Option to opt-out must be surfaced. |

| Block 17: Frontend Experiences |  |
| --- | --- |
| Decision | REUSE — Adapt from Paytm |
| What to Build | Consumer wallet module within Paytm app (iOS + Android), Wallet balance widget, Add money flow, Pay flow (QR + P2P), Transaction history, KYC upgrade journey, Settings & preferences, Merchant wallet dashboard extension |
| PPSL Context | Consumer wallet UI = new module within existing Paytm app. Merchant dashboard = extend existing PPSL merchant portal. Keep UI separate from PPBL wallet (which is being wound down) to avoid confusion. Accessibility (WCAG 2.1 AA) required. |
| API Surface | All APIs served via BFF (Backend for Frontend) — separate BFF for consumer app vs. merchant portal vs. admin panel. |
| Data Models | user_sessions, app_preferences, ui_events (analytics) |
| Technical Notes | BFF pattern: one BFF per client type. Mobile app: React Native (reuse existing Paytm app architecture). Wallet onboarding flow must have <3 taps to first transaction. Deep-link support for payment requests. Biometric auth (fingerprint / face) for high-value transactions. |
| Compliance | RBI PPI Directions Clause 24 — transaction history must be viewable by customer at all times. DPDP Act 2023 — privacy policy and consent UI. |

| Block 18: Platform & Infrastructure |  |
| --- | --- |
| Decision | BUY / Integrate |
| What to Build | Microservices architecture (wallet services as new pods), Event-driven backbone (Kafka), High-availability (multi-AZ), Data isolation (PPSL wallet data ring-fenced from TPAP/PCL), API gateway, Service mesh, Secrets management, Disaster recovery |
| PPSL Context | Paytm runs on a mature cloud-native stack (AWS/GCP). Wallet services = new microservices deployed on the same infra. Key requirement: data isolation between PPSL wallet data and TPAP/PCL data for regulatory ring-fencing. Separate DB schemas / clusters recommended. |
| API Surface | Internal only — API gateway, service mesh, event bus |
| Data Models | Separate PostgreSQL cluster (or schema) for PPSL wallet. Separate Kafka topics prefixed wallet.*. Redis cluster for velocity counters and sessions. |
| Technical Notes | Recommended stack: Node.js / Go for wallet microservices (fast iteration with Claude/Cursor). PostgreSQL (ACID guarantees for ledger). Redis (velocity + session). Kafka (event bus). Kong / AWS API Gateway (API management). Vault (secrets). Target: 99.99% uptime SLA. |
| Compliance | RBI data localisation — all wallet data must reside in India. CERT-In cloud security guidelines. |

# 3. Escrow / Nodal Partner Bank — Evaluation Criteria

Before any wallet development begins, PPSL must secure a nodal/escrow bank tie-up. This is the single most critical business dependency — without it, the wallet cannot hold any customer balances. The following criteria must be reviewed during bank evaluation.

### 3.1  Regulatory & Licensing

- Confirm bank is an RBI-scheduled commercial bank (mandatory for PPI escrow per RBI PPI Directions Clause 7)
- Verify the bank has experience holding PPI / prepaid instrument escrow — ask for reference clients
- Confirm the bank can open a dedicated nodal/escrow account under PPSL entity (not Paytm Payments Bank Ltd)
- Review the bank's own RBI compliance track record — any pending directions or penalties?
- Confirm the escrow agreement structure complies with RBI's prescribed format for PPI issuers

### 3.2  API & Technical Integration

- Availability of a real-time balance API (intraday, not just EOD) — essential for float monitoring
- Inward/outward transaction webhooks or push notifications (not just pull/SFTP)
- Daily bank statement in machine-readable format (MT940 / ISO 20022 / custom API) — no PDF-only options
- API uptime SLA: minimum 99.9% for nodal account APIs (settlement is time-critical)
- Sandbox / UAT environment available for integration testing before go-live
- Support for bulk credits (merchant settlement payouts from nodal account)
- Rate limits on API calls — confirm they support PPSL's expected transaction volumes
- Support for UPI inward credits into nodal account (critical for Add Money via UPI collect)

### 3.3  Operational & Settlement

- Cut-off times for same-day NEFT/RTGS/IMPS credits from nodal account
- Can the bank support multiple sub-accounts or virtual accounts under one nodal account? (useful for ring-fencing per wallet type)
- Dedicated relationship manager / technical support team with <2 hour response SLA
- Escalation matrix for failed transactions or reconciliation breaks
- Experience with high-volume, low-value transaction accounts (PPI accounts have high transaction counts)
- Ability to generate TDS certificates on interest earned in nodal account

### 3.4  Commercial & Contractual

- Fee structure: account maintenance, per-transaction charges, API call fees, statement fees
- Interest rate on nodal account float (this is material at scale — ₹100Cr float @ 4% = ₹4Cr/yr)
- Exclusivity clause: can PPSL hold nodal accounts with multiple banks simultaneously? (redundancy)
- Termination clause: what is the exit process, and what happens to outstanding float during transition?
- Indemnification: bank's liability in case of failed credits or system outages causing settlement delays
- Review SLA for dispute resolution on nodal account transactions

### 3.5  Security & Compliance

- Bank's ISO 27001 / SOC 2 certification status
- Two-factor authentication and IP whitelisting for API access to nodal account
- Data residency: all nodal account data must be stored in India
- AML / KYB: bank must conduct its own KYB on PPSL before opening the nodal account
- Confirm the bank will provide audit support during RBI inspections of PPSL's PPI operations

### 3.6  Recommended Shortlist Approach

Run parallel RFP with at least two banks simultaneously. Evaluate on: (a) API maturity and sandbox availability, (b) interest rate on nodal float, (c) support for virtual sub-accounts, (d) prior PPI escrow experience. Sign with primary bank and maintain a secondary bank relationship for redundancy.

# 4. Step-by-Step Development Plan

**Wallet Load Guard — RBI Compliance Validation**

Wallet Load Guard — RBI PPI Limit Validation

The Wallet Load Guard is a pre-transaction validation layer that enforces RBI PPI limits before any wallet load (add money) transaction is processed. It runs 3 compliance checks and generates Claude AI explanations when a transaction is blocked.

Three RBI Rules Validated:

1. BALANCE_CAP — Maximum wallet balance of ₹1,00,000 for all users

2. MONTHLY_LOAD — Maximum ₹2,00,000 total loads per calendar month

3. MIN_KYC_CAP — Maximum ₹10,000 balance for Minimum KYC users

Architecture:

- Backend: mcp/services/wallet-load-guard.js — validation + Claude Haiku messages

- API: POST /api/wallet/validate-load, GET /api/wallet/load-guard-log

- UI: Add Money page with live helper text, block alerts, "Add max instead" button

- Admin: Load Guard Log panel on Dashboard (last 10 blocked attempts)

Most restrictive rule wins when multiple rules fail.

The development plan is structured for maximum parallel execution. Multiple tracks run simultaneously within each phase. API contracts (OpenAPI specs) for all 18 blocks should be the first output — they serve as the precise input for Claude/Cursor-assisted development and eliminate ambiguity for the engineering team.

| Phase 0: Phase 0 — Pre-conditions & Design  —  Resolve all regulatory and architectural blockers before a single line of code is written |
| --- |

Regulatory (Business / Legal)

- Confirm PPSL holds a valid PPI license from RBI (or initiate application — 3-6 months lead time)
- Initiate escrow/nodal bank evaluation using the bank review criteria in Section 3 — shortlist 2 banks, negotiate in parallel
- Initiate NPCI engagement for PPSL UPI TPAP license or PSP sponsorship — longest regulatory lead time
- Engage UIDAI / licensed KUA for Aadhaar eKYC access under PPSL entity
- Register PPSL as Reporting Entity with FIU-IND (for STR/CTR filing under PMLA)
- Legal review of escrow agreement, KYC data sharing agreements, and NPCI participation agreements

Architecture & Design

- Define data isolation architecture: separate DB cluster/schema for PPSL wallet vs TPAP/PCL
- Design wallet ledger schema (double-entry): accounts, entries, transaction_intents, idempotency_keys
- Define API contract for all 18 blocks (OpenAPI 3.0 specs) — this is the input for Claude/Cursor coding
- Define event schema for Kafka topics (wallet.created, wallet.credited, wallet.debited, kyc.updated, etc.)
- Design RBAC model for admin panel: roles, permissions, approval workflows
- Set up dev, staging, and production environments — separate from TPAP/PCL infra
- Establish coding standards, PR review process, and automated testing requirements

| Phase 1: Phase 1 — Core Foundation (Parallel Tracks)  —  Build the irreducible minimum: ledger, KYC, limits, and nodal integration. All tracks run in parallel. |
| --- |

**All tracks in this phase run in parallel.**

Track A — Wallet Ledger & Architecture (Block 01)

- Sprint 1: Set up DB schema — wallet_accounts, ledger_entries, transaction_intents, idempotency_keys
- Sprint 1: Build Wallet Ledger service — POST /wallet/credit, /debit with double-entry enforcement
- Sprint 2: Build Balance Service — real-time balance derived from ledger, with Redis cache
- Sprint 2: Build Transaction Orchestrator — saga pattern for multi-leg transactions
- Sprint 3: Idempotency layer — idempotency key validation, duplicate detection, response caching
- Sprint 3: Event sourcing — publish wallet.credited / wallet.debited events to Kafka
- Sprint 4: Load testing — 1000 TPS, verify ledger consistency under concurrent writes
- Sprint 4: Ledger audit — verify double-entry integrity, zero-sum checks

Track B — KYC Stack (Block 03)

- Sprint 1: User Profile Service — create, update, get user profile (reuse / adapt from existing Paytm)
- Sprint 2: KYC Orchestrator — state machine: PENDING → AADHAAR_VERIFIED → PAN_VERIFIED → KYC_COMPLETE
- Sprint 2: Aadhaar OTP eKYC integration (via UIDAI / KUA) — test with UIDAI sandbox
- Sprint 3: PAN verification API integration (ITD NSDL / Karza)
- Sprint 3: Risk Tiering Engine — assign Min-KYC / Full-KYC tier based on KYC status
- Sprint 4: Video KYC flow — integrate Video KYC vendor (IDfy / IDFC / others) — async review workflow
- Sprint 4: KYC document encryption at rest, masking in logs, audit trail

Track C — Limits Engine (Block 07)

- Sprint 1: Limits config schema — ppi_types, limit_configs, limit_usage (rolling monthly)
- Sprint 1: Seed limit configs for Min-KYC (₹10K) and Full-KYC (₹2L) — store in DB, not code
- Sprint 2: Limits check API — POST /limits/check — synchronous, <50ms p99
- Sprint 2: Limits commit / rollback API — atomic with ledger debit (distributed lock)
- Sprint 3: Rolling monthly usage tracking — calendar month window, Redis counters
- Sprint 3: KYC upgrade limit reset — on tier change, reset to new tier immediately
- Sprint 4: Admin override API — POST /limits/config with 4-eye approval workflow
- Sprint 4: Limits breach alerting — webhook to risk engine on near-limit

Track D — Nodal/Escrow Bank Integration (Block 05)

- Sprint 1: Nodal bank API integration setup — credentials, IP whitelisting, sandbox environment
- Sprint 2: Inward credit listener — webhook / polling for UPI / NEFT / RTGS inflows into nodal
- Sprint 2: Outward debit (bank transfer) API — for withdrawal and merchant settlement
- Sprint 3: Bank statement ingestion — SFTP / API pull, parse MT940 or bank-specific format
- Sprint 3: Float ledger — internal record of expected float vs actual nodal balance
- Sprint 4: Daily reconciliation job — match internal float ledger vs bank statement
- Sprint 4: Float threshold alerting — alert at 90% utilisation, block Add Money at 98%

| Phase 2: Phase 2 — Money Movement & Security (Parallel Tracks)  —  Enable all money flows: Add Money, Spend, P2P, Withdrawal. Wire in fraud/risk. Enable UPI. |
| --- |

**All tracks in this phase run in parallel.**

Track A — Money Movement Flows (Block 04)

- Sprint 1: Add Money via UPI Collect — initiate UPI collect request, listen for credit, credit wallet on nodal confirmation
- Sprint 2: Add Money via Cards and Netbanking — integrate existing PPSL card/NB PG, credit wallet on PG success
- Sprint 2: Wallet Spend — merchant QR pay, debit wallet, post to nodal for merchant settlement
- Sprint 3: P2P Transfer — wallet-to-wallet, check limits, debit sender, credit receiver atomically
- Sprint 3: Wallet-to-Bank Withdrawal — debit wallet, initiate IMPS/NEFT from nodal, update on confirmation
- Sprint 4: Bill Payment (BBPS) — use PPSL BBPOU status, debit wallet, initiate BBPS payment
- Sprint 4: Refunds Engine — credit wallet on refund trigger, handle source-instrument-vs-wallet routing

Track B — Risk & Fraud Engine (Block 06)

- Sprint 1: Integrate MARS risk scoring into wallet transaction path — sync call in orchestrator
- Sprint 2: Wallet-specific velocity rules — load-to-spend ratio, new-payee velocity, night-time txn rate
- Sprint 2: Velocity counters in Redis — per wallet, per device, per BIN
- Sprint 3: OTP / 2FA service — OTP generation, delivery via CPaaS, verification
- Sprint 3: ATO (account takeover) detection — device change + high-value txn triggers enhanced auth
- Sprint 4: Risk event logging — write all risk decisions to risk_events table
- Sprint 4: Risk dashboard — view wallet risk events, block/unblock wallets from ops panel

Track C — UPI Integration (Block 11)

- Sprint 1: NPCI UPI certification process — functional test cases, security audit (run in parallel with dev)
- Sprint 2: VPA creation service — POST /upi/vpa/create, VPA format, uniqueness validation
- Sprint 2: UPI Pay (outgoing) — link wallet balance as funding source for UPI pay
- Sprint 3: UPI Collect (incoming to wallet) — register wallet UPI ID for collect requests
- Sprint 3: UPI AutoPay mandate — create/manage e-mandate for recurring wallet loads
- Sprint 4: UPI dispute management — integrate UPI chargeback flow with wallet refund engine
- Sprint 4: NPCI certification — submit test results, resolve audit findings

Track D — Notification Engine (Block 13)

- Sprint 1: Notification service setup — integrate with CPaaS/ConnectPlus APIs for SMS and push
- Sprint 2: Transaction confirmation notifications — every debit/credit triggers SMS + push within 2 min
- Sprint 2: OTP delivery — SMS + WhatsApp channel, delivery confirmation tracking
- Sprint 3: Alert templates — pre-approved templates for all wallet event types (DLT-registered)
- Sprint 3: Notification preferences — user opt-in/out for non-mandatory communications
- Sprint 4: Email statements — monthly wallet statement PDF generation and email delivery

| Phase 3: Phase 3 — Operations, Compliance & Merchant  —  Make PPSL operationally ready: admin tools, full recon, regulatory reporting, dispute management, merchant ecosystem. |
| --- |

**All tracks in this phase run in parallel.**

Track A — Admin & Ops Panel (Block 15)

- Sprint 1: Admin user management — RBAC schema, roles (Ops Agent, Compliance, Finance, System Admin)
- Sprint 2: Wallet search + view — search by mobile, wallet ID, name; view balance, lifecycle, transactions
- Sprint 2: Wallet suspension / reactivation — with reason logging and customer notification trigger
- Sprint 3: KYC override UI — ops agent can manually mark KYC verified with reason (audit logged)
- Sprint 3: Limits override — Finance role, dual approval, time-limited override
- Sprint 4: Killswitch / feature flags — per-feature enable/disable without deployment
- Sprint 4: Audit log viewer — complete log of all admin actions, exportable for RBI inspections

Track B — Reconciliation & Reporting (Blocks 08 & 09)

- Sprint 1: Reconciliation engine — match internal ledger transactions vs nodal bank statement (daily job)
- Sprint 2: Recon break identification — auto-flag mismatches, assign to ops queue
- Sprint 2: Recon dashboard — visual summary of breaks, resolution status, aging
- Sprint 3: RBI PPI periodic report — auto-generate monthly/quarterly report from ledger data
- Sprint 3: AML transaction monitoring — flag transactions matching STR criteria
- Sprint 4: STR filing workflow — ops review flagged transactions, approve and file via FIU-IND FINNET 2.0
- Sprint 4: Compliance calendar — task management for all recurring RBI/FIU-IND deadlines

Track C — Dispute Management (Block 14)

- Sprint 1: Dispute schema — disputes, dispute_events, assignments, SLA_log
- Sprint 2: Customer dispute raise — POST /dispute/raise from consumer app
- Sprint 2: Dispute auto-routing — failed txn disputes routed to settlement team, fraud disputes to risk team
- Sprint 3: Ops dispute queue — view, assign, update, resolve disputes with audit trail
- Sprint 3: SLA tracking — auto-escalate disputes approaching 30-day limit
- Sprint 4: RBI Ombudsman case management — escalated disputes, response tracking
- Sprint 4: Customer status updates — automated notifications on dispute status change via CPaaS

Track D — Merchant Ecosystem & Wallet Lifecycle (Blocks 10 & 12)

- Sprint 1: Wallet lifecycle state machine — CREATED, ACTIVE, KYC_PENDING, SUSPENDED, CLOSED
- Sprint 2: Wallet closure flow — ops/customer-initiated, remaining balance transfer, status update
- Sprint 2: Dormancy management — auto-flag wallets with no transaction for 12 months
- Sprint 3: Merchant wallet-acceptance enablement — toggle per merchant, wallet MDR config
- Sprint 3: Wallet QR codes — issue Bharat QR / UPI QR linked to wallet VPA for Full-KYC wallets
- Sprint 4: Wallet merchant settlement track — separate settlement batch for wallet-funded transactions
- Sprint 4: KYC upgrade journey — in-app flow for Min-KYC to Full-KYC upgrade (limit increase)

| Phase 4: Phase 4 — Consumer App & Integration  —  Integrate all wallet functionality into Paytm consumer app. Internal testing, beta, and go-live. |
| --- |

Consumer App Integration & Go-Live

- Sprint 1: BFF (Backend for Frontend) setup — separate BFF for consumer app, merchant portal, admin panel
- Sprint 2: Consumer wallet module — balance display, add money, pay (QR + P2P), transaction history
- Sprint 2: KYC upgrade journey in-app — Min-KYC to Full-KYC, Aadhaar + Video KYC flow
- Sprint 3: Settings & preferences — wallet PIN, biometric auth, notification preferences
- Sprint 3: End-to-end testing — all 18 blocks, all money flows, all compliance scenarios
- Sprint 4: Security audit (VAPT) — mandatory before go-live, fix all critical/high findings
- Sprint 4: RBI / compliance sign-off — internal compliance review, confirm all RBI directions met
- Sprint 5: Soft launch (internal beta) — 500 employee wallets, full monitoring
- Sprint 5: Go-live — phased rollout: Min-KYC first (simpler), Full-KYC in subsequent wave

# 5. API Design Principles for Engineering Team

The following principles must be followed when writing API specs and code for this wallet. These are the exact requirements for Claude/Cursor-assisted development.

### 5.1  Core Rules

- All APIs are REST, JSON, over HTTPS. No exceptions.
- Every POST request requires an Idempotency-Key header. Duplicate requests with same key must return the same response, never double-process.
- All monetary amounts are in paise (integer). Never use float for money. ₹10 = 1000 paise.
- All timestamps are ISO 8601 UTC. Store in UTC, convert to IST only at display layer.
- Every API response includes: { status, data, error_code, error_message, trace_id, timestamp }
- HTTP status codes: 200 (success), 400 (bad request), 422 (validation error), 429 (rate limit), 500 (server error). Never return 200 for errors.
- All wallet mutation APIs (credit, debit, transfer) require: wallet_id, amount (paise), currency (INR), idempotency_key, initiator_type, initiator_id.

### 5.2  Security Requirements

- All APIs must validate JWT token (user) or service-to-service mTLS certificate.
- PAN, Aadhaar, bank account numbers must never appear in API logs — mask before logging.
- Rate limiting: 100 requests/minute per wallet for read APIs, 10 requests/minute for mutation APIs.
- All admin APIs require MFA + RBAC role check. Log every call with actor and reason.

### 5.3  Ledger Contract (Critical)

- A ledger entry is only created when a transaction is in CONFIRMED state — never on PENDING.
- Ledger entries are immutable. Never UPDATE or DELETE ledger rows. Corrections via reversal entries only.
- Balance is always derived: SELECT SUM(credits) - SUM(debits) FROM ledger_entries WHERE wallet_id = ?
- Every ledger entry must have: wallet_id, entry_type (CREDIT/DEBIT), amount_paise, currency, reference_id, transaction_type, created_at.

### 5.4  Event Schema (Kafka)

- Topic naming: wallet.{entity}.{action} — e.g., wallet.account.created, wallet.ledger.credited
- Every event: { event_id, event_type, wallet_id, payload, metadata: { trace_id, timestamp, source_service } }
- Events are at-least-once. Consumers must be idempotent. Use event_id for deduplication.
