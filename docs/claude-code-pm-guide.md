**Claude Code**

for Product Managers

How to use AI-assisted coding to build the PPSL PPI Wallet — without writing a single line of code yourself

| For | Product Managers, Business Owners — no coding experience required |
| --- | --- |
| Project | PPSL PPI Wallet — new build from Engineering Brief v1.0 |
| Tool | Claude Code (command-line AI coding assistant by Anthropic) |
| Time to first output | ~30 minutes from installation to your first working API endpoint |
| What you will own | The ability to initiate, review, and steer an engineering build — directly, without going through a developer first |

The Big Idea  You already wrote the PRD and the Engineering Brief. Those documents contain everything Claude Code needs to build the software. Your job is not to write code — it is to have a conversation with Claude Code about what to build, review what it produces, and steer it when it goes off-track. This guide shows you exactly how.

| 01 | What Is Claude Code? The honest, non-technical explanation |
| --- | --- |

Claude Code is an AI assistant that lives in your computer's terminal — the black window that developers use to run commands. You type instructions to it in plain English, and it reads, writes, and edits software code on your behalf. Think of it as a very fast junior developer who never sleeps, never complains, and needs you to tell it what to build.

| What Claude Code IS | What Claude Code is NOT | PM's role |
| --- | --- | --- |
| An AI that writes, reads, and edits code files on your computer | A magic button that produces a finished product | Describe what to build. Review the output. Catch errors. |
| A pair programmer you direct with English sentences | A replacement for your engineering team | Use it to accelerate your team, not bypass them. |
| A tool that can run tests, install dependencies, and fix its own errors | A tool that always gets it right first time | Review every output. Treat Claude Code like a new hire — it needs your context. |
| Aware of your whole codebase once you open a project folder | Connected to the internet or your live bank systems | Share only your project files. Never paste real API keys or credentials into Claude Code. |

Why This Matters for PPI Wallet Build  The Engineering Brief you now hold contains 50+ API endpoint specs, 8 database table schemas, and a 10-step reconciliation engine design. In a traditional build, a PM hands this to an engineering lead and waits. With Claude Code, you can spin up a working skeleton of any one of these components in under an hour — test it, give feedback, and hand working code to your engineers instead of just a spec document.

| 02 | Installation & Setup One-time setup, takes 20 minutes |
| --- | --- |

Before You Start  You need: a Mac or Windows PC, an internet connection, and an Anthropic account (claude.ai). You do NOT need to know how to code. Every step below has exactly what to type.

| Step 1 | Install Node.js Node.js is the software that Claude Code runs on. Go to nodejs.org in your browser, download the LTS version (the one marked 'Recommended for most users'), and run the installer. Click through all the defaults. 💡  On a Mac, open Terminal (press Cmd+Space, type 'Terminal', press Enter). On Windows, open Command Prompt (press Windows key, type 'cmd', press Enter). This is where you will type all future commands. |
| --- | --- |

| Step 2 | Install Claude Code In your Terminal or Command Prompt, type the following and press Enter: 💡  This downloads Claude Code from the internet and installs it on your computer. It takes about 2 minutes. |
| --- | --- |

| 📋  Type this in Terminal / Command Prompt |
| --- |
| npm install -g @anthropic-ai/claude-code |

| Step 3 | Log in to your Anthropic account Still in your terminal, type the command below and press Enter. It will open a browser window where you log in with your Anthropic account. Once you log in, come back to the terminal — it will confirm you are authenticated. 💡  Use the same email you use for claude.ai. If you do not have an account, create one at claude.ai first. |
| --- | --- |

| 📋  Type this — it launches Claude Code and prompts for login |
| --- |
| claude |

| Step 4 | Create a project folder for PPI Wallet Create a folder on your computer called 'ppsl-ppi-wallet'. This is where all the code will live. Then navigate to it in the terminal. 💡  Keep this folder somewhere easy to find — your Desktop or Documents folder works well. |
| --- | --- |

