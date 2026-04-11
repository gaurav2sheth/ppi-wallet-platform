# PPI WALLET Admin Dashboard

## Product Requirements Document (PRD)

**Version 1.0 | April 2026**

**CONFIDENTIAL -- For Internal Use Only**

| Field | Value |
|-------|-------|
| Document Owner | Product Team |
| Status | Draft |
| Last Updated | 3 April 2026 |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background & Context](#2-background--context)
   - 2.1 [Current Wallet Application](#21-current-wallet-application)
   - 2.2 [Technical Stack](#22-technical-stack)
   - 2.3 [Existing API Endpoints](#23-existing-api-endpoints)
3. [Objectives & Goals](#3-objectives--goals)
4. [Target Users & Personas](#4-target-users--personas)
5. [Feature Modules](#5-feature-modules)
   - 5.1 [Dashboard Home / Overview](#51-dashboard-home--overview)
   - 5.2 [User Management](#52-user-management)
   - 5.3 [KYC Management](#53-kyc-management)
   - 5.4 [Transaction Monitoring](#54-transaction-monitoring)
   - 5.5 [Spend Analytics & Insights](#55-spend-analytics--insights)
   - 5.6 [Revenue & Business Dashboard](#56-revenue--business-dashboard)
   - 5.7 [Customer Journey Funnel](#57-customer-journey-funnel)
   - 5.8 [Engagement & Nudge System](#58-engagement--nudge-system)
   - 5.9 [CST AI Bot Chat Review](#59-cst-ai-bot-chat-review)
   - 5.10 [Product Improvement Insights](#510-product-improvement-insights)
   - 5.11 [Access Control & Admin User Management](#511-access-control--admin-user-management)
6. [UI/UX Design Guidelines](#6-uiux-design-guidelines)
7. [Technical Architecture](#7-technical-architecture)
   - 7.1 [Frontend Stack](#71-frontend-stack)
   - 7.2 [Backend Requirements](#72-backend-requirements)
   - 7.3 [New Admin API Endpoints Required](#73-new-admin-api-endpoints-required)
8. [Data Requirements](#8-data-requirements)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Integration Points](#10-integration-points)
11. [Phased Rollout Plan](#11-phased-rollout-plan)
12. [Success Metrics](#12-success-metrics)
13. [Risks & Mitigations](#13-risks--mitigations)
14. [Appendix](#14-appendix)
    - Appendix A: [Complete API Reference](#appendix-a-complete-api-reference)
    - Appendix B: [MCC Category Mapping](#appendix-b-mcc-category-mapping)
    - Appendix C: [Notification Types](#appendix-c-notification-types)
    - Appendix D: [Wallet States & Transitions](#appendix-d-wallet-states--transitions)
    - Appendix E: [KYC States & Transitions](#appendix-e-kyc-states--transitions)
    - Appendix F: [Transaction Saga Lifecycle](#appendix-f-transaction-saga-lifecycle)
    - Appendix G: [MCP Tool Reference (35 Tools)](#appendix-g-mcp-tool-reference-35-tools)
    - Appendix H: [Render Backend Endpoint Reference](#appendix-h-render-backend-endpoint-reference)

---

## 1. Executive Summary

This document outlines the product requirements for building a comprehensive Admin Dashboard for the PPI (Prepaid Payment Instrument) Wallet application. The dashboard will serve as the central command center for operations, product, business, compliance, and customer support teams to monitor the wallet ecosystem health, manage users, ensure regulatory compliance, and drive revenue growth.

The Admin Dashboard will provide real-time visibility into all wallet users, their KYC status, transaction patterns, and wallet balances. It will enable targeted user engagement through in-app notifications and CPaaS platforms, support AI-powered customer service review, and deliver actionable business intelligence through interactive analytics and funnel analysis.

**Key Capabilities:** User lifecycle management, KYC compliance monitoring, real-time transaction surveillance, spend pattern analytics, revenue tracking with MDR analysis, customer journey optimization, cohort-based engagement campaigns, AI chatbot performance review, and multi-level role-based access control.

---

## 2. Background & Context

### 2.1 Current Wallet Application

The PPI Wallet application is a fully functional prepaid wallet built for RBI-compliant digital payments. The following features are currently live:

**User Onboarding & Authentication**

- Phone-based registration with 10-digit mobile number
- OTP verification (6-digit) for secure authentication
- Profile setup with full name and Wallet ID assignment
- Session persistence via localStorage with automatic hydration

**KYC (Know Your Customer)**

- Two-tier KYC system: MINIMUM and FULL verification
- **MINIMUM KYC:** Wallet balance cap of ₹10,000 with 12-month validity
- **FULL KYC:** Wallet balance cap of ₹2,00,000 via Aadhaar OTP verification
- KYC states: MIN_KYC, FULL_KYC, PENDING, REJECTED, EXPIRED

**Payment & Transaction Features**

- Add Money: Multiple payment methods (Wallet, UPI, Card, Net Banking)
- Send Money (P2P): Requires Full KYC per RBI compliance
- Merchant Payment (Scan & Pay): QR-based with merchant ID
- Bank Transfer: Wallet-to-bank with IFSC validation
- Bill Payment: 6 categories (Electricity, Water, Gas, DTH, Broadband, Insurance)

**Transaction Management**

- Passbook with paginated transaction history and month grouping
- Entry types: CREDIT, DEBIT, HOLD, HOLD_RELEASE
- Saga-based transaction lifecycle: STARTED, RUNNING, COMPLETED, COMPENSATING, COMPENSATED, DLQ
- Idempotency key-based deduplication for all transactions

**Analytics & Engagement**

- Spend analytics with MCC-based category mapping (14 categories)
- Budget controls with monthly caps and category-wise limits
- Scratch card rewards with random cashback (2%-10%)
- In-app notification system (Credit, Debit, Reward, Alert, Info types)
- PIN-based security for all transactions

**Recent Additions (April 2026):**

- MCC-based transaction categorization with 19 spending categories and smart icon mapping
- Wallet Statement download page with date range selection and email delivery
- Spend Analytics page with donut chart, category breakdown, and top merchants
- Budget Manager with category-wise limits and monthly caps
- Rewards system with scratch cards and cashback history
- Payment Gateway screen with split payment support (wallet + UPI/card)
- Data sync to Render backend (sync.ts) for AI agent accuracy
- Total: 19 pages, 18 components, 7 stores, 5 hooks, 22 routes

### 2.2 Technical Stack

| Component | Technology | Details |
|-----------|-----------|---------|
| Frontend Framework | React 18 + TypeScript | Vite build tool, React Router v7 |
| Styling | Tailwind CSS v4 | Mobile-first, max-width 430px |
| State Management | Zustand | 7 stores: auth, wallet, payees, budget, notifications, rewards, pin |
| HTTP Client | Axios | Base URL: localhost:3000/v1, 3s timeout |
| Data Format | Paise (integer) | 1 INR = 100 paise, BigInt arithmetic |
| API Pattern | REST + Saga | Health check, auto-fallback to mock data |
| Storage | localStorage + sessionStorage | Auth in localStorage, temp data in sessionStorage |

### 2.3 Existing API Endpoints

| API Group | Endpoint | Method | Purpose |
|-----------|----------|--------|---------|
| Wallet | /wallet/balance/{walletId} | GET | Fetch wallet balance |
| Wallet | /wallet/ledger/{walletId} | GET | Transaction history (paginated) |
| Wallet | /wallet/status/{walletId} | GET | Wallet status check |
| KYC | /kyc/status/{walletId} | GET | KYC status check |
| KYC | /kyc/initiate | POST | Start KYC process |
| KYC | /kyc/aadhaar/otp-send | POST | Send Aadhaar OTP |
| KYC | /kyc/aadhaar/otp-verify | POST | Verify Aadhaar OTP |
| Limits | /limits/check | POST | Validate transaction limits |
| Limits | /limits/usage/{walletId} | GET | Limit usage details |
| Saga | /saga/add-money | POST | Add money to wallet |
| Saga | /saga/merchant-pay | POST | Merchant payment |
| Saga | /saga/p2p-transfer | POST | P2P transfer |
| Saga | /saga/wallet-to-bank | POST | Bank withdrawal |
| Saga | /saga/bill-pay | POST | Bill payment |
| System | /health | GET | Backend health check |

---

## 3. Objectives & Goals

**Primary Objectives**

- **Centralized Visibility:** Provide a single pane of glass for all wallet users, KYC statuses, balances, and transaction activity across the entire ecosystem
- **Operational Excellence:** Enable the operations team to efficiently monitor and manage the complete user lifecycle from onboarding to dormancy
- **Business Intelligence:** Deliver actionable analytics for revenue tracking, spend patterns, MDR analysis, and month-over-month growth trends
- **Customer Service:** Support AI-powered customer support with chat resolution review and performance monitoring
- **User Engagement:** Enable targeted engagement through in-app notifications and CPaaS platforms for KYC upgrades, reactivation, and profile completion
- **Product Optimization:** Drive data-driven product improvements through funnel analysis, drop-off identification, and automated recommendations
- **Access Security:** Implement multi-level role-based access control ensuring data security and regulatory compliance

**Success Criteria**

- Dashboard becomes the primary tool for all wallet operations within 30 days of launch
- Reduction in manual data lookups by 50% or more
- KYC completion rate improvement of 20% through targeted nudges
- Average transaction issue identification time under 5 minutes

---

## 4. Target Users & Personas

The Admin Dashboard serves six distinct user personas, each with specific access levels and feature requirements:

| Persona | Role | Primary Needs | Access Level |
|---------|------|---------------|-------------|
| Super Admin | CTO / VP Engineering | Full system access, admin user management, system configuration, data exports | Level 1 (Full) |
| Business Admin | Head of Product / Business | Revenue dashboards, MoM trends, funnel analysis, product improvement insights | Level 2 (Analytics) |
| Operations Manager | Operations Lead | User management, KYC review and approval, transaction monitoring, send notifications | Level 3 (Operations) |
| Customer Support Agent | CS Team Member | User lookup, transaction history, AI chat resolution review | Level 4 (Support) |
| Compliance Officer | Compliance Team | KYC audit trail, regulatory reports, suspicious activity monitoring | Level 5 (Compliance) |
| Marketing Manager | Growth / Marketing | Cohort targeting, campaign management, engagement analytics, nudge creation | Level 6 (Marketing) |

---

## 5. Feature Modules

### 5.1 Dashboard Home / Overview

The dashboard home page provides an at-a-glance view of the entire wallet ecosystem with real-time metrics and trend indicators.

**Key Metrics Cards**

- Total Registered Users (with 7-day sparkline trend)
- Active Wallets count (wallets with transactions in last 30 days)
- Total Aggregate Balance across all wallets (in INR)
- Today's Transactions: count and total volume
- KYC Completion Rate: percentage of users with FULL KYC
- Revenue: MDR (Merchant Discount Rate) collected today and MTD

**Quick Stats Bar**

- New signups today
- Pending KYC verifications
- Failed transactions (last 24 hours)
- Currently active sessions

**Alert Banner**

- System health indicators (API latency, error rates)
- Compliance deadline reminders
- Unusual activity detection alerts

**Data Source (Production):**

In the deployed environment, all dashboard metrics are served by the Render backend (ppi-wallet-api.onrender.com) which computes them from the MCP mock-data.js layer. The GET /api/overview endpoint returns aggregated data from getSystemStats() and getKycStats() -- the same functions the AI agent uses. This ensures dashboard metrics and AI agent responses are always consistent. 200 users with seeded PRNG (seed=100) provide consistent data between the dashboard's local generators and the MCP data layer.

### 5.2 User Management

Comprehensive user lifecycle management with search, filtering, detailed views, and bulk operations.

**User List Table**

| Column | Data Type | Sortable | Filterable |
|--------|-----------|----------|-----------|
| User ID | String (UUID) | Yes | Search |
| Name | String | Yes | Search |
| Phone | String (10-digit) | No | Search |
| Wallet ID | String | Yes | Search |
| KYC Tier | MINIMUM / FULL | Yes | Dropdown |
| Balance | INR (from paise) | Yes | Range |
| Wallet Status | ACTIVE / SUSPENDED / DORMANT / EXPIRED / CLOSED | Yes | Dropdown |
| Created Date | DateTime | Yes | Date Range |
| Last Active | DateTime | Yes | Date Range |

**User Detail View**

- Profile information: Name, Phone, Wallet ID, User ID, Registration date
- KYC status with document verification details and audit trail
- Wallet balance breakdown: Available, Held, and Total balance
- Transaction history with pagination (leveraging existing /wallet/ledger API)
- Individual spend analytics with category breakdown
- Rewards earned: Scratch cards history and total cashback
- Notification history: All notifications sent to this user
- Support ticket history and AI chat transcripts

**Bulk Actions**

- Export user list to CSV with selected columns
- Send bulk in-app notifications to filtered user set
- Flag multiple accounts for review

**User Status Management**

- Activate: Re-enable a suspended wallet
- Suspend: Temporarily disable wallet (with reason logging)
- Freeze: Prevent all transactions pending investigation

### 5.3 KYC Management

End-to-end KYC lifecycle management with verification queues, compliance monitoring, and audit trails.

**KYC Dashboard**

- Pie chart: Users distributed by KYC tier (MINIMUM vs FULL)
- Pending verification queue with count and average wait time
- Verification success/failure rate (daily and cumulative)
- Average verification processing time

**KYC States Tracked**

| State | Description | Allowed Transitions |
|-------|------------|-------------------|
| MIN_KYC | Basic KYC completed, ₹10,000 balance limit | FULL_KYC, EXPIRED |
| FULL_KYC | Aadhaar verified, ₹2,00,000 balance limit | EXPIRED (rare) |
| PENDING | KYC submission under review | FULL_KYC, REJECTED |
| REJECTED | KYC submission rejected with reason | PENDING (resubmission) |
| EXPIRED | KYC validity period lapsed (12 months for MIN) | PENDING (re-verification) |

**Verification Queue**

- List of pending KYC submissions sorted by submission date
- Document preview (Aadhaar details, photo, name match)
- Approve/Reject actions with mandatory notes field
- Auto-assignment to available compliance officers

**Audit Trail**

- Every KYC status change logged with: timestamp, admin user, previous state, new state, reason
- Exportable audit reports for regulatory compliance

**Compliance Alerts**

- Wallets approaching KYC expiry (30/15/7 days before)
- Users approaching transaction limits
- Suspicious patterns: Rapid KYC changes, multiple failed attempts

### 5.4 Transaction Monitoring

Real-time transaction surveillance with comprehensive filtering, saga lifecycle tracking, and failure analysis.

**Transaction List**

- Filterable by transaction type: ADD_MONEY, MERCHANT_PAY, P2P_TRANSFER, WALLET_TO_BANK, BILL_PAY
- Filterable by status: COMPLETED, FAILED, PENDING, COMPENSATED, DLQ
- Date range, amount range, and specific user filters
- Sortable by amount, date, and status

**Transaction Detail View**

- Full saga lifecycle visualization: STARTED -> RUNNING -> COMPLETED / COMPENSATED / DLQ
- Amount display in INR (converted from paise) with BigInt precision
- Sender and receiver wallet details
- Idempotency key for deduplication tracking
- Timeline of all status transitions with timestamps

**Real-time Transaction Feed**

- WebSocket-powered live feed of incoming transactions
- Color-coded by type and status for quick scanning
- Pause/resume and filter capabilities

**Failed Transaction Analysis**

- DLQ (Dead Letter Queue) transaction investigation panel
- Compensation transaction review and retry options
- Failure pattern identification across time periods

**Transaction Volume Charts**

- Hourly, daily, weekly, and monthly volume visualization
- Transaction value trends with overlaid success rate
- Split by transaction type for pattern recognition

### 5.5 Spend Analytics & Insights

Aggregate spend analysis across the entire user base with MCC-based category mapping and cohort comparisons.

**Category-based Analysis (MCC Mapping)**

Leveraging the existing 14-category MCC mapping system:

| Category | MCC Codes | Example Merchants |
|----------|-----------|------------------|
| Taxi / Ride | 4121, 7512 | Uber, Ola, Rapido |
| Food & Dining | 5812, 5814 | Swiggy, Zomato, Restaurants |
| Groceries | 5411, 5422 | BigBasket, Blinkit, D-Mart |
| Shopping | 5311, 5651 | Amazon, Flipkart, Myntra |
| Fuel | 5541, 5542 | Petrol pumps, EV charging |
| Travel | 4511, 7011 | Airlines, Hotels, MakeMyTrip |
| Entertainment | 7832, 7922 | Netflix, PVR, BookMyShow |
| Health | 8011, 8099 | Hospitals, Pharmacies, 1mg |
| Education | 8211, 8299 | Schools, Courses, Udemy |
| Utilities | 4900, 4814 | Electricity, Water, Telecom |
| Money Transfer | 6012, 6051 | P2P transfers, Bank transfers |
| Refunds | N/A | Transaction reversals |
| Recharge | 4812, 4813 | Mobile, DTH, Broadband |
| Insurance | 6300, 6399 | Life, Health, Vehicle insurance |

**Time-based Analysis**

- Daily, weekly, monthly spend pattern visualization
- Month-over-Month (MoM) growth charts with percentage change
- Seasonal trend identification and forecasting

**Merchant Intelligence**

- Top merchants by transaction volume and value
- Merchant category distribution analysis
- New merchant acquisition trends

**User Spend Patterns**

- Average transaction value across user segments
- Transaction frequency distribution (power users vs occasional)
- High-value transaction alerts (configurable threshold)

**Cohort Analysis**

- Spend patterns segmented by user signup date
- KYC tier comparison (MINIMUM vs FULL users)
- Geographic distribution analysis (if location data available)

### 5.6 Revenue & Business Dashboard

Business-level metrics for tracking total revenue, growth trends, and key performance indicators.

**Revenue Metrics**

- Total GMV (Gross Merchandise Value): Sum of all transaction values
- MDR Collected: Merchant Discount Rate revenue per transaction type
- Revenue per transaction type breakdown
- ARPU (Average Revenue Per User) calculation

**MoM Trend Charts**

- Total users growth trajectory
- Transaction volume trend (count and value)
- Revenue trend with forecasting
- Average transaction value evolution
- Active user ratio (DAU/MAU)

**Business KPIs**

| KPI | Description | Target | Data Source |
|-----|------------|--------|------------|
| DAU/MAU Ratio | Daily Active / Monthly Active users | > 25% | User activity logs |
| Transaction Success Rate | Completed / Total attempted | > 98% | Saga status data |
| Avg Wallet Balance | Mean available balance across users | Tracking | Wallet balance API |
| Top-up Frequency | Average add-money transactions per user/month | > 3x | Ledger data |
| Withdrawal Frequency | Average bank transfers per user/month | Tracking | Ledger data |
| KYC Upgrade Rate | MIN to FULL KYC conversion | > 40% | KYC status data |
| Churn Rate | Users dormant > 90 days / Total users | < 15% | User activity logs |

**P&L Summary View**

- Revenue breakdown by source (MDR, convenience fees, float income)
- Cost center tracking (infrastructure, compliance, support)
- Net revenue trend with margin analysis

### 5.7 Customer Journey Funnel

Interactive funnel visualization tracking users from first touchpoint through repeat engagement, with drop-off analysis and automated recommendations.

**Funnel Stages**

| Stage | Definition | Key Metric | Current Source |
|-------|-----------|------------|---------------|
| 1. App Download | User installs the application | Install count | App store analytics |
| 2. Registration | Phone + OTP verification completed | Registration rate | Auth store |
| 3. Profile Setup | Name and Wallet ID configured | Setup completion | Auth store |
| 4. Minimum KYC | Basic identity verification done | KYC initiation rate | KYC API |
| 5. First Add Money | First wallet top-up completed | Activation rate | Saga API |
| 6. First Transaction | First payment/transfer made | Transaction rate | Ledger API |
| 7. Full KYC | Aadhaar verification completed | Upgrade rate | KYC API |
| 8. Repeat Usage | 3+ transactions in 30 days | Retention rate | Ledger API |
| 9. Dormancy/Churn | No activity for 90+ days | Churn rate | Activity logs |

**Drop-off Analysis**

- Conversion rate calculated at each stage transition
- Average time spent at each stage before progression or abandonment
- Drop-off reasons categorized where data is available
- Trend comparison: current month vs previous months

**Funnel Visualization**

- Interactive funnel chart with proportional width at each stage
- Click-through to filtered user lists at each stage
- Color-coded health indicators (green/amber/red based on benchmarks)

**Recommendations Engine**

- Auto-generated list of potential fixes for high drop-off stages
- Priority scoring: Users affected multiplied by estimated conversion lift
- Historical tracking of implemented fixes and their measured impact

**A/B Test Tracking**

- Configure and track improvement experiments per funnel stage
- Statistical significance calculator for test results
- Impact visualization: before/after conversion rates

### 5.8 Engagement & Nudge System

Cohort-based user engagement platform supporting in-app notifications, CPaaS integration, and campaign management.

**Cohort Builder**

Create targeted user segments using combinable filter criteria:

- KYC Status: Users with MINIMUM KYC who haven't upgraded
- Balance Range: Low balance (< ₹100), Medium, High
- Activity: Last active date range, transaction frequency
- Spend Preferences: Dominant spend categories
- Funnel Stage: Users stuck at specific journey stages
- Custom: Combine any filters with AND/OR logic

**Notification Channels**

| Channel | Integration | Use Case | Priority |
|---------|-----------|----------|----------|
| In-App Notification | Existing notification store | Active user engagement, feature updates | P0 (MVP) |
| SMS (CPaaS) | Twilio / Gupshup API | KYC reminders, dormant user reactivation | P0 (MVP) |
| Push Notification | FCM / APNs | Time-sensitive alerts, transaction updates | P1 |
| Email | SendGrid / SES | Monthly statements, marketing campaigns | P2 |

**Campaign Management**

- Create campaign: Select cohort, compose message, choose channel, set schedule
- A/B testing: Create message variants with automatic traffic splitting
- Campaign analytics: Delivery rate, open rate, click-through rate, action rate
- Campaign calendar: Visual timeline of scheduled and completed campaigns

**Pre-built Nudge Templates**

- KYC Upgrade Reminder: Target MIN_KYC users with upgrade benefits
- Inactive Wallet Reactivation: Users dormant > 30 days
- Low Balance Top-up: Users with balance < ₹100
- New Feature Announcement: All active users
- Reward/Offer Communication: Eligible cashback and scratch card offers

### 5.9 CST AI Bot Chat Review

Customer support chatbot performance monitoring, conversation review, and training data management. Note: The AI chatbot is a planned feature; this module provides the admin interface for when it is implemented.

**Implementation Status (April 2026):** The AI agent layer is now partially implemented via the MCP chat handler (chat-handler.js). The handler provides:

- 35 tools available for AI-powered customer support (balance queries, transaction lookup, spending analysis, compliance checks)
- Two-role system: "user" role for wallet holder queries, "admin" role for operations team queries
- Multi-turn tool use: Claude can invoke multiple tools sequentially to answer complex questions
- AI Transaction Summarizer: POST /api/summarise-transactions endpoint that analyzes transaction patterns, anomalies, and compliance issues
- Accessible via: (1) Claude Desktop MCP server for direct agent interaction, (2) Render backend /api/chat endpoint for HTTP-based chat, (3) Admin dashboard "Summarize" button on Transactions page

**Chat History View**

- Searchable list of all bot conversations with user details
- Filters: By user, date range, resolution status, topic category
- Quick preview of conversation summary without opening full transcript

**Resolution Analytics**

| Metric | Description | Target |
|--------|------------|--------|
| Total Chats | Daily/weekly/monthly conversation volume | Tracking |
| Resolution Rate | Percentage resolved without human escalation | > 80% |
| Avg Response Time | Bot response latency (seconds) | < 3 seconds |
| Escalation Rate | Conversations transferred to human agents | < 20% |
| CSAT Score | Customer satisfaction rating post-chat | > 4.0/5.0 |

**Conversation Detail**

- Full chat transcript with AI vs human agent responses distinguished
- Resolution status tracking: Resolved, Escalated, Pending, Unresolved
- Conversation duration and message count
- User sentiment indicators per message

**Training & Improvement**

- Flag conversations for AI model retraining
- Identify common unresolved query patterns
- Suggest new FAQ entries based on frequent questions
- Track model performance improvements over retraining cycles

**Bot Performance Dashboard**

- Response accuracy rate across topic categories
- Topics successfully handled vs escalated (heatmap)
- Sentiment analysis trends over time
- Comparison: Bot resolution vs human agent resolution quality

### 5.10 Product Improvement Insights

Data-driven product optimization through metrics matrices, anomaly detection, and actionable improvement recommendations.

**Metrics Matrix**

- All key metrics displayed in a single view with RAG (Red/Amber/Green) status indicators
- Configurable thresholds for each metric to define RAG boundaries
- Historical context: Current value vs 7-day average vs 30-day average

**Automated Insight Generation**

- Anomaly detection using statistical methods on all tracked metrics
- Week-over-week comparison alerts for significant changes (> 10% deviation)
- Trend reversal notifications when metrics change direction
- Correlation analysis: Identify metrics that move together

**Feature Impact Tracking**

- Before/after metrics comparison for feature launches
- User adoption curves for new features (7-day, 30-day, 90-day)
- Feature usage frequency and engagement depth

**User Feedback Integration**

- In-app feedback collection mechanism
- NPS (Net Promoter Score) tracking with trend analysis
- Feature request aggregation and voting
- Feedback categorization by feature area

**Action Items Board**

- Auto-generated improvement suggestions from data patterns
- Impact vs Effort priority matrix for proposed actions
- Assignment to team members with due dates
- Progress tracking and outcome measurement

### 5.11 Access Control & Admin User Management

Multi-level role-based access control (RBAC) with comprehensive audit logging and session security.

**Role Hierarchy**

| Level | Role | Modules Accessible | Key Permissions |
|-------|------|-------------------|----------------|
| 1 | Super Admin | All modules | Create/manage admins, system config, full data export, all CRUD operations |
| 2 | Business Admin | Dashboard, Analytics, Revenue, Funnel, Insights | View all analytics, export reports, read-only user data |
| 3 | Operations Manager | Dashboard, Users, KYC, Transactions, Notifications | User management, KYC approve/reject, send notifications, transaction investigation |
| 4 | Customer Support | Dashboard (limited), User Lookup, Transactions, Chat Review | View user details (limited PII), view transactions, review chat history |
| 5 | Compliance Officer | KYC Management, Audit Logs, Reports | KYC audit access, generate regulatory reports, view suspicious activity, read-only |
| 6 | Marketing Manager | Cohort Builder, Campaigns, Engagement Analytics | Create/manage campaigns, view aggregate analytics, no direct PII access |

**Permission Granularity**

- Module-level access: Which sections of the dashboard are visible
- Action-level permissions: View, Create, Edit, Delete, Export for each module
- Data-level access: PII masking for roles below Level 3 (phone numbers, names partially masked)
- Record-level: Compliance officers see all KYC data; CS agents see only their assigned users

**Admin Activity Audit Log**

- Every admin action logged: timestamp, admin user ID, admin role, IP address, action performed, affected resource
- Immutable audit trail (append-only, no deletion)
- Exportable audit reports with date range filtering
- Real-time alerts for suspicious admin activity patterns

**Session Security**

- MFA (Multi-Factor Authentication) required for all admin logins
- Session timeout after 30 minutes of inactivity (configurable per role)
- Concurrent session limit: 1 active session per admin user
- IP whitelisting option for Super Admin and Compliance roles
- Session revocation capability for Super Admins

---

## 6. UI/UX Design Guidelines

### 6.1 Design Language

The Admin Dashboard follows Paytm for Business design principles, emphasizing clarity, trust, and data accessibility.

**Color System**

| Color | Hex Code | Usage |
|-------|---------|-------|
| Dark Midnight Blue | #002E6E | Primary headers, navigation background, important CTAs |
| Process Cyan | #00B9F1 | Accent elements, links, active states, chart primary color |
| Light Blue Background | #E8F4FD | Card backgrounds, hover states, alternating table rows |
| Success Green | #12B76A | Positive metrics, completed status, success alerts |
| Warning Amber | #F79009 | Warning indicators, pending status, approaching limits |
| Error Red | #F04438 | Failed transactions, critical alerts, negative trends |
| Text Primary | #1A1A2E | Body text, table content, metric values |
| Text Secondary | #666666 | Labels, descriptions, helper text, timestamps |
| Border Grey | #D0D5DD | Table borders, card borders, dividers |
| Background | #F5F7FA | Page background, subtle separation |

**Layout Structure**

- Collapsible sidebar navigation (240px expanded, 64px collapsed with icons)
- Top bar with breadcrumbs, global search, notifications bell, and admin profile
- Main content area with responsive grid layout
- Sticky metric cards at top of scrollable content areas

**Component Library**

- Ant Design (antd) as the base component library, customized with Paytm color theme
- Apache ECharts for all chart and visualization components
- Custom metric card components with sparkline integration
- Responsive data tables with virtual scrolling for large datasets

**Responsive Breakpoints**

- Desktop: 1280px+ (primary target)
- Tablet: 768px - 1279px (sidebar collapses, cards reflow)
- Mobile: Not targeted (admin dashboard is desktop-first)

---

## 7. Technical Architecture

### 7.1 Frontend Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Framework | React 18 + TypeScript | Consistency with existing wallet app codebase |
| UI Library | Ant Design (antd) v5 | Enterprise-grade components, theming support, data tables |
| State Management | Zustand | Lightweight, consistent with wallet app pattern |
| Charting | Apache ECharts | High-performance, rich chart types, large dataset support |
| HTTP Client | Axios | Consistent with wallet app, interceptor support for auth |
| Build Tool | Vite | Fast builds, HMR support, consistent with wallet app |
| Routing | React Router v7 | Nested layouts, route guards for RBAC |
| Real-time | WebSocket (native) | Live transaction feed, instant notifications |
| Export | SheetJS + jsPDF | CSV and PDF report generation on client side |

### 7.2 Backend Requirements

- JWT-based authentication with refresh token rotation
- RBAC middleware validating permissions on every API endpoint
- WebSocket server for real-time transaction feed
- PostgreSQL / TimescaleDB for time-series analytics data
- Redis caching layer for frequently accessed dashboard metrics
- API gateway with rate limiting and request logging
- Circuit breaker pattern for resilient service-to-service calls

**Production Backend (Render -- Current Implementation):**

The deployed admin dashboard connects to an Express.js backend on Render (ppi-wallet-api.onrender.com) that serves MCP data. This backend provides 13 REST endpoints including:

- GET /api/users -- Paginated user list from MCP data (transforms to WalletUser format)
- GET /api/users/:userId -- User detail with KYC profile, recent transactions, spending summary
- GET /api/transactions -- All transactions across all users, paginated with filters
- GET /api/overview -- Dashboard analytics from getSystemStats() + getKycStats()
- GET /api/kyc/stats -- KYC tier distribution statistics
- GET /api/kyc/queue -- Pending KYC verification queue
- POST /api/chat -- AI chat endpoint proxying to chat-handler.js
- POST /api/summarise-transactions -- AI transaction summarizer
- POST /api/wallet/transact -- Transaction sync from wallet app

The frontend client.ts detects VITE_API_URL (set during build) to determine whether to use the Render backend or local mock data. Health check timeout is 15 seconds for Render free tier cold starts.

Both frontends are deployed on GitHub Pages (gaurav2sheth.github.io/paytm-wallet-app and gaurav2sheth.github.io/ppi-wallet-admin) with VITE_API_URL set at build time.

### 7.3 New Admin API Endpoints Required

| Endpoint Group | Endpoint | Method | Description |
|---------------|----------|--------|------------|
| Users | /admin/users | GET | List users with pagination, search, filters |
| Users | /admin/users/{userId} | GET | Detailed user profile with all related data |
| Users | /admin/users/{userId}/status | PATCH | Update user/wallet status |
| Analytics | /admin/analytics/overview | GET | Dashboard overview metrics |
| Analytics | /admin/analytics/spend | GET | Aggregate spend analytics with filters |
| Analytics | /admin/analytics/revenue | GET | Revenue and MDR metrics |
| Analytics | /admin/analytics/funnel | GET | Customer journey funnel data |
| Campaigns | /admin/campaigns | GET/POST | List and create engagement campaigns |
| Campaigns | /admin/campaigns/{id} | GET/PATCH/DELETE | Campaign CRUD operations |
| Campaigns | /admin/campaigns/{id}/send | POST | Execute campaign delivery |
| Chat | /admin/chat-history | GET | List AI bot conversations |
| Chat | /admin/chat-history/{id} | GET | Full conversation transcript |
| Audit | /admin/audit-log | GET | Admin activity audit trail |
| RBAC | /admin/roles | GET/POST | List and create admin roles |
| RBAC | /admin/admins | GET/POST | List and create admin users |
| RBAC | /admin/admins/{id} | PATCH/DELETE | Update/remove admin users |

---

## 8. Data Requirements

| Data Point | Source Service | Refresh Rate | Storage | Retention |
|-----------|---------------|-------------|---------|-----------|
| User Profiles | User Service DB | Real-time | PostgreSQL | Lifetime |
| KYC Status & Documents | KYC Service | Real-time | PostgreSQL + S3 | 7 years (regulatory) |
| Wallet Balances | Wallet Service | Real-time | PostgreSQL | Lifetime |
| Transaction Ledger | Ledger Service | Real-time | TimescaleDB | 7 years |
| Spend Analytics | Analytics Pipeline | Hourly aggregation | TimescaleDB | 3 years |
| Revenue / MDR | Payment Processing | Daily aggregation | PostgreSQL | 7 years |
| Funnel Metrics | Event Tracking | Daily computation | PostgreSQL | 2 years |
| Chat Transcripts | AI Bot Service | Real-time | MongoDB / S3 | 1 year |
| Campaign Data | Notification Service | Real-time | PostgreSQL | 2 years |
| Admin Audit Logs | Dashboard Backend | Real-time | Append-only DB | 5 years |

---

## 9. Non-Functional Requirements

### 9.1 Performance

| Metric | Requirement | Measurement |
|--------|-----------|-------------|
| Dashboard Initial Load | < 3 seconds | Time to first meaningful paint |
| Chart Rendering | < 1 second | Time from data receipt to chart visible |
| Table Pagination | < 500ms | Time to load next page of results |
| Real-time Feed Delay | < 2 seconds | Transaction to dashboard display latency |
| Search Results | < 1 second | Time from query to results displayed |
| Export Generation | < 10 seconds | For datasets up to 100K rows |

### 9.2 Security

- All data encrypted in transit using TLS 1.3
- PII masked by default; reveal-on-click with audit log entry
- Role-based data access enforced at API and UI levels
- Admin activity audit trail for all state-changing operations
- CSRF protection on all form submissions
- Rate limiting: 100 requests/minute per admin user
- Content Security Policy (CSP) headers

### 9.3 Scalability

- Support 10,000+ concurrent admin users
- Handle 1M+ wallet user records with efficient pagination
- Virtual scrolling for large tables and real-time feeds
- Lazy loading for charts and non-critical dashboard sections

### 9.4 Availability

- 99.9% uptime SLA (< 8.76 hours downtime per year)
- Graceful degradation: Dashboard functional even if individual services are unavailable
- Cached data display with staleness indicators when live data unavailable

### 9.5 Compliance

- RBI PPI Master Directions adherence
- Data retention policies per regulatory requirements
- Audit-ready reports exportable at any time
- PII handling per IT Act and data protection guidelines

---

## 10. Integration Points

### 10.1 Existing APIs (Reuse)

- Wallet APIs: /wallet/balance, /wallet/ledger, /wallet/status
- KYC APIs: /kyc/status, /kyc/initiate, /kyc/aadhaar/*
- Limits APIs: /limits/check, /limits/usage
- Saga APIs: /saga/add-money, /saga/merchant-pay, /saga/p2p-transfer, /saga/wallet-to-bank, /saga/bill-pay
- Health Check: /health

### 10.2 New APIs Required

- Admin User Management: CRUD for admin users and roles
- Aggregate Analytics: Pre-computed metrics and trend data
- Campaign Management: CRUD for engagement campaigns with delivery
- Audit Logging: Append-only admin action recording
- Chat History: AI bot conversation retrieval and analysis
- Funnel Analytics: Customer journey stage computation
- Revenue Metrics: MDR calculation and business KPIs

### 10.3 Third-party Integrations

- CPaaS Platform: Twilio or Gupshup for SMS campaign delivery
- Push Notifications: Firebase Cloud Messaging (FCM) for mobile push
- Email Service: SendGrid or AWS SES for email campaigns
- Analytics: Optional Mixpanel/Amplitude integration for product analytics

### 10.4 MCP Integration (AI Agent Layer)

The admin dashboard integrates with the MCP (Model Context Protocol) ecosystem:

- **MCP Server:** 35 tools accessible via Claude Desktop for natural language wallet management
- **Chat Handler:** Agentic AI with multi-turn tool use for admin and user queries
- **Render Backend:** Serves MCP data (200 users, 500+ transactions) to the dashboard in PaginatedResponse format
- **Transaction Sync:** Wallet app mutations sync to Render backend, keeping MCP data and dashboard in sync
- **AI Summarizer:** Claude API integration for transaction pattern analysis (POST /api/summarise-transactions)

---

## 11. Phased Rollout Plan

### Phase 1: MVP (Weeks 1-4)

Core operational dashboard with essential user and transaction management.

- Dashboard Home with key metrics cards and quick stats
- User Management: List, search, filter, detail view, status management
- Transaction Monitoring: List, filters, detail view with saga lifecycle
- Basic KYC Management: Status overview, verification queue
- Access Control: Super Admin and Operations Manager roles

**Deliverable:** Functional admin dashboard for operations team to manage day-to-day wallet operations.

### Phase 2: Analytics (Weeks 5-7)

Business intelligence and revenue tracking capabilities.

- Spend Analytics with category breakdown and time-based analysis
- Revenue & Business Dashboard with GMV, MDR, and KPIs
- MoM Trend Charts for all key metrics
- Business KPI tracking with targets

**Deliverable:** Data-driven business dashboard for product and leadership teams.

### Phase 3: Engagement (Weeks 8-10)

Customer journey optimization and targeted engagement tools.

- Customer Journey Funnel with drop-off analysis
- Cohort Builder for targeted user segmentation
- Nudge System with in-app notifications and CPaaS integration
- Campaign Management with scheduling and A/B testing

**Deliverable:** Growth tools for marketing and product teams to drive user engagement.

### Phase 4: AI & Insights (Weeks 11-12)

Intelligent insights and customer support optimization.

- CST AI Bot Chat Review with conversation analysis
- Product Improvement Insights with RAG metrics matrix
- Automated Recommendations engine
- Anomaly detection and trend alerts

**Deliverable:** AI-powered insights platform for continuous product improvement.

### Phase 5: Polish & Scale (Weeks 13-14)

Production hardening, full RBAC, and enterprise features.

- Full RBAC implementation with all 6 roles
- Comprehensive audit logging
- CSV/PDF export for all data views and reports
- Performance optimization, caching, and virtual scrolling
- User acceptance testing and documentation

**Deliverable:** Production-ready admin dashboard with enterprise-grade security and performance.

---

## 12. Success Metrics

### 12.1 Operational Efficiency

| Metric | Current State | Target | Measurement |
|--------|-------------|--------|-------------|
| Manual User Lookup Time | 15-20 minutes | < 2 minutes (50% reduction) | Time from request to resolution |
| KYC Reviews via Dashboard | 0% (manual process) | 80% of all reviews | Dashboard KYC actions / Total reviews |
| Transaction Issue Identification | 30+ minutes | < 5 minutes | Time from alert to root cause |
| User Status Changes | Database direct access | 100% via dashboard | Dashboard actions / Total changes |

### 12.2 Business Impact

| Metric | Baseline | Target | Timeline |
|--------|---------|--------|----------|
| KYC Completion Rate | Current rate | +20% improvement | 90 days post-nudge launch |
| Dormant Wallet Reactivation | Current rate | +15% reactivation | 60 days post-campaign launch |
| CS Escalation Rate | Current rate | -10% reduction | 90 days post-AI bot review |
| Revenue Visibility | Manual reports | Real-time dashboard | Phase 2 completion |

### 12.3 Product Quality

- Internal NPS score > 70 from dashboard admin users
- Less than 5 bugs reported per sprint after stabilization
- Dashboard uptime: 99.9% availability
- All critical user flows completable without errors

---

## 13. Risks & Mitigations

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|------------|-------------------|
| Data volume causing slow dashboard queries | High | Medium | Materialized views for aggregate metrics, cursor-based pagination, Redis caching for frequently accessed data, TimescaleDB for time-series optimization |
| PII exposure through admin dashboard | High | Low | RBAC with PII masking by default, reveal-on-click with audit logging, encrypted data at rest and in transit, regular access reviews |
| Real-time feed overwhelming frontend | Medium | Medium | WebSocket throttling, virtual scrolling for large lists, configurable refresh intervals, backpressure mechanisms |
| Multiple backend service integration failures | Medium | High | API gateway with circuit breakers, graceful degradation with cached data display, service health monitoring, fallback to last-known-good data |
| Admin credential compromise | High | Low | MFA enforcement, session timeout, concurrent session limits, IP whitelisting for elevated roles, suspicious activity alerts |
| Scope creep delaying delivery | Medium | High | Phased rollout with clear MVP scope, weekly stakeholder reviews, feature prioritization using MoSCoW method |
| CPaaS integration reliability | Medium | Medium | Multi-provider setup, failover configuration, delivery receipt tracking, retry mechanisms with exponential backoff |

---

## 14. Appendix

### Appendix A: Complete API Reference

All API endpoints from the PPI Wallet backend. Base URL: `http://localhost:3000/v1`

Timeout: 3000ms | Format: JSON | Amounts: All monetary values in paise (integer)

#### A.1 Wallet APIs

**GET /wallet/balance/{walletId}**

Fetches real-time wallet balance including held and available amounts.

**Path Params:** walletId (string) - Unique wallet identifier

**Response Fields:**

| Field | Type | Description |
|-------|------|------------|
| wallet_id | string | Unique wallet identifier |
| user_id | string | Associated user identifier |
| balance_paise | string (BigInt) | Total balance in paise |
| held_paise | string (BigInt) | Amount currently held/frozen |
| available_paise | string (BigInt) | Spendable balance (balance - held) |
| kyc_tier | enum | 'MINIMUM' \| 'FULL' |
| is_active | boolean | Whether wallet is active |
| updated_at | string (ISO) | Last balance update timestamp |

**GET /wallet/ledger/{walletId}**

Retrieves paginated transaction history with cursor-based pagination.

**Path Params:** walletId (string)

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|---------|------------|
| cursor | string | No | Pagination cursor from previous response |
| limit | number | No | Results per page (default: 20) |
| entry_type | string | No | Filter: CREDIT \| DEBIT \| HOLD \| HOLD_RELEASE |
| from | string (ISO) | No | Start date filter |
| to | string (ISO) | No | End date filter |

**Response: LedgerEntry[] with pagination**

| Field | Type | Description |
|-------|------|------------|
| entries[].id | string | Unique ledger entry ID |
| entries[].entry_type | enum | 'CREDIT' \| 'DEBIT' \| 'HOLD' \| 'HOLD_RELEASE' |
| entries[].amount_paise | string (BigInt) | Transaction amount in paise |
| entries[].balance_after_paise | string (BigInt) | Balance after this entry |
| entries[].held_paise_after | string (BigInt) | Held amount after this entry |
| entries[].transaction_type | string | E.g., ADD_MONEY, MERCHANT_PAY, P2P_TRANSFER |
| entries[].reference_id | string \| null | External reference (saga_id) |
| entries[].description | string \| null | Human-readable description |
| entries[].idempotency_key | string | UUID for deduplication |
| entries[].hold_id | string \| null | Related hold entry ID |
| entries[].created_at | string (ISO) | Entry creation timestamp |
| pagination.next_cursor | string \| null | Cursor for next page (null if last) |
| pagination.has_more | boolean | Whether more entries exist |

**GET /wallet/status/{walletId}**

Returns comprehensive wallet status including state, KYC, balances, and activity timestamps.

**Response Fields:**

| Field | Type | Description |
|-------|------|------------|
| wallet_id | string | Unique wallet identifier |
| user_id | string | Associated user identifier |
| state | enum | 'ACTIVE' \| 'SUSPENDED' \| 'DORMANT' \| 'EXPIRED' \| 'CLOSED' |
| kyc_tier | enum | 'MINIMUM' \| 'FULL' |
| balance_paise | string (BigInt) | Total balance |
| held_paise | string (BigInt) | Held/frozen amount |
| available_paise | string (BigInt) | Spendable balance |
| is_active | boolean | Whether wallet is operational |
| wallet_expiry_date | string \| null | KYC expiry date (MIN_KYC: 12 months) |
| last_activity_at | string \| null | Last transaction timestamp |
| created_at | string (ISO) | Wallet creation date |
| updated_at | string (ISO) | Last modification date |

#### A.2 KYC APIs

**GET /kyc/status/{walletId}**

Retrieves current KYC verification status and tier information.

**Response Fields:**

| Field | Type | Description |
|-------|------|------------|
| wallet_id | string | Wallet identifier |
| kyc_state | enum | 'MIN_KYC' \| 'FULL_KYC' \| 'PENDING' \| 'REJECTED' \| 'EXPIRED' |
| kyc_tier | enum | 'MINIMUM' \| 'FULL' |
| wallet_expiry_date | string \| null | Wallet validity expiry date |
| ckyc_number | string \| null | Central KYC number (masked) |
| pan_masked | string \| null | PAN card number (masked, e.g., XXXX1234X) |
| aadhaar_verified | boolean | Whether Aadhaar OTP was verified |

**POST /kyc/initiate**

Initiates the KYC verification process for a wallet.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Target wallet ID |
| name | string | Yes | Full legal name as per ID document |
| dob | string | Yes | Date of birth (YYYY-MM-DD format) |
| gender | string | No | Gender (optional) |

**Response:** kyc_profile_id (string), kyc_state (string), next_step (string)

**POST /kyc/aadhaar/otp-send**

Sends OTP to the mobile number linked with the provided Aadhaar number.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Target wallet ID |
| aadhaar_number | string | Yes | 12-digit Aadhaar number |
| transaction_id | string | Yes | Unique transaction reference for this OTP session |

**Response:** transaction_id (string), expires_in_seconds (number)

**POST /kyc/aadhaar/otp-verify**

Verifies the OTP received on Aadhaar-linked mobile to complete Full KYC.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Target wallet ID |
| aadhaar_number | string | Yes | 12-digit Aadhaar number |
| otp | string | Yes | 6-digit OTP received on Aadhaar mobile |
| transaction_id | string | Yes | Transaction ID from otp-send response |

**Response:** new_state (string - 'FULL_KYC'), wallet_expiry_date (string)

#### A.3 Limits APIs

**POST /limits/check**

Validates whether a transaction is permitted under the current wallet limits (KYC-tier based).

**Request Body:**

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Wallet to check limits for |
| txn_type | string | Yes | Transaction type (ADD_MONEY, P2P_TRANSFER, etc.) |
| amount_paise | number | Yes | Transaction amount in paise |

**Response Fields:**

| Field | Type | Description |
|-------|------|------------|
| allowed | boolean | Whether the transaction is permitted |
| wallet_id | string | Wallet identifier |
| kyc_tier | enum | 'MINIMUM' \| 'FULL' |
| txn_type | string | Transaction type checked |
| amount_paise | string | Amount validated |

**GET /limits/usage/{walletId}**

Returns current limit utilization for the wallet including MTD and YTD figures.

**Response Fields:**

| Field | Type | Description |
|-------|------|------------|
| wallet_id | string | Wallet identifier |
| kyc_tier | enum | 'MINIMUM' \| 'FULL' |
| current_balance_paise | string (BigInt) | Current wallet balance |
| monthly_p2p_mtd_paise | string (BigInt) | P2P transfers month-to-date |
| annual_load_ytd_paise | string (BigInt) | Wallet loads year-to-date |

**Limit Thresholds (per RBI PPI Master Directions):**

| Limit | MINIMUM KYC | FULL KYC |
|-------|------------|----------|
| Maximum Balance | ₹10,000 | ₹2,00,000 |
| Monthly P2P Transfer | Not Allowed | ₹2,00,000 |
| Annual Wallet Load | ₹1,20,000 | ₹12,00,000 |
| Per Transaction | ₹10,000 | ₹2,00,000 |
| Wallet Validity | 12 months from issuance | Indefinite (subject to re-KYC) |

#### A.4 Saga/Transaction APIs

All transaction endpoints use the Saga pattern for distributed consistency. Every request requires an idempotency_key (UUID v4) to prevent duplicate processing.

**Common Response Type: SagaResponse**

| Field | Type | Description |
|-------|------|------------|
| saga_id | string | Unique saga/transaction identifier |
| saga_type | enum | 'ADD_MONEY' \| 'MERCHANT_PAY' \| 'P2P_TRANSFER' \| 'WALLET_TO_BANK' \| 'BILL_PAY' |
| status | enum | 'STARTED' \| 'RUNNING' \| 'COMPLETED' \| 'COMPENSATING' \| 'COMPENSATED' \| 'DLQ' |
| result | object \| null | Transaction result data (on success) |
| error | string \| null | Error message (on failure) |

**POST /saga/add-money**

Initiates a wallet top-up (add money) transaction.

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Target wallet to credit |
| amount_paise | number | Yes | Amount to add in paise (e.g., 50000 = ₹500) |
| idempotency_key | string (UUID) | Yes | Auto-generated unique key for deduplication |

**POST /saga/merchant-pay**

Initiates a payment to a merchant (Scan & Pay / QR payment).

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Payer wallet ID |
| merchant_id | string | Yes | Merchant identifier (from QR or manual entry) |
| amount_paise | number | Yes | Payment amount in paise |
| idempotency_key | string (UUID) | Yes | Deduplication key |

**POST /saga/p2p-transfer**

Initiates a person-to-person wallet transfer. Requires FULL KYC per RBI compliance.

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Sender wallet ID |
| beneficiary_wallet_id | string | Yes | Recipient wallet ID |
| amount_paise | number | Yes | Transfer amount in paise |
| idempotency_key | string (UUID) | Yes | Deduplication key |

**POST /saga/wallet-to-bank**

Initiates a withdrawal from wallet to a bank account.

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Source wallet ID |
| amount_paise | number | Yes | Withdrawal amount in paise |
| bank_account | string | Yes | Destination bank account number (up to 20 digits) |
| ifsc | string | Yes | Bank IFSC code (e.g., HDFC0001234) |
| idempotency_key | string (UUID) | Yes | Deduplication key |

**POST /saga/bill-pay**

Initiates a bill payment for utilities (Electricity, Water, Gas, DTH, Broadband, Insurance).

| Field | Type | Required | Description |
|-------|------|---------|------------|
| wallet_id | string | Yes | Payer wallet ID |
| biller_id | string | Yes | Biller/Provider identifier |
| biller_ref | string | Yes | Consumer/Reference number at the biller |
| amount_paise | number | Yes | Bill amount in paise |
| idempotency_key | string (UUID) | Yes | Deduplication key |

#### A.5 System APIs

**GET /health**

Backend health check endpoint. Called silently at app startup to determine if the real backend is available. If this fails (timeout: 3000ms), the app automatically falls back to mock data stored in sessionStorage.

**Response:** 200 OK if healthy, timeout/error triggers mock mode

#### A.6 Common Error Response Format

All API errors return a consistent structure:

| Field | Type | Description |
|-------|------|------------|
| code | string | Error code (e.g., NETWORK_ERROR, INSUFFICIENT_BALANCE, KYC_REQUIRED) |
| message | string | Human-readable error description |
| statusCode | number | HTTP status code (400, 401, 403, 404, 500) |

**Common Error Codes:**

- `NETWORK_ERROR` - Backend unreachable or timeout
- `INSUFFICIENT_BALANCE` - Available balance less than transaction amount
- `KYC_REQUIRED` - Operation requires higher KYC tier
- `LIMIT_EXCEEDED` - Transaction exceeds KYC-tier limit
- `WALLET_INACTIVE` - Wallet is suspended, dormant, expired, or closed
- `DUPLICATE_REQUEST` - Idempotency key already processed
- `INVALID_BENEFICIARY` - Target wallet ID not found
- `INVALID_IFSC` - Bank IFSC code format validation failed

### Appendix B: MCC Category Mapping

The wallet app uses 14 spending categories derived from Merchant Category Codes (MCC): Taxi/Ride, Food & Dining, Groceries, Shopping, Fuel, Travel, Entertainment, Health, Education, Utilities, Money Transfer, Refunds, Recharge, Insurance.

Smart category detection is performed from transaction descriptions and MCC codes via the mcc.ts utility module.

> **Updated April 2026:** The MCC mapping system now includes 19 categories (expanded from 14) with smart keyword detection from transaction descriptions. Categories: Taxi/Ride, Food & Dining, Groceries, Shopping, Fuel, Travel, Entertainment, Health, Education, Utilities, Money Transfer, Refunds, Recharge, Insurance, Bank Transfer, Wallet Top-up, Subscription, Government, and Others. 23 merchant names are used for realistic transaction descriptions.

### Appendix C: Notification Types

| Type | Color | Usage | Example |
|------|-------|-------|---------|
| Credit | Green | Money received notifications | Received ₹500 from Wallet_ABC |
| Debit | Red | Payment/transfer notifications | Paid ₹200 to Merchant_XYZ |
| Reward | Amber | Cashback and reward notifications | You won ₹50 cashback! |
| Alert | Orange | Warnings and important notices | Your KYC expires in 7 days |
| Info | Blue | General informational messages | New feature: Budget Controls |

### Appendix D: Wallet States & Transitions

| State | Description | Allowed Actions |
|-------|------------|----------------|
| ACTIVE | Fully functional wallet | All transactions, KYC upgrade |
| SUSPENDED | Temporarily disabled by admin | View only, no transactions |
| DORMANT | No activity for 90+ days | Reactivation via transaction or admin action |
| EXPIRED | KYC validity lapsed | Re-verification required, limited access |
| CLOSED | Permanently closed | No actions, balance refunded |

### Appendix E: KYC States & Transitions

- MIN_KYC -> FULL_KYC (via Aadhaar verification)
- MIN_KYC -> EXPIRED (after 12 months)
- PENDING -> FULL_KYC (approved) or REJECTED (with reason)
- REJECTED -> PENDING (resubmission)
- EXPIRED -> PENDING (re-verification initiated)

### Appendix F: Transaction Saga Lifecycle

All monetary transactions follow the Saga pattern for distributed consistency:

1. **STARTED:** Transaction initiated, idempotency key generated
2. **RUNNING:** Processing in progress, funds may be held
3. **COMPLETED:** Transaction successful, ledger updated, balance adjusted
4. **COMPENSATING:** Failure detected, rollback initiated
5. **COMPENSATED:** Rollback complete, funds returned
6. **DLQ (Dead Letter Queue):** Unresolvable failure, requires manual investigation

### Appendix G: MCP Tool Reference (35 Tools)

The MCP server exposes 35 tools for AI agent interaction, organized into 9 categories:

- **User & Wallet (10):** get_wallet_balance (+ include_runway), get_transaction_history, get_spending_summary (+ group_by=merchant), search_transactions, get_user_profile (+ include_limits), compare_spending, detect_recurring_payments, compare_users, generate_report, flag_suspicious_transaction, unflag_transaction
- **Admin & Platform (6):** get_system_stats, search_users, get_flagged_transactions, suspend_user, get_failed_transactions, get_kyc_stats
- **Transaction Operations (5):** add_money, pay_merchant, transfer_p2p, pay_bill, request_refund
- **Compliance (1):** check_compliance (+ include_risk_score)
- **Disputes & Support (3):** raise_dispute, get_dispute_status, get_refund_status
- **Notifications (2):** get_notifications, set_alert_threshold
- **KYC Actions (3):** approve_kyc, reject_kyc, request_kyc_upgrade
- **Analytics (2):** get_peak_usage, get_monthly_trends
- **KYC Expiry & Renewal (2):** query_kyc_expiry (+ urgency, sort_by, include_expired), generate_kyc_renewal_report

**KYC Expiry Alert System (API Endpoints):**

- `GET /api/kyc-alerts/preview` - Returns at-risk user count and total balance (no Claude API call)
- `POST /api/kyc-alerts/run` - Runs full alert engine: detects critical expirations, generates AI messages via Claude Haiku, simulates SMS delivery, returns ops summary
- **Dashboard Integration:** KycAlertPanel component on KYC Management page with status bar, Run Alert button, alert table (Name, Phone, Expiry, Days Left, Balance, AI Message, Status), and Ops Summary modal

### Appendix H: Render Backend Endpoint Reference

Base URL: `https://ppi-wallet-api.onrender.com`

- `GET /health` -- Health check ({ ok: true, tools: 35, version: "1.0.0" })
- `GET /api/users?page=&pageSize=&search=&kyc_tier=&status=` -- Paginated user list
- `GET /api/users/:userId` -- User detail with KYC, transactions, spending
- `GET /api/transactions?page=&pageSize=&type=&status=&from=&to=&search=` -- Paginated transactions
- `GET /api/overview` -- Dashboard analytics (total users, active wallets, balances, volumes)
- `GET /api/kyc/stats` -- KYC distribution and completion rates
- `GET /api/kyc/queue` -- Pending KYC queue
- `GET /api/wallet/balance/:walletId` -- Wallet balance
- `GET /api/wallet/ledger/:walletId` -- Transaction history
- `GET /api/wallet/status/:walletId` -- Wallet status
- `POST /api/chat` -- AI chat (body: { message, role, context })
- `POST /api/summarise-transactions` -- AI summarizer (body: { transactions })
- `POST /api/wallet/transact` -- Transaction sync (body: { wallet_id, saga_type, amount_paise, description })

---

*End of Document*
