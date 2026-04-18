# Compliance Reference — Regulatory Framework Summary

> **Scope of this document: FRAMEWORK SUMMARY, not current circular citation.**
>
> This document summarises the RBI / PMLA / DPDP regulatory framework that shapes the PPI Wallet reference implementation's design. It is **not** a primary-source citation of currently-in-force circular paragraphs. Author lacks access to the current RBI Master Directions PDF and cannot verify each clause reference as this document sits.
>
> **Use this document for:**
> - Understanding which regulation shapes which implementation choice
> - Following the link structure across other platform docs (README, CLAUDE.md, `.claude/rules/compliance.md`, PRDs, ADRs) as their shared numbers table
> - Running a compliance verification pass (each row below is a checklist item)
>
> **Do NOT use this document for:**
> - Any production deployment — the author's summary of regulatory limits is not a substitute for a lawyer reading the current RBI text
> - Any external claim of RBI compliance — at minimum, every `⚠️ TO VERIFY` marker must be closed out against primary source
>
> **Last verified against primary source:** *Not yet verified.* **Target verification date:** *TBD by the author.* When verification is complete, this section becomes: *"Last verified against: RBI Master Directions on Prepaid Payment Instruments dated [date], as amended by circular [ref]. Next review due: [date + 6 months]."*

---

## PPI Issuance Limits

| Rule | Small PPI (Min-KYC) | Full-KYC PPI | Source paragraph |
| --- | --- | --- | --- |
| Balance outstanding (max at any point) | ₹10,000 | ₹2,00,000 | ⚠️ TO VERIFY — PPI-MD Cl. 9 |
| Monthly load (cumulative) | ₹10,000 | ₹2,00,000 | ⚠️ TO VERIFY — PPI-MD Cl. 9 |
| Annual load cap (Small PPI only) | ₹1,20,000/year | N/A | ⚠️ TO VERIFY — PPI-MD Cl. 9.2 |
| P2P transfer to another PPI holder | Prohibited | ₹1,00,000/month aggregate | ⚠️ TO VERIFY — PPI-MD Cl. 15 |
| P2P transfer to bank account | Prohibited | ₹1,00,000/month aggregate | ⚠️ TO VERIFY — PPI-MD Cl. 15 |
| Cash withdrawal from Full-KYC PPI | Not allowed | ₹2,000/txn; ₹10,000/month | ⚠️ TO VERIFY — PPI-MD |
| KYC validity (Small PPI) | 12 months, then 60-day grace, then expiry | — | ⚠️ TO VERIFY — PPI-MD Cl. 9.2 |
| KYC validity (Full-KYC) | — | Aligned with bank KYC policy (RBI KYC-MD) | ⚠️ TO VERIFY |

## NCMC (National Common Mobility Card)

| Rule | Value | Source |
| --- | --- | --- |
| Balance cap | ₹3,000 | ⚠️ TO VERIFY — RBI NCMC framework |
| Offline transaction cap (per txn) | ₹200 | ⚠️ TO VERIFY — RBI NCMC framework |
| Offline aggregate before online re-auth | ₹2,000 | ⚠️ TO VERIFY — RBI NCMC framework |
| Category scope | Transit-only (closed loop for interoperable fare media) | ⚠️ TO VERIFY |

## FASTag

| Rule | Value | Source |
| --- | --- | --- |
| Minimum security deposit (per vehicle, used in this reference impl) | ₹300 | ⚠️ TO VERIFY — NPCI NETC guidelines; may differ by vehicle class |
| Minimum threshold balance for toll transactions | ₹150 typical | ⚠️ TO VERIFY — NPCI NETC; vehicle-class-dependent |

## Sub-Wallet Monthly Limits (Reference Implementation Choices)

These are **product choices** by this reference implementation, not regulatory mandates. Documented here for completeness; source is the platform's own policy.