| 📋  Create and enter your project folder |
| --- |
| # Mac / Linux: mkdir ~/Desktop/ppsl-ppi-wallet cd ~/Desktop/ppsl-ppi-wallet  # Windows: mkdir %USERPROFILE%\Desktop\ppsl-ppi-wallet cd %USERPROFILE%\Desktop\ppsl-ppi-wallet |

| Step 5 | Open Claude Code inside your project folder With your terminal pointing at the ppsl-ppi-wallet folder, type 'claude' and press Enter. Claude Code will start up, read your folder, and wait for your first instruction. You will see a > prompt. You are ready. 💡  Claude Code now knows about any files in this folder. When you paste Engineering Brief prompts, it will create files here. |
| --- | --- |

| 📋  Launch Claude Code in your project folder |
| --- |
| claude |

| 03 | Your First 30 Minutes From zero to a working wallet API skeleton |
| --- | --- |

This section walks you through your first Claude Code session, step by step. By the end, you will have a runnable Node.js project with your first PPI Wallet API endpoint — POST /wallet/credit — fully scaffolded with Prisma schema and a compliance test. No coding required from you.

## 3.1  Setting the Scene (Context Prompt)

The first thing you do in every Claude Code session is give it context. Think of it as the briefing you give a new team member on Day 1. Paste this prompt exactly:

| 📋  Session context prompt — paste this first |
| --- |
| I am the Product Manager at Paytm Payment Services Limited (PPSL). We are building a new RBI-regulated PPI (Prepaid Payment Instrument) wallet product. I have a detailed Engineering Brief that specifies the API endpoints, database schemas, and reconciliation engine we need to build.  For this session, I want you to: 1. Scaffold a new Node.js + TypeScript + Fastify + Prisma project 2. Set up PostgreSQL schema for wallet_accounts and ledger_entries 3. Implement POST /wallet/credit as the first endpoint  Key compliance rules you must follow: - All monetary amounts are stored as BIGINT in paise (never DECIMAL or FLOAT) - ledger_entries is append-only — no UPDATE or DELETE ever - POST /wallet/credit must write to ledger_entries BEFORE returning a response - Every endpoint needs a unique idempotency_key to prevent duplicate entries  Start by creating the project structure and package.json. |

## 3.2  What Claude Code Will Do Next

After you paste that prompt, Claude Code will start working. You will see it create files in your folder. Here is what it is doing and what you should watch for:

| What you see in the terminal | What Claude Code is actually doing | What to check |
| --- | --- | --- |
| Creating files: package.json, tsconfig.json... | Setting up the Node.js project skeleton — the equivalent of a blank notebook with the right stationery. | A new file should appear in your ppsl-ppi-wallet folder. Open it in any text editor to see. |
| Running: npm install... | Downloading the libraries the project needs (Fastify, Prisma, TypeScript). Like buying ingredients before cooking. | Normal. Takes 1-2 minutes. A node_modules folder will appear — ignore it. |
| Creating: prisma/schema.prisma... | Writing the database schema — defining what the wallet_accounts and ledger_entries tables will look like. | IMPORTANT: Check that amount fields show BigInt, not Float or Decimal. |
| Creating: src/routes/wallet.ts... | Writing the actual API endpoint code for POST /wallet/credit. | Ask Claude Code: 'Does this endpoint write to ledger_entries before returning?' It should say yes. |
| Running: npx prisma generate... | Generating the database access layer from your schema file. | If you see errors here, paste the error message back to Claude Code and ask it to fix them. |

## 3.3  Your First Compliance Verification

Once Claude Code has created the endpoint, ask it to prove it satisfies the RBI compliance rule. This is a key PM skill — you do not need to read the code, but you do need to verify the behaviour.

| 📋  Ask Claude Code to write compliance tests |
| --- |
| Good. Now write an automated test that proves POST /wallet/credit satisfies the RBI PPI compliance rule: the ledger entry must be created BEFORE the endpoint returns HTTP 200.  The test should: 1. Send a POST /wallet/credit request 2. Intercept the response 3. Query the ledger_entries table 4. Assert that the ledger entry exists BEFORE the response was returned  Also write a test that proves a duplicate request with the same idempotency_key returns the same result but does NOT create a second ledger entry. |

