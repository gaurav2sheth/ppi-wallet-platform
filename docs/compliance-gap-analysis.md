# PPI Wallet Platform - Compliance Gap Analysis

**Document Version:** 2.0
**Date:** 2026-04-11
**Regulatory Framework:** RBI Master Directions on Prepaid Payment Instruments (PPIs) - Updated 2024
**Scope:** End-to-end compliance assessment of the PPI Wallet Platform across 12 regulatory categories

---

## Table of Contents

1. [Licensing & Authorization](#1-licensing--authorization)
2. [KYC / CDD (Customer Due Diligence)](#2-kyc--cdd-customer-due-diligence)
3. [Transaction Limits](#3-transaction-limits)
4. [Escrow & Settlement](#4-escrow--settlement)
5. [Interoperability](#5-interoperability)
6. [AML/CFT (Anti-Money Laundering / Counter Financing of Terrorism)](#6-amlcft-anti-money-laundering--counter-financing-of-terrorism)
7. [Security & Data Protection](#7-security--data-protection)
8. [Reporting & Audit](#8-reporting--audit)
9. [Wallet Lifecycle Management](#9-wallet-lifecycle-management)
10. [Merchant Onboarding & Compliance](#10-merchant-onboarding--compliance)
11. [Dispute Resolution & Grievance Redressal](#11-dispute-resolution--grievance-redressal)
12. [NPCI / UPI Integration](#12-npci--upi-integration)

---

## 1. Licensing & Authorization

### RBI References
- RBI Master Direction on PPIs (MD-PPIs), Chapter II - Authorization
- Payment and Settlement Systems Act, 2007 (PSS Act), Section 7
- RBI Circular on PPI License Renewal Requirements

### Current State
The platform is designed as an RBI-regulated PPI Wallet system. Architecture assumes valid PPI authorization is in place to operate as a prepaid instrument issuer.

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 1.1 | PPI Authorization Certificate Display | Medium | The platform does not display the RBI authorization certificate number or validity date in the app UI or admin dashboard. RBI mandates that PPI issuers display their Certificate of Authorization (CoA) details prominently. |
| 1.2 | License Renewal Tracking | High | No automated tracking or alerting system for license renewal timelines. The admin dashboard lacks a compliance calendar for CoA renewal deadlines. |
| 1.3 | Authorized Activities Scope | Medium | The sub-wallet system (Food, NCMC, FASTag, Gift, Fuel) introduces corporate benefit wallets. Each sub-wallet type must be validated against the authorized activities specified in the CoA. No mapping exists between sub-wallet types and authorized instrument categories. |
| 1.4 | Net-Worth Requirement Monitoring | Low | RBI requires PPI issuers to maintain minimum net-worth (currently Rs 5 crore for bank PPIs, Rs 15 crore for non-bank PPIs). No dashboard widget or alert tracks net-worth compliance. |

### Action Items
- [ ] Add RBI CoA details (certificate number, issue date, validity) to app Profile page and admin Settings page
- [ ] Implement license renewal calendar with 90-day, 60-day, and 30-day advance alerts in admin dashboard
- [ ] Create a mapping document for each sub-wallet type against CoA-authorized activities
- [ ] Add net-worth compliance tracker with quarterly review alerts

---

## 2. KYC / CDD (Customer Due Diligence)

### RBI References
- RBI Master Direction on KYC, 2016 (updated 2023)
- RBI MD-PPIs, Chapter IV - KYC Requirements
- PMLA (Prevention of Money Laundering Act), Section 12
- RBI Circular on Video-based Customer Identification Process (V-CIP)

### Current State
The platform implements a two-tier KYC model:

| Rule | Min-KYC | Full-KYC |
|------|---------|----------|
| Balance cap | Rs 10,000 | Rs 2,00,000 |
| Monthly load cap | Rs 10,000 | Rs 1,00,000 |
| Annual load cap | Rs 1,20,000 | -- |
| P2P transfer | PROHIBITED | Rs 1,00,000/month |
| Wallet validity | 12 months (then EXPIRED) | Unlimited |
| Cash withdrawal | Not allowed | Rs 2,000/txn, Rs 10,000/month |

**KYC State Machine:**
```
UNVERIFIED -> MIN_KYC (mobile OTP)
MIN_KYC -> FULL_KYC_PENDING (Aadhaar OTP submitted)
FULL_KYC_PENDING -> FULL_KYC (approved)
FULL_KYC_PENDING -> REJECTED (denied)
Any -> SUSPENDED (fraud/compliance)
MIN_KYC -> EXPIRED (12-month validity lapsed, 60-day grace)
```

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 2.1 | V-CIP Support | High | The platform only supports Aadhaar OTP-based verification for Full KYC. RBI also recognizes Video-based Customer Identification Process (V-CIP) as a valid KYC method. No V-CIP workflow exists. |
| 2.2 | CKYC (Central KYC) Registry Integration | High | RBI mandates upload of KYC records to the Central KYC Records Registry (CKYCR). No integration with CKYC registry for uploading or downloading KYC records. |
| 2.3 | Periodic KYC Refresh | Medium | Full-KYC customers require periodic KYC refresh (risk-based: 2 years for high-risk, 5 years for medium, 10 years for low-risk). No automated refresh scheduling or reminder system exists. |
| 2.4 | PEP/Sanctions Screening | High | No screening against Politically Exposed Persons (PEP) lists, UN sanctions lists, or OFAC/EU sanctions databases during onboarding or on an ongoing basis. |
| 2.5 | Aadhaar Consent & Audit Trail | Medium | Aadhaar OTP-based KYC requires explicit customer consent with audit trail per UIDAI regulations. The mock implementation does not capture or store consent artifacts. |
| 2.6 | KYC Rejection Reason Communication | Medium | When KYC is rejected (FULL_KYC_PENDING -> REJECTED), the platform does not mandate a reason code or communicate the specific rejection reason to the customer per RBI fair practice guidelines. |
| 2.7 | DigiLocker Integration | Low | RBI encourages acceptance of DigiLocker documents for KYC. No DigiLocker integration pathway exists. |
| 2.8 | Re-KYC for Dormant Accounts | Medium | Wallets transitioning from DORMANT back to ACTIVE should undergo re-KYC verification. No re-KYC trigger exists on dormant wallet reactivation. |

### Action Items
- [ ] Implement V-CIP workflow with video recording, liveness detection, and agent-assisted verification
- [ ] Integrate with CKYC Registry API for uploading KYC records (within 10 days of completion) and downloading existing records
- [ ] Build periodic KYC refresh scheduler based on customer risk categorization
- [ ] Integrate PEP/sanctions screening (e.g., Dow Jones, Refinitiv World-Check) at onboarding and on periodic batch basis
- [ ] Add explicit Aadhaar consent capture screen with timestamp, consent text, and audit log storage
- [ ] Implement rejection reason codes and customer notification flow for KYC rejections
- [ ] Evaluate DigiLocker API integration for document verification
- [ ] Add re-KYC trigger in the wallet lifecycle state machine for DORMANT -> ACTIVE transitions

---

## 3. Transaction Limits

### RBI References
- RBI MD-PPIs, Chapter V - Limits on PPIs
- RBI Circular on Enhancement of PPI Transaction Limits (2022)
- RBI Circular on Cash Withdrawal from Full-KYC PPIs

### Current State
The platform enforces the following via the Wallet Load Guard (3 rules):

1. **BALANCE_CAP** - Total wallet balance (main + all sub-wallets) must not exceed Rs 1,00,000
2. **MONTHLY_LOAD** - Calendar month total loads must not exceed Rs 2,00,000
3. **MIN_KYC_CAP** - Min-KYC wallets: total balance must not exceed Rs 10,000

Additional limits:
- Min-KYC monthly load: Rs 10,000
- Min-KYC annual load: Rs 1,20,000
- Full-KYC monthly load: Rs 1,00,000
- Cash withdrawal (Full-KYC): Rs 2,000/txn, Rs 10,000/month
- P2P (Full-KYC only): Rs 1,00,000/month

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 3.1 | Full-KYC Balance Cap Discrepancy | Critical | The BALANCE_CAP rule enforces Rs 1,00,000 but the KYC tier table shows Full-KYC balance cap as Rs 2,00,000. RBI permits up to Rs 2,00,000 for Full-KYC PPIs. The Load Guard rule must be updated to enforce the correct tier-specific cap. |
| 3.2 | Per-Transaction Limit Enforcement | Medium | No per-transaction amount limit is enforced for merchant payments or P2P transfers. RBI may prescribe per-transaction ceilings that must be configurable. |
| 3.3 | Daily Transaction Limits | Medium | No daily aggregate transaction limit (debit-side) is enforced. Only monthly load caps are checked. Daily velocity limits are needed for fraud prevention and compliance. |
| 3.4 | Annual Load Cap Enforcement for Min-KYC | High | The Rs 1,20,000 annual load cap for Min-KYC wallets is defined in the data model but not included in the Load Guard's 3 rules. It must be added as a 4th rule. |
| 3.5 | Cash Withdrawal Limit Tracking | Medium | Cash withdrawal limits (Rs 2,000/txn, Rs 10,000/month) are specified but no implementation of tracking or enforcement is visible in the mock API or Load Guard. |
| 3.6 | Sub-Wallet Limit Interaction | Medium | NCMC Transit has a Rs 3,000 balance cap, and Food/Fuel wallets have monthly limits (Rs 3,000 and Rs 2,500 respectively). These sub-wallet limits are independent from RBI PPI limits. The interaction model (whether sub-wallet loads count toward monthly PPI load caps) must be clarified and enforced. |
| 3.7 | Limit Configuration Externalization | Low | All limits are hardcoded. RBI may revise limits via circular at any time. Limits should be externalized to a configuration service or admin-editable settings. |

### Action Items
- [ ] **CRITICAL:** Fix BALANCE_CAP in Load Guard to enforce Rs 2,00,000 for Full-KYC and Rs 10,000 for Min-KYC (tier-aware)
- [ ] Add annual load cap (Rs 1,20,000) as a 4th Load Guard rule for Min-KYC wallets
- [ ] Implement daily aggregate transaction limits (configurable per KYC tier)
- [ ] Build cash withdrawal tracking and limit enforcement module
- [ ] Define and document the interaction between sub-wallet limits and RBI PPI limits
- [ ] Externalize all limit values to a configuration table with admin UI for updates
- [ ] Add per-transaction ceiling enforcement with configurable limits

---

## 4. Escrow & Settlement

### RBI References
- RBI MD-PPIs, Chapter VII - Safeguarding of Money Collected
- RBI Circular on Escrow Account Requirements for PPI Issuers
- Payment and Settlement Systems Act, 2007 - Section 23A

### Current State
The platform stores wallet balances in an in-memory/localStorage mock data layer. No escrow account integration or settlement system is implemented.

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 4.1 | Escrow Account Integration | Critical | RBI mandates that the outstanding balance of PPIs must be maintained in an escrow account with a scheduled commercial bank. No escrow account linkage exists in the platform. |
| 4.2 | End-of-Day Settlement | Critical | PPI issuers must settle merchant payments within T+1 business day. No settlement engine or batch processing system exists. |
| 4.3 | Escrow Balance Reconciliation | Critical | Daily reconciliation between outstanding PPI balances and escrow account balance is mandatory. No reconciliation module exists. |
| 4.4 | Dormant/Closed Wallet Balance Transfer | High | When wallets move to CLOSED state, the system specifies "balance to suspense" and "T+1 balance refund via NEFT/IMPS." No integration with banking systems for actual fund transfer from escrow exists. |
| 4.5 | Interest on Escrow | Medium | Interest earned on escrow account balances must be handled per RBI guidelines. No interest calculation or distribution mechanism exists. |
| 4.6 | Multiple Escrow Account Support | Medium | If multiple banks are used for escrow (for redundancy), inter-escrow reconciliation is needed. Architecture does not address multi-bank escrow. |

### Action Items
- [ ] **CRITICAL:** Design and implement escrow account integration with a scheduled commercial bank
- [ ] Build T+1 settlement engine for merchant payments with batch processing
- [ ] Implement daily automated reconciliation between PPI outstanding balances and escrow account
- [ ] Build fund transfer module (NEFT/IMPS/RTGS) for wallet closure refunds from escrow
- [ ] Design interest handling mechanism for escrow balances per RBI norms
- [ ] Document escrow account architecture and failover strategy

---

## 5. Interoperability

### RBI References
- RBI MD-PPIs, Chapter VIII - Interoperability
- RBI Circular on Interoperability of Full-KYC PPIs (2022)
- NPCI UPI Guidelines for PPI Issuers

### Current State
The platform supports UPI-based payments (payment source shows "UPI - HDFC Bank 7125") and merchant QR code payments. The system references interoperability implicitly through UPI integration.

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 5.1 | Full-KYC PPI Interoperability Mandate | Critical | RBI mandates that all Full-KYC PPIs must be interoperable (usable at any merchant, not just issuer's network). While UPI integration is referenced, no explicit interoperability layer ensures acceptance at all UPI-enabled merchants and ATMs. |
| 5.2 | RuPay Card Issuance | High | RBI requires Full-KYC PPI issuers to offer card-based PPIs (physical/virtual) on RuPay or authorized card networks for interoperability. No card issuance module exists. |
| 5.3 | ATM Withdrawal for Full-KYC | Medium | Full-KYC PPIs with interoperability must allow ATM cash withdrawal via RuPay card. Cash withdrawal limits are defined (Rs 2,000/txn, Rs 10,000/month) but no ATM integration exists. |
| 5.4 | Cross-PPI Transfer | Medium | Interoperability should enable transfers between PPIs of different issuers via UPI. The P2P transfer currently validates against internal UUID wallet IDs only, not UPI VPAs or other PPI identifiers. |
| 5.5 | QR Code Standards | Medium | Merchant payment uses a generic QR placeholder. Must support Bharat QR and UPI QR standards per RBI/NPCI specifications. |

### Action Items
- [ ] **CRITICAL:** Implement full interoperability layer ensuring Full-KYC PPIs work at all UPI-enabled acceptance points
- [ ] Design and implement virtual/physical RuPay card issuance for Full-KYC wallets
- [ ] Integrate ATM withdrawal capability via RuPay card network
- [ ] Extend P2P transfer to support UPI VPA addresses and cross-PPI transfers
- [ ] Implement Bharat QR and UPI QR code generation and scanning per NPCI standards

---

## 6. AML/CFT (Anti-Money Laundering / Counter Financing of Terrorism)

### RBI References
- Prevention of Money Laundering Act (PMLA), 2002
- RBI Master Direction on KYC, Chapter V - Monitoring of Transactions
- FATF Recommendations on Virtual Payment Products
- RBI Circular on Suspicious Transaction Reporting

### Current State
The platform implements:
- CTR (Cash Transaction Report): Mandatory for cash > Rs 10 lakh/day
- STR (Suspicious Transaction Report): Filed within 7 days
- EDD (Enhanced Due Diligence): Triggered at Rs 50,000/month load
- Transaction flagging via MCP tool (`flag_suspicious_transaction`)
- Admin dashboard for flagged transaction review (`get_flagged_transactions`)

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 6.1 | Automated Transaction Monitoring Rules | High | While manual flagging exists via MCP tools, no automated rule-based transaction monitoring engine runs in real-time. RBI expects automated detection of structuring, rapid movement, unusual patterns, etc. |
| 6.2 | STR Filing Integration with FIU-IND | Critical | STRs must be filed electronically with the Financial Intelligence Unit - India (FIU-IND) via the FINnet 2.0 portal. No FIU-IND integration exists. The 7-day SLA is defined but no automated filing workflow is implemented. |
| 6.3 | CTR Automated Filing | High | CTR for cash transactions > Rs 10 lakh/day must be filed monthly with FIU-IND by the 15th of the following month. No automated CTR generation or filing system exists. |
| 6.4 | Transaction Velocity Rules | High | No velocity-based rules for detecting money laundering patterns (e.g., multiple small loads just below threshold, rapid load-and-transfer cycles, round-tripping). |
| 6.5 | Beneficiary Screening | High | For P2P transfers, the recipient must be screened against sanctions lists. No beneficiary screening is performed. |
| 6.6 | Risk Categorization Engine | Medium | While EDD is triggered at Rs 50,000/month load, no comprehensive customer risk categorization (Low/Medium/High) based on profile, geography, transaction behavior, and occupation exists. |
| 6.7 | Tipping-Off Controls | Medium | Staff and systems must not "tip off" customers about STR filings. No access controls or audit trails prevent disclosure of STR status to customer-facing channels. |
| 6.8 | Record of Internal Suspicious Reports | Medium | The admin dashboard shows flagged transactions but no formal internal suspicious activity report (SAR) workflow with investigation notes, escalation, and disposition tracking exists. |
| 6.9 | Wire Transfer Rule Compliance | Low | For transfers above Rs 50,000, originator and beneficiary information must be captured and transmitted per FATF wire transfer rules. Current P2P transfer captures minimal information. |

### Action Items
- [ ] Build real-time automated transaction monitoring engine with configurable rules (structuring, velocity, round-tripping, dormant-to-active spike)
- [ ] **CRITICAL:** Integrate with FIU-IND FINnet 2.0 portal for electronic STR and CTR filing
- [ ] Implement automated CTR generation for cash transactions exceeding Rs 10 lakh/day with monthly filing
- [ ] Add velocity-based detection rules for money laundering patterns
- [ ] Implement beneficiary screening for all outbound transfers (P2P, bank transfer)
- [ ] Build customer risk categorization engine (Low/Medium/High) with periodic reassessment
- [ ] Implement tipping-off controls: restrict STR status visibility to compliance officers only
- [ ] Build formal SAR workflow with investigation notes, escalation chains, and disposition tracking
- [ ] Capture and transmit full originator/beneficiary details for transfers above Rs 50,000

---

## 7. Security & Data Protection

### RBI References
- RBI Circular on Cyber Security Framework for Banks/NBFCs (applied to PPI issuers)
- IT Act, 2000 - Section 43A (Reasonable Security Practices)
- RBI Guidelines on Information Security, Electronic Banking, and Technology Risk Management
- Digital Personal Data Protection Act (DPDPA), 2023
- PCI-DSS Requirements (for card-based PPIs)
- RBI Circular on Storage of Payment System Data (Data Localisation)

### Current State
- PIN is a 4-digit code stored in localStorage (default: 1234)
- PII masking toggle exists in admin dashboard
- Data localisation requirement noted: "All data localised to India"
- Authentication: Phone + OTP login (mock: any 10-digit number)
- CORS whitelist for GitHub Pages + localhost

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 7.1 | PIN Storage in localStorage | Critical | The 4-digit wallet PIN is stored in plain text in localStorage (`wallet_pin`). This is a severe security vulnerability. PINs must be hashed (bcrypt/Argon2) and stored server-side, never in client-side storage. |
| 7.2 | Default PIN Value | Critical | Default PIN of "1234" is hardcoded. Users must be forced to set a unique PIN during onboarding. No PIN complexity enforcement or change-after-first-use flow exists. |
| 7.3 | Authentication Strength | Critical | Mock login accepts any 10-digit number without real OTP verification. Production must integrate with SMS gateway for real OTP delivery with rate limiting, expiry (2-5 minutes), and retry limits. |
| 7.4 | Session Management | High | Authentication state stored in localStorage (`wallet_auth`) without session tokens, expiry, or refresh mechanisms. No session timeout after inactivity. |
| 7.5 | Encryption at Rest | High | Sensitive data (wallet balances, transaction history, KYC artifacts) stored in localStorage/in-memory without encryption. Production requires AES-256 encryption at rest. |
| 7.6 | Encryption in Transit | Medium | CORS is configured but no mention of TLS 1.2+ enforcement, certificate pinning, or HSTS headers. All API communication must use TLS 1.2 or higher. |
| 7.7 | Data Localisation Verification | High | "All data localised to India" is stated but the backend is deployed on Render (which may use non-India servers). Data localisation per RBI mandate requires verified India-only data residency. |
| 7.8 | PII Data Handling | Medium | PII masking toggle exists in admin but no data classification, access logging, or DLP (Data Loss Prevention) controls exist. DPDPA compliance requires purpose limitation, data minimisation, and consent management. |
| 7.9 | API Security | High | No API rate limiting, input validation beyond Zod schemas, SQL injection protection, or WAF (Web Application Firewall) mentioned. API endpoints must be hardened per OWASP guidelines. |
| 7.10 | Vulnerability Assessment | Medium | No mention of VAPT (Vulnerability Assessment and Penetration Testing) schedule. RBI mandates periodic VAPT and IT audit by CERT-IN empanelled auditors. |
| 7.11 | Multi-Factor Authentication for Admin | High | Admin dashboard uses username/password only (admin/admin123). Compliance and operations roles must use MFA (TOTP/hardware token). |
| 7.12 | KYC Artifact Storage Security | High | KYC artifacts (Aadhaar data) require 10-year retention with encryption and access controls per UIDAI regulations. No secure KYC document vault is implemented. |
| 7.13 | Incident Response Plan | Medium | No documented incident response plan for security breaches, data leaks, or system compromises. RBI requires a board-approved cyber security policy and incident response framework. |

### Action Items
- [ ] **CRITICAL:** Remove PIN from localStorage; implement server-side PIN storage with bcrypt/Argon2 hashing
- [ ] **CRITICAL:** Remove default PIN; enforce mandatory PIN creation during onboarding with complexity rules
- [ ] **CRITICAL:** Integrate production SMS/OTP gateway with rate limiting and expiry
- [ ] Implement JWT-based session management with short-lived access tokens, refresh tokens, and inactivity timeout
- [ ] Implement AES-256 encryption at rest for all sensitive data stores
- [ ] Enforce TLS 1.2+, HSTS, and certificate pinning for all API communications
- [ ] Verify and document data localisation (India-only hosting) for all data stores and processing
- [ ] Implement data classification framework and DLP controls per DPDPA requirements
- [ ] Add API rate limiting, WAF, input sanitization, and OWASP Top 10 protections
- [ ] Schedule quarterly VAPT by CERT-IN empanelled auditors
- [ ] Implement MFA for all admin dashboard roles (mandatory for SUPER_ADMIN, COMPLIANCE_OFFICER, OPS_MANAGER)
- [ ] Build encrypted KYC document vault with access logging and 10-year retention
- [ ] Develop and document incident response plan with board approval

---

## 8. Reporting & Audit

### RBI References
- RBI MD-PPIs, Chapter IX - Reporting Requirements
- RBI Circular on Returns/Statements to be Submitted by PPI Issuers
- PMLA Rules - Record Keeping Requirements
- RBI Circular on Audit Trail Requirements

### Current State
- Data retention: 5 years ledger/admin, 10 years KYC artifacts
- Append-only ledger with idempotency keys
- Admin dashboard with transaction monitoring and KYC stats
- MCP tools for generating reports and system stats
- Wallet statement download (1M/3M/6M/1Y/Custom) via email

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 8.1 | RBI Returns Automation | Critical | PPI issuers must submit periodic returns to RBI (monthly, quarterly, annually) covering outstanding PPI balances, transaction volumes, fraud incidents, complaints, etc. No automated return generation or submission system exists. |
| 8.2 | Audit Trail Completeness | High | The append-only ledger captures financial transactions but no audit trail exists for administrative actions (KYC approvals/rejections, user suspensions, limit changes, configuration changes). |
| 8.3 | Board/Management Reporting | Medium | RBI expects periodic reporting to the issuer's board on PPI operations, compliance, fraud, and risk. No board reporting module or template exists. |
| 8.4 | Concurrent Audit | Medium | RBI may require concurrent/internal audit of PPI operations. No audit-facilitating features (sampling, extraction, read-only audit access) exist. |
| 8.5 | Transaction Log Immutability | High | While the ledger is "append-only," the current localStorage/in-memory implementation provides no guarantees of immutability. Production requires immutable audit logs (e.g., write-once storage, blockchain-anchored hashes, or database-level immutability). |
| 8.6 | Data Retention Enforcement | Medium | 5-year and 10-year retention periods are specified but no automated archival, retention policy enforcement, or secure deletion after retention period exists. |
| 8.7 | Regulatory Change Tracking | Low | No system to track and flag RBI circular updates, master direction amendments, or new compliance requirements as they are issued. |

### Action Items
- [ ] **CRITICAL:** Build automated RBI return generation system (monthly/quarterly/annual) with submission tracking
- [ ] Implement comprehensive audit trail for all administrative actions with who/what/when/why
- [ ] Create board reporting templates and automated generation for quarterly board presentations
- [ ] Build audit access module with read-only role, sampling tools, and data extraction capabilities
- [ ] Implement immutable transaction log in production (append-only database with cryptographic chaining or write-once storage)
- [ ] Build data lifecycle management: automated archival, retention enforcement, and certified deletion
- [ ] Subscribe to RBI notification service and build regulatory change tracker in admin dashboard

---

## 9. Wallet Lifecycle Management

### RBI References
- RBI MD-PPIs, Chapter VI - Validity and Closure of PPIs
- RBI Circular on Treatment of PPIs Remaining Unused
- RBI Circular on Unclaimed Amounts in PPIs

### Current State
**Wallet Lifecycle States:**
```
ACTIVE -> SUSPENDED (fraud/compliance hold)
ACTIVE -> DORMANT (12 months no transactions)
DORMANT -> CLOSED (24 months, balance to suspense)
MIN_KYC -> EXPIRED (12-month validity, 60-day grace)
Any -> CLOSED (user-initiated, T+1 balance refund via NEFT/IMPS)
```

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 9.1 | Dormancy Notification | High | Before transitioning a wallet to DORMANT, the customer must be notified via SMS/email/in-app (at least 30 days advance notice). No automated dormancy notification system exists. |
| 9.2 | Grace Period Handling for Min-KYC Expiry | Medium | The 60-day grace period for MIN_KYC expiry is defined but no implementation details exist for what operations are permitted during the grace period (e.g., withdrawal only, no new loads). |
| 9.3 | Unclaimed Balances Transfer | High | Wallets CLOSED after 24 months of dormancy with remaining balance must have unclaimed amounts transferred per RBI/IEPF (Investor Education and Protection Fund) or DEA (Department of Economic Affairs) guidelines. No unclaimed balance transfer mechanism exists. |
| 9.4 | Wallet Reactivation Flow | Medium | No defined process for reactivating a DORMANT or SUSPENDED wallet. Reactivation may require re-KYC, additional verification, or compliance officer approval. |
| 9.5 | User-Initiated Closure SLA | Medium | T+1 refund is specified for user-initiated closure, but no SLA monitoring, escalation on breach, or customer communication of refund status exists. |
| 9.6 | Partial Closure / Balance Transfer | Low | No option for customers to transfer balance to another PPI or bank account before closure (other than full closure refund). Interoperable PPIs should allow balance portability. |
| 9.7 | Automated State Transitions | High | State transitions (ACTIVE -> DORMANT at 12 months, DORMANT -> CLOSED at 24 months) require automated batch jobs. No scheduled job or cron system is implemented for lifecycle transitions. |

### Action Items
- [ ] Implement automated dormancy notification system (SMS + email + in-app) with 30-day advance notice
- [ ] Define and enforce grace period rules for Min-KYC expiry (withdrawal-only mode during 60-day grace)
- [ ] Build unclaimed balance transfer module per RBI/IEPF guidelines
- [ ] Design wallet reactivation workflow with re-KYC trigger and compliance approval for SUSPENDED wallets
- [ ] Implement closure refund SLA monitoring with T+1 tracking and escalation
- [ ] Add balance transfer/portability option before wallet closure
- [ ] **HIGH:** Build automated batch job system for wallet lifecycle state transitions with audit logging

---

## 10. Merchant Onboarding & Compliance

### RBI References
- RBI MD-PPIs, Chapter III - Types of PPIs and Permitted Transactions
- RBI Guidelines on Merchant Onboarding (Payment Aggregator Circular)
- NPCI Merchant Registration Guidelines
- RBI Circular on MDR (Merchant Discount Rate)

### Current State
- Merchant payments supported via QR code scanning
- MCC (Merchant Category Code) mapping with 19 spending categories
- Sub-wallet eligibility check based on merchant category keywords
- Cascade spend logic: specific sub-wallet -> Gift wallet -> main wallet

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 10.1 | Merchant KYC/Onboarding | High | No merchant onboarding or KYC process exists. PPI issuers accepting merchants must verify merchant identity, business registration, PAN, GST, and bank account details. |
| 10.2 | MCC Code Verification | Medium | The 19-category MCC mapping uses keyword detection from transaction descriptions. This is insufficient for compliance; merchants must be assigned verified MCC codes during onboarding, not inferred from transaction text. |
| 10.3 | Merchant Risk Assessment | Medium | No merchant risk categorization (Low/Medium/High) based on business type, transaction volume, chargeback history, or compliance record. |
| 10.4 | Merchant Agreement / T&C | Medium | No digital merchant agreement, terms of service, or chargeback liability framework exists. |
| 10.5 | MDR (Merchant Discount Rate) Management | Medium | No MDR configuration, calculation, or settlement system. PPI-to-merchant transactions may attract MDR as per RBI/NPCI guidelines. |
| 10.6 | Merchant Transaction Limits | Low | No per-merchant transaction limits, daily settlement caps, or velocity controls for merchant payments. |
| 10.7 | Prohibited Merchant Categories | Medium | No enforcement of RBI-prohibited merchant categories (e.g., gambling, cryptocurrency trading, banned goods). Merchants should be screened against prohibited MCC codes. |

### Action Items
- [ ] Build merchant onboarding module with KYC (PAN, GST, bank account verification, business registration)
- [ ] Implement verified MCC code assignment during merchant onboarding (replace keyword-based inference)
- [ ] Build merchant risk categorization and periodic review system
- [ ] Create digital merchant agreement workflow with electronic signature
- [ ] Implement MDR configuration, calculation, and settlement module
- [ ] Add per-merchant transaction limits and velocity controls
- [ ] Implement prohibited merchant category blacklist enforcement

---

## 11. Dispute Resolution & Grievance Redressal

### RBI References
- RBI Circular on Limiting Liability of Customers in Unauthorised Electronic Banking Transactions (2017)
- RBI Integrated Ombudsman Scheme, 2021
- RBI MD-PPIs, Chapter X - Customer Protection and Grievance Redressal
- Consumer Protection Act, 2019

### Current State
- Dispute SLA: 30 days per RBI, auto-close in customer's favour at day 30+1
- MCP tools: `raise_dispute`, `get_dispute_status`, `get_refund_status`
- Refund request tool: `request_refund`

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 11.1 | Liability Framework Implementation | Critical | RBI's zero-liability and limited-liability framework for unauthorized transactions is not implemented. Customers must not be liable for unauthorized transactions reported within 3 working days. The platform has no liability calculation or automatic reversal based on reporting timeline. |
| 11.2 | TAT (Turn Around Time) Based Provisional Credit | High | For disputes on unauthorized transactions, RBI mandates provisional credit within 10 working days if investigation is not completed. No provisional credit mechanism exists. |
| 11.3 | Grievance Escalation Matrix | Medium | No defined escalation matrix (Level 1: Customer Service -> Level 2: Nodal Officer -> Level 3: RBI Ombudsman). Customer must be informed of escalation path and Ombudsman contact details. |
| 11.4 | Dispute Categorization | Medium | No categorization of disputes (unauthorized transaction, merchant dispute, service failure, failed transaction, etc.) which is needed for proper routing and SLA tracking. |
| 11.5 | RBI Ombudsman Integration | Medium | RBI Integrated Ombudsman Scheme requires PPI issuers to integrate with the CMS (Complaint Management System). No integration exists. |
| 11.6 | Auto-Reversal for Failed Transactions | High | For failed transactions where money is debited but service not delivered, RBI mandates auto-reversal within T+5 working days. No failed transaction monitoring or auto-reversal system exists. |
| 11.7 | Compensation for Delayed Resolution | Medium | RBI mandates compensation (Rs 100/day) for delayed dispute resolution beyond the stipulated TAT. No automated compensation calculation exists. |
| 11.8 | Dispute Communication Trail | Medium | All dispute communications (acknowledgment, updates, resolution) must be logged with timestamps. No structured communication log for disputes exists. |
| 11.9 | Customer Awareness | Low | RBI requires PPI issuers to display the grievance redressal mechanism, nodal officer details, and RBI Ombudsman link prominently. Not present in app UI. |

### Action Items
- [ ] **CRITICAL:** Implement RBI zero-liability/limited-liability framework with automatic reversal based on reporting timeline (3-day/4-7 day/beyond 7-day buckets)
- [ ] Implement provisional credit mechanism for disputes pending beyond 10 working days
- [ ] Build and display grievance escalation matrix (L1 -> L2 Nodal Officer -> L3 RBI Ombudsman)
- [ ] Implement dispute categorization with type-specific routing and SLA tracking
- [ ] Integrate with RBI Integrated Ombudsman CMS portal
- [ ] Build auto-reversal engine for failed transactions with T+5 monitoring
- [ ] Implement automated compensation calculation (Rs 100/day) for SLA breaches
- [ ] Build structured dispute communication log with customer-facing status updates
- [ ] Add grievance redressal information, nodal officer details, and RBI Ombudsman link to app Profile/Help pages

---

## 12. NPCI / UPI Integration

### RBI References
- NPCI UPI Procedural Guidelines
- NPCI Circular on UPI Transaction Limits
- RBI Circular on UPI for PPIs
- NPCI Third-Party App Provider (TPAP) Guidelines
- NPCI Circular on UPI Lite and UPI 123PAY

### Current State
- UPI referenced as a payment source ("UPI - HDFC Bank 7125")
- QR code-based merchant payment flow exists
- Bank transfer via IFSC validation
- No direct NPCI/UPI API integration (mock implementation)

### Gap Findings

| # | Gap | Severity | Description |
|---|-----|----------|-------------|
| 12.1 | UPI PSP/TPAP Registration | Critical | To operate as a UPI-enabled PPI, the issuer must be registered with NPCI as a PSP (Payment Service Provider) or through a sponsor bank. No NPCI registration or PSP agreement is in scope. |
| 12.2 | UPI Handle / VPA Generation | High | Full-KYC PPI wallets must have a UPI VPA (Virtual Payment Address) for interoperability. No UPI handle generation (e.g., user@ppiwallet) is implemented. |
| 12.3 | UPI Transaction Limit Compliance | Medium | NPCI prescribes UPI transaction limits (currently Rs 1,00,000 per transaction for most categories, Rs 5,00,000 for specific categories like capital markets). No NPCI limit enforcement layer exists alongside RBI PPI limits. |
| 12.4 | UPI Mandate / Recurring Payments | Medium | UPI AutoPay (mandate) for recurring payments (subscriptions, bills) is not implemented. The auto top-up feature ("Auto add Rs 2,000 when balance below Rs 200") would require UPI mandate registration. |
| 12.5 | NPCI Dispute Resolution (UDIR) | High | NPCI operates the UPI Dispute Resolution (UDIR) system for handling UPI transaction disputes. No UDIR integration exists for automated dispute routing and resolution. |
| 12.6 | UPI Lite Integration | Low | NPCI's UPI Lite allows small-value offline transactions (up to Rs 500/txn, Rs 4,000 wallet limit). Could complement the PPI wallet for micro-payments. Not implemented. |
| 12.7 | BQR / UPI QR Standards | Medium | The QR code implementation must conform to NPCI Bharat QR (BQR) and UPI QR specifications including EMV QR code standards. Current implementation uses a generic QR placeholder. |
| 12.8 | NPCI Reporting & Returns | Medium | PPI issuers on UPI must submit periodic returns to NPCI on transaction volumes, success rates, decline rates, and technical failures. No NPCI reporting integration exists. |
| 12.9 | UPI Callback / Notification Handling | Medium | UPI transactions generate callbacks (credit/debit notifications). No UPI callback handler or real-time transaction notification infrastructure exists. |
| 12.10 | Collect Request Handling | Low | UPI Collect requests (merchant-initiated payment requests) are not supported. Full interoperability requires handling both pay and collect flows. |

### Action Items
- [ ] **CRITICAL:** Initiate NPCI PSP/TPAP registration process and establish sponsor bank relationship
- [ ] Implement UPI VPA generation and management for Full-KYC wallets
- [ ] Build NPCI transaction limit enforcement layer (configurable per transaction category)
- [ ] Implement UPI AutoPay mandate registration for recurring payments and auto top-up
- [ ] Integrate with NPCI UDIR system for UPI dispute resolution
- [ ] Evaluate UPI Lite implementation for micro-payment use cases
- [ ] Implement BQR and UPI QR code generation/scanning per EMV and NPCI standards
- [ ] Build NPCI return generation and submission module
- [ ] Implement UPI callback handler for real-time transaction status updates
- [ ] Add UPI Collect request handling for merchant-initiated payment flows

---

## Summary: Gap Severity Distribution

| Severity | Count | Categories Affected |
|----------|-------|---------------------|
| Critical | 12 | Licensing, Transaction Limits, Escrow, Interoperability, AML/CFT, Security, Reporting, Dispute, NPCI/UPI |
| High | 24 | All 12 categories |
| Medium | 32 | All 12 categories |
| Low | 8 | Licensing, KYC/CDD, Transaction Limits, Reporting, Merchant, NPCI/UPI |
| **Total** | **76** | |

## Priority Remediation Roadmap

### Phase 1: Critical (0-3 months)
1. Fix PIN storage security vulnerability (remove from localStorage, implement server-side hashing)
2. Fix BALANCE_CAP discrepancy (Rs 1,00,000 vs Rs 2,00,000 for Full-KYC)
3. Implement escrow account integration and settlement engine
4. Integrate with FIU-IND for STR/CTR filing
5. Implement zero-liability framework for unauthorized transactions
6. Initiate NPCI PSP/TPAP registration
7. Build RBI return generation system
8. Implement production-grade authentication (real OTP, session management)
9. Implement Full-KYC interoperability mandate

### Phase 2: High Priority (3-6 months)
1. CKYC Registry integration
2. PEP/sanctions screening
3. Automated transaction monitoring engine
4. Wallet lifecycle automation (dormancy, closure batch jobs)
5. Merchant onboarding and KYC module
6. Provisional credit for disputes
7. Auto-reversal for failed transactions
8. Admin MFA implementation
9. Comprehensive audit trail for administrative actions
10. Data localisation verification and remediation
11. UPI VPA generation and management
12. RuPay card issuance for Full-KYC wallets
13. Annual load cap enforcement for Min-KYC
14. NPCI UDIR dispute integration

### Phase 3: Medium Priority (6-12 months)
1. V-CIP (Video KYC) workflow
2. Periodic KYC refresh scheduling
3. Customer risk categorization engine
4. Board reporting module
5. Data lifecycle management (archival, retention, deletion)
6. MDR management system
7. UPI AutoPay mandate integration
8. Grievance escalation matrix and Ombudsman integration
9. BQR/UPI QR standards implementation
10. Dispute categorization and routing
11. NPCI reporting integration

### Phase 4: Enhancements (12+ months)
1. DigiLocker integration
2. UPI Lite for micro-payments
3. Regulatory change tracker
4. Multi-bank escrow architecture
5. UPI Collect request handling
6. Balance portability on closure

---

*This document should be reviewed and updated quarterly, or upon issuance of new RBI circulars affecting PPI operations.*
