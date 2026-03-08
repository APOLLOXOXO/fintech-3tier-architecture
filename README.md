# Fintech 3-Tier Network Architecture
**VirtualBox · pfSense · Ubuntu Server · Nginx · Python Flask · PostgreSQL · Wireshark**

---

I want to build a fintech platform someday. Not integrate M-PESA's API into someone else's template - actually build one. Own the infrastructure underneath it.

But before I touch any business logic, I needed to understand what holds a financial system together at the network level. So I built it. From scratch. Locally. No cloud account required.

This is that build.

---

## What this is

A working simulation of the network architecture that runs under production mobile money platforms. Three isolated tiers. One pfSense firewall enforcing every single rule between them. Zero trust between layers.

The same logic that keeps your M-PESA balance unreachable from the internet - that's what I replicated here, manually, so I could understand every piece of it before trusting a cloud provider to abstract it away.

---

## The architecture

```
SIMULATED INTERNET
        |
  [pfSense Firewall]
   192.168.100.1
   Rules:
   Tier 1 → Tier 2 : port 8000 only
   Tier 2 → Tier 3 : port 5432 only
   Everything else : DENY
        |
        |
┌──────────────────────────────────────────────────┐
│                                                  │
│   TIER 1 — API GATEWAY          192.168.10.0/24  │
│   Ubuntu Server + Nginx                          │
│   What it does: accepts requests from outside.   │
│   Rate limiting. TLS. Idempotency key per txn.   │
│   The only tier the internet can reach.          │
│                                                  │
│            │ port 8000 only                      │
│            ▼                                     │
│   TIER 2 — TRANSACTION ENGINE   192.168.20.0/24  │
│   Ubuntu Server + Python Flask                   │
│   What it does: processes every transaction.     │
│   State machine: PENDING → COMPLETED/FAILED.     │
│   Fraud check runs before touching the database. │
│   No internet access. Ever.                      │
│                                                  │
│            │ port 5432 only                      │
│            ▼                                     │
│   TIER 3 — DATA VAULT           192.168.30.0/24  │
│   Ubuntu Server + PostgreSQL 15                  │
│   What it does: stores everything that matters.  │
│   Balances. Transactions. KYC. Float accounts.   │
│   Reachable from one place only: Tier 2.         │
│   The internet has no path here. Not blocked —   │
│   there is literally no route that reaches it.   │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## How a transaction moves through this

Someone sends KES 500:

```
1. Request hits Tier 1 API Gateway
2. Nginx validates format, applies rate limiting
3. Unique transaction ID generated — idempotency key
4. Request forwarded to Tier 2 on port 8000
   pfSense checks: port 8000 from Tier 1? Allow.
5. Transaction Engine sets state: PENDING
6. Fraud check: amount normal? velocity ok? account clean?
   If anything looks wrong → state: FAILED. Stop here.
7. If clean → query Tier 3 database on port 5432
   pfSense checks: port 5432 from Tier 2? Allow.
8. PostgreSQL checks sender balance, receiver status
9. Debit sender. Credit receiver. Atomic — both or neither.
10. State: COMPLETED
11. Response travels back up through Tier 2, then Tier 1
12. User sees confirmation

The database was never directly reachable from outside.
Not once. Not even for a millisecond.
```

---

## Why each component exists

**pfSense** — This is the brain of the whole thing. Every firewall rule is written manually here. When I say Tier 3 is unreachable from Tier 1 — pfSense is what makes that true. Not a setting in a dashboard. An explicit rule I wrote, that I can read, that I can prove with Wireshark captures.

On AWS this same logic lives inside Security Groups and Route Tables. Building it manually in pfSense first means I understand what AWS is actually doing when it says "connection refused."

**Nginx on Tier 1** — The public-facing layer. Handles incoming requests, applies rate limiting so nobody can flood the system, terminates TLS, then proxies clean requests to the transaction engine. It never connects to the database. That's by design and by firewall rule.

**Python Flask on Tier 2** — The transaction engine. Every transaction goes through a state machine: starts as PENDING, ends as COMPLETED, FAILED or REVERSED. Fraud detection runs first — if the check fails, the database is never touched. The idempotency check runs before everything else — if this transaction ID already exists, return the cached result. No double charges.

**PostgreSQL on Tier 3** — The vault. Five tables: users, wallets, transactions, float_accounts, kyc_records. The float_accounts table is the detail that most tutorials skip — it mirrors how M-PESA actually segregates funds into working capital, settlement and commission pools. That separation is what makes reconciliation possible and regulators happy.

---

## The database schema

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number VARCHAR(15) UNIQUE NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    kyc_status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE wallets (
    wallet_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id),
    balance DECIMAL(15,2) DEFAULT 0.00,
    currency VARCHAR(3) DEFAULT 'KES',
    is_active BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE transactions (
    txn_id UUID PRIMARY KEY,
    sender_wallet UUID REFERENCES wallets(wallet_id),
    receiver_wallet UUID REFERENCES wallets(wallet_id),
    amount DECIMAL(15,2) NOT NULL,
    state VARCHAR(20) NOT NULL,
    idempotency_key VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE TABLE float_accounts (
    account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_type VARCHAR(50),
    balance DECIMAL(15,2) NOT NULL,
    last_reconciled TIMESTAMP
);

CREATE TABLE kyc_records (
    kyc_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id),
    id_number VARCHAR(20),
    verified_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'PENDING'
);
```

---

## Proving the security boundaries actually work

These aren't assumptions. Every boundary was tested and captured.

