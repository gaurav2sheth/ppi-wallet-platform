# Compliance Gap Analysis — PPSL PPI Wallet

Cross-reference: PRD vs RBI Master Directions on Prepaid Payment Instruments (Oct 2023) and RBI PA-PG Guidelines (Mar 2022)

Entity: Paytm Payment Services Limited (PPSL)  |  Prepared: March 2026  |  Status: Draft — Pending Legal Review

## Legend & Priority Flags

| ⚠ CRITICAL | Regulatory requirement with no PRD coverage, OR a gap that could block launch or invite RBI enforcement. Requires legal/compliance sign-off before build begins. | GAP | Partial coverage or an open question that must be resolved before / during build. |
| --- | --- | --- | --- |
| Covered | PRD coverage is adequate. Minor implementation verification may still be needed. | Items requiring legal / RBI clarification are flagged in the Gap column. | Engage RBI DPSS, NPCI, or legal counsel before resolving. |

## Section A — Licensing & Authorization

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| PPI License — PPSL must hold valid RBI authorization as PPI issuer before any wallet issuance (PPI-MD Clause 4) | PRD flags as P0 blocker: 'PPSL must hold a valid RBI PPI license before any development begins.' No detail on current license status. | CRITICAL: Confirm whether PPSL has applied for / holds PPI license. If not, initiate application (3–6 month RBI lead time). Legal sign-off required before Sprint 1. | ⚠ CRITICAL |
| Net Worth — Minimum ₹15 Cr net worth for non-bank PPI issuers at authorization, ₹25 Cr by end of 3rd FY (PPI-MD Clause 5) | PRD does not explicitly address PPSL's net worth position. | Obtain CFO certification of PPSL net worth as on date. If PPSL is a subsidiary, clarify whether parent guarantee is accepted by RBI. | GAP |
| Entity Type — Only scheduled commercial banks and authorized non-bank entities can issue PPIs (PPI-MD Clause 4.1) | PRD correctly identifies PPSL as the issuing entity under PA-PG license, distinct from TPAP/PCL. | Verify RBI authorization explicitly covers PPI issuance (PA license ≠ PPI license). Legal clarification required. | GAP |

## Section B — KYC & Customer Due Diligence

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Minimum KYC — Mobile number + self-declaration of name + unique ID. Balance cap ₹10,000. Monthly load ₹10,000. Annual load ₹1.2 lakh. Valid 12 months (PPI-MD Clause 9.2) | PRD Block 02 and Block 07 address Min-KYC wallet type, ₹10,000 balance cap, and monthly load limit. Wallet validity period of 12 months not explicitly mentioned. | Add 12-month validity logic to Wallet Lifecycle (Block 12). Build auto-expiry notification and UX re-KYC upgrade flow before expiry. | GAP |
| Full KYC — Aadhaar OTP / biometric or Video KYC. Balance cap ₹2 lakh. Must comply with RBI KYC Master Direction (PPI-MD Clause 9.3) | PRD Block 03 covers Aadhaar OTP, PAN, Video KYC with ₹2 lakh cap. CKYC check mentioned. | PRD does not specify CKYC registry lookup before Video KYC (mandatory per KYC-MD Oct 2023 update). Add CKYC check to KYC Orchestrator sprint plan. | GAP |
| Gift / Corporate PPIs — Issued in favour of a person against specific value. Different issuance rules apply (PPI-MD Clause 9.1) | PRD Block 02 mentions Gift/Corporate wallet type in PPI type registry. | Gift wallet issuance rules (issuing entity must maintain records, no KYC on recipient if below ₹10,000) are not detailed in PRD. Define rules for gift PPI issuance and merchant PPI before build. | GAP |
| KYC Record Retention — 10 years under PMLA Section 12. (PPI-MD Clause 17, PMLA 2002) | PRD Block 03 notes 'Retain KYC artefacts for 10 years per PMLA'. | Covered. Verify encryption-at-rest and PMLA-compliant deletion policy are implemented at infrastructure layer. | Covered |
| CKYC Integration — Mandatory CKYC registry check and upload at onboarding (RBI KYC-MD 2016, Oct 2023 update) | PRD mentions 'CKYC check must be run before Video KYC' in Technical Notes for Block 03. | CKYC upload API to CERSAI not explicitly in sprint plan. Add as explicit sprint task under Block 03 with API contract. | GAP |

