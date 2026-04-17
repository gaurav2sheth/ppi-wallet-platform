# Sequence Diagrams

Mermaid diagrams for the platform's critical flows. GitHub renders these inline; for local preview use the Mermaid CLI or any Markdown viewer with Mermaid support.

## Table of Contents

- [1. Load Guard Validation](#1-load-guard-validation)
- [2. Cascade Spend (Merchant Pay)](#2-cascade-spend-merchant-pay)
- [3. FASTag Toll Transaction](#3-fastag-toll-transaction)
- [4. NCMC Direct Load from UPI](#4-ncmc-direct-load-from-upi)
- [5. KYC Upgrade Agent Flow](#5-kyc-upgrade-agent-flow)
- [6. Support Agent Flow with Escalation](#6-support-agent-flow-with-escalation)
- [7. 3-Tier API Fallback](#7-3-tier-api-fallback)
- [8. Saga Transaction Lifecycle](#8-saga-transaction-lifecycle)

---

## 1. Load Guard Validation

Every Add Money request runs 3 Load Guard rules before the saga starts.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant WA as Wallet App
    participant API as Express API
    participant LG as Load Guard
    participant MW as Main Wallet
    participant SW as Sub-Wallets

    U->>WA: Tap "Add ₹5,000"
    WA->>API: POST /api/wallet/validate-load<br/>{user_id, amount: 500000}
    API->>LG: validateLoadAmount(user_id, 500000)

    LG->>MW: read balance_paise
    LG->>SW: read all sub-wallet balances
    LG->>LG: Rule 1: BALANCE_CAP<br/>main + Σ(sub) + 500000 ≤ ₹1,00,000?
    alt BALANCE_CAP exceeded
        LG-->>API: { allowed: false, blocked_by: BALANCE_CAP,<br/>user_message, max_allowed }
        API-->>WA: 200 { allowed: false, ... }
        WA-->>U: "Can only add ₹X more"
    else Within cap
        LG->>LG: Rule 2: MONTHLY_LOAD<br/>loads_this_month + 500000 ≤ ₹2,00,000?
        alt Monthly limit exceeded
            LG-->>API: { allowed: false, blocked_by: MONTHLY_LOAD }
            API-->>WA: 200 { allowed: false }
            WA-->>U: "Monthly limit reached"
        else Within monthly limit
            LG->>LG: Rule 3: MIN_KYC_CAP (if Min-KYC)<br/>main + 500000 ≤ ₹10,000?
            alt Min-KYC cap exceeded
                LG-->>API: { allowed: false, blocked_by: MIN_KYC_CAP,<br/>suggestion: "Upgrade KYC" }
                API-->>WA: 200 { allowed: false }
                WA-->>U: "Upgrade KYC to add more"
            else All rules pass
                LG-->>API: { allowed: true, new_balance }
                API-->>WA: 200 { allowed: true }
                WA->>API: POST /api/wallet/add-money (actual saga)
                Note over API: See Saga diagram §8
            end
        end
    end
```

### Key properties

- **Order matters** — BALANCE_CAP runs first (cheapest check, fails most commonly)
- **Read-heavy** — sub-wallet aggregation reads 5+ records per request
- **Stateless** — Load Guard doesn't mutate; only the saga does
- **Race window** — between validate and actual save; in production, re-check inside the saga

---

## 2. Cascade Spend (Merchant Pay)

Merchant Pay auto-detects category and cascades across applicable sub-wallets before falling back to main.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant WA as Wallet App
    participant MP as Merchant Pay
    participant MCC as MCC Matcher
    participant SW as Sub-Wallets
    participant MW as Main Wallet
    participant L as Ledger

    U->>WA: Scan Swiggy QR, enter ₹500
    WA->>MP: payMerchant("Swiggy", 50000)
    MP->>MCC: matchCategory("Swiggy")
    MCC-->>MP: "Food Delivery"

    MP->>SW: getEligibleSubWallets("Food Delivery")
    SW-->>MP: [Food Wallet (₹400), Gift Wallet (₹2,000)]

    Note over MP: Priority: specific wallets first, then Gift as fallback

    MP->>SW: Food Wallet balance = ₹400, need ₹500
    alt Food balance ≥ amount
        MP->>SW: debit ₹500 from Food
        SW-->>MP: success
        MP->>L: append SUB_WALLET_TXN(FOOD, -500)
    else Food balance < amount (₹400 < ₹500)
        MP->>SW: debit full ₹400 from Food
        SW-->>MP: success
        MP->>MW: need remainder ₹100, main balance = ₹80
        alt Main balance ≥ remainder
            MP->>MW: debit ₹100 from main
            MP->>L: append DEBIT(-100, MAIN) + SUB_WALLET_TXN(FOOD, -400)
            MP-->>WA: success { split: { food: 400, main: 100 } }
        else Insufficient (₹80 < ₹100)
            MP->>SW: REVERT Food debit (compensation)
            MP-->>WA: fail { reason: INSUFFICIENT_FUNDS, shortfall: 20 }
            WA-->>U: "Short ₹20 — add money first"
        end
    end

    WA-->>U: Show result screen
```

### Notes
- **Rollback on partial failure** is critical — never leave Food debited if main couldn't cover remainder
- **Gift fallback** only triggers if no category-specific wallet matches (universal eligibility)
- **NCMC is NEVER in the eligible list for non-transit** — hard-rejected at `getEligibleSubWallets`

---

## 3. FASTag Toll Transaction

Toll hits deduct from main wallet first; security deposit is the safety net.

```mermaid
sequenceDiagram
    autonumber
    participant T as Toll Plaza Reader
    participant API as Backend
    participant MW as Main Wallet
    participant FT as FASTag Sub-Wallet

    T->>API: tollPayment(vehicle_no, ₹150)
    API->>MW: read balance_paise
    alt Main balance ≥ ₹150
        API->>MW: debit ₹150
        API-->>T: success { source: MAIN_WALLET }
    else Main balance < ₹150
        API->>FT: read security_deposit (balance_paise)
        alt Deposit ≥ ₹150
            API->>FT: consume ₹150 from deposit<br/>increment security_deposit_used_paise
            API->>MW: balance unchanged (₹0)
            API-->>T: success { source: SECURITY_DEPOSIT,<br/>warning: "Refill wallet soon" }
        else Deposit < ₹150 (partial or empty)
            API->>MW: attempt partial debit from main + rest from deposit
            Note over API: Whatever is feasible, else hard reject
            alt Combined funds still insufficient
                API-->>T: reject { code: INSUFFICIENT_FUNDS_TOLL }
                Note over T: Vehicle denied at boom barrier
            end
        end
    end
```

### On next Add Money

```mermaid
sequenceDiagram
    participant U as User
    participant WA as Wallet App
    participant FT as FASTag
    participant MW as Main Wallet

    U->>WA: Add Money ₹500 (from FASTag page, any payment source)
    WA->>FT: read security_deposit_used_paise (e.g., 10000 = ₹100)
    FT-->>WA: ₹100 used
    WA->>FT: refill ₹100 to deposit first<br/>(min of amount and deposit_used)
    WA->>MW: remainder ₹400 to main
    WA-->>U: "₹100 refilled to deposit + ₹400 to main"
```

---

## 4. NCMC Direct Load from UPI

NCMC can load directly from UPI/DC/NB, bypassing main wallet (ADR-005 amended).

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant WA as Wallet App
    participant NC as NCMC Sub-Wallet
    participant UPI as UPI Gateway (mocked)
    participant L as Ledger

    U->>WA: On NCMC page, select "UPI", enter ₹500
    WA->>NC: read balance_paise, max_balance_paise
    alt current + 500 > ₹3,000 cap
        WA-->>U: "Can only add ₹X more (max ₹3,000)"
    else Within cap
        WA->>UPI: charge ₹500 via UPI
        UPI-->>WA: success { ref: UPI-xxx }
        WA->>NC: credit ₹500 to NCMC sub-wallet
        WA->>L: append CREDIT ledger entry<br/>{type: ADD_MONEY, source: "UPI - HDFC 7125",<br/>balance_after_paise: main_unchanged}
        Note over L: Main wallet balance NOT changed<br/>Money flows: UPI → NCMC directly
        WA-->>U: "₹500 added to NCMC"
    end
```

### Why this is safe re: RBI cap

- `main + Σ(sub_wallets)` still enforced on load (Load Guard runs before credit)
- NCMC cap (₹3,000) enforced separately
- Combined cap (₹1,00,000) would have rejected earlier if violated
- Ledger entry preserves audit trail (even though main balance unchanged)

---

## 5. KYC Upgrade Agent Flow

The 7-step autonomous loop run daily at 8 AM IST.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cron (8 AM IST)
    participant A as KYC Agent
    participant DB as Mock Data
    participant S as Claude Sonnet
    participant H as Claude Haiku
    participant E as Escalation Store
    participant N as Notification Store

    C->>A: runKycUpgradeAgent()

    Note over A,DB: 1. PERCEIVE
    A->>DB: queryKycExpiry(days_ahead: 7)
    A->>DB: enrich with balance, transactions, sub-wallets
    DB-->>A: [userContexts]

    Note over A,S: 2. REASON (5-shot)
    loop per user
        A->>S: classify(user + 5 exemplars)
        S-->>A: { priority, strategy, tone, offer, escalate_immediately }
    end

    Note over A: 3. PLAN
    A->>A: sort by priority (P1 first); build actions[]

    Note over A,H: 4. ACT
    loop per user with outreach
        A->>H: draft SMS (≤160 chars)
        H-->>A: SMS text
        A->>H: draft in-app notification (JSON)
        H-->>A: { title, body, cta_button }
        A->>N: store notification
        opt escalate_immediately == true
            A->>E: escalateToOps({source: KYC_AGENT, priority: P1})
        end
    end

    Note over A: 5. OBSERVE (simulated)
    A->>A: check each user's response status

    Note over A,E: 6. FOLLOW-UP / ESCALATE
    loop per user observation
        alt UPGRADED
            A->>A: close case, grant reward
        else CTA_TAPPED_NO_UPGRADE
            A->>H: draft follow-up SMS
        else READ_NO_ACTION && P1/P2
            A->>H: draft stronger SMS
            A->>E: escalateToOps(source: KYC_AGENT)
        else UNREAD && critical
            A->>E: escalateToOps(source: KYC_AGENT)
        end
    end

    Note over A,H: 7. SUMMARY
    A->>H: draft 4-line Slack ops summary
    H-->>A: summary text
    A->>A: store run in agentRunHistory (last 10)
```

---

## 6. Support Agent Flow with Escalation

Dynamic 5-step flow per chat message.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant WA as Wallet App
    participant API as Express API
    participant SA as Support Agent
    participant S as Sonnet
    participant H as Haiku
    participant T as Tools
    participant E as Escalation Store
    participant TK as Ticket Store

    U->>WA: "Why was my payment blocked?"
    WA->>API: POST /api/support/chat<br/>{ user_id, message, session_id, context: {balance, txns} }
    API->>SA: handleSupportChat(...)

    Note over SA,S: 1. UNDERSTAND
    SA->>S: classify intent (+ last 3 turns)
    S-->>SA: { intent: PAYMENT_BLOCKED, confidence: 0.93,<br/>sentiment: FRUSTRATED, urgency: HIGH }

    Note over SA,T: 2. INVESTIGATE (context-preferred)
    SA->>SA: balance ← context.balance_paise<br/>txns ← context.recent_transactions
    SA->>T: getSubWalletData(user_id)
    SA->>T: getBlockedAttempts(user_id)
    T-->>SA: { sub_wallets, last_blocked_by: LOAD_GUARD }

    Note over SA: 3. RESOLVE
    SA->>SA: root_cause = "LOAD_GUARD",<br/>can_resolve = true

    alt Can resolve (within-turn)
        Note over SA,H: 4. RESPOND
        SA->>H: draft response (FRUSTRATED tone)
        H-->>SA: "I understand the frustration. Your payment was..."
        SA-->>API: { response_text, suggested_actions, resolved: true }
        API-->>WA: 200 { response_text, ... }
        WA-->>U: Shows response + suggested-action pills
    else Cannot resolve OR user says "human"
        Note over SA,E: 5. ESCALATE
        SA->>E: escalateToOps({source: SUPPORT_AGENT, priority: P2})
        SA->>TK: createTicket({sla: 2hr, priority: MEDIUM})
        SA->>H: draft response with ticket + helpline
        H-->>SA: response text
        SA-->>API: { response_text, ticket_id, escalated: true,<br/>helpline_info }
        API-->>WA: 200 { ... }
        WA-->>U: Response + ticket banner (amber)
    end

    Note over SA: Session persistence (30-min TTL)<br/>Auto-escalate if same intent fails ≥2x
```

---

## 7. 3-Tier API Fallback

Covers ADR-001. Each frontend request tries tiers in order.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant C as Component
    participant API as apiClient.ts
    participant V as Vite Middleware
    participant E as Express API (Render)
    participant M as mock.ts (localStorage)

    C->>API: getBalance(walletId)

    alt Tier 1: Vite dev middleware (npm run dev)
        API->>V: GET /api/wallet/balance
        V-->>API: 200 { balance_paise: "23611" }
        API-->>C: balance
    else Tier 2: Express API (production)
        API->>E: GET /api/wallet/balance
        alt Render responds
            E-->>API: 200 { balance_paise }
            API-->>C: balance
        else Render down / timeout / 5xx
            Note over API: Catch error
            API->>M: mockApi.getBalance(walletId)
            M->>M: read from localStorage.__mock_balance
            M-->>API: { balance_paise: "23611" }
            API-->>C: balance
            Note over C: console.warn("Mock mode")
        end
    else Tier 3: Mock-only (GitHub Pages demo)
        API->>E: GET /api/wallet/balance
        E--xAPI: network error (no backend configured)
        API->>M: mockApi.getBalance(walletId)
        M-->>API: balance
    end
```

---

## 8. Saga Transaction Lifecycle

Covers ADR-002. Any money-moving operation as a saga.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant WA as Wallet App
    participant API as API
    participant SA as Saga Orchestrator
    participant L as Ledger
    participant PB as Partner Bank (mock)
    participant N as Notification

    U->>WA: Confirm "Transfer ₹5,000 to bank"
    WA->>API: POST /api/wallet/to-bank (idempotency_key: UUID)
    API->>SA: start saga WALLET_TO_BANK

    SA->>L: Step 1: HOLD ₹5,000 on main wallet
    L-->>SA: hold_id created, balance_after reflects hold
    SA->>SA: status: RUNNING

    SA->>PB: Step 2: initiate bank transfer (NEFT/IMPS)
    alt PB acknowledges
        PB-->>SA: transfer_ref, status: IN_PROGRESS

        SA->>L: Step 3: DEBIT (release hold, create debit entry)
        L-->>SA: DEBIT ledger entry

        SA->>PB: Step 4: poll for settlement
        PB-->>SA: SETTLED

        SA->>N: Step 5: notify user
        N-->>SA: notified
        SA->>SA: status: COMPLETED
        SA-->>API: { saga_id, status: COMPLETED, balance_after }
        API-->>WA: 200 success
        WA-->>U: "Transfer successful"

    else PB rejects OR timeout OR bank returns
        PB--xSA: failure
        Note over SA: Compensating actions run in reverse

        SA->>L: Compensate Step 1: release hold
        L-->>SA: hold released, balance restored
        SA->>SA: status: COMPENSATED
        SA-->>API: { saga_id, status: COMPENSATED, error }
        API-->>WA: 200 { success: false, reason }
        WA-->>U: "Transfer failed — balance restored"
    end

    opt Compensation itself fails
        Note over SA: status: DLQ (dead-letter)
        SA->>N: alert ops
        SA-->>API: { status: DLQ, saga_id }
        API-->>WA: 502 — ask user to contact support
    end
```

### Saga state machine

```mermaid
stateDiagram-v2
    [*] --> STARTED
    STARTED --> RUNNING: first step runs
    RUNNING --> COMPLETED: all steps succeed
    RUNNING --> COMPENSATING: any step fails
    COMPENSATING --> COMPENSATED: rollback succeeds
    COMPENSATING --> DLQ: rollback itself fails
    COMPLETED --> [*]
    COMPENSATED --> [*]
    DLQ --> [*]: manual intervention
```
