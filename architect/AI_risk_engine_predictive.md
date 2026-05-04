AI Risk Engine (Predictive)

Your AI risk brain — uses machine learning to predict fraud, sanctions exposure, behavioural anomalies, counterparty risk, and transaction laundering patterns in real time. Feeds actionable risk intelligence into the Compliance Dashboard, AI Compliance Agent, and the AML Risk Engine.

This engine consumes the unified event stream from the Event Bus, enriches it with on‑chain transaction data and off‑chain watchlists, and runs specialised models. It produces predictive risk scores, anomaly alerts, and network‑aware risk assessments that go beyond static rules. Outputs integrate directly with the existing AML Risk Engine’s on‑chain risk profiles and the AI Compliance Agent’s triage pipeline.

---

1. High‑Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       AI Risk Engine (Off‑Chain)                              │
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                     Model Serving & Inference                          │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │   │
│  │  │ Fraud       │  │ Sanctions    │  │ Behavioural  │  │ Transaction│  │   │
│  │  │ Predictor   │  │ Exposure     │  │ Anomaly      │  │ Laundering │  │   │
│  │  │             │  │ Model        │  │ Detector     │  │ Detector   │  │   │
│  │  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └──────┬─────┘  │   │
│  │         └────────────────┴─────────────────┴─────────────────┘       │   │
│  │                                   │                                   │   │
│  │  ┌────────────────────────────────▼────────────────────────────────┐  │   │
│  │  │                  Counterparty Risk Scorer                        │  │   │
│  │  │  (Graph‑based propagation of risk across transaction networks)   │  │   │
│  │  └─────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────┬────────────────────────────────────────────┘   │
│                              │                                               │
│  ┌───────────────────────────▼──────────────────────────────────────────┐    │
│  │                       Risk Action Dispatcher                          │    │
│  │  • Update on‑chain AML risk scores (via CPI to AML Risk Engine)       │    │
│  │  • Emit predictive alerts to Event Bus                                │    │
│  │  • Feed AI Compliance Agent with enriched risk signals                │    │
│  │  • Trigger real‑time blocking (via Policy Engine) if risk extreme      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 │  Reads / Writes
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Event Bus + Indexer Supernode                         │
│   • Unified events (compliance, transactions, cross‑chain)                    │
│   • PostgreSQL (profiles, transaction history) + ClickHouse (analytics)       │
└──────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 │  Integrates with
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     On‑Chain Modules (via SDK / API)                           │
│  • AML Risk Engine (update SubjectRiskProfile PDA)                            │
│  • Compliance Supergraph (read profiles, update risk fields)                  │
│  • Policy Engine (instant block if risk > emergency threshold)                │
│  • Cross‑Chain Bridge (monitor cross‑chain patterns)                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

2. Subsystem Breakdown

Subsystem Responsibility
Fraud Predictor Supervised model (XGBoost / deep learning) trained on historical fraud labels. Predicts the probability that a subject will commit fraud based on behavioural features, credential usage, transaction velocity, and graph centrality.
Sanctions Exposure Model Combines real‑time watchlist screening with predictive scoring — estimates the likelihood that a subject is a sanctioned entity using fuzzy name matching, network proximity to known sanctioned addresses, and cross‑chain footprint.
Behavioural Anomaly Detector Unsupervised (Isolation Forest, autoencoders) to detect deviations from normal behaviour: sudden changes in transaction patterns, credential usage spikes, unusual cross‑chain attestations.
Transaction Laundering Detector Graph‑based ML (GraphSAGE, RGCN) that identifies layered transaction flows, peeling chains, and mixing patterns indicative of laundering.
Counterparty Risk Scorer Uses network propagation (PageRank‑like) to assign a risk score to every wallet based on its transaction counterparties. A wallet interacting with high‑risk entities inherits elevated risk.
Risk Action Dispatcher Translates model outputs into actions: updating on‑chain risk scores (via AML Engine CPI), generating predictive alerts, feeding the AI Compliance Agent, or triggering emergency policy blocks.
Feedback Loop Ingests outcomes from human reviewers and on‑chain resolutions (e.g., SAR filed) to retrain models.
Model Registry Versioned models stored and served via MLflow / Triton.