PM Insight  You may not understand the test code Claude Code writes. That is fine. What matters is that you can read the test descriptions — they should say things like 'should create ledger entry before returning 200' and 'should not create duplicate entry for same idempotency_key'. If the test descriptions do not match the compliance requirement, tell Claude Code to fix the description.

| 04 | Using the Engineering Brief as Your Prompt Source Every table in the brief maps to a Claude Code prompt |
| --- | --- |

The Engineering Brief (PPI_Wallet_Engineering_Brief_v1.0.docx) is not just a document for your engineering team. Every section in it was written to be a Claude Code instruction. Here is how to map each section to a session:

| Engineering Brief Section | What to ask Claude Code | Example starter prompt |
| --- | --- | --- |
| Section 2.1 — Wallet Core API | Build all 7 Wallet Core endpoints with Prisma + Fastify | "Implement all endpoints from the Wallet Core service: POST /wallet/credit, POST /wallet/debit, GET /wallet/balance/{id}, GET /wallet/ledger/{id}, POST /wallet/hold, POST /wallet/release, POST /wallet/idempotency/check. Use the ledger-first pattern — ledger write must happen before any external call." |
| Section 2.3 — Limits Engine | Build the limits check service with Redis counters | "Build POST /limits/check. Hard rule: if the initiating wallet is MIN_KYC tier and txn_type is P2P, return HTTP 403 with reason_code PPI_P2P_NOT_ALLOWED immediately — do not check any other limit. For all other cases, check balance ceiling and monthly P2P cap from the compliance_rules table. Use Redis INCR for MTD counters." |
| Section 3.1 — Ledger Schema | Generate Prisma migrations for all ledger tables | "Generate Prisma schema and migration for ledger_entries. Requirements: id is UUID, amount_paise is BigInt (NOT Decimal), idempotency_key has a unique constraint, there is a PostgreSQL row-security policy that denies UPDATE and DELETE for all roles except superuser. Add an index on wallet_id and created_at." |
| Section 4 — Recon Engine | Build the 10-step reconciliation Saga | "Build the daily reconciliation engine as a Kafka consumer. The engine must process a recon.trigger message and execute 10 steps: (1) create recon_runs row, (2) compute wallet_system_total from ledger_entries, (3) ingest bank statement, (4) normalise, (5) match on reference_id, (6) classify breaks into the 4 break types, (7) compute variance, (8) auto-resolve timing differences, (9) write results, (10) emit Kafka event and alert if variance > 0." |
| Section 6.2 — Compliance Tests | Generate all 10 mandatory compliance integration tests | "Write integration tests for all 10 compliance test cases listed in Section 6.2 of the Engineering Brief: MIN_KYC P2P block, balance ceiling, duplicate idempotency, debit-first compensation, float monitor killswitch, CKYC lookup bypass, wallet expiry block, recon idempotency, UPI FULL_KYC gate, PAN log scrubber." |

## 4.1  The Anatomy of a Good Claude Code Prompt

The quality of what Claude Code builds is directly proportional to the quality of your prompt. Here is the structure that works best for technical builds:

| Prompt Part | What it does | Example |
| --- | --- | --- |
| Context / Role | Reminds Claude Code which project and what rules apply | "We are building the PPSL PPI Wallet. All money amounts are BIGINT paise." |
| What to build | Specific endpoint, schema, or component | "Implement POST /money/transfer-p2p from the Money Movement Service." |
| Non-negotiable rules | Compliance constraints that must not be violated | "Hard rule: MIN_KYC wallets must return HTTP 403 PPI_P2P_NOT_ALLOWED. This is not configurable." |
| Verify behaviour | Ask for proof via a test or explanation | "Write a test that proves the MIN_KYC block fires before the limits check." |
| Ask what is missing | Forces Claude Code to surface gaps proactively | "What edge cases or error scenarios have you not handled in this endpoint?" |

