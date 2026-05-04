Compliance Dashboard (Internal Admin Console)

The control center for compliance officers — identity oversight, KYC issuer management, AML alerts & case management, ZK proof logs, governance rule editing, and sanctions/risk monitoring, all in one web interface.

This dashboard sits atop the GitDigital compliance stack (Supergraph, Policy Engine, ZK MetaProof, Cross‑Chain Bridge) and provides authorised personnel with a unified, real‑time operational view.

---

1. High‑Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Compliance Dashboard (Web UI)                        │
│                   React + Tailwind + Recharts                           │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ HTTPS / WSS
┌────────────────────────────────▼────────────────────────────────────────┐
│                       Dashboard API (REST + WS)                         │
│                     Node.js / Express / PostgreSQL                      │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       Service Integration                        │  │
│  │                                                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │  │
│  │  │ Supergraph   │  │ AML Alerts   │  │ Governance Rule Editor   │ │  │
│  │  │ Query Layer  │  │ & Case Mgmt  │  │ (Policy Engine CRUD)     │ │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────┘ │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │  │
│  │  │ Identity     │  │ KYC Issuer   │  │ ZK Proof Explorer        │ │  │
│  │  │ Registry     │  │ Management   │  │ (MetaProof logs)         │ │  │
│  │  │ Explorer     │  │              │  │                          │ │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────┘ │  │
│  │  ┌──────────────────────────────────────────────────────────┐     │  │
│  │  │  Sanctions & Risk Monitoring (Watchlists + AML scores)    │     │  │
│  │  └──────────────────────────────────────────────────────────┘     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                        Data Sources & Indexers                          │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │ Supergraph       │  │ AML Risk Engine  │  │ Policy Engine    │      │
│  │ SQL Indexer      │  │ Event Bus        │  │ Event Logs       │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │ Identity Registry│  │ KYC Engine Events│  │ Cross‑chain      │      │
│  │ (on‑chain reads) │  │ (issuer updates) │  │ Attestations     │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

2. Subsystem Breakdown

Module Responsibility
Dashboard UI React SPA with responsive design, real‑time updates via WebSockets, and role‑based views (Compliance Officer, AML Analyst, Governance Admin).
Dashboard API Backend REST/WS server that aggregates data from multiple sources, enforces access control, and provides CRUD operations for cases, alerts, and configurations.
Identity Registry Explorer Search, view, and audit on‑chain identity records. Shows linked KYC/AML profile, status, and history.
KYC Issuer Management List of authorised KYC issuers, their status, issued credentials count, expiry trends, and ability to revoke issuer permissions.
AML Alerts & Case Management Ingest AML alerts from the risk engine, allow analysts to triage, create cases, attach evidence, assign investigators, and track resolution.
ZK Proof Logs Browse all MetaProof verifications (on‑chain records), filter by nullifier, policy, chain; detect anomalies.
Governance Rule Editor Visual interface for creating and modifying compliance policies (Policy Engine rules) and network‑wide thresholds. Proposes transactions via multisig.
Sanctions & Risk Monitoring Real‑time watchlist screening results, risk score trends, heatmaps by jurisdiction, and configurable alert thresholds.
Event/ Audit Stream Aggregated, filterable log of all compliance events (profile syncs, access grants/denials, overrides, delegations).

---

3. API Schema (Dashboard Backend)

REST Endpoints

```
── Auth & Session ──────────────────────────────────────────────────────
POST   /api/auth/login                  → { token, user }
POST   /api/auth/refresh                → { token }

── Identity Explorer ───────────────────────────────────────────────────
GET    /api/identities?q=pubkey&status=...   → IdentityResult[]
GET    /api/identities/:pubkey               → IdentityDetail (with profile)

── KYC Issuer Management ───────────────────────────────────────────────
GET    /api/issuers                      → Issuer[]
POST   /api/issuers                      → { txSig }   (register a new issuer)
PUT    /api/issuers/:pubkey/status       → { txSig }   (activate/suspend)
GET    /api/issuers/:pubkey/stats        → IssuerStats

── AML Alerts ──────────────────────────────────────────────────────────
GET    /api/aml/alerts?status=new&severity=high → Alert[]
PUT    /api/aml/alerts/:id               → { ... }     (triage, assign)
POST   /api/aml/cases                    → { caseId }   (create case from alert)
PUT    /api/aml/cases/:id                → update case status/notes
GET    /api/aml/cases?assignee=me        → Case[]

── ZK Proof Logs ───────────────────────────────────────────────────────
GET    /api/zk/proofs?policy=&chain=&since= → MetaProofRecord[]
GET    /api/zk/proofs/:nullifier            → Status

── Governance Rules ────────────────────────────────────────────────────
GET    /api/policies                     → PolicySummary[]
GET    /api/policies/:name               → PolicyDetail
POST   /api/policies                     → { txSig } (create)
PUT    /api/policies/:name               → { txSig } (update)
DELETE /api/policies/:name               → { txSig } (deactivate)

── Sanctions & Risk Monitoring ─────────────────────────────────────────
GET    /api/risk/heatmap?jurisdiction=   → RiskHeatmap
GET    /api/risk/alerts?score_above=     → HighRiskSubject[]

── Event Stream ────────────────────────────────────────────────────────
GET    /api/events?types=AccessDenied,OverrideApplied&from=... → Event[]
WS     /api/events/stream               → real‑time event push

── Dashboard Stats ─────────────────────────────────────────────────────
GET    /api/stats/overview               → { totalProfiles, blocked, overrides, ... }
GET    /api/stats/trends?days=30         → { dailyCompliant, dailyBlocked, ... }
```