---

3. Solana Program Design (AML Risk Engine Extension)

The existing AML Risk Engine has an on‑chain SubjectRiskProfile PDA with a composite_score. The AI Risk Engine acts as an authorised updater of that score, appending predictive dimensions.

Minimal on‑chain addition to support AI‑driven updates:

```rust
// Existing AML Risk Engine instruction (already defined):
pub fn update_risk_score(
    ctx: Context<UpdateRiskScore>,
    new_score: u8,
    reason: RiskUpdateReason,
) -> Result<()>;

// Extended reason enum (already in AML engine):
pub enum RiskUpdateReason {
    ManualReview,
    AutomatedScreening,
    // New variants:
    AIPredictiveFraud,
    AISanctionsExposure,
    AIBehaviouralAnomaly,
    AITransactionLaundering,
    AICounterpartyPropagation,
}
```

No new PDA required. The AI Risk Engine uses the same update_risk_score instruction via a delegated hot wallet (or CPI from the AI Compliance Agent).

---

4. PDA Map (Reuse)

PDA Usage
SubjectRiskProfile (aml_risk_engine) The composite_score, risk_tier, aml_status, and flag_reason are updated by the AI Risk Engine (with proper authority).
AccessGateRecord (supergraph) If extreme risk detected, policy engine can be instructed to mark subject as blocked.

---

5. API Schema (AI Risk Engine Service)

The AI Risk Engine exposes an internal API for the Compliance Dashboard and AI Compliance Agent.

REST Endpoints

```
── Predictive Risk Scores ──────────────────────────────────────────────
GET    /api/risk-engine/score/:subject              → PredictiveRiskScore
POST   /api/risk-engine/scores/batch               → PredictiveRiskScore[]
  Body: { subjects: ["pubkey1", "pubkey2"] }

── Alerts ──────────────────────────────────────────────────────────────
GET    /api/risk-engine/alerts?type=fraud&from=...  → PredictiveAlert[]
GET    /api/risk-engine/alerts/:id                  → AlertDetail

── Network Risk ────────────────────────────────────────────────────────
GET    /api/risk-engine/network/:subject            → NetworkRiskView
  Returns the subject’s risk propagation, top counterparty clusters.

── Model Insights ──────────────────────────────────────────────────────
GET    /api/risk-engine/insights/feature-importance → FeatureImportanceReport
GET    /api/risk-engine/insights/model-drift        → DriftMetrics

── Manual Triggers ─────────────────────────────────────────────────────
POST   /api/risk-engine/trigger/full-scan           → (re‑evaluates all subjects)
POST   /api/risk-engine/trigger/repropagate         → (re‑runs network propagation)
POST   /api/risk-engine/trigger/retrain             → (initiates training pipeline)
```

Core Request Types

```typescript
interface PredictiveRiskScore {
  subject: string;
  models: {
    fraud: { score: number; confidence: number };
    sanctions_exposure: { score: number; confidence: number; matched_list?: string };
    behavioural_anomaly: { score: number; z_score: number };
    transaction_laundering: { score: number; confidence: number };
    counterparty_risk: { score: number; propagation_level: number };
  };
  composite_predictive_score: number;   // 0–100, weighted combination
  timestamp: number;
}

interface PredictiveAlert {
  id: string;
  subject: string;
  alert_type: 'FRAUD' | 'SANCTIONS_EXPOSURE' | 'ANOMALY' | 'LAUNDERING' | 'COUNTERPARTY_RISK';
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  score: number;
  reason: string;
  recommended_action: 'UPDATE_RISK_SCORE' | 'CREATE_CASE' | 'TRIGGER_BLOCK' | 'MONITOR';
  created_at: number;
}
```

---

6. Data Models (PostgreSQL + ClickHouse)

PostgreSQL (Operational)

