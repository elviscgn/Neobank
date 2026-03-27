# NeoBank SA — Project Master Brief

> SA Digital Banking Platform · 4-Person Microservices Project · BBD Portfolio

---

## Final Project Scope

| Decision | Choice |
|---|---|
| Ledger | Event sourcing — balance derived from ledger entries |
| Account types | Cheque · Savings · Fixed Deposit |
| Fraud rules | Amount anomaly · Velocity check · New beneficiary large payment · Geographic anomaly |
| Auth features | TOTP from scratch (RFC 6238) · Device trust scoring · Step-up auth on large transactions |
| Payments | Internal transfers · Simulated interbank EFT · Proof of payment · Immediate settlement |
| Notifications | Mailtrap (simulated SMTP) |
| Statements | PDF via iText |
| Inter-service comms | REST only |
| Testing | JUnit 5 + Mockito (unit tests) |
| Repo structure | Mono-repo · 4 Maven modules |
| Deployment | Docker Compose (local) |
| Spend analytics | Full category breakdown per month |

---

## 1. Project Overview

NeoBank SA is a distributed digital banking backend modelled on how South African retail banks like Capitec and TymeBank actually operate. The platform is built as four independent Spring Boot microservices in a Maven mono-repo, communicating over REST, secured with bank-grade authentication, and protected by a real-time fraud detection engine.

It is designed specifically to demonstrate the engineering depth that BBD looks for: financial domain knowledge, distributed systems thinking, design patterns applied to real problems, and SA-specific regulatory awareness.

**Tech stack:** Spring Boot · PostgreSQL · Docker Compose · Maven Mono-repo · Swagger/OpenAPI · JUnit 5 + Mockito

### System Architecture

```
           [ Swagger UI / Postman ]
                     |
   ┌─────────────────┼─────────────────┐
   │                 │                 │
Identity Svc    Core Banking     Payments Svc
(Port 8081)      (Port 8082)      (Port 8083)
   │                 │                 │
   └─────────────────┼─────────────────┘
                     │
           Notifications Svc
              (Port 8084)
                     │
          [ PostgreSQL × 4 schemas ]
```

Each service owns its own PostgreSQL schema. No service queries another's database directly. All cross-service communication goes through REST calls. Each service exposes Swagger UI for API documentation.

---

## 2. Service Specifications

### 2.1 Identity & Authentication Service
**Port:** 8081 · **Owner:** Person 1

The security backbone of the entire platform. Every request to any other service must carry a JWT issued by this service. This service also implements bank-grade authentication including TOTP built from scratch using HMAC-SHA1 and RFC 6238 — the same algorithm behind Google Authenticator — without calling an external library.

#### Responsibilities

- Customer registration with SA ID number validation via the Luhn algorithm
- Date of birth, gender, and citizenship extraction from SA ID digits
- FICA verification simulation against a mock Home Affairs response
- KYC document upload and verification tracking
- Login and JWT token issuance with refresh token rotation
- **TOTP two-factor authentication implemented from scratch (RFC 6238 / HMAC-SHA1)**
- Device registration and trust scoring (fingerprint: user-agent, IP subnet, screen hash)
- Step-up authentication demanding TOTP when a transaction exceeds R5 000
- Session anomaly detection — invalidates session on device fingerprint change
- Password reset with time-limited OTP via Mailtrap

#### Key Endpoints

```
POST /api/customers/register
POST /api/customers/verify-fica
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/totp/setup
POST /api/auth/totp/verify
POST /api/auth/reset-password
GET  /api/customers/{id}/profile
```

#### Database Tables

```
customers         (id, id_number, full_name, email, phone, fica_status, created_at)
documents         (id, customer_id, doc_type, file_path, verified)
auth_credentials  (id, customer_id, password_hash, totp_secret, last_login)
devices           (id, customer_id, fingerprint, trust_score, registered_at)
refresh_tokens    (id, customer_id, token_hash, expires_at, revoked)
```

> ⭐ **Technical highlight:** TOTP algorithm implemented from scratch — no library. Luhn + demographic extraction from SA ID number.

---

### 2.2 Core Banking Service
**Port:** 8082 · **Owner:** Person 2

The financial heart of the platform. Manages all accounts and maintains the general ledger using event sourcing — account balance is never stored as a column that gets updated. Every financial event is an immutable record and the balance is always derived by summing the event log. This is how real banking cores work and it eliminates an entire class of concurrency bugs.

#### Responsibilities

- Create and manage accounts: Cheque, Savings, Fixed Deposit
- Double-entry ledger — every transaction debits one account and credits another
- **Event sourcing: balance derived by summing immutable ledger entries, never stored directly**
- Periodic balance snapshots for query performance as the ledger grows
- Optimistic locking to prevent double-spend under concurrent requests
- Account number generation in a realistic SA bank format
- Minimum balance enforcement, daily withdrawal limits, account status management
- Full event log replay to reconstruct any account's complete financial history