## Section C — Transaction Limits

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Min-KYC wallet: balance ≤ ₹10,000 at any time; monthly load ≤ ₹10,000; P2P transfer prohibited (PPI-MD Clause 9.2, 15.1) | PRD Block 07 covers balance cap and monthly load. P2P prohibition for Min-KYC not explicitly stated. | Add explicit rule: Min-KYC wallets CANNOT initiate P2P wallet-to-wallet transfers. Block 04 Money Movement flow must enforce this. | GAP |
| Full-KYC wallet: balance ≤ ₹2 lakh; monthly load ≤ ₹1 lakh (post April 2024 amendment); P2P ≤ ₹1 lakh/month (PPI-MD Clause 9.3) | PRD Block 07 covers ₹2 lakh balance cap. Monthly load and P2P limits for Full-KYC mentioned in Block 04. | Confirm alignment with April 2024 RBI amendment on monthly load limits. PRD may reflect pre-amendment limits. Legal/compliance to verify current applicable limits before coding. | GAP |
| Cash withdrawal — Full KYC semi-closed PPI: ≤ ₹2,000/transaction; ≤ ₹10,000/month (PPI-MD Clause 10.3) | PRD mentions wallet-to-bank withdrawal but does not distinguish cash withdrawal via ATM/PoS from bank transfer. | Clarify whether PPSL intends to enable ATM/PoS cash withdrawal. If yes, add separate limits config for cash withdrawal. If no, document in compliance register that this feature is intentionally excluded. | GAP |

## Section D — Escrow & Nodal Account

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Escrow with Scheduled Commercial Bank — All outstanding e-money must be held in escrow with a scheduled commercial bank (PPI-MD Clause 7.1) | PRD Block 05 correctly identifies this as P0 blocker and covers escrow API integration design. | Covered in design intent. Execution gap: bank partner not yet identified. See Brief 2 for bank shortlist. Escrow agreement must be RBI-format compliant — legal review required. | Covered |
| Float = Outstanding E-Money — Total wallet balance must always equal escrow float (PPI-MD Clause 7.2) | PRD Block 05 addresses float management and 95% utilisation threshold alerts. | PRD uses 95% threshold for alerts but does not define the breach response SOP. Define SOP: at 100% float utilisation, block new Add Money transactions. Document as compliance control. | GAP |
| Daily Reconciliation — Mandatory daily reconciliation of outstanding e-money vs escrow balance (PPI-MD Clause 7.3) | PRD Block 08 covers daily reconciliation engine with EOD automated run. | Covered. Confirm reconciliation report format matches RBI-prescribed format and is retained for 5 years. | Covered |
| Interest on Escrow Float — No interest on escrow account per PA guidelines. PPI-MD allows separate core-portion interest account. | PRD Block 03 (Escrow criteria Section 3.2) mentions interest rate on nodal float. Section 3.4 notes 'Interest rate on nodal account float (material at scale)'. | Clarify with partner bank and RBI counsel: PPI escrow accounts may not earn interest directly. Separate 'core portion' arrangement per PA guidelines may be available. Legal opinion required before contracting. | GAP |

