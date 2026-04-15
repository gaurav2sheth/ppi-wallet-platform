# AI Agents — PPI Wallet Platform

Comprehensive documentation for the autonomous AI agents powering the PPI Wallet's KYC compliance, customer support, and operational monitoring.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [1. KYC Upgrade Agent](#1-kyc-upgrade-agent)
- [2. Customer Support Agent](#2-customer-support-agent)
- [3. Escalation Manager](#3-escalation-manager)
- [4. Support Ticket Manager](#4-support-ticket-manager)
- [5. KYC Alert Service](#5-kyc-alert-service)
- [6. Scheduler](#6-scheduler)
- [API Endpoints](#api-endpoints)
- [Data Flow Diagrams](#data-flow-diagrams)
- [Claude Model Usage & Cost](#claude-model-usage--cost)
- [Error Handling & Fallback Strategies](#error-handling--fallback-strategies)
- [Business Rules & Constraints](#business-rules--constraints)
- [Observability & Monitoring](#observability--monitoring)
- [Configuration & Deployment](#configuration--deployment)

---

## Overview

The PPI Wallet platform includes **3 autonomous AI agents** and **2 supporting services** that handle operational workflows without human intervention:

| Agent | Architecture | Trigger | Purpose |
|-------|-------------|---------|---------|
| KYC Upgrade Agent | Fixed 7-step loop | Cron (daily 8 AM IST) + manual | Detect expiring KYC, decide intervention, draft outreach, follow up |
| Customer Support Agent | Dynamic tool selection | User chat message | Classify intent, investigate wallet data, resolve or escalate |
| KYC Alert Service | 4-step pipeline | Cron (daily 9 AM IST) + manual | Detect at-risk KYC users, generate alert messages |

Supporting modules:

| Module | Purpose |
|--------|---------|
| Escalation Manager | Shared in-memory store for ops escalations (used by both agents) |
| Support Ticket Manager | In-memory ticket store with SLA tracking |
| Scheduler | Cron-based orchestration of all autonomous jobs |

All agents use **Claude Sonnet** for reasoning/classification and **Claude Haiku** for text generation, with full **keyword/template fallback** when the API is unavailable.

---

## Architecture

```
                         ┌──────────────────────────────┐
                         │        Cron Scheduler         │
                         │  8 AM: KYC Upgrade Agent      │
                         │  9 AM: KYC Alert Service      │
                         │  6-hourly: Follow-up Checker  │
                         └──────────────┬───────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              ▼                         ▼                         ▼
   ┌─────────────────┐     ┌─────────────────────┐    ┌──────────────────┐
   │ KYC Upgrade      │     │ Customer Support     │    │ KYC Alert        │
   │ Agent             │     │ Agent                │    │ Service          │
   │                   │     │                      │    │                  │
   │ PERCEIVE          │     │ UNDERSTAND           │    │ FETCH            │
   │ REASON (Sonnet)   │     │ (Sonnet intent)      │    │ GENERATE (Haiku) │
   │ PLAN              │     │ INVESTIGATE          │    │ SIMULATE         │
   │ ACT (Haiku SMS)   │     │ (dynamic tools)      │    │ SUMMARY (Haiku)  │
   │ OBSERVE           │     │ RESOLVE              │    │                  │
   │ FOLLOW-UP         │     │ RESPOND (Haiku)      │    └──────────────────┘
   │ SUMMARY (Haiku)   │     │ ESCALATE (if needed) │
   └────────┬──────────┘     └──────────┬───────────┘
            │                           │
            ▼                           ▼
   ┌──────────────────────────────────────────────┐
   │            Escalation Manager                 │
   │     (shared in-memory escalation store)       │
   └──────────────────────────────────────────────┘
            │                           │
            ▼                           ▼
   ┌──────────────────┐     ┌──────────────────────┐
   │ Notification      │     │ Support Ticket       │
   │ Store             │     │ Manager              │
   └──────────────────┘     └──────────────────────┘
```

### Key Design Decisions

- **Fixed vs Dynamic architecture**: The KYC agent uses a fixed 7-step loop (processes all users per run). The Support agent uses dynamic tool selection (adapts per query).
- **Dual-model strategy**: Sonnet for reasoning (high accuracy), Haiku for generation (fast, cheap). Cost per 100 support chats: ~$0.29.
- **Graceful degradation**: Every Claude API call has a keyword/template fallback. Agents work at $0.00 cost in fallback mode.
- **Shared escalation store**: Both agents write to the same escalation manager, with a `source` field to distinguish origin.
- **Session management**: Support agent maintains conversation sessions (30-min expiry) for contextual follow-up.

---

## 1. KYC Upgrade Agent

**File:** `mcp/agents/kyc-upgrade-agent.js`

An autonomous 7-step agent that detects users with expiring KYC, makes prioritised intervention decisions, drafts personalised outreach, and handles follow-ups or escalations.

### Step-by-Step Architecture

```
Step 1: PERCEIVE    → Scan for users with KYC expiring ≤7 days
Step 2: REASON      → Claude Sonnet decides priority + strategy per user
Step 3: PLAN        → Build execution plan (SMS, in-app, follow-up, escalate)
Step 4: ACT         → Claude Haiku drafts personalised SMS + in-app notifications
Step 5: OBSERVE     → Check if user responded (upgraded, tapped CTA, ignored)
Step 6: FOLLOW-UP   → Send follow-up or escalate based on response
Step 7: SUMMARY     → Claude Haiku generates ops summary for Slack
```

### Step 1: PERCEIVE — `perceiveAtRiskUsers()`

Scans the mock database for users with KYC expiring within 7 days and gathers full context for each:

| Data gathered | Source | Purpose |
|--------------|--------|---------|
| Wallet balance | `getWalletBalance()` | Quantify amount at risk |
| Transaction history (30d) | `getTransactionHistory()` | Calculate activity score |
| Sub-wallet data | `getSubWalletData()` | Include benefit balances |
| User profile | `getUserProfile()` | Name, phone, KYC state |

**Derived fields:**

| Field | Logic |
|-------|-------|
| `activity_score` | HIGH (>10 txns), MEDIUM (4-10), LOW (1-3), DORMANT (0) |
| `is_high_value` | Balance > 500,000 paise (₹5,000) |
| `days_until_expiry` | Calendar days from today to `wallet_expiry_date` |

**Returns:** `{ userContexts: [...], auditLog: [...] }`

### Step 2: REASON — `reasonAboutUsers(userContexts, apiKey)`

Uses Claude Sonnet to make a complex multi-factor decision for each user.

**Claude model:** `claude-sonnet-4-20250514` (2048 max tokens)

**Decision output per user:**

| Field | Values |
|-------|--------|
| `priority` | P1_CRITICAL, P2_HIGH, P3_MEDIUM, P4_LOW |
| `intervention_strategy` | URGENT_MULTI_TOUCH, STANDARD_OUTREACH, GENTLE_REMINDER, MONITOR_ONLY |
| `message_tone` | URGENT, FRIENDLY, INFORMATIONAL |
| `key_motivator` | BALANCE_AT_RISK, BENEFITS_EXPIRY, HABIT_CONTINUITY, COMPLIANCE |
| `suggested_offer` | NONE, CASHBACK_50, CASHBACK_100, BONUS_SCRATCH_CARD |
| `follow_up_hours` | 24, 48, 72 |
| `escalate_immediately` | boolean |

**Fallback (rule-based decision matrix):**

| Condition | Priority | Strategy | Offer |
|-----------|----------|----------|-------|
| ≤2 days OR balance >₹50,000 | P1_CRITICAL | URGENT_MULTI_TOUCH | CASHBACK_100 |
| ≤4 days OR balance >₹10,000 | P2_HIGH | STANDARD_OUTREACH | CASHBACK_50 |
| ≤6 days AND (HIGH/MEDIUM activity) | P3_MEDIUM | GENTLE_REMINDER | BONUS_SCRATCH_CARD |
| All others | P4_LOW | MONITOR_ONLY | NONE |

### Step 3: PLAN — `buildExecutionPlan(userContexts, decisions)`

Converts decisions into executable action plans, sorted by priority (P1 first):

| Action | Timing | Condition |
|--------|--------|-----------|
| `SEND_SMS` | Immediate | All users except P4 |
| `SEND_IN_APP` | Immediate | All users except P4 |
| `FOLLOW_UP_SMS` | After N hours | Based on `follow_up_hours` |
| `ESCALATE_TO_OPS` | Immediate | If `escalate_immediately` is true |

### Step 4: ACT — `executeOutreach(executionPlans, apiKey)`

**SMS generation:**
- **Model:** `claude-haiku-4-5-20251001`
- **Constraint:** <160 characters, include expiry date, balance, CTA
- **Fallback template:** `"Hi {name}, your wallet KYC expires on {date}. ₹{balance} at risk. Upgrade now to keep your wallet active: [LINK]"`

**In-app notification generation:**
- **Model:** `claude-haiku-4-5-20251001`
- **Output:** JSON with `title`, `body`, `cta_button`
- **Fallback template:** Generic title/body/button

**Immediate escalations:** Calls `escalateToOps()` for P1/P2 users with `escalate_immediately: true`.

### Step 5: OBSERVE — `observeUserResponse(userId)`

Simulates checking if the user responded to outreach:

| Response Status | Meaning |
|----------------|---------|
| `UPGRADED` | User completed KYC upgrade |
| `CTA_TAPPED_NO_UPGRADE` | User engaged but didn't complete |
| `READ_NO_ACTION` | Notification read, no action |
| `UNREAD` | Notification not opened |

### Step 6: FOLLOW-UP — `handleFollowUpOrEscalation(userId, observeResult, actionPlan, apiKey)`

| User Response | Agent Action |
|--------------|-------------|
| UPGRADED | Close case, grant reward if offer was given |
| CTA_TAPPED_NO_UPGRADE | Send encouragement follow-up SMS |
| READ_NO_ACTION | Send stronger SMS; escalate if P1/P2 |
| UNREAD | Escalate if ≤2 days to expiry or balance >₹50,000 |

### Step 7: SUMMARY — `generateAgentSummary(runLog, apiKey)`

Generates a 4-line Slack-ready summary using Claude Haiku with key metrics:
- Users processed, escalated, offers deployed
- Total balance protected
- Duration in seconds

### Main Orchestrator — `runKycUpgradeAgent(apiKey)`

Executes all 7 steps sequentially, stores last 10 runs in memory, and returns a complete run report.

### Accessor Functions

| Function | Returns |
|----------|---------|
| `getAgentRunHistory()` | Last 10 agent runs with full audit trails |
| `getActiveNotifications()` | All notifications created by the agent |
| `getNotificationsByUser(userId)` | Notifications for a specific user |
| `markNotificationRead(notificationId)` | Marks notification as read |
| `markNotificationActionTaken(notificationId)` | Records CTA action |

---

## 2. Customer Support Agent

**File:** `mcp/agents/customer-support-agent.js`

A dynamic AI agent that handles customer queries via chat. Unlike the KYC agent's fixed loop, this agent decides its own tool sequence based on what the user asks.

### Step-by-Step Architecture

```
Step 1: UNDERSTAND   → Classify intent + extract entities (Claude Sonnet)
Step 2: INVESTIGATE  → Dynamically call wallet tools based on intent
Step 3: RESOLVE      → Build resolution with root cause analysis
Step 4: RESPOND      → Draft natural language response (Claude Haiku)
Step 5: ESCALATE     → Create ticket + ops escalation (if needed)
```

### Step 1: UNDERSTAND — `understandQuery(userId, message, conversationHistory, apiKey)`

**Claude model:** `claude-sonnet-4-20250514` (512 max tokens)

Uses last 3 conversation turns as context for accurate classification.

**13 supported intents:**

| Intent | Trigger examples |
|--------|-----------------|
| `BALANCE_QUERY` | "What's my balance?", "kitna paisa hai?" |
| `PAYMENT_BLOCKED` | "Payment failed", "transaction declined" |
| `TRANSACTION_INQUIRY` | "Show my transactions", "when did I pay?" |
| `SUB_WALLET_QUERY` | "Food wallet balance", "NCMC status" |
| `KYC_STATUS` | "What's my KYC status?", "is my KYC expired?" |
| `KYC_UPGRADE_HELP` | "How to upgrade KYC?" |
| `MERCHANT_ELIGIBILITY` | "Can I pay at Swiggy with food wallet?" |
| `REWARD_QUERY` | "Show my rewards" |
| `CASHBACK_QUERY` | "Where's my cashback?" |
| `SCRATCH_CARD_QUERY` | "Any scratch cards?" |
| `ESCALATION_REQUEST` | "Talk to a human", "I want to complain" |
| `GENERAL_HELP` | "What can you do?", "help" |
| `OUT_OF_SCOPE` | Questions unrelated to wallet |

**Entity extraction:**
- `amount` — Parsed from "₹500" or "Rs 1,000" patterns
- `merchant` — Merchant name
- `merchant_category` — Category type
- `sub_wallet_type` — FOOD, NCMC, FASTAG, GIFT, FUEL
- `transaction_id` — Transaction reference
- `time_period_days` — "last 7 days" → 7

**Fallback:** `keywordIntentMatch()` — regex-based classification with 0.6 confidence.

### Step 2: INVESTIGATE — `investigate(userId, intentResult, context)`

Dynamically selects and calls wallet tools based on the detected intent.

**Context-aware data sourcing:** When the frontend sends a `context` object (balance, transactions), the agent uses that real app data instead of server-side mock data. This ensures AI responses always match what the user sees in the app. The `context` parameter is optional — server-side mock data is the fallback.

| Intent | Tools Called |
|--------|-------------|
| BALANCE_QUERY | `clientContext:balance` OR `getWalletBalance()`, `getSubWalletData()` |
| PAYMENT_BLOCKED | `clientContext:balance+transactions` OR `getTransactionHistory()` + `getWalletBalance()`, `getSubWalletData()`, `getBlockedAttempts()` |
| TRANSACTION_INQUIRY | `clientContext:transactions` OR `getTransactionHistory()`, optionally `searchTransactions()` |
| KYC_STATUS / KYC_UPGRADE_HELP | `getUserProfile()`, `clientContext:balance` OR `getWalletBalance()`, `getEscalations()` |
| SUB_WALLET_QUERY | `getSubWalletData()`, `clientContext:balance` OR `getWalletBalance()` |
| MERCHANT_ELIGIBILITY | `getSubWalletData()`, `clientContext:balance` OR `getWalletBalance()` |
| REWARD_QUERY | `clientContext:balance+transactions` OR `getWalletBalance()` + `getTransactionHistory()`, `getNotifications()` |
| ESCALATION_REQUEST | `getUserProfile()`, `clientContext:balance` OR `getWalletBalance()` |

**Root cause analysis:**
- Identifies specific blockers (e.g., `INSUFFICIENT_BALANCE`, `LOAD_GUARD`, `KYC_EXPIRED`)
- Sets `can_resolve` flag based on whether the issue is actionable

### Step 3: RESOLVE — `resolve(userId, intent, investigationResult)`

Builds a resolution object with:
- `resolved` — Whether the agent can fully address the query
- `resolution_data` — Structured data specific to the intent (balances, transactions, KYC state)
- `suggested_actions` — Quick reply options for the user

**Example resolution data by intent:**

| Intent | Resolution Data |
|--------|----------------|
| BALANCE_QUERY | `main_balance`, `sub_wallets[]`, `sub_wallet_total`, `grand_total` |
| PAYMENT_BLOCKED | `root_cause`, `balance`, `explanation` |
| TRANSACTION_INQUIRY | `total_transactions`, `transactions[]` (last 5) |
| KYC_STATUS | `kyc_state`, `kyc_tier`, `wallet_expiry`, `balance` |

### Step 4: RESPOND — `draftResponse(userId, intent, resolution, sentiment, apiKey)`

**Claude model:** `claude-haiku-4-5-20251001` (300 max tokens)

**Response rules:**
- Match tone to sentiment: FRUSTRATED → extra empathetic, NEUTRAL → professional, POSITIVE → warm
- Max 4 sentences for simple queries, 6 for complex
- Always end with one clear next step
- Never use RBI jargon — plain language only
- Include real ₹ amounts from investigation data

**Fallback:** `templateResponse()` — hardcoded per-intent response using resolution data.

### Step 5: ESCALATE — `triggerFullEscalation(userId, conversationHistory, investigationResult, apiKey)`

Three-part escalation:

| Action | Details |
|--------|---------|
| **Ops escalation** | Creates record via `escalateToOps()` with P2_HIGH priority, source: `SUPPORT_AGENT` |
| **Support ticket** | Creates ticket via `createTicket()` with conversation history and investigation data |
| **Helpline info** | Returns number (1800-XXX-XXXX), hours (9 AM - 9 PM IST), WhatsApp, expected resolution (2 hours) |

**Auto-escalation triggers:**

| Trigger | Condition |
|---------|-----------|
| Unresolvable issue | `resolved = false` after investigation |
| Explicit request | User says "talk to a human" / "complaint" |
| Repeated failure | Same intent unresolved >=2 times in conversation |

### Session Management

| Function | Purpose |
|----------|---------|
| `getOrCreateSession(userId, sessionId)` | Creates session or retrieves existing (30-min expiry) |
| `addTurn(session, ...)` | Appends conversation turn with intent, tools, response |
| `countUnresolvedSameIntent(session, intent)` | Counts unresolved turns for auto-escalation detection |
| `getSession(sessionId)` | Retrieves active session |

**Session structure:**
```json
{
  "session_id": "SESS-1776238395403-user_001",
  "user_id": "user_001",
  "turns": [
    {
      "turn_number": 1,
      "timestamp": "15/4/2026, 1:03:15 pm",
      "user_message": "What is my balance?",
      "intent": "BALANCE_QUERY",
      "tools_used": ["getWalletBalance", "getSubWalletData"],
      "agent_response": "Your main wallet balance is ₹236.11...",
      "resolved": true
    }
  ],
  "created_at": "15/4/2026, 1:03:15 pm",
  "escalation_triggered": false,
  "ticket_id": null
}
```

### Analytics

| Metric | Description |
|--------|-------------|
| `total_chats_today` | Total support conversations |
| `resolved_by_agent` | Count resolved without escalation |
| `escalated` | Count escalated to ops |
| `intent_counts` | Distribution of intents (e.g., BALANCE_QUERY: 15) |
| `sentiment_counts` | FRUSTRATED / NEUTRAL / POSITIVE counts |
| `avg_turns_to_resolve` | Average conversation turns before resolution |
| `resolution_rate` | Percentage resolved by agent (resolved / total * 100) |

---

## 3. Escalation Manager

**File:** `mcp/agents/escalation-manager.js`

Shared in-memory store for tracking users who need manual ops intervention. Used by both the KYC Upgrade Agent and the Customer Support Agent.

### Functions

| Function | Description |
|----------|-------------|
| `escalateToOps(userId, context)` | Creates escalation record with priority, reasoning, recommended action |
| `getEscalations(filters)` | Query by status (OPEN/IN_PROGRESS/RESOLVED) and priority |
| `resolveEscalation(id, resolvedBy, notes)` | Marks as RESOLVED with resolution notes |
| `updateEscalationStatus(id, status)` | Updates status (e.g., OPEN → IN_PROGRESS) |
| `getEscalationStats()` | Aggregate counts: total, open, in_progress, resolved, by_priority |

### Escalation Record Structure

```json
{
  "escalation_id": "ESC-1776238395403-user_001",
  "user_id": "user_001",
  "name": "Kavita Thakur",
  "phone": "+91-9XXXXXXXXX",
  "priority": "P1_CRITICAL",
  "days_until_expiry": 1,
  "total_at_risk": "₹48,000.00",
  "escalation_reason": "KYC expiring in 1 day with high balance",
  "outreach_attempts": 1,
  "agent_reasoning": "User has ₹48K at risk with only 1 day left...",
  "recommended_action": "Manual outreach recommended",
  "status": "OPEN",
  "created_at": "15/4/2026, 8:00:00 am",
  "assigned_to": null
}
```

### Priority Levels

| Priority | Typical Trigger |
|----------|----------------|
| P1_CRITICAL | ≤2 days to KYC expiry OR balance >₹50,000 |
| P2_HIGH | ≤4 days OR support agent escalation |
| P3_MEDIUM | ≤6 days with moderate activity |
| P4_LOW | All others |

---

## 4. Support Ticket Manager

**File:** `mcp/agents/support-ticket-manager.js`

In-memory ticket store for customer support issues with SLA deadline tracking.

### Functions

| Function | Description |
|----------|-------------|
| `createTicket(userId, issueData)` | Creates ticket with auto-calculated SLA deadline |
| `getTicket(ticketId)` | Retrieve single ticket |
| `updateTicket(ticketId, updates)` | Partial update |
| `getUserTickets(userId)` | All tickets for a user (newest first) |
| `getOpenTickets(priority)` | Open/in-progress tickets with overdue flagging |
| `resolveTicket(ticketId, resolvedBy, notes)` | Mark as RESOLVED |
| `getTicketStats()` | Aggregate: total, open, in_progress, resolved, overdue, by_priority, by_type |

### SLA Deadlines

| Priority | Resolution SLA |
|----------|---------------|
| HIGH | 1 hour |
| MEDIUM | 2 hours |
| LOW | 4 hours |

### Ticket Structure

```json
{
  "ticket_id": "TKT-1776238395403-user_001",
  "user_id": "user_001",
  "name": "Aarav Sharma",
  "issue_type": "PAYMENT_BLOCKED",
  "issue_summary": "Auto-escalated from support chat. Root cause: INSUFFICIENT_BALANCE.",
  "conversation_history": [
    { "user": "Why was my payment blocked?", "agent": "...", "intent": "PAYMENT_BLOCKED" }
  ],
  "investigation_data": {
    "tools_used": ["getTransactionHistory", "getWalletBalance"],
    "root_cause": "INSUFFICIENT_BALANCE"
  },
  "priority": "HIGH",
  "status": "OPEN",
  "created_at": "15/4/2026, 1:05:00 pm",
  "sla_resolve_by": "15/4/2026, 2:05:00 pm",
  "is_overdue": false,
  "resolved_at": null,
  "resolution_notes": null
}
```

---

## 5. KYC Alert Service

**File:** `mcp/services/kyc-alert-service.js`

A 4-step pipeline that detects KYC expiring within 7 days, generates personalised alert messages, simulates delivery, and produces an ops summary.

### Pipeline

| Step | Function | Description |
|------|----------|-------------|
| 1. FETCH | `previewAtRiskUsers()` | Query users with KYC expiring ≤7 days |
| 2. GENERATE | `generateAlertMessage()` | Claude Haiku drafts <160 char SMS per user |
| 3. SIMULATE | `simulateSend()` | Logs delivery to console |
| 4. SUMMARY | `generateOpsSummary()` | Claude Haiku drafts 3-line ops summary |

**Main orchestrator:** `runKycAlerts(apiKey)`

**Returns:**
```json
{
  "run_at": "2026-04-15T02:30:00.000Z",
  "users_alerted": 4,
  "total_at_risk_balance": "₹54,573.89",
  "alerts": [...],
  "ops_summary": "4 users alerted, ₹54K at risk. 2 critical (≤2 days)..."
}
```

---

## 6. Scheduler

**File:** `mcp/services/scheduler.js`

Cron-based orchestration using `node-cron` for all autonomous jobs.

### Jobs

| Job | Cron Expression | IST Time | Function |
|-----|----------------|----------|----------|
| KYC Upgrade Agent | `30 2 * * *` | Daily 8:00 AM | `runKycUpgradeAgent()` |
| KYC Alert Service | `30 3 * * *` | Daily 9:00 AM | `runKycAlerts()` |
| Follow-up Checker | `0 */6 * * *` | Every 6 hours | Iterates notifications, calls `observeUserResponse()` + `handleFollowUpOrEscalation()` |

### Functions

| Function | Description |
|----------|-------------|
| `startScheduler()` | Starts all 3 cron jobs |
| `stopScheduler()` | Stops all 3 jobs |
| `startKycAlertJob()` / `stopKycAlertJob()` | Individual job control |
| `startKycUpgradeAgentJob()` / `stopKycUpgradeAgentJob()` | Individual job control |
| `startFollowUpCheckerJob()` / `stopFollowUpCheckerJob()` | Individual job control |
| `getSchedulerStatus()` | Returns job list with enabled status and cron expressions |

### Standalone Execution

```bash
# Start all jobs
node services/scheduler.js

# Start all jobs and run KYC alerts immediately
node services/scheduler.js --run-now
```

---

## API Endpoints

### KYC Upgrade Agent Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/kyc-agent/run` | Trigger a full KYC upgrade agent run |
| GET | `/api/kyc-agent/runs` | Returns last 10 agent runs with metrics |
| GET | `/api/kyc-agent/escalations` | Query escalations (filters: `?status=OPEN&priority=P1_CRITICAL`) |
| PATCH | `/api/kyc-agent/escalations/:id` | Resolve (`{ resolved_by, notes }`) or update status (`{ status }`) |
| GET | `/api/kyc-agent/notifications/:userId` | Get notifications for a user |
| PATCH | `/api/kyc-agent/notifications/:id` | Mark read (`{ read: true }`) or actioned (`{ action_taken: true }`) |
| GET | `/api/kyc-agent/audit/:runId` | Full audit trail for a specific run |

### Customer Support Agent Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/support/chat` | AI-powered support chat (`{ user_id, message, session_id, context? }`) |
| GET | `/api/support/tickets` | List open tickets (filters: `?status=OPEN&priority=HIGH`) |
| GET | `/api/support/tickets/user/:userId` | Tickets for a specific user |
| GET | `/api/support/tickets/:ticketId` | Single ticket details |
| PATCH | `/api/support/tickets/:ticketId` | Resolve ticket (`{ resolved_by, resolution_notes }`) |
| GET | `/api/support/sessions/:sessionId` | Chat session details with conversation history |
| GET | `/api/support/analytics` | Support analytics + ticket stats |

### KYC Alert Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/kyc-alerts/preview` | Preview at-risk KYC users (no outreach) |
| POST | `/api/kyc-alerts/run` | Run full alert pipeline with SMS generation |

### Support Chat Request Format

The frontend sends actual app data as `context` so AI responses always match what the user sees:

```json
{
  "user_id": "demo-wallet",
  "message": "What is my balance?",
  "session_id": "SESS-xxx",
  "context": {
    "balance_paise": "23611",
    "balance_formatted": "₹236.11",
    "user_name": "Aarav",
    "kyc_tier": "FULL",
    "recent_transactions": [
      { "entry_type": "DEBIT", "amount_paise": "5000", "amount_formatted": "₹50.00", "description": "Swiggy Order", "transaction_type": "MERCHANT_PAY", "created_at": "2026-04-14T10:00:00Z" }
    ]
  }
}
```

### Support Chat Response Format

```json
{
  "session_id": "SESS-1776238395403-user_001",
  "response_text": "Your main wallet balance is ₹236.11. You also have 5 sub-wallets...",
  "suggested_actions": ["Add money", "View sub-wallets", "Transaction history"],
  "tools_used": ["clientContext:balance", "getSubWalletData"],
  "intent_detected": "BALANCE_QUERY",
  "confidence": 0.95,
  "urgency": "LOW",
  "sentiment": "NEUTRAL",
  "resolved": true,
  "ticket_id": null,
  "escalated": false,
  "helpline_info": null,
  "response_time_ms": 450
}
```

---

## Data Flow Diagrams

### Flow 1: KYC Expiry Detection & Outreach

```
Scheduler (daily 8 AM IST)
  │
  ▼
runKycUpgradeAgent(apiKey)
  │
  ├─ 1. perceiveAtRiskUsers()
  │     └─ queryKycExpiry() → users expiring ≤7 days
  │     └─ getWalletBalance(), getTransactionHistory(), getSubWalletData()
  │
  ├─ 2. reasonAboutUsers(userContexts, apiKey)
  │     └─ Claude Sonnet → priority, strategy, tone, offer per user
  │     └─ [Fallback: rule-based decision matrix]
  │
  ├─ 3. buildExecutionPlan(userContexts, decisions)
  │     └─ SMS, in-app notification, follow-up, escalation actions
  │
  ├─ 4. executeOutreach(plans, apiKey)
  │     ├─ Claude Haiku → personalised SMS (<160 chars)
  │     ├─ Claude Haiku → in-app notification (JSON)
  │     └─ escalateToOps() for immediate escalations
  │
  ├─ 5. observeUserResponse(userId)
  │     └─ UPGRADED / CTA_TAPPED / READ_NO_ACTION / UNREAD
  │
  ├─ 6. handleFollowUpOrEscalation()
  │     ├─ UPGRADED → close case, grant reward
  │     ├─ CTA_TAPPED → follow-up SMS
  │     ├─ READ_NO_ACTION → stronger SMS, escalate P1/P2
  │     └─ UNREAD → escalate if critical
  │
  └─ 7. generateAgentSummary()
        └─ Claude Haiku → 4-line Slack summary
        └─ Store in agentRunHistory (last 10)
```

### Flow 2: Customer Support Query

```
POST /api/support/chat { user_id, message, session_id, context? }
  │
  ▼
handleSupportChat(userId, message, sessionId, apiKey, context)
  │
  ├─ getOrCreateSession() → 30-min TTL
  │
  ├─ 1. understandQuery()
  │     └─ Claude Sonnet → intent, entities, sentiment, urgency
  │     └─ [Fallback: keyword regex matcher]
  │
  ├─ 2. investigate(userId, intentResult, context)
  │     └─ Dynamic tool selection per intent
  │     └─ Prefer client context (balance, txns) over mock-data.js
  │     └─ getSubWalletData(), getBlockedAttempts() etc. from server
  │
  ├─ 3. resolve()
  │     └─ Build resolution_data + suggested_actions
  │     └─ Set can_resolve flag
  │
  ├─ [if needs escalation?]
  │   │  (unresolvable OR explicit request OR same intent failed ≥2x)
  │   │
  │   ├─ 5. triggerFullEscalation()
  │   │     ├─ escalateToOps() → P2_HIGH
  │   │     ├─ createTicket() → with SLA
  │   │     └─ helpline info
  │   │
  │   └─ 4. draftResponse() + append ticket info
  │
  ├─ [else]
  │   └─ 4. draftResponse()
  │         └─ Claude Haiku → sentiment-matched reply
  │         └─ [Fallback: templateResponse()]
  │
  ├─ addTurn() → session history
  ├─ updateAnalytics()
  │
  └─ return { session_id, response, tools_used, intent, escalated, ticket_id }
```

### Flow 3: Escalation Resolution (Admin)

```
Admin Dashboard → Support Panel → Open Tickets tab
  │
  ├─ GET /api/support/tickets → list open tickets
  │
  ├─ Admin clicks "Resolve"
  │     └─ PATCH /api/support/tickets/:id { resolved_by, resolution_notes }
  │           └─ resolveTicket() → status: RESOLVED, resolved_at timestamp
  │
  └─ GET /api/kyc-agent/escalations → view ops escalations
        └─ PATCH /api/kyc-agent/escalations/:id { resolved_by, notes }
              └─ resolveEscalation() → status: RESOLVED
```

---

## Claude Model Usage & Cost

### Models Used

| Model | Usage | Tokens (in/out) |
|-------|-------|-----------------|
| `claude-sonnet-4-20250514` | Intent classification, KYC decision reasoning | ~600 / ~200 |
| `claude-haiku-4-5-20251001` | SMS drafting, notification text, response generation, ops summaries | ~400 / ~150 |

### Cost Estimates

| Operation | Sonnet Calls | Haiku Calls | Est. Cost |
|-----------|-------------|-------------|-----------|
| 1 KYC Agent Run (4 users) | 1 | ~10 | ~$0.01 |
| 1 Support Chat | 1 | 1 | ~$0.003 |
| 100 Support Chats | 100 | 100 | ~$0.29 |
| 1 KYC Alert Run (4 users) | 0 | ~5 | ~$0.003 |

**Fallback mode cost:** $0.00 — all operations work without an API key using keyword classifiers and template responses.

---

## Error Handling & Fallback Strategies

Every Claude API call has a graceful fallback. No agent operation fails fatally.

| Component | Primary Path | Fallback | Impact |
|-----------|-------------|----------|--------|
| KYC Reasoning (Sonnet) | Claude API call | Rule-based priority matrix | Lower personalisation |
| SMS Generation (Haiku) | Claude API call | Template: "Hi {name}, KYC expires {date}..." | Less engaging |
| In-app Notification (Haiku) | Claude API call | Template with generic title/body/button | Functional |
| Intent Classification (Sonnet) | Claude API call | `keywordIntentMatch()` regex (0.6 confidence) | 80% accuracy |
| Response Drafting (Haiku) | Claude API call | `templateResponse()` per intent | Less natural |
| Ops Summary (Haiku) | Claude API call | Structured summary with metrics | Less readable |
| Escalation Creation | `escalateToOps()` | Fallback escalation ID | Always tracked |
| Ticket Creation | `createTicket()` | Fallback ticket ID | Always tracked |

### Error Logging

All errors logged with `[Agent Name]` prefix:
```
[Support Agent] Step 1 — UNDERSTAND: Classifying user intent...
[Support Agent]   Claude API fallback — using keyword matcher: API key not set
[Support Agent]   Intent: BALANCE_QUERY (confidence: 0.6)
```

---

## Business Rules & Constraints

### RBI Compliance (Enforced by Agents)

| Rule | Enforcement |
|------|-------------|
| Min-KYC 12-month validity | KYC agent detects expiry, triggers upgrade outreach |
| 60-day grace period | Agent tracks days_until_expiry for grace period users |
| Balance freeze on expiry | Agent communicates balance-at-risk amounts |
| P2P prohibition (Min-KYC) | Support agent explains restrictions when queried |

### KYC Agent Priority Matrix

| Priority | Criteria | Intervention |
|----------|----------|-------------|
| P1_CRITICAL | ≤2 days to expiry OR balance >₹50,000 | Immediate multi-channel + ops escalation |
| P2_HIGH | ≤4 days OR balance >₹10,000 | Standard outreach + follow-up |
| P3_MEDIUM | ≤6 days AND recent activity | Gentle reminder |
| P4_LOW | >6 days OR no activity | Monitor only |

### Support Agent Escalation Rules

| Condition | Action |
|-----------|--------|
| Cannot resolve issue | Auto-escalate with ticket |
| User says "human" / "agent" / "complaint" | Immediate escalation |
| Same intent unresolved 2+ times | Auto-escalate |
| FRUSTRATED sentiment + HIGH urgency | Priority P2_HIGH escalation |

### SLA Deadlines

| Ticket Priority | Resolution Target |
|----------------|-------------------|
| HIGH | 1 hour |
| MEDIUM | 2 hours |
| LOW | 4 hours |

---

## Observability & Monitoring

### Metrics Captured

**KYC Agent Runs:**
- `run_id`, `started_at`, `completed_at`, `duration_seconds`
- `users_processed`, `decisions[]`, `actions_taken[]`
- `escalations[]`, `offers_deployed`, `summary`
- Full `audit_trail` per run

**Support Chats:**
- `session_id`, `intent`, `confidence`, `urgency`, `sentiment`
- `tools_used[]`, `response_time_ms`
- `escalated`, `ticket_id`

**Escalations:**
- `escalation_id`, `priority`, `user_id`, `status`
- `days_until_expiry`, `total_at_risk`, `agent_reasoning`

**Tickets:**
- `ticket_id`, `issue_type`, `priority`, `status`
- `sla_resolve_by`, `is_overdue`, `resolved_at`

### Admin Dashboard Panels

| Panel | Location | Data Source |
|-------|----------|-------------|
| KYC Agent Panel | KYC Management page | `/api/kyc-agent/runs`, `/api/kyc-agent/escalations` |
| Support Agent Panel | Support page | `/api/support/analytics`, `/api/support/tickets`, `/api/kyc-agent/escalations` |
| AI Chat (Admin) | Dashboard page | `/api/support/chat`, `/api/chat` |

### Consumer App Integration

| Feature | Location | Data Source |
|---------|----------|-------------|
| AI Support Chat | Home page "Ask AI" card | `/api/support/chat` |
| My Tickets | `/support/tickets` route | `/api/support/tickets/user/:userId` |
| KYC Notifications | Notifications page | Agent-generated in-app notifications |

---

## Configuration & Deployment

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes (for Claude) | Anthropic API key for Sonnet + Haiku calls |
| `PORT` | No (default: 3001) | API server port |

### File Locations

```
mcp/
├── agents/
│   ├── kyc-upgrade-agent.js      # KYC Upgrade Agent (34K)
│   ├── customer-support-agent.js  # Customer Support Agent (34K)
│   ├── escalation-manager.js      # Shared Escalation Manager (3.4K)
│   └── support-ticket-manager.js  # Support Ticket Manager (4.2K)
├── services/
│   ├── kyc-alert-service.js       # KYC Alert Service
│   ├── scheduler.js               # Cron Scheduler
│   ├── wallet-load-guard.js       # RBI Load Guard
│   └── sub-wallet-service.js      # Sub-wallet CRUD
├── mock-data.js                   # 200 users, 500+ txns, BigInt paise
├── chat-handler.js                # Claude chat with MCP tool orchestration
└── wallet-mcp-server.js           # 49 MCP tools via Zod schemas
```

### Deployment

The API server (with all agents) deploys to Render via `ppi-wallet-api-deploy/`:

```bash
# Copy agent files to deploy repo
cp mcp/agents/*.js ppi-wallet-api-deploy/mcp/agents/

# Push to GitHub → Render auto-deploys
cd ppi-wallet-api-deploy
git add -A && git commit -m "Update agents" && git push
```

### Testing

```bash
# Run agent tests (59 tests)
cd mcp && node --experimental-vm-modules ./node_modules/.bin/vitest run

# Smoke test support agent (no API key needed)
node -e "
import { handleSupportChat } from './mcp/agents/customer-support-agent.js';
handleSupportChat('user_001', 'What is my balance?', null, null).then(r => {
  console.log('Intent:', r.intent_detected);
  console.log('Resolved:', r.resolved);
  console.log('Response:', r.response_text);
});
"

# Live test with API key (on Render)
curl -X POST https://ppi-wallet-api.onrender.com/api/support/chat \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"user_001","message":"What is my wallet balance?"}'
```