```sql
-- Predictive risk scores (latest per subject)
CREATE TABLE predictive_risk_scores (
    subject               TEXT PRIMARY KEY,
    fraud_score           REAL,
    sanctions_exposure    REAL,
    behavioural_anomaly   REAL,
    transaction_laundering REAL,
    counterparty_risk     REAL,
    composite_predictive  REAL,
    updated_at            TIMESTAMPTZ DEFAULT NOW()
);

-- Predictive alerts
CREATE TABLE predictive_alerts (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject       TEXT NOT NULL,
    alert_type    TEXT NOT NULL,
    severity      TEXT NOT NULL,
    score         REAL,
    reason        TEXT,
    recommended_action TEXT,
    status        TEXT DEFAULT 'NEW',
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Network risk graph cache (edges with risk weights)
CREATE TABLE transaction_network (
    source        TEXT NOT NULL,
    target        TEXT NOT NULL,
    volume_usd    REAL,
    tx_count      INTEGER,
    last_updated  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (source, target)
);
```

ClickHouse (Analytics)

```sql
CREATE TABLE risk_score_history (
    subject       String,
    model_name    LowCardinality(String),
    score         Float32,
    timestamp     DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (subject, model_name, timestamp);
```

---

7. Client SDK

```typescript
// sdk/src/AIRiskEngineClient.ts
export class AIRiskEngineClient {
  constructor(private apiBase: string, private authToken: string) {}

  async getPredictiveScore(subject: string): Promise<PredictiveRiskScore> {
    return fetch(`${this.apiBase}/api/risk-engine/score/${subject}`, {
      headers: { Authorization: `Bearer ${this.authToken}` }
    }).then(r => r.json());
  }

  async getPredictiveAlerts(filter?: { type?: string; severity?: string }): Promise<PredictiveAlert[]> {
    const params = new URLSearchParams(filter);
    return fetch(`${this.apiBase}/api/risk-engine/alerts?${params}`, {
      headers: { Authorization: `Bearer ${this.authToken}` }
    }).then(r => r.json());
  }

  async getNetworkRisk(subject: string): Promise<NetworkRiskView> {
    return fetch(`${this.apiBase}/api/risk-engine/network/${subject}`, {
      headers: { Authorization: `Bearer ${this.authToken}` }
    }).then(r => r.json());
  }
}
```

---

8. Workflows

Fraud Prediction & Prevention

```
1. Real‑time: each new transaction event (or access request) triggers a feature extraction.
2. Fraud Predictor returns a fraud probability.
3. If probability > high threshold (e.g., 0.8):
   - Dispatcher creates a predictive alert (FRAUD, CRITICAL).
   - Simultaneously, if policy allows, the AI Risk Engine calls
     the AML Risk Engine’s update_risk_score to raise the composite_score.
   - The Policy Engine may instantly block the subject from further actions
     (via assertCompliant CPI, which now sees elevated risk).
4. Alert feeds into AI Compliance Agent for triage.
```

Sanctions Exposure Prediction

```
1. Integrates with external sanctions list (OFAC, UN, etc.) via API.
2. For each subject, the model computes:
   - Fuzzy name match scores (if name known off‑chain)
   - Wallet address screen (direct matches)
   - Network distance to known sanctioned addresses (using graph DB)
3. Produces a sanctions_exposure score.
4. If score > threshold, dispatcher raises on‑chain AML status to Flagged or Blocked,
   and creates a SANCTIONS_EXPOSURE alert.
```

Behavioural Anomaly Detection

```
1. Baseline per‑subject behaviour: typical transaction volume, frequency, programs used,
   counterparty clusters, time‑of‑day patterns.
2. Unsupervised model continuously monitors for deviations.
3. On detecting anomaly (e.g., sudden large cross‑chain move), the model generates
   an ANOMALY alert. The score influences the composite_predictive_score but does not
   automatically block (requires human review).
```

Transaction Laundering Detection (Graph‑Based)