| Sub-wallet | Monthly limit | Notes |
| --- | --- | --- |
| Food | ₹3,000 | Employer-loaded; eligible at Restaurants / Cafes / Food delivery / Swiggy / Zomato / Canteen |
| Fuel | ₹2,500 | Employer-loaded; eligible at HP / IOCL / BPCL / Shell |
| Gift | No hard cap | Self + employer; subject to combined-balance cap with expiry |
| NCMC | ₹3,000 balance (not monthly) | See NCMC section above — this is an RBI-derived cap |
| FASTag deposit | ₹300/vehicle | See FASTag section above |

## PMLA / AML Reporting Thresholds

| Rule | Value | Source |
| --- | --- | --- |
| STR (Suspicious Transaction Report) | Any suspicious txn, no amount floor | ⚠️ TO VERIFY — PMLA Rule 3 |
| CTR (Cash Transaction Report) | Single cash txn > ₹10L, or connected txns aggregating > ₹10L in a month | ⚠️ TO VERIFY — PMLA Rule 3 |
| STR filing deadline | Within 7 working days of establishing suspicion | ⚠️ TO VERIFY — PMLA Rules |
| EDD (Enhanced Due Diligence) trigger | ₹50,000/month load (reference-impl choice) | Implementation choice; not an RBI-set amount |
| Record retention — transaction ledger | 5 years from transaction date | ⚠️ TO VERIFY — PMLA Rules |
| Record retention — KYC artefacts | 10 years | ⚠️ TO VERIFY — KYC-MD |

## DPDP Act 2023

| Rule | Value | Source |
| --- | --- | --- |
| Consent requirement | Specific, informed, unambiguous, affirmative | ⚠️ TO VERIFY — DPDP Act §6 |
| Data principal rights | Access, correction, erasure, grievance | ⚠️ TO VERIFY — DPDP Act §11-14 |
| Data fiduciary registration | Required if notified as Significant Data Fiduciary | ⚠️ TO VERIFY — DPDP Act §10 |
| Breach notification | To Data Protection Board and affected data principals | ⚠️ TO VERIFY — DPDP Act §8(6) |

## Reference-Implementation Implementation Choices

These document where our code diverges from the regulatory ceiling — intentionally. Not gaps; design choices.

| Implementation constant | Value | Regulatory ceiling | Rationale |
| --- | --- | --- | --- |
| `BALANCE_CAP_PAISE` in Load Guard | ₹1,00,000 (100,00,000 paise) | ₹2,00,000 Full-KYC | Conservative baseline for the reference implementation. Tighter than RBI to reduce blast radius if code is ever run against real money. Documented in `.claude/rules/compliance.md`. |
| `MONTHLY_LOAD_LIMIT_PAISE` | ₹2,00,000 | ₹2,00,000 Full-KYC | Matches regulation. |
| `MIN_KYC_BALANCE_CAP_PAISE` | ₹10,000 | ₹10,000 Small PPI | Matches regulation. |
| FASTag security deposit | ₹300 / vehicle | Vehicle-class-dependent per NETC | Single flat value used for demo simplicity. Real FASTag uses per-class minimums. |

---

## How to Use This Document

**When writing or updating any doc that mentions a regulatory limit:**
1. Look up the value here.
2. Link to this doc. Do not restate the number unless you also need to say where it comes from.
3. If the number here is wrong, update it here first, then update the dependent docs.
4. Add an entry to the Change Log below.

**When reviewing a PR that touches compliance numbers:**
1. Check every changed number against this doc.
2. Verify this doc's values against the current RBI / PMLA / DPDP text.
3. If this doc itself is being updated, require a source-paragraph reference for every change.

## Change Log

| Date | Change | Verified By | Circular Ref |
| --- | --- | --- | --- |
| 2026-04-17 | Initial authoritative table created. All entries marked ⚠️ TO VERIFY pending primary-source check. | — (scaffolded, not verified) | — |