WebSocket Events

```json
{ "type": "AML_ALERT", "data": { "subject": "...", "score": 85, "trigger": "SANCTIONS_MATCH" } }
{ "type": "ACCESS_DENIED", "data": { "subject": "...", "reason": "KycExpired", "program": "..." } }
{ "type": "OVERRIDE_APPLIED", "data": { "subject": "...", "officer": "..." } }
{ "type": "KYC_ISSUER_UPDATE", "data": { "issuer": "...", "action": "SUSPENDED" } }
```

---

4. Data Models (Dashboard DB – PostgreSQL)

Tables beyond the existing compliance_profiles, compliance_events, etc. used by the supergraph indexer.

```sql
-- Dashboard users & roles
CREATE TABLE dashboard_users (
    id          SERIAL PRIMARY KEY,
    username    TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role        TEXT NOT NULL DEFAULT 'viewer',  -- viewer, analyst, officer, admin
    pubkey      TEXT,         -- associated Solana wallet for signing proposals
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- AML Alerts (from risk engine events)
CREATE TABLE aml_alerts (
    id              SERIAL PRIMARY KEY,
    subject         TEXT NOT NULL,
    alert_type      TEXT NOT NULL,  -- SANCTIONS_MATCH, RISK_SCORE_SPIKE, PEP_DETECTED
    severity        TEXT NOT NULL,  -- LOW, MEDIUM, HIGH, CRITICAL
    description     TEXT,
    raw_payload     JSONB,
    status          TEXT NOT NULL DEFAULT 'NEW',  -- NEW, ACKNOWLEDGED, CASE_CREATED, RESOLVED
    acknowledged_by TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- AML Cases (linked to one or more alerts)
CREATE TABLE aml_cases (
    id              SERIAL PRIMARY KEY,
    subject         TEXT NOT NULL,
    title           TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'OPEN',  -- OPEN, IN_PROGRESS, RESOLVED, SAR_FILED
    priority        TEXT NOT NULL DEFAULT 'MEDIUM',
    assigned_to     TEXT,             -- dashboard_users.username
    notes           TEXT,
    evidence_uris   TEXT[],
    sar_filing_ref  TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ
);

-- Case-Alert mapping
CREATE TABLE case_alerts (
    case_id   INTEGER REFERENCES aml_cases(id),
    alert_id  INTEGER REFERENCES aml_alerts(id),
    PRIMARY KEY (case_id, alert_id)
);

-- Sanctions watchlist results (cached)
CREATE TABLE sanctions_hits (
    id          SERIAL PRIMARY KEY,
    subject     TEXT NOT NULL,
    list_name   TEXT NOT NULL,
    matched_name TEXT NOT NULL,
    match_score REAL,
    screened_at TIMESTAMPTZ DEFAULT NOW()
);

-- KYC Issuer registry
CREATE TABLE kyc_issuers (
    pubkey            TEXT PRIMARY KEY,
    name              TEXT NOT NULL,
    jurisdiction      TEXT,
    credentials_issued INTEGER DEFAULT 0,
    status            TEXT NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, SUSPENDED, REVOKED
    added_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Governance proposal drafts (to be submitted as multisig txs)
CREATE TABLE governance_drafts (
    id            SERIAL PRIMARY KEY,
    policy_name   TEXT,
    new_rules     JSONB,
    reason        TEXT,
    proposed_by   TEXT,
    multisig_tx   TEXT,   -- serialized transaction
    status        TEXT DEFAULT 'DRAFT',
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Dashboard audit log (all sensitive actions performed via UI)
CREATE TABLE dashboard_audit (
    id            SERIAL PRIMARY KEY,
    username      TEXT NOT NULL,
    action        TEXT NOT NULL,  -- e.g., CREATE_CASE, APPROVE_OVERRIDE, UPDATE_POLICY
    details       JSONB,
    timestamp     TIMESTAMPTZ DEFAULT NOW()
);
```

---

5. UI Components (Key Screens)

1. Overview Dashboard

· KPI cards: Total Profiles, Fully Compliant, Blocked, Overrides Active, KYC Expiring in 30 days.
· Real‑time event feed (WebSocket) of recent Access Denied, Override, AML alerts.
· Risk heatmap (choropleth) by jurisdiction.
· Trend chart: daily compliant vs. non‑compliant over 30 days.

2. Identity Registry Explorer

· Search by DID, Solana pubkey, or jurisdiction.
· Subject detail panel: full compliance profile, linked identity/KYC/AML records, event timeline.
· “Request Sync” button to trigger a profile re‑sync.

3. KYC Issuer Management