**Tier 1 responds to the internet:**
```bash
curl http://192.168.10.10/health
→ {"status": "ok", "tier": "api-gateway"}
```

**Tier 2 does not:**
```bash
curl http://192.168.20.10:8000/health
→ curl: (7) Failed to connect: Connection refused
```

**Tier 3 unreachable from Tier 1:**
```bash
# From inside Tier 1 VM
psql -h 192.168.30.10 -U postgres
→ could not connect to server: Connection timed out
```

**Tier 3 reachable from Tier 2 on the correct port only:**
```bash
# From inside Tier 2 VM
psql -h 192.168.30.10 -U postgres -p 5432 fintech_db
→ connected successfully
```

**Full transaction end to end:**
```bash
curl -X POST http://192.168.10.10/transaction \
  -H "Content-Type: application/json" \
  -d '{"sender":"0712000001","receiver":"0712000002","amount":500}'

→ {
    "txn_id": "a3f8c2d1-...",
    "status": "COMPLETED",
    "amount": 500,
    "currency": "KES",
    "timestamp": "2026-03-08T14:22:31Z"
  }
```

Wireshark filter on the Tier 2 → Tier 3 link:
```
tcp.port == 5432 and ip.dst == 192.168.30.10
```
Result: only PostgreSQL protocol traffic. Nothing else reaches that VM.

---

## What broke and what I learned from it

**pfSense rule ordering**
pfSense reads firewall rules top to bottom and stops at the first match. I had a broad ALLOW rule sitting above my specific DENY rules. Traffic I meant to block was getting through because it matched the ALLOW before it ever reached the DENY. Took 40 minutes to find.

Lesson: in pfSense, order is everything. The most specific rules go first.

**PostgreSQL not listening on the right interface**
PostgreSQL defaults to listening only on localhost — 127.0.0.1. Tier 2 was sending requests to 192.168.30.10 and getting nothing back, even after the firewall rules were correct. The firewall was fine. The service just wasn't listening on that interface.

Fix: edited `/etc/postgresql/15/main/postgresql.conf`
Changed `listen_addresses = 'localhost'` to `listen_addresses = '192.168.30.10'`

Lesson: a correctly configured firewall means nothing if the service behind it isn't listening on the right address.

**Idempotency — the expensive lesson**
During testing, a simulated network timeout caused the API gateway to retry a request. Without idempotency keys implemented, the transaction engine processed it as a new transaction. KES 500 charged twice. Same sender, same receiver, same amount — two separate database entries.

This is exactly how real platforms lose customer trust. Fixed by checking for an existing transaction ID as the absolute first step in every handler — before fraud check, before database query, before anything.

Lesson: idempotency is not a nice-to-have in fintech. It is table stakes.

---

## How this maps to AWS

If this project moved to AWS today:

| This project | AWS equivalent |
|---|---|
| pfSense firewall rules | Security Groups + Network ACLs |
| VirtualBox Host-Only Network 1 | Public Subnet 10.0.1.0/24 |
| VirtualBox Host-Only Network 2 | Private Subnet 10.0.2.0/24 |
| VirtualBox Host-Only Network 3 | Isolated Subnet 10.0.3.0/24 |
| Ubuntu + Nginx VM | EC2 in Public Subnet |
| Ubuntu + Flask VM | EC2 in Private Subnet |
| Ubuntu + PostgreSQL VM | RDS in Isolated Subnet |
| No internet route on Tier 3 | No route table entry pointing to IGW |

Building this manually means I understand what AWS is abstracting. When a security group says "connection refused" I know exactly what rule triggered it and why.

---

## Tools used

| Tool | Purpose |
|---|---|
| VirtualBox 7.0 | Hypervisor — runs all four VMs |
| pfSense CE 2.7 | Firewall — enforces every inter-tier rule |
| Ubuntu Server 22.04 LTS | OS for all three tier VMs |
| Nginx 1.24 | Reverse proxy and API gateway on Tier 1 |
| Python Flask 3.0 | Transaction engine REST API on Tier 2 |
| PostgreSQL 15 | Financial data store on Tier 3 |
| Wireshark 4.0 | Packet capture — proves security boundaries hold |

---

## Repository structure

```
fintech-3tier-architecture/
│
├── README.md
├── screenshots/
│   ├── pfsense-firewall-rules.png
│   ├── tier1-nginx-running.png
│   ├── tier2-flask-running.png
│   ├── tier3-postgres-schema.png
│   ├── test-tier2-blocked.png
│   ├── test-tier3-blocked-from-tier1.png
│   ├── test-full-transaction.png
│   └── wireshark-tier2-tier3.png
├── config/
│   ├── nginx/api-gateway.conf
│   └── postgresql/pg_hba.conf
├── src/
│   ├── transaction_engine/
│   │   ├── app.py
│   │   ├── models.py
│   │   ├── fraud_check.py
│   │   └── requirements.txt
│   └── database/
│       └── schema.sql
└── docs/
    └── transaction-flow.png
```

---

## Part of a bigger picture

This is one project inside a 30-day hands-on challenge I started in March 2026 — building and documenting one real project per day across network virtualization and AI engineering.

The goal isn't just a certificate or a job. It's understanding the infrastructure deeply enough to eventually build something on top of it.

Follow the daily build on LinkedIn → [Your LinkedIn URL]

---

## Connect

GitHub: github.com/APOLLOXOXO
LinkedIn: [Your LinkedIn URL]

Telecom & InfoTech · Network Virtualization & AI Engineering
Cisco · VMware · Python · Linux · AWS

---

*"Most people want to build a fintech app.*
*I started with the network."*