```
1. Builds a dynamic transaction graph from all on‑chain movements.
2. Trains a graph neural network (or simpler heuristics) on known laundering patterns:
   - “Peel chains” where large amounts are gradually split and consolidated.
   - Use of mixers / tumblers.
3. Flags wallets involved in suspicious patterns.
4. Dispatcher updates the AML risk profile and generates LAUNDERING alerts.
```

Counterparty Risk Propagation

```
1. Start with a set of known high‑risk wallets (e.g., sanctioned, flagged).
2. Run a risk propagation algorithm (Personalized PageRank) to assign a counterparty risk score
   to every wallet in the network.
3. Scores updated periodically (batch) and stored in predictive_risk_scores.
4. If a user’s counterparty risk suddenly spikes (e.g., they interact with a newly sanctioned
   wallet), a COUNTERPARTY_RISK alert is raised.
```

---

9. Security Considerations

Threat Mitigation
Manipulated transaction data Data sourced from trusted RPC providers and validated Solana signatures. Model features are computed from immutable on‑chain records.
Model inversion / adversarial inputs Models do not expose raw features via API; only final scores. Regular adversarial robustness testing.
False positives causing unwarranted blocking Extreme actions (blocking) require human approval via the AI Compliance Agent or are limited to temporary, reversible blocks.
Unauthorised updating of on‑chain risk scores The AI Risk Engine’s hot wallet is permissioned in the AML Risk Engine (multisig or designated updater role). Each update is logged.

---

10. Compliance Considerations

Requirement Implementation
Explainability Every predictive alert includes a reason string derived from model explanation (SHAP/LIME) for the top contributing features.
Bias monitoring Model performance dashboards continuously monitor fairness metrics across jurisdictions and user types.
Right to human review Blocking and SAR recommendations always require human confirmation.
Data retention Graph data retained for regulatory periods; raw transaction history purged according to policy.

---

11. Integration Points

Module Integration
AML Risk Engine The AI Risk Engine is an authorised updater of on‑chain risk profiles.
Compliance Supergraph Reads compliance profiles; its Access Gate is affected by elevated risk scores.
Policy Engine Receives emergency block signals from the Dispatcher.
Event Bus + Indexer Supernode Primary source of transaction and compliance events.
AI Compliance Agent Consumes predictive alerts and enriches triage decisions.
Cross‑Chain Bridge Provides cross‑chain transaction data for laundering and anomaly detection.
Off‑Chain Watchlist Providers Pulled into sanctions exposure model.

---

12. Deployment & Tech Stack

· Model Serving: NVIDIA Triton Inference Server or FastAPI with ONNX Runtime.
· ML Frameworks: PyTorch (graph neural nets), XGBoost, scikit‑learn.
· Graph DB: Neo4j or Apache AGE for network storage and propagation.
· Stream Processing: Python/TypeScript consumers on Redpanda, feeding features to models.
· Training Pipeline: Apache Airflow / Dagster; data from ClickHouse.
· Infrastructure: Docker + Kubernetes.

---

13. Documentation

Events (Internal Agent Log)

```typescript
interface PredictiveRiskUpdated {
  subject: string;
  new_composite_score: number;
  model_triggers: string[];
  timestamp: number;
}
```

Repository Structure (Addition)

```
ai-risk-engine/
├── models/
│   ├── fraud/                  # Fraud predictor training & serving
│   ├── sanctions/              # Sanctions exposure model
│   ├── anomaly/                # Behavioural anomaly detector
│   ├── laundering/             # Graph‑based laundering detector
│   └── counterparty/           # Risk propagation
├── dispatcher/                 # Risk Action Dispatcher (TypeScript)
├── api/                        # REST/WS service for scores and alerts
├── graph_db/                   # Neo4j configuration and queries
├── training/                   # Training pipelines
└── tests/
```

---

Status: AI Risk Engine (Predictive) architecture complete.
Provides real‑time ML‑driven predictions for fraud, sanctions, anomalies, laundering, and counterparty risk, integrating directly with the on‑chain AML Risk Engine and the AI Compliance Agent. Ready for model development and integration with the Event Bus and Indexer Supernode.
