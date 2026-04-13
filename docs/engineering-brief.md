**PPSL PPI Wallet**

Engineering Brief

API Specification · Database Schema · Escrow Reconciliation Engine

| Document Type | Engineering Brief — Claude Code Ready |
| --- | --- |
| Product | PPSL PPI Wallet (New Build) |
| Entity | Paytm Payment Services Limited (PPSL) |
| Audience | Engineering leads, Backend engineers, DevOps, Security |
| Version | 1.0 — March 2026 |
| Based On | PPI_Wallet_Product_Requirements_v1.1.docx |
| Compliance Ref | Compliance_Gap_Analysis_v2.docx |
| Classification | Confidential — Internal Use Only |

Purpose  This brief translates the PPSL PPI Wallet PRD (v1.1) into an engineering-ready specification across three dimensions: (1) a complete REST API contract with compliance annotations, (2) a canonical database schema covering all 18 product blocks, and (3) a step-by-step escrow reconciliation engine design. Each section includes ready-to-use Claude Code prompts so engineers can generate production-quality stubs directly from this document.

⚠ Pre-Build Gate  This brief is for planning and code generation ONLY. Engineering must not begin Sprint 1 until all P0 compliance blockers are cleared: (1) RBI PPI license held by PPSL, (2) FIU-IND registration as PMLA Reporting Entity, (3) partner bank escrow agreement signed, (4) NPCI TPAP membership application filed, (5) legal opinion on PA vs PPI escrow separation obtained. See Compliance_Gap_Analysis_v2.docx for full pre-build checklist.

| 01 | How To Use This Brief With Claude Code |
| --- | --- |

This document is structured as Claude Code-ready engineering specifications. Each sub-section contains a Claude Code starter prompt that engineers can paste directly into Claude Code (or Cursor) to generate production-quality boilerplate. Follow the workflow below for each module:

| Step | Action | Detail |
| --- | --- | --- |
| 1 | Read the API contract | Each endpoint table shows method, path, auth requirement, and the exact compliance clause it must satisfy. |
| 2 | Run the Claude Code prompt | Copy the starter prompt verbatim into Claude Code. Add your framework and language preference as a prefix (e.g. 'Using Node.js + Fastify + Prisma...'). |
| 3 | Validate schema & migrations | Run schema rows through your ORM migration generator. Cross-check NOT NULL / foreign keys against the Nullable column. |
| 4 | Add compliance assertions | Every endpoint marked with an RBI clause reference must have a corresponding integration test asserting the compliance behaviour (e.g. P2P blocked for Min-KYC, balance cap enforcement). |
| 5 | Wire to ledger-first pattern | Every mutation endpoint that moves money (POST /wallet/debit, POST /money/*) must debit the ledger BEFORE any external API call. This is non-negotiable. See Section 3. |

### Technology Stack Assumptions

The API contracts and schema designs in this brief are stack-agnostic. The Claude Code prompts assume the following defaults — override in your prompt prefix if different:

- Runtime: Node.js 20 LTS (TypeScript 5.x) — override to Java/Spring Boot or Python/FastAPI as needed
- ORM / Query Builder: Prisma 5 (PostgreSQL dialect) — schema definitions use Prisma DSL
- Database: PostgreSQL 16 (primary), Redis 7 (velocity counters, idempotency cache, session store)
- API Style: REST + OpenAPI 3.1 — gRPC for internal service-to-service calls
- Auth: JWT (RS256) for consumer-facing APIs, mTLS for bank/escrow integrations, API key for internal services
- Message Queue: Kafka 3.x (event sourcing backbone, recon triggers, async enrichment)
- Infra: Kubernetes (GKE or EKS), Terraform, ArgoCD
- Observability: OpenTelemetry → Grafana/Tempo; structured JSON logs → Loki; PagerDuty alerting

| 02 | REST API Specification |
| --- | --- |

The following endpoint tables cover all P0 and P1 blocks from the PRD. Endpoints are grouped by service domain. The Compliance column cites the specific RBI directive clause that each endpoint is designed to satisfy.

## 2.1  Wallet Core & Ledger Service

Claude Code Prompt  Generate a production-ready TypeScript + Fastify service for a PPI wallet ledger. The service must implement a double-entry append-only ledger. POST /wallet/credit and POST /wallet/debit must be wrapped in a database transaction that inserts a ledger_entries row BEFORE returning. Implement idempotency via a unique constraint on (idempotency_key). GET /wallet/balance must derive balance from SUM(credits) - SUM(debits) — never store balance as a mutable column. Include Zod request validation, Prisma ORM, and a Kafka event emit on each mutation. Add OpenAPI 3.1 annotations.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /wallet/credit | Credit wallet balance — add e-money to wallet. Atomic ledger insert. | JWT + HMAC | PPI-MD Cl.8 — e-money issuance record |
| POST | /wallet/debit | Debit wallet balance — reduce e-money. Ledger-first; external call follows. | JWT + HMAC | PPI-MD Cl.8 — audit trail mandatory |
| GET | /wallet/balance/{id} | Return real-time derived balance (SUM credits - debits). Never cached stale state. | JWT | PPI-MD Cl.9 — balance cap enforcement |
| GET | /wallet/ledger/{id} | Paginated transaction history with debit/credit/balance running total. | JWT | PPI-MD Cl.18 — 5yr audit retention |
| POST | /wallet/hold | Reserve funds for pending transaction. Creates a HOLD ledger entry. | JWT + HMAC | Saga pattern — prevents double-spend |
| POST | /wallet/release | Release a previously held amount back to available balance. | JWT + HMAC | Saga compensating transaction |
| POST | /wallet/idempotency/check | Check if idempotency key was already processed. Returns prior result if so. | API Key | Prevents duplicate money movement |

## 2.2  KYC & Identity Service

Claude Code Prompt  Build a KYC orchestrator service in TypeScript. Implement a state machine with states: UNVERIFIED → MIN_KYC → FULL_KYC_PENDING → FULL_KYC → REJECTED → SUSPENDED. POST /kyc/aadhaar/otp-verify must call the UIDAI OTP verification API and on success transition the wallet to MIN_KYC. POST /kyc/video-kyc/session must: (1) call CERSAI CKYC API to check if KYC record exists before initiating video KYC, (2) if record exists, accept it and skip video KYC. On Full-KYC completion, upload KYC record to CERSAI CKYC registry. All KYC documents must be encrypted at rest (AES-256). PAN must be masked in all logs. Set wallet_expiry_date = now + 12 months for MIN_KYC wallets.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /kyc/initiate | Start KYC flow for a user. Returns KYC session ID and required document list. | JWT | PPI-MD Cl.9 — KYC tier determines limits |
| POST | /kyc/aadhaar/otp-send | Trigger Aadhaar OTP via UIDAI eKYC API. | JWT | UIDAI circular — consent logging required |
| POST | /kyc/aadhaar/otp-verify | Verify OTP + eKYC data. Transitions wallet to MIN_KYC on success. | JWT | PPI-MD Cl.9.2 — Min-KYC upgrade |
| POST | /kyc/pan-verify | Verify PAN via NSDL/UTIITSL API. Required for Full-KYC. | JWT | PMLA — identity verification |
| POST | /kyc/video-kyc/session | Initiate Video KYC session. Must check CKYC registry first. | JWT | RBI KYC-MD Oct 2023 — CKYC lookup mandatory |
| POST | /kyc/ckyc/upload | Upload completed KYC record to CERSAI CKYC registry. | mTLS | RBI KYC-MD — upload mandatory post Full-KYC |
| GET | /kyc/status/{user_id} | Return current KYC state machine status + tier. | JWT | PPI-MD Cl.9 — tier governs limit engine |
| GET | /kyc/tier/{user_id} | Return MIN_KYC / FULL_KYC / UNVERIFIED enum. | API Key | Internal use by Limits Engine |
| PATCH | /kyc/suspend/{user_id} | Suspend KYC (AML hold). Blocks all wallet operations. | Admin | PMLA Sec.12A — freeze on suspicion |

## 2.3  PPI Limits & Compliance Engine

Claude Code Prompt  Build a stateless limits enforcement service that evaluates every money movement request before the ledger write. The service must: (1) return HTTP 403 with PPI_P2P_NOT_ALLOWED if a MIN_KYC wallet attempts a P2P transfer — this is a hard RBI regulatory block, not a configurable limit. (2) Enforce monthly P2P cap of Rs.1,00,000 for FULL_KYC wallets and Rs.0 for MIN_KYC. (3) Enforce wallet balance ceiling of Rs.10,000 for MIN_KYC and Rs.2,00,000 for FULL_KYC. (4) Track annual load totals against Rs.1,20,000 annual cap for MIN_KYC wallets. Store all limit configs in PostgreSQL with effective_from versioning — never hardcode in application code. Redis counters for real-time monthly tracking with TTL = end of calendar month.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /limits/check | Evaluate a money movement against all applicable limits. Returns ALLOW or DENY + reason code. | API Key | PPI-MD Cl.9 — limit enforcement |
| GET | /limits/config/{kyc_tier} | Return current limit config for a KYC tier (balance ceiling, monthly P2P, annual load). | API Key | PPI-MD Cl.9.2/9.3 |
| GET | /limits/usage/{wallet_id} | Return current MTD and YTD usage counters for a wallet. | JWT | PPI-MD Cl.9 — usage tracking |
| POST | /limits/config | Create or update a limit config with effective_from date. Requires 4-eye approval in audit log. | Admin | PPI-MD Cl.9 — hot-reload without deploy |
| GET | /compliance/wallet-types | Return supported PPI types (MIN_KYC, FULL_KYC, GIFT, CORPORATE) with properties. | API Key | PPI-MD Cl.2 — PPI type definitions |
| GET | /compliance/interop-status/{id} | Return whether a wallet is eligible for UPI interoperability. | JWT | PPI-MD Cl.10 + Dec 2024 amendment |

## 2.4  Money Movement Service

Claude Code Prompt  Generate a Saga-orchestrated money movement service. Each transaction type (add_money, merchant_pay, p2p_transfer, wallet_to_bank, bill_pay, refund) must be a Saga with defined compensating transactions. The Saga must: (1) call POST /limits/check before any ledger operation, (2) call POST /wallet/debit (ledger first) before initiating any external payment rail, (3) emit a Kafka event on completion or failure. For P2P transfers: validate that the initiating wallet is FULL_KYC or return HTTP 403 PPI_P2P_NOT_ALLOWED. For refunds: the refund must credit back to the original wallet (not the source instrument), and must be idempotent — the same refund_ref must never debit the merchant wallet twice. Include dead-letter queue handling for failed Sagas.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /money/add | Load money into wallet from UPI/card/netbanking. Ledger-first pattern. | JWT | PPI-MD Cl.20 — permitted load methods |
| POST | /money/spend | Wallet spend at merchant QR / payment link / in-app. | JWT+HMAC | PPI-MD Cl.21 — merchant acceptance |
| POST | /money/transfer-p2p | Wallet-to-wallet P2P. FULL_KYC only. Monthly cap enforced. | JWT+HMAC | PPI-MD Cl.15.1 — P2P prohibition for Min-KYC |
| POST | /money/withdraw | Wallet-to-bank withdrawal. Debits nodal account. FULL_KYC only. | JWT+HMAC | PPI-MD Cl.22 — withdrawal rules |
| POST | /money/bill-pay | BBPS bill payment from wallet. Uses PPSL BBPOU membership. | JWT | BBPS operating guidelines |
| POST | /refund/initiate | Initiate merchant refund back to wallet. Idempotent on refund_ref. | API Key | PPI-MD Cl.22 — refund to source |
| GET | /refund/status/{ref} | Return refund saga state + leg details. | JWT | Audit trail |
| GET | /money/transaction/{id} | Full transaction detail with all saga legs and compliance metadata. | JWT | PPI-MD Cl.18 — 5yr retention |

## 2.5  Escrow & Nodal Bank Service

Claude Code Prompt  Build an escrow/nodal account management service. The service must: (1) integrate with the partner bank's API for real-time balance feeds and statement ingestion (daily SFTP + intraday API), (2) expose GET /nodal/balance returning the live nodal balance, (3) compute float_utilisation = SUM(all active wallet balances) / nodal_balance and emit a Kafka alert event when utilisation crosses 90% and 95%, (4) at 95% utilisation, call POST /feature-flags/disable with key=add_money to block new wallet loads, (5) POST /nodal/recon/trigger initiates the daily reconciliation Saga, (6) all nodal transactions must be written to a float_ledger table with double-entry bookkeeping mirroring the wallet ledger. The PA-PG escrow and PPI nodal accounts are SEPARATE bank accounts — never commingle funds.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /nodal/debit | Initiate debit from nodal account (e.g. wallet-to-bank withdrawal). | mTLS | PPI-MD Cl.7 — escrow debit rules |
| POST | /nodal/credit | Credit nodal account (e.g. Add Money inflow from payment rail). | mTLS | PPI-MD Cl.7 — escrow credit |
| GET | /nodal/balance | Live nodal account balance from bank feed. Returns balance + float_utilisation %. | Admin | PPI-MD Cl.7.2 — float adequacy |
| GET | /nodal/statement/{date} | Parsed bank statement for a given date. Normalised to internal transaction format. | Admin | RBI recon mandate |
| POST | /nodal/recon/trigger | Start daily reconciliation run for a given date. Async — returns run_id. | Admin | PPI-MD Cl.7 — daily recon mandatory |
| GET | /nodal/recon/status/{date} | Return recon run status: RUNNING / COMPLETE / BREAKS_FOUND + break count. | Admin | Compliance dashboard feed |
| GET | /nodal/recon/breaks/{date} | List all reconciliation breaks for a date with category and amount. | Admin | Finance ops resolution workflow |
| POST | /nodal/float-alert | Internal: float monitor calls this when utilisation threshold crossed. | API Key | PPI-MD Cl.7.2 — float breach SOP |

## 2.6  UPI Integration & VPA Management

Claude Code Prompt  Build a UPI / VPA management service under PPSL's NPCI TPAP membership. The service must: (1) POST /upi/vpa/create generates a UPI handle in format user@ppsl and registers it with NPCI via the TPAP API, (2) GET /upi/vpa/{wallet_id} returns the wallet's linked VPA, (3) POST /upi/collect handles inbound UPI collect requests — validates the wallet is FULL_KYC before accepting, (4) POST /upi/third-party/link enables a Full-KYC wallet to be linked as a funding source in third-party UPI apps via NPCI inter-PSP protocol (Dec 2024 RBI amendment compliance), (5) all UPI transactions must produce ledger entries, (6) implement 2FA for UPI transactions per RBI Cl.14. FULL_KYC is mandatory for all UPI operations.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /upi/vpa/create | Create and register UPI VPA (user@ppsl) with NPCI TPAP. FULL_KYC required. | JWT | NPCI TPAP guidelines — VPA registration |
| GET | /upi/vpa/{wallet_id} | Return linked VPA for a wallet. | JWT | UPI interoperability |
| DELETE | /upi/vpa/{wallet_id} | Deregister VPA — called on wallet closure or KYC downgrade. | JWT | PPI-MD Cl.10 — interop management |
| POST | /upi/collect | Accept inbound UPI collect request. Validate FULL_KYC, limits, 2FA. | JWT+OTP | PPI-MD Cl.14 — 2FA mandatory |
| POST | /upi/pay | Initiate outbound UPI payment from wallet to VPA. | JWT+OTP | PPI-MD Cl.14 + Cl.21 |
| POST | /upi/third-party/link | Enable wallet as funding source in third-party UPI apps (inter-PSP). | JWT | Dec 2024 RBI amendment — 3rd party link |
| DELETE | /upi/third-party/unlink | Revoke third-party UPI app access for this wallet. | JWT | Customer consent management |
| GET | /upi/mandate/list/{id} | List active UPI AutoPay mandates for a wallet. | JWT | NPCI AutoPay guidelines |
| DELETE | /upi/mandate/{mandate_id} | Revoke a UPI AutoPay mandate. | JWT | Customer control requirement |

## 2.7  AML, Reporting & FIU-IND Integration

Claude Code Prompt  Build a compliance reporting service for FIU-IND FINNET 2.0 integration. The service must: (1) automatically aggregate daily Cash Transaction Reports (CTR) — any wallet load from cash equivalent instrument > Rs.10,000 in a day, (2) generate Suspicious Transaction Reports (STR) when the risk engine raises a suspicion flag — must be filed within 7 days of suspicion arising, (3) file reports via FIU-IND FINNET 2.0 portal API with PPSL's registered Reporting Entity credentials, (4) maintain immutable 10-year retention of all KYC and transaction records per PMLA Section 12, (5) POST /reports/rbi-monthly generates the monthly PPI statistical return for RBI (due 10th of following month).

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| POST | /reports/ctr | Generate and file Cash Transaction Report with FIU-IND FINNET 2.0. | Admin | PMLA Sec.12 — CTR filing |
| POST | /reports/str | Generate and file Suspicious Transaction Report within 7 days of suspicion. | Admin | PMLA Sec.12A — STR filing |
| GET | /reports/status/{report_id} | Check FIU-IND acknowledgement status for a filed report. | Admin | FINNET 2.0 filing confirmation |
| POST | /reports/rbi-monthly | Generate monthly PPI statistical return for RBI submission. | Admin | RBI PPI-MD Cl.29 — monthly reporting |
| GET | /reports/audit-trail/{id} | Immutable audit log for a wallet — 10yr retention. Admissible evidence format. | Admin | PMLA Sec.12 — 10yr record retention |
| POST | /reports/edd-flag/{user_id} | Trigger Enhanced Due Diligence workflow for high-risk customer. | Admin | RBI KYC-MD — EDD for PEP/HNI/sanctioned |

## 2.8  Wallet Lifecycle & Notifications

Claude Code Prompt  Build a wallet lifecycle state machine service with states: ACTIVE, SUSPENDED, DORMANT, EXPIRED, CLOSED. Implement: (1) automated dormancy transition — wallet not used for 1 year transitions to DORMANT (per PPI-MD Cl.19); dormant wallet can be reactivated with OTP, (2) MIN_KYC wallet expiry — wallet_expiry_date = created_at + 12 months; at T-30, T-7, T-1 days send upgrade nudge notifications via POST /notifications/send; on expiry_date transition to EXPIRED and block all transactions, (3) wallet closure — on CLOSED, initiate balance refund to the linked bank account if available, else hold as unclaimed balance with 60-day grace period, (4) all state transitions must produce a wallet_state_history entry with timestamp, reason, and operator_id.

| Method | Endpoint | Description | Auth | Compliance |
| --- | --- | --- | --- | --- |
| GET | /wallet/status/{id} | Return wallet state (ACTIVE/DORMANT/SUSPENDED/EXPIRED/CLOSED) + flags. | JWT | PPI-MD Cl.18/19 — lifecycle |
| POST | /wallet/reactivate/{id} | Reactivate dormant wallet after OTP verification. | JWT+OTP | PPI-MD Cl.19 — dormancy reactivation |
| POST | /wallet/close/{id} | Initiate wallet closure. Triggers balance refund or unclaimed hold. | JWT | PPI-MD Cl.18 — closure + balance return |
| POST | /wallet/upgrade-kyc/{id} | Trigger KYC upgrade flow from MIN_KYC to FULL_KYC. | JWT | PPI-MD Cl.9.2 — upgrade path |
| GET | /wallet/expiry/{id} | Return wallet expiry date and days remaining. | JWT | PPI-MD Cl.9.2 — 12-month MIN_KYC window |
| POST | /notifications/send | Send notification event to communication engine (SMS/push/email). | API Key | PPI-MD Cl.24 — transaction alerts mandatory |

| 03 | Database Schema Design |
| --- | --- |

The schema is organised into five domains: Core Ledger, Identity & KYC, Money Movement, Escrow & Nodal, and Compliance. All tables use UUID v4 primary keys, created_at / updated_at timestamps, and soft-delete via deleted_at. Fields marked N (Not Null) in the schema tables enforce database-level constraints.

Claude Code Prompt — Generate All Migrations  Generate Prisma schema definitions and PostgreSQL migration files for all tables listed in this section. Requirements: (1) all PKs are uuid (gen_random_uuid()), (2) all monetary amounts are stored as BIGINT in paise (1/100 rupee) — never DECIMAL or FLOAT, (3) status fields use PostgreSQL ENUMs, (4) all tables have created_at TIMESTAMPTZ DEFAULT now(), updated_at TIMESTAMPTZ DEFAULT now(), deleted_at TIMESTAMPTZ (nullable for soft delete), (5) add row-level security (RLS) policies on tables containing PII, (6) generate indexes for all foreign key columns and high-cardinality filter columns (wallet_id, user_id, status, created_at), (7) all KYC document fields must be encrypted using pgcrypto.

## 3.1  Core Ledger Domain

### wallet_accounts

Master wallet record — one row per wallet issuance. Balance is never stored here; it is derived from ledger_entries.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key — wallet identifier. |  |
| user_id | uuid | N | FK → user_profiles.id — owner of the wallet. | PPI-MD — single wallet per KYC tier per user |
| wallet_type | enum | N | MIN_KYC \| FULL_KYC \| GIFT \| CORPORATE | PPI-MD Cl.2 — PPI type classification |
| status | enum | N | ACTIVE \| SUSPENDED \| DORMANT \| EXPIRED \| CLOSED | PPI-MD Cl.18/19 — lifecycle states |
| kyc_tier | enum | N | UNVERIFIED \| MIN_KYC \| FULL_KYC | PPI-MD Cl.9 — tier governs all limits |
| currency | char(3) | N | ISO currency code. Always 'INR' for PPI. |  |
| wallet_expiry_at | timestamptz | Y | NULL for FULL_KYC. Set to +12 months for MIN_KYC at creation. | PPI-MD Cl.9.2 — 12-month MIN_KYC window |
| issuer_id | uuid | N | FK → ppsl_entity — legal issuer (PPSL entity reference). | PPI-MD Cl.4 — issuer identity |
| vpa | varchar(64) | Y | UPI VPA string e.g. user@ppsl. NULL until FULL_KYC. | NPCI TPAP — VPA registration |
| last_txn_at | timestamptz | Y | Timestamp of last transaction — used for dormancy detection. | PPI-MD Cl.19 — 1yr dormancy threshold |
| created_at | timestamptz | N | Wallet creation timestamp. |  |
| updated_at | timestamptz | N | Last update timestamp (auto-update trigger). |  |
| deleted_at | timestamptz | Y | Soft delete — set on CLOSED. |  |

### ledger_entries

Append-only immutable double-entry ledger. Every money movement produces exactly two rows — one debit (DEBIT) and one credit (CREDIT) in a single atomic transaction. Balance is derived from SUM at query time.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key — entry identifier. |  |
| wallet_id | uuid | N | FK → wallet_accounts.id — affected wallet. | PPI-MD Cl.8 — e-money issuance audit |
| txn_id | uuid | N | FK → transaction_intents.id — parent transaction. |  |
| entry_type | enum | N | DEBIT \| CREDIT \| HOLD \| RELEASE | Double-entry bookkeeping |
| amount_paise | bigint | N | Amount in paise (1/100 INR). Always positive. | Never store as DECIMAL/FLOAT |
| running_balance | bigint | N | Running balance after this entry (for reporting). | Derived but stored for performance |
| idempotency_key | varchar(255) | N | Unique constraint — prevents duplicate ledger entries. | Idempotency guarantee |
| reference_id | varchar(255) | Y | External reference (UPI UTR, bank ref, NPCI txn ID). | Reconciliation anchor |
| narration | text | Y | Human-readable description for ledger statement. | PPI-MD Cl.18 — transaction statement |
| created_at | timestamptz | N | Entry creation timestamp. Immutable after insert. | PPI-MD Cl.18 — 5yr audit retention |

### transaction_intents

Orchestrator-layer record of every attempted money movement — including failed and compensated transactions. A transaction intent has one or more ledger_entries.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key. |  |
| wallet_id | uuid | N | Initiating wallet. |  |
| txn_type | enum | N | ADD_MONEY \| SPEND \| P2P \| WITHDRAW \| BILL_PAY \| REFUND |  |
| status | enum | N | PENDING \| PROCESSING \| COMPLETED \| FAILED \| COMPENSATED | Saga orchestration states |
| amount_paise | bigint | N | Requested amount in paise. |  |
| fee_paise | bigint | N | Platform fee in paise. 0 if no fee. |  |
| counterparty_id | varchar(255) | Y | VPA, account number, merchant ID, or wallet ID of counterparty. |  |
| idempotency_key | varchar(255) | N | Client-generated unique key. Unique constraint. | Idempotency — prevents double execution |
| metadata | jsonb | Y | Arbitrary saga context (instrument details, risk score, etc). |  |
| kyc_tier_at_txn | enum | N | KYC tier of initiating wallet at transaction time. | Immutable audit — tier may change later |
| risk_score | smallint | Y | Risk score (0-100) from risk engine at transaction time. | PPI-MD Cl.27 — fraud monitoring |
| created_at | timestamptz | N | Creation timestamp. |  |
| completed_at | timestamptz | Y | NULL until terminal state reached. |  |

## 3.2  Identity & KYC Domain

### user_profiles

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key. |  |
| mobile | varchar(15) | N | E.164 format mobile number. Unique index. | PPI-MD — mobile as primary identifier |
| name_encrypted | bytea | Y | AES-256 encrypted full name. | PMLA — PII encryption at rest |
| dob_encrypted | bytea | Y | AES-256 encrypted date of birth. | PMLA — PII encryption |
| pan_hash | varchar(64) | Y | SHA-256 hash of PAN — for deduplication only. Never store raw. | PMLA — PAN deduplication |
| aadhaar_token | varchar(72) | Y | UIDAI virtual ID token — never store actual Aadhaar number. | UIDAI — VID tokenisation mandatory |
| ckyc_number | varchar(14) | Y | 14-digit CKYC record number from CERSAI registry. | RBI KYC-MD — CKYC lookup & upload |
| kyc_tier | enum | N | UNVERIFIED \| MIN_KYC \| FULL_KYC \| SUSPENDED | PPI-MD Cl.9 — tier drives all limits |
| risk_band | enum | N | LOW \| MEDIUM \| HIGH \| PEP \| SANCTIONED | RBI KYC-MD — customer risk classification |
| edd_required | boolean | N | True if Enhanced Due Diligence workflow triggered. | RBI KYC-MD — EDD trigger |
| created_at | timestamptz | N | Registration timestamp. |  |
| kyc_updated_at | timestamptz | Y | Last KYC tier transition timestamp. |  |
| deleted_at | timestamptz | Y | Soft delete — never hard delete per PMLA 10yr retention. | PMLA Sec.12 — 10yr retention |

### kyc_submissions

Each KYC attempt (OTP, PAN, Video KYC) creates a row. Multiple submissions per user are allowed.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key. |  |
| user_id | uuid | N | FK → user_profiles.id |  |
| kyc_type | enum | N | AADHAAR_OTP \| PAN \| VIDEO_KYC \| CKYC_FETCH |  |
| status | enum | N | PENDING \| SUCCESS \| FAILED \| EXPIRED |  |
| result_tier | enum | Y | Target KYC tier if successful. NULL if failed. |  |
| provider | varchar(50) | Y | UIDAI \| NSDL \| UTIITSL \| CERSAI — upstream provider. |  |
| provider_ref | varchar(255) | Y | Upstream reference ID for the verification call. | Audit trail |
| consent_text | text | N | Verbatim consent text shown to user. | UIDAI — consent logging mandatory |
| consent_at | timestamptz | N | Timestamp of user consent capture. | UIDAI — consent timestamp |
| created_at | timestamptz | N | Submission timestamp. | PMLA — 10yr retention |
| expires_at | timestamptz | Y | OTP/session expiry timestamp. |  |

## 3.3  Escrow & Nodal Domain

### nodal_accounts

Registry of nodal/escrow bank accounts. PPSL must maintain separate accounts for PPI float (this table) and PA-PG merchant float. Commingling is a regulatory violation.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key. |  |
| account_type | enum | N | PPI_NODAL \| PA_ESCROW — must never be commingled. | PPI-MD Cl.7 + PA-PG Guidelines |
| bank_name | varchar(100) | N | Partner bank name (e.g. Yes Bank, IndusInd). |  |
| account_number | varchar(20) | N | Bank account number. Encrypted at rest. |  |
| ifsc_code | char(11) | N | Bank IFSC code. |  |
| current_balance | bigint | N | Last known nodal balance in paise from bank feed. | PPI-MD Cl.7.2 — float adequacy |
| balance_as_of | timestamptz | N | Timestamp of last balance refresh from bank. |  |
| float_utilisation | numeric(5,2) | N | SUM(wallet balances) / current_balance × 100. Computed field. | PPI-MD Cl.7.2 — float breach SOP |
| alert_threshold_pct | smallint | N | Alert when float_utilisation exceeds this %. Default 90. |  |
| block_threshold_pct | smallint | N | Block new Add Money when float_utilisation exceeds this %. Default 95. |  |
| is_active | boolean | N | Only one PPI_NODAL account should be active at a time. |  |
| created_at | timestamptz | N | Account onboarding timestamp. |  |

### recon_runs

One row per daily reconciliation execution. Stores run metadata and aggregate break counts.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key. |  |
| recon_date | date | N | The business date being reconciled. Unique constraint. | PPI-MD Cl.7 — daily recon |
| status | enum | N | RUNNING \| COMPLETE \| BREAKS_FOUND \| FAILED |  |
| wallet_system_total | bigint | N | Sum of all wallet balances from ledger_entries (paise). | Internal source of truth |
| nodal_bank_total | bigint | N | Nodal account balance per bank statement (paise). | External source of truth |
| variance_paise | bigint | N | ABS(wallet_system_total - nodal_bank_total). | Must be 0 at day end |
| break_count | integer | N | Number of individual recon breaks found. |  |
| started_at | timestamptz | N | Run start timestamp. |  |
| completed_at | timestamptz | Y | Run completion timestamp. NULL if still running. |  |
| initiated_by | varchar(100) | N | Operator ID or 'SCHEDULER' for automated runs. | Audit trail |

### recon_breaks

Individual reconciliation discrepancy records. Each break requires a resolution action before the next recon cycle.

| Field | Type | Null? | Description | Compliance |
| --- | --- | --- | --- | --- |
| id | uuid | N | Primary key. |  |
| recon_run_id | uuid | N | FK → recon_runs.id |  |
| break_type | enum | N | TXNS_IN_LEDGER_MISSING_IN_BANK \| TXNS_IN_BANK_MISSING_IN_LEDGER \| AMOUNT_MISMATCH \| TIMING_DIFFERENCE |  |
| ledger_ref | varchar(255) | Y | Ledger entry ID or transaction intent ID for the break. |  |
| bank_ref | varchar(255) | Y | Bank statement reference for the break. |  |
| amount_paise | bigint | N | Discrepant amount in paise. |  |
| description | text | Y | Auto-generated break description. |  |
| resolution | enum | Y | NULL \| TIMING \| MANUAL_MATCH \| ESCALATED \| WRITTEN_OFF |  |
| resolved_by | varchar(100) | Y | Operator who resolved the break. |  |
| resolved_at | timestamptz | Y | Resolution timestamp. |  |
| created_at | timestamptz | N | Break detection timestamp. |  |

| 04 | Escrow Reconciliation Engine Design |
| --- | --- |

The reconciliation engine is the most compliance-critical component of the escrow layer. Its mandate is to guarantee that the total outstanding e-money across all PPSL wallets is at all times backed by an equal or greater balance in the PPSL PPI nodal account. A float shortfall is a violation of RBI PPI-MD Clause 7.2 and must never occur in production. This section specifies the complete engine design.

Claude Code Prompt — Full Engine  Build a daily reconciliation engine as a Saga-orchestrated Kafka consumer service. The engine: (1) subscribes to a recon.trigger Kafka topic — on message, starts a recon run for the given date, (2) Step 1: compute wallet_system_total as SELECT SUM(running_balance) FROM ledger_entries WHERE entry_type='CREDIT' minus entry_type='DEBIT' as of EOD snapshot, (3) Step 2: ingest bank statement for the date via SFTP pickup or bank API call, normalise to internal format, compute nodal_bank_total, (4) Step 3: transaction-level matching — inner join on reference_id; categorise unmatched rows into break types, (5) Step 4: write recon_runs and recon_breaks rows, (6) Step 5: if variance > 0, emit recon.breaks.found Kafka event; if variance = 0, emit recon.clean Kafka event, (7) Step 6: send Slack/PagerDuty alert to Finance ops team if break count > 0 or variance > Rs.1000, (8) all steps must be idempotent — re-running the same date must produce the same result.

## 4.1  Reconciliation Engine — Step-by-Step Flow

| # | Step | Input | Output | On Break |
| --- | --- | --- | --- | --- |
| 1 | Trigger | recon.trigger Kafka message with recon_date | recon_runs row created with status=RUNNING | Duplicate trigger for same date: skip (idempotent) |
| 2 | Wallet Snapshot | ledger_entries table as-of EOD for recon_date | wallet_system_total (paise) — sum of all live wallet balances | Empty ledger: total=0, log warning, continue |
| 3 | Bank Statement Ingestion | SFTP pickup or bank API for recon_date statement | Parsed list of {bank_ref, amount, direction, value_date} rows | SFTP timeout: retry 3x with backoff. Failure: FAILED status, PD alert |
| 4 | Normalisation | Raw bank statement rows | normalised_bank_txns temp table: internal format, duplicates removed | Unrecognised format: escalate to Engineering, abort run |
| 5 | Transaction Matching | ledger_entries + normalised_bank_txns matched on reference_id | Matched set + unmatched residuals on each side | Low match rate (<80%): alert Finance + Engineering immediately |
| 6 | Break Classification | Unmatched residuals from Step 5 | recon_breaks rows with break_type categorised | Amount mismatch >Rs.1,000: ESCALATED status, immediate PD alert |
| 7 | Aggregate Totals | Matched + break amounts | wallet_system_total, nodal_bank_total, variance_paise computed | variance_paise > 0: recon status=BREAKS_FOUND |
| 8 | Auto-Resolve Timing Diffs | recon_breaks with break_type=TIMING_DIFFERENCE | If same txn appears in next-day bank statement: auto-resolve | Timing diff unresolved after T+2: escalate to Finance ops |
| 9 | Persist Results | All computed data | recon_runs updated to COMPLETE or BREAKS_FOUND, recon_breaks inserted | DB write failure: retry, then alert — recon result must persist |
| 10 | Notify & Emit | recon_runs record + break list | Kafka event emitted; Slack/PD alert if breaks > 0 or variance > Rs.1,000 | Notification failure: log, continue — do not fail the recon run itself |

## 4.2  Break Type Taxonomy

The engine classifies every reconciliation discrepancy into one of four categories. Each category has a defined resolution owner and SLA.

| Break Type | Description | Resolution SLA | Owner |
| --- | --- | --- | --- |
| TXNS_IN_LEDGER_MISSING_IN_BANK | Transaction in PPSL ledger but absent from bank statement. Possible delayed settlement or bank processing lag. | T+1 (auto-check); T+3 escalate | Finance Ops → Bank RM if unresolved by T+3 |
| TXNS_IN_BANK_MISSING_IN_LEDGER | Transaction in bank statement but absent from PPSL ledger. CRITICAL — indicates possible fund leakage or phantom credit. | Immediate — same-day escalation | Engineering + Finance + Compliance. PD P1 alert. |
| AMOUNT_MISMATCH | Transaction present on both sides but amounts differ. May indicate fee deduction, FX rounding, or data corruption. | T+1 investigation | Finance Ops investigates; Engineering if data corruption suspected. |
| TIMING_DIFFERENCE | Transaction booked on different dates in ledger vs. bank (cut-off timing). Common for late-night transactions. | Auto-resolved at T+1 check. Manual if T+2. | Automated engine (T+1). Finance Ops manual resolution if T+2. |

## 4.3  Float Monitor — Continuous Alerting

In addition to the daily reconciliation, a real-time float monitor runs every 5 minutes and computes float utilisation. This is separate from the daily recon and provides intraday early warning.

| float-monitor.ts — Float Utilisation Check |
| --- |
| // Float Monitor — runs every 5 minutes via cron async function checkFloatUtilisation() {   const walletTotal   = await db.ledgerEntries.aggregate({ sum: { amount_paise: true } });   const nodalBalance  = await bankApiClient.getLiveBalance(); // or last SFTP refresh   const utilisation   = (walletTotal / nodalBalance) * 100;    await db.nodalAccounts.update({     where: { account_type: 'PPI_NODAL', is_active: true },     data: { float_utilisation: utilisation, balance_as_of: new Date() }   });    if (utilisation >= 100) {     await featureFlags.disable('add_money');         // hard block     await pagerDuty.createIncident({ severity: 'P0',       title: `CRITICAL: Float breach — ${utilisation.toFixed(2)}%` });     await notifyFinance('FLOAT_BREACH', utilisation);   } else if (utilisation >= 95) {     await featureFlags.disable('add_money');         // preventive block     await pagerDuty.createIncident({ severity: 'P1',       title: `Float at ${utilisation.toFixed(2)}% — Add Money blocked` });   } else if (utilisation >= 90) {     await notifyFinance('FLOAT_HIGH', utilisation);  // advisory alert only   } } |

⚠ RBI Compliance Note — Float Breach  A float_utilisation of 100% or above is a violation of RBI PPI-MD Clause 7.2 (escrow adequacy). This must never occur in production. The 95% threshold block provides a 5-percentage-point safety margin. Finance must maintain a pre-arranged overdraft facility with the nodal bank to replenish float within 2 business hours of a 95% alert. Engineering must ensure the Add Money killswitch is tested in staging monthly.

## 4.4  Bank Statement Ingestion — Normalisation Contract

Bank statements arrive either via daily SFTP (file-based, end-of-day) or intraday bank API (webhook or polling). Both formats must be normalised to the internal format before reconciliation.

| bank-statement-normaliser.ts — Interface Contract |
| --- |
| // Target normalised format for all bank statement sources interface NormalisedBankTransaction {   bank_ref:        string;      // Bank's unique transaction reference   ppsl_ref:        string \| null; // PPSL reference if included by bank   amount_paise:    bigint;       // Always positive; direction from entry_type   entry_type:      'CREDIT' \| 'DEBIT';   value_date:      Date;         // Banking date (may differ from booking date)   booking_date:    Date;         // Actual processing date   narration:       string;       // Bank's description   instrument_type: 'NEFT' \| 'RTGS' \| 'IMPS' \| 'UPI' \| 'INTERNAL' \| 'UNKNOWN';   raw:             Record<string, unknown>; // Original source row for audit }  // Statement ingestion must: // 1. Detect duplicate bank_refs within the same statement date // 2. Convert all amounts to paise (multiply by 100) // 3. Validate sum(credits) - sum(debits) matches bank's closing balance // 4. Store raw JSON alongside normalised record for 5yr audit retention |

| 05 | Sprint Plan — Claude Code Delivery Map |
| --- | --- |

The sprint plan maps deliverables to two-week sprints and provides a Claude Code prompt starter for each sprint's primary output. Sprints are designed for parallel execution by a 6–8 person backend team.

Sprint 0 (Pre-Build Gate)  No code is written in Sprint 0. Sprint 0 exists to clear all P0 compliance blockers. Sprint 1 cannot begin until: RBI PPI license confirmed active, FIU-IND registration filed, partner bank escrow agreement signed, NPCI TPAP application lodged, legal opinion on PA/PPI escrow separation obtained, CERSAI CKYC registration initiated.

| Sprint | Deliverables | Claude Code Prompts (starter) | Acceptance Gate |
| --- | --- | --- | --- |
| S1 | Repo setup, CI/CD pipeline, Terraform base infra PostgreSQL + Redis provisioned (dev/staging) Kafka cluster + topics: wallet.events, recon.trigger, recon.breaks.found Prisma migrations: wallet_accounts, ledger_entries, transaction_intents | "Generate Prisma schema + migrations for wallet_accounts, ledger_entries, transaction_intents. All monetary amounts as BIGINT paise. Append-only constraint on ledger_entries (no UPDATE/DELETE). Idempotency unique index on ledger_entries.idempotency_key." | Migration runs clean on fresh DB Duplicate ledger entry rejected (constraint test) |
| S2 | Wallet Core Service: POST /wallet/credit, /debit, GET /balance, /ledger POST /wallet/hold, /release (Saga support) Idempotency cache in Redis (24hr TTL) | "Implement POST /wallet/debit in TypeScript + Fastify. The endpoint must: (1) acquire a DB row-lock on wallet_accounts, (2) call POST /limits/check, (3) insert ledger_entries row in the same DB transaction, (4) emit wallet.debit Kafka event, (5) return 200 only if DB commit succeeds. If limit check fails, return 403 with error code. Never return 200 before DB commit." | Balance derived = SUM ledger (no stored balance column) Concurrent debit stress test: 100 concurrent requests, 0 duplicates in ledger |
| S3 | Limits Engine: all PPI limit rules, Redis MTD/YTD counters P2P hard block for MIN_KYC wallets PPI type registry with hot-reload config | "Build a limits enforcement service. POST /limits/check must be synchronous p99 < 50ms. Use Redis INCR + EXPIRE for MTD counters with TTL = seconds until end of calendar month. Return {decision: 'ALLOW'\|'DENY', reason_code, limit_type, current_usage, limit_value}. MIN_KYC P2P must return DENY with reason_code PPI_P2P_NOT_ALLOWED regardless of amount." | MIN_KYC P2P returns 403 PPI_P2P_NOT_ALLOWED in all scenarios Limit config hot-reload tested without service restart |
| S4 | KYC Service: state machine, UIDAI Aadhaar OTP, PAN verify CERSAI CKYC lookup + upload integration MIN_KYC wallet expiry logic (12-month + expiry scheduler) | "Implement KYC state machine with states UNVERIFIED → MIN_KYC → FULL_KYC_PENDING → FULL_KYC. Integrate CERSAI CKYC API: on POST /kyc/video-kyc/session, call CKYC lookup with PAN. If record found, accept and skip Video KYC. On Full-KYC completion, call POST /kyc/ckyc/upload to register the record. All PAN stored as SHA-256 hash only. Aadhaar stored as UIDAI VID token only." | CKYC lookup tested against CERSAI sandbox PAN never appears in logs (log scrubbing test) |
| S5 | Escrow & Nodal Service: nodal account registry, bank API integration Float monitor (5-min cron), 90%/95%/100% thresholds Reconciliation engine: Steps 1–5 (trigger → break classification) | "Build the daily reconciliation Saga as described in Section 4. The recon engine must be idempotent — re-running for the same date must not create duplicate recon_runs rows (use upsert on recon_date). Transaction matching uses reference_id as the join key. Generate unit tests for all four break types using mock ledger and bank statement data." | Recon engine idempotency test (run same date twice → same result) All 4 break types correctly classified in unit tests |
| S6 | Money Movement Sagas: Add Money, Spend, P2P, Withdraw Refund engine with idempotency Dead-letter queue for failed Sagas | "Generate all Money Movement Saga implementations. Each Saga must call POST /limits/check BEFORE any ledger write. POST /wallet/debit must be called before any external rail call (debit-first pattern). Compensating transactions must fully reverse the ledger if the external call fails. Generate integration tests simulating external rail failure mid-Saga." | Debit-first proven: external failure leaves ledger clean Same refund_ref processed twice → only one debit |
| S7–8 | UPI VPA registration + NPCI TPAP integration Third-party UPI link (Dec 2024 RBI amendment) FIU-IND FINNET 2.0 integration (CTR/STR filing) Wallet lifecycle scheduler (dormancy, expiry) | "Build UPI VPA management service. POST /upi/vpa/create must validate FULL_KYC before calling NPCI TPAP API. POST /upi/third-party/link must implement NPCI inter-PSP protocol to enable wallet as a funding source in third-party UPI apps. Include 2FA (OTP) middleware for all UPI payment endpoints." | VPA creation blocked for MIN_KYC wallet FINNET 2.0 sandbox CTR filing acknowledged |

| 06 | Security & Compliance Engineering Requirements |
| --- | --- |

## 6.1  Data Handling Rules

The following are non-negotiable engineering requirements derived from the compliance gap analysis. These must be enforced at the code review gate — no PR that violates these rules should be merged.

| Rule | Implementation Requirement | Clause |
| --- | --- | --- |
| Aadhaar storage prohibition | Never store Aadhaar number. Store only UIDAI Virtual ID (VID) token. Any code path that writes Aadhaar to DB must be rejected at review. | UIDAI circular — VID tokenisation |
| PAN in logs | PAN must be masked in all structured logs (replace with ****NN). Add a log scrubber middleware that intercepts all log output. CI gate: regex scan for PAN pattern in logs. | PMLA — PII in logs prohibited |
| Monetary amounts — no float | All amounts stored as BIGINT in paise. Never use DECIMAL, FLOAT, or DOUBLE for monetary values anywhere in the codebase. PR linting rule must reject float/decimal monetary types. | Financial precision — rounding errors in DECIMAL/FLOAT cause real-money bugs |
| KYC document encryption | All KYC documents (photos, ID copies) stored encrypted with AES-256-GCM. Encryption key stored in AWS KMS / GCP KMS — never in application config or codebase. | PMLA Sec.12 + RBI data security |
| Data localisation | All PPI transaction data must be stored on servers located in India. No transaction data may be processed or stored on foreign cloud regions. Infra must be validated before Sprint 1. | RBI April 2018 data localisation circular |
| Audit log immutability | ledger_entries must have no UPDATE or DELETE routes. Enforce via PostgreSQL row-security policy: DENY UPDATE/DELETE on ledger_entries for all roles except superuser. Verify in migration. | PPI-MD Cl.18 — 5yr immutable audit |
| Escrow account separation | Application layer must enforce that PPI_NODAL and PA_ESCROW nodal accounts are distinct. Any code path that would transfer funds between these two accounts must be rejected at build time. | PPI-MD Cl.7 + PA-PG Guidelines — commingling violation |

## 6.2  Compliance Test Suite Requirements

The following test cases must be present in the integration test suite before any production deployment. These are regulatory compliance tests, not optional quality improvements.

- MIN_KYC P2P: POST /money/transfer-p2p with MIN_KYC wallet → assert HTTP 403, reason_code = PPI_P2P_NOT_ALLOWED
- Balance ceiling: wallet load that would exceed Rs.10,000 (MIN_KYC) → assert HTTP 403, reason_code = BALANCE_CEILING_BREACH
- Duplicate ledger entry: two requests with same idempotency_key → second returns prior result, ledger has only one entry
- Debit-first: simulate external rail failure after wallet debit → assert wallet balance is fully restored via compensating transaction
- Float monitor: set mock nodal balance to produce 96% utilisation → assert Add Money feature flag is disabled within one monitor cycle
- CKYC lookup: POST /kyc/video-kyc/session with user who has existing CKYC record → assert video KYC not initiated, status = FULL_KYC
- MIN_KYC expiry: wallet_expiry_at = now - 1 minute → assert all transaction endpoints return HTTP 403, reason_code = WALLET_EXPIRED
- Recon idempotency: run reconciliation for same date twice → assert exactly one recon_runs row exists for that date
- UPI FULL_KYC gate: POST /upi/vpa/create with MIN_KYC wallet → assert HTTP 403
- PAN log scrubber: transaction with PAN in metadata → assert PAN does not appear in any log output

## 7.  MCP & AI Agent Layer — Current Implementation (April 2026)

While the above specification targets the production-grade architecture, the following demo/prototype layer has been built to validate the product and enable AI-powered operations:

### 7.1  MCP Server (35 Tools)

A Model Context Protocol server (wallet-mcp-server.js) exposes 35 tools to Claude Desktop and other MCP clients, organized across 9 categories: User & Wallet (10), Admin & Platform (6), Transaction Operations (5), Compliance (1), Disputes & Support (3), Notifications (2), KYC Actions (3), Analytics (2), KYC Expiry & Renewal (2). All tools validate inputs via Zod schemas and route to mock-data.js functions.

### 7.2  Data Layer (200 Users, 500+ Transactions)

mock-data.js provides the data foundation: 200 users via seeded PRNG (seed=100, matching admin dashboard generators.ts), 500+ transactions (seed=200), 41 exported functions. All amounts stored as BigInt in paise. RBI PPI limits (MINIMUM ₹10,000 / FULL ₹2,00,000) and P2P caps enforced. KYC expiry tracking for MINIMUM KYC users (365 days from creation) with urgency bands.

**Wallet Load Guard Service**

Wallet Load Guard — RBI PPI Limit Validation

The Wallet Load Guard is a pre-transaction validation layer that enforces RBI PPI limits before any wallet load (add money) transaction is processed.

Three RBI Rules Validated:

1. BALANCE_CAP — Maximum wallet balance of ₹1,00,000

2. MONTHLY_LOAD — Maximum ₹2,00,000 total loads per calendar month

3. MIN_KYC_CAP — Maximum ₹10,000 balance for Minimum KYC users

Architecture:

- Backend: mcp/services/wallet-load-guard.js — validation + Claude Haiku messages

- API: POST /api/wallet/validate-load, GET /api/wallet/load-guard-log

- UI: Add Money page with live helper text, block alerts, "Add max instead" button

- Admin: Load Guard Log panel on Dashboard (last 10 blocked attempts)

### 7.3  Render Backend (13 REST Endpoints)

Express.js on Render (ppi-wallet-api.onrender.com) serves MCP data to deployed GitHub Pages frontends. Key endpoints: /api/users, /api/users/:id, /api/transactions, /api/overview, /api/kyc/stats, /api/kyc/queue, /api/wallet/balance/:id, /api/wallet/ledger/:id, /api/wallet/status/:id, /api/wallet/transact, /api/chat, /api/summarise-transactions.

### 7.4  Agentic Chat Handler

chat-handler.js provides an HTTP-accessible AI interface using Claude's tool_use capability. Supports user and admin roles with multi-turn tool invocation. Powers the admin dashboard AI transaction summarizer and the /api/chat endpoint.

### 7.5  Production Deployment

Wallet App: GitHub Pages (gaurav2sheth.github.io/paytm-wallet-app) — 19 pages, 22 routes

Admin Dashboard: GitHub Pages (gaurav2sheth.github.io/ppi-wallet-admin) — 10 pages, AI summarizer

Backend API: Render (ppi-wallet-api.onrender.com) — 13 endpoints, auto-deploy from GitHub

MCP Server: Local via Claude Desktop (35 tools over stdio transport)

7.2  KYC Expiry Alert Service

An automated alert engine (mcp/services/kyc-alert-service.js) detects MINIMUM KYC users expiring within 7 days and generates personalised SMS/WhatsApp messages using Claude Haiku (claude-haiku-4-5-20251001) to minimise API cost. The service exposes two API endpoints: GET /api/kyc-alerts/preview (no API call) and POST /api/kyc-alerts/run (full alert cycle). A node-cron scheduler (mcp/services/scheduler.js) optionally runs the service daily at 9:00 AM IST. The admin dashboard KYC page includes a KycAlertPanel widget with real-time status, alert table, and AI-generated ops summary.

Transaction Sync: Wallet app sync.ts → Render /api/wallet/transact → mock-data.js in memory

| 07 | Open Questions & Decisions Required |
| --- | --- |

The following technical decisions are outstanding and block engineering implementation. Each must be resolved before the relevant sprint begins.

| # | Question | Options | Owner | Blocks Sprint |
| --- | --- | --- | --- | --- |
| 1 | Which partner bank for PPI nodal account? This determines bank API integration spec and SFTP format. | Yes Bank, IndusInd, RBL — see Partner Bank Evaluation Brief | Business / Finance | Sprint 5 (Recon Engine) |
| 2 | BBPOU registration status for PPSL. If PPSL is not a registered BBPOU, bill payment via BBPS cannot be offered. | (a) PPSL applies for BBPOU, (b) integrate via existing Paytm BBPOU, (c) drop bill pay from MVP | Business / Legal | Sprint 6 (Money Movement) |
| 3 | Event sourcing depth: full event-sourced ledger (Kafka + CQRS) vs. append-only PostgreSQL table? Full event sourcing adds complexity but enables replay. | (a) Full event-sourced (Kafka as source of truth), (b) PostgreSQL append-only (simpler, proven) | Engineering Lead | Sprint 1 (Architecture) |
| 4 | KYC provider for Video KYC: in-house UIDAI KUA registration or third-party licensed provider (IDfy, IDFC FIRST, etc.)? | (a) Register PPSL as KUA with UIDAI, (b) white-label from licensed provider | Legal / Product | Sprint 4 (KYC) |
| 5 | NPCI TPAP membership: should PPSL apply independently or operate as a sub-member under PCL's existing NPCI membership? | (a) Independent PPSL TPAP membership (preferred, separate legal entity), (b) sub-member under PCL | Legal / NPCI Liaison | Sprint 7 (UPI) |
| 6 | Encryption key management: AWS KMS vs. GCP KMS vs. HashiCorp Vault for KYC document encryption keys? | Depends on cloud provider choice. Must be decided before Sprint 4. | Engineering / Infra | Sprint 4 (KYC) |

PPSL PPI Wallet — Engineering Brief v1.0 — March 2026

Confidential — Internal Use Only — Paytm Payment Services Limited

Based on PPI_Wallet_Product_Requirements_v1.1.docx + Compliance_Gap_Analysis_v2.docx