## Section E — Interoperability

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Full-KYC PPIs must be interoperable — via UPI for wallet-form PPIs; via authorised card networks for card-form PPIs (PPI-MD Clause 6.5, mandatory from March 2022) | PRD Block 11 covers UPI integration. Block 02 mentions 'Full-KYC wallets must be interoperable via UPI from Day 1 of Full-KYC upgrade'. | Covered for UPI. Card-form interoperability (Visa/Mastercard) not addressed in PRD — confirm whether PPSL intends to issue PPI as card. If yes, card network partnership and BIN issuance required. | GAP |
| Third-party UPI app interoperability — Full-KYC PPI holders must be able to transact via any TPAP (RBI Dec 2024 amendment to PPI-MD) | PRD Block 11 covers UPI integration but does not specifically address PPSL wallet being linkable to third-party UPI apps (Google Pay, PhonePe, etc.). | CRITICAL GAP: Dec 2024 RBI amendment mandates this. PPSL wallet must be enrollable as a funding source in third-party UPI apps. Requires NPCI technical certification and UPI handle registration. Add to Block 11 sprint plan. | ⚠ CRITICAL |

## Section F — AML / CFT Compliance

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| FIU-IND Registration — PPSL must register as Reporting Entity under PMLA before commencing PPI issuance (PMLA 2002 Sec 12, FIU-IND guidelines) | PRD Block 09 identifies FIU-IND FINNET 2.0 portal and STR/CTR filing as a requirement. | FIU-IND registration timeline not in sprint plan. This is a pre-launch regulatory step. Add to 'Regulatory (Business/Legal)' track with 4-6 week lead time. | GAP |
| STR Filing — Suspicious Transaction Reports must be filed within 7 days of suspicion (PMLA Sec 12A) | PRD Block 09 covers STR workflow with 7-day SLA. | Covered. Confirm FINNET 2.0 integration is tested end-to-end in staging environment before go-live. | Covered |
| AML Monitoring — Enhanced Due Diligence for wallets with cumulative load > ₹50,000/month (PPI-MD Clause 18) | PRD Block 09 mentions '₹50,000/month load flag for enhanced due diligence'. | Covered in design. Confirm AML rule is implemented as automatic flag (not just dashboard indicator) and triggers ops review queue. | Covered |
| CTR Filing — Cash transactions > ₹10 lakh must be reported to FIU-IND (PMLA Sec 12) | PRD Block 09 mentions CTR filings. | Wallet-funded transactions are typically non-cash. Clarify whether PPI top-up via cash (if enabled) triggers CTR. If cash loading is not enabled, document explicitly. | GAP |

## Section G — Security & Data

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Two-factor authentication for all wallet transactions (PPI-MD Clause 14) | PRD Block 06 covers OTP/2FA with CPaaS integration. | Covered. CERT-In guidelines on OTP delivery channels should be referenced in implementation. | Covered |
| Fraud Monitoring — Continuous transaction monitoring with escalation to Law Enforcement (PPI-MD Clause 27) | PRD Block 06 covers MARS integration and wallet-specific velocity rules. | LEA escalation workflow not detailed in PRD. Add escalation SOP to Block 06 and link to Dispute/Compliance module. | GAP |
| Data Localisation — All payment data must be stored in India (RBI circular Apr 2018, PA-PG guidelines 2022) | PRD Block 03 mentions 'Data residency: all nodal account data must be stored in India' in escrow criteria. | Localisation requirement applies to all PPI transaction data, not just nodal. Confirm infrastructure layer (Block 18) enforces India-only data residency for all wallet data stores. | GAP |
| RBI Periodic Reports — Monthly and quarterly PPI transaction data reports to RBI DPSS (PPI-MD Annex / RBI circular) | PRD Block 09 covers RBI periodic report generation. | RBI report formats (as per PPI-MD Annex) must be implemented exactly. Engage RBI DPSS for prescribed format before building report module. | GAP |