#### Key Endpoints

```
POST /api/accounts
GET  /api/accounts/{accountNumber}/balance
POST /api/accounts/{accountNumber}/deposit
POST /api/accounts/{accountNumber}/withdraw
GET  /api/accounts/{accountNumber}/transactions
GET  /api/accounts/{accountNumber}/mini-statement
GET  /api/accounts/{accountNumber}/full-history
```

#### Database Tables

```
accounts          (id, account_number, customer_id, account_type, status, created_at)
ledger_entries    (id, transaction_ref, account_id, entry_type, amount, description, created_at)
account_snapshots (id, account_id, balance, snapshot_at)
transaction_limits(id, account_type, daily_limit, min_balance)
```

> ⭐ **Technical highlight:** Append-only ledger is tamper-evident by design. Balance is computed, never mutated.

---

### 2.3 Payments & Fraud Detection Service
**Port:** 8083 · **Owner:** Person 3

Handles all money movement and runs every transaction through a real-time fraud detection pipeline before settlement. The fraud engine is implemented as a chain of responsibility — each rule is an independent handler class. Adding a new fraud rule means adding a new class, not modifying existing ones. Classic SOLID in action.

#### Responsibilities

- Internal EFT between accounts on the platform
- Simulated interbank payments using SA bank code routing
- Beneficiary management with payment limit enforcement
- Proof of payment generation as a structured response
- Immediate payment (real-time settlement) flow
- **Fraud rule chain: Amount Anomaly · Velocity Check · New Beneficiary Large Payment · Geographic Anomaly**
- Per-rule risk scoring aggregated into a composite score
- Low risk → settles immediately | Medium → settles with alert | High → held for manual review
- Fraud review queue with approve/reject decision endpoint
- Rules configurable via the database — no redeployment to tune thresholds

#### Key Endpoints

```
POST   /api/payments/eft
POST   /api/payments/immediate
GET    /api/payments/{ref}/status
GET    /api/payments/{ref}/proof
POST   /api/beneficiaries
GET    /api/beneficiaries/{customerId}
DELETE /api/beneficiaries/{id}
GET    /api/fraud/review-queue
PUT    /api/fraud/{ref}/decision
```

#### Database Tables

```
payments       (id, ref, from_account, to_account, amount, status, risk_score, created_at)
beneficiaries  (id, customer_id, account_number, bank_code, name, nickname, created_at)
fraud_rules    (id, rule_name, weight, threshold, enabled)
fraud_events   (id, payment_ref, rule_name, score, reason, created_at)
review_queue   (id, payment_ref, assigned_to, decision, decided_at)
```

> ⭐ **Technical highlight:** Chain of Responsibility fraud engine. Open/Closed Principle — new rules never touch existing code.

---

### 2.4 Notifications & Statements Service
**Port:** 8084 · **Owner:** Person 4

Handles all customer-facing communication and financial reporting. Triggered by REST calls from the rest of the platform: sends emails via Mailtrap, generates PDF statements with iText, and produces spend analytics broken down by category per month.

#### Responsibilities

- Email notifications via Mailtrap for: payment confirmed, fraud alert, new device login, low balance
- **Monthly statement generation as a downloadable PDF using iText**
- Transaction categorisation via keyword matching on descriptions (groceries, fuel, transfers, etc.)
- Full spend analytics: category breakdown per month per customer
- Balance threshold alerts configurable per customer
- Notification history endpoint per customer
- Dead-letter handling for failed notification delivery with retry logic

#### Key Endpoints

```
GET  /api/statements/{accountNumber}?month=YYYY-MM
GET  /api/statements/{accountNumber}/download
GET  /api/analytics/{customerId}/spend?month=YYYY-MM
POST /api/alerts/configure
GET  /api/notifications/{customerId}/history
```

#### Database Tables

```
notifications      (id, customer_id, channel, type, message, sent_at, status)
statement_runs     (id, account_number, period, generated_at, file_path)
spend_categories   (id, keyword, category_name)
alert_configs      (id, customer_id, alert_type, threshold, enabled)
```

> ⭐ **Technical highlight:** iText PDF generation. Keyword-based transaction categorisation. Full spend analytics API.

---

## 3. Team Roles & Service Ownership

Each team member owns one service end-to-end: database schema, business logic, unit tests, Swagger documentation, and Docker configuration. No one should be blocked by another — agree on API contracts in Week 1 and then build independently.