| 05 | The PM Review Workflow How to review code without reading code |
| --- | --- |

You do not need to understand every line of code Claude Code writes. Your job is to verify that it built the right thing. Here are five review techniques that require zero coding knowledge:

## Technique 1 — Ask For a Plain English Summary

| 📋  Review prompt — ask for plain English explanation |
| --- |
| Explain what the code in src/routes/wallet.ts does in plain English. Tell me: (1) what it does when a request arrives, (2) what database tables it reads or writes, (3) what happens if the database write fails, and (4) what compliance rules it enforces. Use no technical jargon. |

## Technique 2 — Ask About the Failure Mode

Compliance failures often happen in edge cases. Always ask about what happens when things go wrong:

| 📋  Review prompt — failure mode check |
| --- |
| Walk me through exactly what happens if the database goes down halfway through processing a POST /wallet/debit request.  Specifically: does the customer's money get debited without the ledger entry being created? If yes, fix it. The ledger entry MUST be atomic with the debit operation. |

## Technique 3 — Run the Compliance Test Suite

The Engineering Brief contains 10 mandatory compliance tests. Ask Claude Code to run them and show you a pass/fail summary:

| 📋  Run compliance tests and report |
| --- |
| Run all the compliance integration tests and show me a summary of which tests pass and which fail. For any failing test, tell me: (1) what compliance rule it is testing, (2) why it is failing, and (3) what you will fix. |

## Technique 4 — The Security Checklist

Ask Claude Code to self-audit against the security rules in Section 6.1 of the Engineering Brief:

| 📋  Security self-audit prompt |
| --- |
| Audit the entire codebase against these 7 security rules: 1. No Aadhaar number stored anywhere — only UIDAI VID token 2. PAN never appears in log output — masked as ****NN 3. All monetary amounts are BigInt, never Decimal or Float 4. KYC documents encrypted with AES-256-GCM 5. ledger_entries has no UPDATE or DELETE routes 6. PPI_NODAL and PA_ESCROW accounts are never transferred between 7. All infrastructure is India-region only  For each rule: tell me PASS (with file reference) or FAIL (with fix needed). |

## Technique 5 — Ask For the API Contract

Ask Claude Code to document what it built. You can then compare this against the Engineering Brief to check for gaps:

| 📋  Generate OpenAPI documentation |
| --- |
| Generate an OpenAPI 3.1 spec for all endpoints you have built so far. For each endpoint, include: method, path, request body schema, response codes, and a one-line description of the compliance rule it satisfies. Export it as openapi.yaml in the project root. |

| 06 | Do's and Don'ts for PMs Using Claude Code Lessons from running AI-assisted builds |
| --- | --- |

| ✅  DO | ❌  DON'T |
| --- | --- |
| Start every session by pasting the full context prompt (project, rules, compliance constraints). Claude Code has no memory between sessions. | Assume Claude Code remembers what you told it last time. It does not. Each new terminal session starts fresh. |
| Give Claude Code one specific component to build per prompt. 'Build the Limits Engine' works better than 'Build the whole wallet'. | Ask Claude Code to build everything at once. Large vague prompts produce large vague code. |
| Ask Claude Code to explain what it built in plain English before moving on. This catches misunderstandings early. | Blindly approve everything Claude Code produces without checking it matches your spec. |
| Paste error messages directly back to Claude Code and say 'Fix this'. It can debug its own output. | Try to fix errors yourself in the code. You might break something. Let Claude Code fix its own work. |
| Use the compliance test prompts from Section 6.2 of the Engineering Brief as your acceptance criteria. If the test passes, the feature is done. | Mark a feature as complete just because Claude Code says it is done. Tests are the truth. |
| Keep a running notes document of prompts that worked well. Build a library of reusable prompts for this project. | Retype prompts from scratch every time. Save the ones that worked. |
| Loop in your engineering lead to review the output of each sprint. Claude Code builds fast; a senior engineer catches subtle issues. | Ship Claude Code output directly to production without engineering review. Use it to accelerate your team, not replace review. |
| Tell Claude Code which document it is working from: 'Use the schema from Section 3.3 of the Engineering Brief'. | Assume Claude Code has read your documents. It has not seen them unless you paste the relevant section. |