## Section H — Reporting & Audit

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Audit Log Retention — All transaction logs retained for 5 years (PPI-MD Clause 8.4, IT Act 2000) | PRD Block 01 notes '5-year audit log retention' for ledger. | Covered for ledger. Confirm admin action logs (Block 15) and KYC audit trail (Block 03) also have 5-year retention policy in data architecture. | Covered |
| PPSL's PA-PG license — Wallet balance cannot be used as pass-through for PA settlement. PPI and PA escrow accounts must be distinct (RBI PA-PG guidelines 2022 / 2025 update) | PRD correctly identifies data isolation as a core requirement: 'separate DB cluster/schema for PPSL wallet vs TPAP/PCL'. | CRITICAL: PA escrow and PPI escrow must be SEPARATE bank accounts. Commingling is a regulatory violation. Confirm with RBI counsel that PPSL holding both PA and PPI licenses does not require ring-fencing beyond separate accounts. | ⚠ CRITICAL |

## Section I — PA-PG License Overlap & NPCI

| Regulatory Requirement
(Clause / Direction) | Our PRD Coverage | Gap / Action Needed | Status |
| --- | --- | --- | --- |
| Merchant KYC / Onboarding — PA guidelines require merchant due diligence before onboarding (PA-PG guidelines Clause 8) | PRD Block 10 mentions merchant onboarding and settlement but does not detail merchant KYC/AML compliance for wallet-acceptance enablement. | Wallet-accepting merchants must be subject to same KYC/AML diligence as PA merchants. Confirm Block 10 reuses PPSL's existing PA merchant onboarding KYC stack. | GAP |
| NPCI UPI Membership / TPAP License — To issue UPI handles under PPSL entity, PPSL needs NPCI PSP/TPAP participation (NPCI UPI Procedural Guidelines) | PRD Block 11 identifies 'NPCI UPI certification process' as Sprint 1 task under UPI track. | CRITICAL: NPCI TPAP license is distinct from RBI PPI license and has longest lead time (6-12 months). Initiate NPCI engagement immediately. Without this, Full-KYC wallets cannot achieve UPI interoperability. | ⚠ CRITICAL |

# Critical Items Requiring Action Before Build Begins

The following items are CRITICAL — they either block development or carry material regulatory risk. Each requires resolution before Sprint 1.

| # | Item | Required Action | Owner | Deadline |
| --- | --- | --- | --- | --- |
| 1 | PPI License Confirmation | Confirm PPSL holds active RBI PPI authorization. If not, initiate application immediately. | Legal / CEO | Pre-Sprint 1 |
| 2 | PA + PPI Escrow Separation | Obtain RBI/legal opinion confirming separate escrow accounts for PA and PPI. Document in compliance register. | Legal / Finance | Pre-Sprint 1 |
| 3 | NPCI TPAP Membership | Initiate NPCI engagement for UPI TPAP participation. 6–12 month lead time — do not wait. | Business / Tech Lead | Immediate |
| 4 | Third-Party UPI App Linkage | Add Dec 2024 RBI amendment requirement to Block 11 sprint plan: Full-KYC wallets must be linkable in Google Pay, PhonePe etc. | Product / Engineering | Block 11 Sprint 1 |
| 5 | FIU-IND Registration | Register PPSL as Reporting Entity with FIU-IND before go-live. 4–6 week process. | Compliance | Pre Go-Live |
| 6 | Escrow Interest Structure | Get legal opinion on whether PPI escrow float can earn interest. Negotiate core-portion arrangement with partner bank. | Legal / Finance | During Bank RFP |
| 7 | CKYC Upload Integration | Add CKYC registry lookup + upload to Block 03 KYC Orchestrator. Mandatory per Oct 2023 KYC-MD update. | Engineering | Block 03 Sprint 4 |

Note: This analysis is based on RBI PPI Master Directions (issued 2021, updated October 2023), RBI PA-PG Guidelines (March 2022 / 2025 update), PMLA 2002, and the PPSL PPI Wallet PRD (March 2026). This document does not constitute legal advice. All critical items should be reviewed by qualified legal counsel and PPSL's compliance team before action.