| Person | Service | Key Technical Challenge | Integration Dependency |
|---|---|---|---|
| Person 1 | Identity & Auth (8081) | TOTP from scratch using HMAC-SHA1 + RFC 6238. Device fingerprinting. | All other services validate JWTs from this service. Critical path — finish JWT issuing by end of Week 2. |
| Person 2 | Core Banking (8082) | Event sourcing ledger. Balance never stored — always computed. Snapshot pattern. | Payments calls this service to debit/credit accounts. Expose REST endpoints by end of Week 3. |
| Person 3 | Payments & Fraud (8083) | Chain of Responsibility fraud engine. Each rule is an independent class. Risk scoring pipeline. | Calls Core Banking to move money. Calls Identity for step-up auth on large transactions. |
| Person 4 | Notifications & Statements (8084) | iText PDF statement generation. Keyword-based transaction categorisation. Spend analytics API. | Triggered by REST calls from Payments when events occur. Calls Core Banking for statement data. |

### Shared Team Responsibilities

- Week 1 API contract agreement — everyone defines request/response models before writing business logic
- Each person writes their own unit tests (JUnit 5 + Mockito, target 70%+ coverage on core logic)
- Each person writes their own Swagger/OpenAPI annotations
- Pull request review — at least one other team member reviews before merging to main
- Docker Compose: each person adds their service's container config to the shared compose file
- Week 6 integration testing — all four people present for the first full end-to-end demo run
- Week 8 demo rehearsal — run the full demo script at least twice before the actual presentation

---

## 4. 8-Week Project Timeline

The timeline is front-loaded with planning and contracts. The biggest risk in a 4-person microservices project is integration — services that work independently but break each other. The mitigation is agreeing on exact API contracts before anyone writes business logic.

### Week 1 — API Contracts & Project Setup
- Team agrees on ALL API contracts: request/response JSON models for every endpoint
- Define inter-service call contracts: what Payments sends to Core Banking, what Payments sends to Identity for step-up
- Database schemas finalised for all 4 services — no schema changes after this week without team discussion
- GitHub mono-repo created with 4 Maven module structure
- Docker Compose skeleton running (Postgres ×4, all 4 service shells on correct ports)
- GitHub Actions CI pipeline configured (build + test on every PR)
- **Deliverable:** `contracts.md` in repo root that every service is built against

### Week 2 — Scaffolding & Basic CRUD
- Each person: Spring Boot project set up, database connected, Flyway migrations running
- Person 1: Register + Login endpoints returning real JWTs
- Person 2: Create account + deposit + withdraw working against the event sourcing ledger
- Person 3: Basic EFT endpoint (no fraud engine yet) calling Core Banking to move funds
- Person 4: Mailtrap connected, basic notification on REST trigger
- All services: Swagger UI live on each port
- **Deliverable:** All 4 services start, connect to their DB, and basic happy-path flows work

### Week 3 — Core Business Logic
- Person 1: SA ID Luhn validator + demographic extraction. TOTP algorithm from scratch (RFC 6238). Device fingerprinting.
- Person 2: Full event sourcing implementation. Balance snapshot pattern. Optimistic locking for concurrent requests.
- Person 3: Fraud rule chain (all 4 rules). Risk scoring aggregation. Review queue.
- Person 4: iText PDF statement generation. Transaction keyword categorisation. Spend analytics logic.
- **Deliverable:** Each service's core domain logic is complete and unit tested

### Week 4 — Auth Integration
- JWT validation middleware added to Core Banking, Payments, and Notifications
- All services reject requests without a valid JWT from the Identity Service
- Person 1 + Person 3: Step-up auth flow wired — Payments calls Identity to demand TOTP on transactions > R5 000
- Device trust scoring active — new device login triggers additional verification
- **Deliverable:** Full auth flow working end-to-end across all services

### Week 5 — Service Integration
- Payments → Core Banking: debit/credit calls for every payment type
- Notifications ← Payments: REST trigger on payment completion, fraud flag, low balance
- Notifications ← Core Banking: statement data pull for PDF generation
- First full payment lifecycle tested: register → login → open account → pay → notification
- **Deliverable:** The complete demo flow runs without manual intervention

### Week 6 — End-to-End Testing & Bug Fixes
- Run the full demo script as a team — document every break point
- Fix integration bugs surfaced during the full flow run
- Each person: write unit tests for all business logic not yet covered
- Test unhappy paths: invalid SA ID, insufficient funds, fraud-flagged payment, new device login
- **Deliverable:** Full demo flow runs cleanly. Core logic has 70%+ unit test coverage.

### Week 7 — Polish & Documentation
- README with setup instructions — someone who has never seen the project must be able to run it
- `docker-compose up` brings up all 4 services, all 4 databases with seed data
- Swagger docs complete and accurate on all 4 services
- Error handling and meaningful error responses on all endpoints
- Seed data script that populates a demo customer, account, and transaction history
- **Deliverable:** A stranger can clone the repo, run `docker-compose up`, and hit the demo flow

### Week 8 — Demo Preparation
- Rehearse the demo script at least twice as a full team
- Each person prepares a 2-minute deep-dive on their service's most impressive technical feature
- Prepare answers to likely BBD interview questions (see Section 6)
- Record a backup demo video in case of live technical issues
- **Deliverable:** Team is confident and the demo runs in under 10 minutes

---