| 07 | Sprint-by-Sprint PM Playbook What you do at the start and end of each sprint |
| --- | --- |

The Engineering Brief maps build work to 8 sprints. As PM, your role with Claude Code is different at the start of a sprint versus the end. Here is the cadence:

| Sprint | PM start-of-sprint action | Claude Code session goal | PM end-of-sprint review |
| --- | --- | --- | --- |
| S0 (Pre-build) | Confirm all 6 compliance blockers cleared. Do NOT open Claude Code yet. | No code. Sprint 0 is purely legal and licensing work. | Write a one-line confirmation that each blocker is cleared. Share with engineering lead before Sprint 1 kick-off. |
| S1 | Paste the project structure and Prisma schema prompts from Section 3.1 of Engineering Brief. | Project scaffold + Prisma migrations for wallet_accounts, ledger_entries, transaction_intents. | Ask: 'Show me the migration file for ledger_entries.' Check that amount_paise is BigInt. Check for unique constraint on idempotency_key. |
| S2 | Paste the Wallet Core API prompts from Section 2.1 of Engineering Brief. | All 7 Wallet Core endpoints working with ledger-first pattern. | Run: 'Execute the duplicate idempotency compliance test. Show me the test result.' |
| S3 | Paste Limits Engine prompts from Section 2.3. Emphasise the MIN_KYC P2P hard block. | Limits Engine with Redis counters and hot-reload config. | Run: 'Execute the MIN_KYC P2P compliance test. Confirm it returns 403 PPI_P2P_NOT_ALLOWED.' |
| S4 | Paste KYC Service prompts from Section 2.2. Emphasise CKYC lookup and 10-year retention. | KYC state machine, CERSAI CKYC integration, MIN_KYC expiry scheduler. | Ask: 'Show me where PAN is stored. Confirm it is only a SHA-256 hash, never the raw PAN.' |
| S5 | Paste Recon Engine prompts from Section 4 of Engineering Brief. This is the most critical sprint. | Full 10-step reconciliation Saga + float monitor + break classification. | Ask: 'Run the recon idempotency test. Run the float monitor killswitch test. Show me results.' |
| S6 | Paste Money Movement Saga prompts from Section 2.4. | All Money Movement Sagas with debit-first pattern and compensating transactions. | Ask: 'Simulate an external rail failure mid-Saga. Show me the ledger state before and after compensation.' |
| S7–8 | Paste UPI and FIU-IND integration prompts from Sections 2.6 and 2.7. | UPI VPA management, third-party link, FIU-IND FINNET 2.0 CTR/STR filing. | Run the full compliance test suite (Section 6.2). All 10 tests must pass before you consider build complete. |

| 08 | When Things Go Wrong Troubleshooting for non-technical PMs |
| --- | --- |

Claude Code will sometimes produce incorrect output, hit errors, or go off-spec. Here is how to handle the most common situations without needing an engineer:

| What you see | What it means | What to say to Claude Code |
| --- | --- | --- |
| Error: Cannot find module / npm install fails | A library is missing or the project setup is incomplete. | "You got an error during setup. Here is the exact error message: [paste error]. Fix it and try again." |
| Compliance test fails: 'expected 403 but got 200' | A compliance rule is not being enforced. This is a real bug, not a cosmetic issue. | "The MIN_KYC P2P compliance test is failing. The endpoint is returning 200 instead of 403. This is a regulatory requirement — it cannot fail. Find and fix the root cause." |
| amount stored as Decimal or Float in schema | Claude Code ignored the BigInt rule. This will cause rounding errors in real money. | "You used Decimal for amount_paise. This is wrong. All monetary amounts must be BigInt in paise. Fix the Prisma schema and regenerate the migration. Check all other amount fields too." |
| Claude Code starts building something you did not ask for | Claude Code made an assumption and started going off-spec. | "Stop. You are building something I did not ask for. Undo the last changes and only build what I specified: [restate your exact requirement]." |
| Claude Code says 'I cannot do that' or 'that is not possible' | Claude Code is stuck or misunderstood your request. | "OK, let us approach it differently. Instead of [what you asked], can you [break it into a smaller step]?" If still stuck, ask an engineer to review. |
| The code builds but the test takes more than 5 minutes | A test may be hanging on a database connection or external service. | "The test is taking too long. Is it waiting for a real database? Replace any external calls in tests with mocks — tests should run in under 10 seconds total." |

| 09 | Frequently Asked Questions From PMs who have used Claude Code on fintech builds |
| --- | --- |

| Can Claude Code connect to our real bank or RBI systems? | No. Claude Code only writes and runs code on your local computer. It has no access to external networks, live APIs, or real banking systems unless you explicitly configure your project to make API calls. In development, all external services are mocked. This is safe and correct. |
| --- | --- |
| What if Claude Code builds something that is not RBI compliant? | This is why the compliance tests exist. Claude Code does not know RBI law — it only knows what you tell it. Use the compliance test suite from Section 6.2 of the Engineering Brief as your acceptance gate. If a compliance test fails, it means Claude Code got it wrong and must fix it. |
| Do I need to share the Engineering Brief document with Claude Code? | You do not upload documents to Claude Code. Instead, you copy and paste the relevant section of the brief into the chat. Claude Code reads what you paste in the current session. For long sections, break them into smaller chunks and paste one at a time. |
| How do I know if the code is production-ready? | Code from Claude Code is always a starting point, not a finished product. Your engineering team must review every output before it goes to staging or production. Think of Claude Code as producing a very good first draft that engineers then refine, harden, and deploy safely. |
| What happens to the code between sessions? Does it disappear? | No. The code files Claude Code creates are saved in your ppsl-ppi-wallet folder on your computer. They persist between sessions. Claude Code itself has no memory of what you discussed, but the files remain. |
| Can two people on the team use Claude Code on the same project? | Yes. Put the project folder in a shared Git repository (GitHub, GitLab, Bitbucket). Each person runs Claude Code in their own copy of the folder. Engineers commit and merge changes as normal. Claude Code is just a tool that creates files — it works with any standard team workflow. |
| Will Claude Code ever make a mistake that costs us real money? | Claude Code runs in your local development environment, not in production. It cannot touch a live database or real customer wallets. The risk of Claude Code making a mistake that reaches production exists only if your team deploys its output without review — which is why the engineering review gate is non-negotiable. |
| The Engineering Brief mentions 'Node.js'. What if my team uses a different language? | Tell Claude Code at the start of your session: 'We use Python / Java / Go — please use that instead of Node.js.' Claude Code will adapt. The Engineering Brief prompts are language-agnostic; the Node.js default was an assumption, not a constraint. |
| How do I hand off Claude Code output to my engineering team? | Commit the files to your Git repository and open a Pull Request. Write in the PR description: 'Generated with Claude Code from Engineering Brief Section X. Compliance tests pass: [list tests]. Please review for production readiness.' Your engineers will then review, improve, and merge. |
| Is the code Claude Code produces secure enough for a PPI product? | Claude Code follows the security rules you give it — it implemented the AES-256 encryption, PAN masking, and Aadhaar VID tokenisation because the Engineering Brief specified them. However, a security audit by a qualified fintech security engineer is mandatory before any PPI product handles real customer data. |

**You now have everything you need to run an AI-assisted build as a non-technical PM.**

The PRD defined what to build. The Engineering Brief defined how to build it.

This guide shows you how to drive the build — one prompt at a time.

PPSL PPI Wallet — Claude Code PM Guide v1.0 — March 2026 — Confidential