· Table of all issuers: name, pubkey, jurisdiction, issued count, status.
· Actions: Activate / Suspend / Revoke (multisig integration).
· Issuer stats: credentials issued over time, expiry distribution.

4. AML Alerts & Case Management

· Alert Queue: filterable list (severity, status). Each alert shows subject, type, time, and a quick “Acknowledge” / “Create Case” button.
· Case Workspace: For a selected case, shows linked alerts, subject profile snippet, notes thread, evidence attachments. Buttons to assign, change status, mark for SAR filing.
· Sanctions Screen: dedicated view of all recent sanctions hits, ability to mark false positives.

5. ZK Proof Logs

· Search by nullifier, policy, verifier pubkey, or time range.
· Table: nullifier (truncated), policy, chain, verified at, expiry, status (valid/revoked/expired).
· Nullifier revocation form (admin).

6. Governance Rule Editor

· List of active policies (from Policy Engine) with a YAML/JSON editor (code editor component).
· “Create Policy” wizard with fields for compliance tier, risk cap, required credentials, jurisdiction allowlist.
· Draft proposals stored in governance_drafts, then proposed as multisig transactions (via Squads or similar). The dashboard can interact with the multisig SDK to submit and track.

7. Sanctions & Risk Monitoring

· Live feed of watchlist screening results.
· Configurable thresholds: alert when subject’s AML risk score > X, or when screened name similarity > Y.
· Risk trend chart for flagged subjects.

8. Audit Trail

· Filterable, exportable log of all dashboard actions (create case, override, policy change, etc.).
· Compliance events aggregated from on‑chain event logs.

---

6. Security Considerations

Threat Mitigation
Unauthorised access JWT authentication + RBAC (viewer, analyst, officer, admin). Admin functions require multisig approval on‑chain as well.
Data leakage API enforces row‑level access; sensitive fields like full KYC data are never exposed – only digest/hashes are shown.
Session hijacking HTTPS with secure, httpOnly cookies; token refresh rotation; short expiry.
Injection attacks Parameterised queries; input validation and sanitisation.
Internal abuse Every sensitive action recorded in dashboard_audit and linked to the authenticated user. Multisig required for high‑impact changes.
Supply chain (frontend) Bundled React app with Subresource Integrity; dependency scanning.

---

7. Compliance Considerations

Requirement Implementation
Segregation of duties RBAC ensures only authorised roles can triage alerts, edit policies, or revoke issuers.
Audit trail completeness On‑chain events + dashboard_audit table capture all human and programmatic decisions.
SAR support AML cases can be escalated to SAR filing with appropriate evidence logged.
Data minimisation Dashboard shows only status codes and hashes for identity/KYC; raw PII is never displayed.
Right to view / correction Administrators can trigger forced re‑sync of a subject’s profile to correct errors. Override functionality documented.

---

8. Integration Points

Module Integration
Supergraph Indexer Main data source for profiles, events, and stats (PostgreSQL).
AML Risk Engine Real‑time alert ingestion via WebSocket or polling.
Policy Engine REST API for CRUD operations on policies; transaction building for governance.
Multisig (Squads) SDK to propose and execute on‑chain transactions (policy updates, overrides, issuer management).
ZK MetaProof Indexer Query verified proofs and nullifier status.
Cross‑Chain Bridge Attestations Included in event feed and export logs.
Identity Registry / KYC Engine On‑chain reads for detailed subject view.

---

9. Deployment & Tech Stack

· Frontend: React, Tailwind CSS, Recharts (or D3), Monaco editor (for policies), WebSocket client.
· Backend: Node.js (Express) with TypeScript, pg for PostgreSQL, @solana/web3.js and Anchor for on‑chain interactions, @coral-xyz/anchor types.
· Database: PostgreSQL (shared with supergraph indexer, with additional dashboard tables).
· Auth: JWT with bcrypt password hashing. RBAC middleware.
· Hosting: Could be containerised (Docker) and deployed on a cloud service with access restricted to internal VPN/IP.

---

10. Repository Structure (Addition)

```
compliance-dashboard/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Overview/
│   │   │   ├── IdentityExplorer/
│   │   │   ├── KycIssuers/
│   │   │   ├── AmlCases/
│   │   │   ├── ZkProofs/
│   │   │   ├── PolicyEditor/
│   │   │   └── AuditTrail/
│   │   ├── hooks/
│   │   ├── services/          # API client, WebSocket
│   │   └── App.tsx
│   └── package.json
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── auth.ts
│   │   │   ├── identities.ts
│   │   │   ├── issuers.ts
│   │   │   ├── aml.ts
│   │   │   ├── zk.ts
│   │   │   ├── policies.ts
│   │   │   ├── risk.ts
│   │   │   └── events.ts
│   │   ├── services/
│   │   ├── middleware/
│   │   ├── db/
│   │   │   ├── migrations/
│   │   │   └── queries.ts
│   │   └── app.ts
│   └── package.json
└── docker-compose.yml
```

---

Status: Compliance Dashboard architecture complete.
Provides the centralised internal admin console for the entire GitDigital compliance suite.
Ready for UI/UX design and backend implementation.
