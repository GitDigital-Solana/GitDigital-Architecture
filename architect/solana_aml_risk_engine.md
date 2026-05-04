# architect solana-aml-risk-engine

---

## 1. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      AML Risk Engine                                 │
│                                                                      │
│  ┌──────────────┐  ┌─────────────────┐  ┌───────────────────────┐  │
│  │  Screening   │  │  Risk Scoring   │  │   Anchor Program      │  │
│  │  Providers   │  │  Orchestrator   │  │   (On-chain State)    │  │
│  │  (External)  │  │  (Off-chain TS) │  │                       │  │
│  └──────┬───────┘  └────────┬────────┘  └──────────┬────────────┘  │
│         │                   │                       │               │
│         └───────────────────▼───────────────────────▼               │
│                    ┌─────────────────────────────────┐              │
│                    │        PDA Accounts             │              │
│                    │  - RiskEngineConfig             │              │
│                    │  - SubjectRiskProfile           │              │
│                    │  - ScreeningRecord              │              │
│                    │  - TransactionRiskRecord        │              │
│                    │  - SanctionsListRecord          │              │
│                    │  - AlertRecord                  │              │
│                    │  - CaseRecord                   │              │
│                    └─────────────────────────────────┘              │
│                                    │                                │
│           ┌────────────────────────┼────────────────────┐          │
│           ▼                        ▼                    ▼           │
│  ┌──────────────┐      ┌─────────────────┐   ┌──────────────────┐  │
│  │  Rule Engine │      │  Alert Manager  │   │  Case Management │  │
│  │  (Off-chain) │      │  (Off-chain)    │   │  (Off-chain)     │  │
│  └──────────────┘      └─────────────────┘   └──────────────────┘  │
│           │                        │                    │           │
│           └────────────────────────▼────────────────────┘          │
│                          ┌──────────────────┐                       │
│                          │  SQL Indexer +   │                       │
│                          │  Event Bus       │                       │
│                          └──────────────────┘                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Subsystem Breakdown

| Subsystem | Responsibility |
|---|---|
| **Risk Engine Config** | Governance-controlled thresholds, rule weights, jurisdiction rules |
| **Subject Risk Profiler** | Aggregate and score subject wallet risk across all vectors |
| **Transaction Screener** | Screen individual transactions for risk patterns |
| **Sanctions Registry** | On-chain list of sanctioned addresses and entities |
| **Alert Manager** | Generate, escalate, and resolve risk alerts |
| **Case Management** | Structured investigation workflow for flagged subjects |
| **Rule Engine** | Off-chain configurable rules that feed on-chain risk scores |
| **Screening Providers** | External feed integration (Chainalysis, Elliptic, TRM Labs) |
| **ZK Risk Proof** | Prove risk score is below threshold without revealing score |
| **Indexer** | Event-driven SQL for audit trail and dashboard queries |

---

## 3. Solana Program Design

```rust
// programs/aml_risk_engine/src/lib.rs

declare_id!("AMLRisk1111111111111111111111111111111111111");

#[program]
pub mod aml_risk_engine {
    use super::*;

    // ── Engine Admin ───────────────────────────────────────────
    pub fn initialize_engine(
        ctx: Context<InitEngine>,
        config: RiskEngineConfigParams,
    ) -> Result<()>;

    pub fn update_config(
        ctx: Context<UpdateConfig>,
        config: RiskEngineConfigParams,
    ) -> Result<()>;

    pub fn pause_engine(ctx: Context<PauseEngine>) -> Result<()>;

    pub fn resume_engine(ctx: Context<ResumeEngine>) -> Result<()>;

    // ── Sanctions Registry ─────────────────────────────────────
    pub fn add_sanctions_entry(
        ctx: Context<AddSanctions>,
        params: SanctionsParams,
    ) -> Result<()>;

    pub fn remove_sanctions_entry(
        ctx: Context<RemoveSanctions>,
        reason: String,
    ) -> Result<()>;

    pub fn update_sanctions_entry(
        ctx: Context<UpdateSanctions>,
        params: SanctionsUpdateParams,
    ) -> Result<()>;

    // ── Subject Risk Profile ───────────────────────────────────
    pub fn initialize_risk_profile(
        ctx: Context<InitRiskProfile>,
        subject: Pubkey,
    ) -> Result<()>;

    pub fn update_risk_score(
        ctx: Context<UpdateRiskScore>,
        params: RiskScoreParams,
    ) -> Result<()>;

    pub fn flag_subject(
        ctx: Context<FlagSubject>,
        params: FlagParams,
    ) -> Result<()>;

    pub fn clear_subject_flag(
        ctx: Context<ClearFlag>,
        resolution: FlagResolution,
    ) -> Result<()>;

    // ── Transaction Screening ──────────────────────────────────
    pub fn screen_transaction(
        ctx: Context<ScreenTransaction>,
        params: TxScreenParams,
    ) -> Result<()>;

    pub fn flag_transaction(
        ctx: Context<FlagTransaction>,
        reason: TxFlagReason,
    ) -> Result<()>;

    // ── Alerts ─────────────────────────────────────────────────
    pub fn create_alert(
        ctx: Context<CreateAlert>,
        params: AlertParams,
    ) -> Result<()>;

    pub fn escalate_alert(
        ctx: Context<EscalateAlert>,
        reason: String,
    ) -> Result<()>;

    pub fn resolve_alert(
        ctx: Context<ResolveAlert>,
        resolution: AlertResolution,
    ) -> Result<()>;

    // ── Case Management ────────────────────────────────────────
    pub fn open_case(
        ctx: Context<OpenCase>,
        params: CaseParams,
    ) -> Result<()>;

    pub fn update_case(
        ctx: Context<UpdateCase>,
        params: CaseUpdateParams,
    ) -> Result<()>;

    pub fn close_case(
        ctx: Context<CloseCase>,
        outcome: CaseOutcome,
    ) -> Result<()>;

    pub fn file_sar(
        ctx: Context<FileSar>,
        sar_hash: [u8; 32],
    ) -> Result<()>;

    // ── ZK Risk Proof ──────────────────────────────────────────
    pub fn verify_risk_zk(
        ctx: Context<VerifyRiskZk>,
        proof: Vec<u8>,
        public_inputs: Vec<[u8; 32]>,
    ) -> Result<()>;
}
```

---

## 4. PDA Map

```
Seed Pattern → Account Type → Description
─────────────────────────────────────────────────────────────────────
["risk_config"]
  → RiskEngineConfig
  → Global thresholds, rule weights, jurisdiction overrides, paused flag

["risk_profile", subject_wallet]
  → SubjectRiskProfile
  → Composite risk score, flag status, screening history summary

["screening", subject_wallet, screening_id]
  → ScreeningRecord
  → Single screening event: provider, score, result, timestamp

["tx_risk", tx_signature_hash]
  → TransactionRiskRecord
  → Risk assessment of a specific Solana transaction

["sanctions", target_pubkey]
  → SanctionsListRecord
  → Sanctioned address entry: source, list type, severity, added_at

["alert", subject_wallet, alert_nonce]
  → AlertRecord
  → Generated risk alert: trigger, severity, status

["case", case_id]
  → CaseRecord
  → Investigation case: subject, alerts, status, SAR hash

["sar", case_id]
  → SarRecord
  → Suspicious Activity Report hash anchor

["zk_risk_proof", subject_wallet, verifier_pubkey]
  → ZkRiskProofRecord
  → On-chain anchor for a verified ZK risk proof
```

```rust
// PDA derivation patterns
let (risk_profile_pda, bump) = Pubkey::find_program_address(
    &[b"risk_profile", subject.as_ref()],
    &program_id
);

let (sanctions_pda, bump) = Pubkey::find_program_address(
    &[b"sanctions", target_pubkey.as_ref()],
    &program_id
);

let (alert_pda, bump) = Pubkey::find_program_address(
    &[b"alert", subject.as_ref(), &alert_nonce.to_le_bytes()],
    &program_id
);

let (case_pda, bump) = Pubkey::find_program_address(
    &[b"case", case_id.as_bytes()],
    &program_id
);
```

---

## 5. API Schema

### REST Endpoints

```
── Engine Admin ────────────────────────────────────────────────────
POST   /v1/engine/initialize              → initialize engine
PUT    /v1/engine/config                  → update config
POST   /v1/engine/pause                   → pause engine
POST   /v1/engine/resume                  → resume engine
GET    /v1/engine/config                  → fetch config
GET    /v1/engine/health                  → engine health + stats

── Sanctions ───────────────────────────────────────────────────────
POST   /v1/sanctions/add                  → { tx_sig, sanctions_pda }
DELETE /v1/sanctions/{pubkey}/remove      → { tx_sig }
PUT    /v1/sanctions/{pubkey}/update      → { tx_sig }
GET    /v1/sanctions/{pubkey}             → SanctionsListRecord
GET    /v1/sanctions/check/{pubkey}       → { sanctioned: bool, severity }
GET    /v1/sanctions                      → SanctionsListRecord[] (paginated)

── Risk Profiles ───────────────────────────────────────────────────
POST   /v1/risk/profile/init              → { tx_sig, profile_pda }
PUT    /v1/risk/profile/{subject}/score   → { tx_sig }
POST   /v1/risk/profile/{subject}/flag    → { tx_sig }
POST   /v1/risk/profile/{subject}/clear   → { tx_sig }
GET    /v1/risk/profile/{subject}         → SubjectRiskProfile
GET    /v1/risk/profile/{subject}/history → ScreeningRecord[]

── Transaction Screening ───────────────────────────────────────────
POST   /v1/risk/transaction/screen        → { tx_sig, tx_risk_pda }
POST   /v1/risk/transaction/{sig}/flag    → { tx_sig }
GET    /v1/risk/transaction/{sig}         → TransactionRiskRecord

── Alerts ──────────────────────────────────────────────────────────
POST   /v1/alerts/create                  → { tx_sig, alert_pda }
POST   /v1/alerts/{id}/escalate           → { tx_sig }
POST   /v1/alerts/{id}/resolve            → { tx_sig }
GET    /v1/alerts/{id}                    → AlertRecord
GET    /v1/alerts?subject={wallet}        → AlertRecord[]
GET    /v1/alerts?status=open             → AlertRecord[]

── Cases ───────────────────────────────────────────────────────────
POST   /v1/cases/open                     → { tx_sig, case_pda }
PUT    /v1/cases/{id}/update              → { tx_sig }
POST   /v1/cases/{id}/close               → { tx_sig }
POST   /v1/cases/{id}/sar                 → { tx_sig, sar_pda }
GET    /v1/cases/{id}                     → CaseRecord
GET    /v1/cases?subject={wallet}         → CaseRecord[]
GET    /v1/cases?status=open              → CaseRecord[]

── ZK ──────────────────────────────────────────────────────────────
POST   /v1/zk/prove-risk                  → { proof, public_inputs }
POST   /v1/zk/verify-risk                 → { valid: bool, proof_pda }
```

### Request/Response Types

```typescript
interface RiskScoreParams {
  subject:           string;
  composite_score:   number;       // 0–100 aggregate
  sanctions_score:   number;       // 0–100
  behavioral_score:  number;       // 0–100
  counterparty_score: number;      // 0–100
  geographic_score:  number;       // 0–100
  provider:          string;
  evidence_hash:     string;       // sha256 of evidence bundle
  screened_at:       number;
}

interface SanctionsParams {
  target_pubkey:  string;
  list_source:    SanctionsSource;
  list_type:      SanctionsListType;
  severity:       SanctionsSeverity;
  entity_name:    string;
  program_name:   string;          // e.g. "OFAC SDN"
  added_at:       number;
  evidence_uri:   string;
}

interface TxScreenParams {
  tx_sig_hash:       string;       // sha256 of tx signature
  subject:           string;
  counterparties:    string[];
  amount_usd:        number;
  risk_score:        number;
  risk_flags:        TxRiskFlag[];
  screened_at:       number;
}

interface AlertParams {
  subject:        string;
  alert_type:     AlertType;
  severity:       AlertSeverity;
  trigger_score:  number;
  description:    string;
  evidence_hash:  string;
}

interface CaseParams {
  case_id:      string;
  subject:      string;
  alert_pdas:   string[];
  case_type:    CaseType;
  priority:     CasePriority;
  assigned_to:  string;
  description:  string;
}
```

---

## 6. Data Models

### On-chain (Rust)

```rust
#[account]
pub struct RiskEngineConfig {
    pub authority:              Pubkey,
    pub auto_flag_threshold:    u8,     // score >= this → auto-flag
    pub auto_block_threshold:   u8,     // score >= this → block txs
    pub alert_threshold:        u8,     // score >= this → create alert
    pub sanctions_weight:       u8,     // 0–100 weight in composite
    pub behavioral_weight:      u8,
    pub counterparty_weight:    u8,
    pub geographic_weight:      u8,
    pub blocked_jurisdictions:  Vec<String>, // ISO codes, max 32
    pub high_risk_jurisdictions: Vec<String>,
    pub sar_required_threshold: u8,     // score >= this → SAR required
    pub review_timeout_secs:    i64,
    pub paused:                 bool,
    pub version:                u8,
    pub bump:                   u8,
}

#[account]
pub struct SubjectRiskProfile {
    pub subject:            Pubkey,
    pub composite_score:    u8,         // 0–100 weighted average
    pub sanctions_score:    u8,
    pub behavioral_score:   u8,
    pub counterparty_score: u8,
    pub geographic_score:   u8,
    pub risk_tier:          RiskTier,
    pub flag_status:        FlagStatus,
    pub flag_reason:        Option<FlagReason>,
    pub flag_set_at:        Option<i64>,
    pub last_screened_at:   i64,
    pub screening_count:    u32,
    pub open_alert_count:   u8,
    pub open_case_count:    u8,
    pub sar_filed:          bool,
    pub created_at:         i64,
    pub updated_at:         i64,
    pub bump:               u8,
}

#[account]
pub struct ScreeningRecord {
    pub subject:        Pubkey,
    pub provider:       String,         // max 64
    pub composite_score: u8,
    pub sanctions_score: u8,
    pub behavioral_score: u8,
    pub counterparty_score: u8,
    pub geographic_score: u8,
    pub result:         ScreeningResult,
    pub evidence_hash:  [u8; 32],
    pub screened_at:    i64,
    pub bump:           u8,
}

#[account]
pub struct TransactionRiskRecord {
    pub tx_sig_hash:    [u8; 32],
    pub subject:        Pubkey,
    pub risk_score:     u8,
    pub risk_flags:     u64,            // bitmask of TxRiskFlag
    pub result:         TxRiskResult,
    pub screened_at:    i64,
    pub bump:           u8,
}

#[account]
pub struct SanctionsListRecord {
    pub target:         Pubkey,
    pub list_source:    SanctionsSource,
    pub list_type:      SanctionsListType,
    pub severity:       SanctionsSeverity,
    pub entity_name:    String,         // max 128
    pub program_name:   String,         // max 64
    pub evidence_uri:   String,         // max 256
    pub added_at:       i64,
    pub removed_at:     Option<i64>,
    pub active:         bool,
    pub bump:           u8,
}

#[account]
pub struct AlertRecord {
    pub subject:        Pubkey,
    pub alert_type:     AlertType,
    pub severity:       AlertSeverity,
    pub trigger_score:  u8,
    pub description:    String,         // max 256
    pub evidence_hash:  [u8; 32],
    pub status:         AlertStatus,
    pub resolution:     Option<AlertResolution>,
    pub created_at:     i64,
    pub resolved_at:    Option<i64>,
    pub escalated:      bool,
    pub bump:           u8,
}

#[account]
pub struct CaseRecord {
    pub case_id:        String,         // max 64
    pub subject:        Pubkey,
    pub case_type:      CaseType,
    pub priority:       CasePriority,
    pub status:         CaseStatus,
    pub outcome:        Option<CaseOutcome>,
    pub assigned_to:    Pubkey,
    pub description:    String,         // max 512
    pub sar_hash:       Option<[u8; 32]>,
    pub sar_filed_at:   Option<i64>,
    pub opened_at:      i64,
    pub closed_at:      Option<i64>,
    pub bump:           u8,
}

// ── Enums ─────────────────────────────────────────────────────────
#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum RiskTier {
    Low,        // 0–29
    Medium,     // 30–59
    High,       // 60–79
    Critical,   // 80–100
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum FlagStatus { Clean, Monitoring, Flagged, Blocked }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum FlagReason {
    SanctionsMatch,
    HighRiskScore,
    SuspiciousActivity,
    PEPMatch,
    GeographicRisk,
    CounterpartyRisk,
    ManualFlag,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum FlagResolution { Cleared, FalsePositive, Escalated }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum ScreeningResult { Clean, Review, Flagged, Blocked }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum SanctionsSource { OFAC, UN, EU, HMT, AUSTRAC, Custom }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum SanctionsListType { SDN, Sectoral, PEP, AdverseMedia }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum SanctionsSeverity { Critical, High, Medium, Low }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum AlertType {
    SanctionsHit,
    ScoreThreshold,
    SuspiciousPattern,
    HighValueTx,
    RapidMovement,
    Structuring,
    GeographicAlert,
    CounterpartyAlert,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum AlertSeverity { Critical, High, Medium, Low, Informational }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum AlertStatus { Open, InReview, Resolved, Escalated }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum AlertResolution { TruePositive, FalsePositive, Inconclusive }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum CaseType { SAR, Investigation, EnhancedDueDiligence, Regulatory }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum CasePriority { Critical, High, Medium, Low }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum CaseStatus { Open, InReview, PendingSAR, Closed }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum CaseOutcome { SARFiled, Cleared, Blocked, RegulatoryReferral }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum TxRiskResult { Clean, Review, Blocked }

bitflags::bitflags! {
    pub struct TxRiskFlag: u64 {
        const HIGH_VALUE          = 0b000001;
        const STRUCTURING         = 0b000010;
        const RAPID_MOVEMENT      = 0b000100;
        const SANCTIONED_CPTY     = 0b001000;
        const HIGH_RISK_GEO       = 0b010000;
        const MIXER_INTERACTION   = 0b100000;
        const DARK_MARKET_LINK    = 0b1000000;
        const LAYERING_PATTERN    = 0b10000000;
    }
}
```

### Off-chain SQL

```sql
-- Engine config cache
CREATE TABLE risk_engine_config (
    id                      SERIAL PRIMARY KEY,
    authority               TEXT NOT NULL,
    auto_flag_threshold     SMALLINT NOT NULL DEFAULT 80,
    auto_block_threshold    SMALLINT NOT NULL DEFAULT 95,
    alert_threshold         SMALLINT NOT NULL DEFAULT 70,
    sar_required_threshold  SMALLINT NOT NULL DEFAULT 85,
    paused                  BOOLEAN DEFAULT FALSE,
    version                 SMALLINT DEFAULT 1,
    updated_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Subject risk profiles
CREATE TABLE subject_risk_profiles (
    pda                 TEXT PRIMARY KEY,
    subject             TEXT NOT NULL UNIQUE,
    composite_score     SMALLINT NOT NULL DEFAULT 0,
    sanctions_score     SMALLINT NOT NULL DEFAULT 0,
    behavioral_score    SMALLINT NOT NULL DEFAULT 0,
    counterparty_score  SMALLINT NOT NULL DEFAULT 0,
    geographic_score    SMALLINT NOT NULL DEFAULT 0,
    risk_tier           TEXT NOT NULL DEFAULT 'Low',
    flag_status         TEXT NOT NULL DEFAULT 'Clean',
    flag_reason         TEXT,
    flag_set_at         TIMESTAMPTZ,
    last_screened_at    TIMESTAMPTZ,
    screening_count     INTEGER DEFAULT 0,
    open_alert_count    SMALLINT DEFAULT 0,
    sar_filed           BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Screening records
CREATE TABLE screening_records (
    pda                 TEXT PRIMARY KEY,
    subject             TEXT NOT NULL,
    provider            TEXT NOT NULL,
    composite_score     SMALLINT NOT NULL,
    sanctions_score     SMALLINT NOT NULL,
    behavioral_score    SMALLINT NOT NULL,
    counterparty_score  SMALLINT NOT NULL,
    geographic_score    SMALLINT NOT NULL,
    result              TEXT NOT NULL,
    evidence_hash       TEXT NOT NULL,
    screened_at         TIMESTAMPTZ NOT NULL
);

-- Transaction risk records
CREATE TABLE transaction_risk_records (
    pda             TEXT PRIMARY KEY,
    tx_sig_hash     TEXT NOT NULL UNIQUE,
    subject         TEXT NOT NULL,
    risk_score      SMALLINT NOT NULL,
    risk_flags      BIGINT NOT NULL DEFAULT 0,
    result          TEXT NOT NULL,
    screened_at     TIMESTAMPTZ NOT NULL
);

-- Sanctions list
CREATE TABLE sanctions_list (
    pda             TEXT PRIMARY KEY,
    target          TEXT NOT NULL UNIQUE,
    list_source     TEXT NOT NULL,
    list_type       TEXT NOT NULL,
    severity        TEXT NOT NULL,
    entity_name     TEXT NOT NULL,
    program_name    TEXT NOT NULL,
    evidence_uri    TEXT,
    added_at        TIMESTAMPTZ NOT NULL,
    removed_at      TIMESTAMPTZ,
    active          BOOLEAN DEFAULT TRUE
);

-- Alert records
CREATE TABLE alert_records (
    pda             TEXT PRIMARY KEY,
    subject         TEXT NOT NULL,
    alert_type      TEXT NOT NULL,
    severity        TEXT NOT NULL,
    trigger_score   SMALLINT NOT NULL,
    description     TEXT,
    evidence_hash   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'Open',
    resolution      TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,
    escalated       BOOLEAN DEFAULT FALSE
);

-- Case records
CREATE TABLE case_records (
    pda             TEXT PRIMARY KEY,
    case_id         TEXT NOT NULL UNIQUE,
    subject         TEXT NOT NULL,
    case_type       TEXT NOT NULL,
    priority        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'Open',
    outcome         TEXT,
    assigned_to     TEXT NOT NULL,
    description     TEXT,
    sar_hash        TEXT,
    sar_filed_at    TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ DEFAULT NOW(),
    closed_at       TIMESTAMPTZ
);

-- ZK risk proof anchors
CREATE TABLE zk_risk_proofs (
    pda             TEXT PRIMARY KEY,
    subject         TEXT NOT NULL,
    verifier        TEXT NOT NULL,
    proof_hash      TEXT NOT NULL,
    public_inputs   JSONB,
    verified_at     TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_risk_profile_score      ON subject_risk_profiles(composite_score);
CREATE INDEX idx_risk_profile_flag       ON subject_risk_profiles(flag_status);
CREATE INDEX idx_risk_profile_tier       ON subject_risk_profiles(risk_tier);
CREATE INDEX idx_screening_subject       ON screening_records(subject);
CREATE INDEX idx_screening_result        ON screening_records(result);
CREATE INDEX idx_tx_risk_subject         ON transaction_risk_records(subject);
CREATE INDEX idx_tx_risk_result          ON transaction_risk_records(result);
CREATE INDEX idx_sanctions_active        ON sanctions_list(active);
CREATE INDEX idx_sanctions_severity      ON sanctions_list(severity);
CREATE INDEX idx_alert_subject           ON alert_records(subject);
CREATE INDEX idx_alert_status            ON alert_records(status);
CREATE INDEX idx_alert_severity          ON alert_records(severity);
CREATE INDEX idx_case_subject            ON case_records(subject);
CREATE INDEX idx_case_status             ON case_records(status);
CREATE INDEX idx_case_priority           ON case_records(priority);
```

---

## 7. Client SDK

```typescript
// sdk/src/AmlRiskEngineSDK.ts

import { Connection, PublicKey }       from '@solana/web3.js';
import { AnchorProvider, Program, BN } from '@coral-xyz/anchor';
import { AmlRiskEngine }               from './types';
import { IDL }                         from './idl';

export const PROGRAM_ID =
  new PublicKey('AMLRisk1111111111111111111111111111111111111');

export class AmlRiskEngineSDK {
  program: Program<AmlRiskEngine>;

  constructor(connection: Connection, wallet: any) {
    const provider = new AnchorProvider(connection, wallet, {});
    this.program   = new Program(IDL, PROGRAM_ID, provider);
  }

  // ── PDA Helpers ─────────────────────────────────────────────
  findRiskProfilePDA(subject: PublicKey): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('risk_profile'), subject.toBuffer()],
      PROGRAM_ID
    );
  }

  findSanctionsPDA(target: PublicKey): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('sanctions'), target.toBuffer()],
      PROGRAM_ID
    );
  }

  findAlertPDA(subject: PublicKey, nonce: number): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [
        Buffer.from('alert'),
        subject.toBuffer(),
        Buffer.from(new Uint32Array([nonce]).buffer),
      ],
      PROGRAM_ID
    );
  }

  findCasePDA(caseId: string): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('case'), Buffer.from(caseId)],
      PROGRAM_ID
    );
  }

  findScreeningPDA(subject: PublicKey, screeningId: string): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('screening'), subject.toBuffer(), Buffer.from(screeningId)],
      PROGRAM_ID
    );
  }

  // ── Sanctions ────────────────────────────────────────────────
  async addSanctionsEntry(params: SanctionsParams): Promise<string> {
    const [sanctionsPDA] = this.findSanctionsPDA(
      new PublicKey(params.target_pubkey)
    );
    return this.program.methods
      .addSanctionsEntry(params)
      .accounts({ sanctionsRecord: sanctionsPDA })
      .rpc();
  }

  async checkSanctions(target: PublicKey): Promise<{
    sanctioned: boolean;
    severity:   string | null;
    source:     string | null;
  }> {
    try {
      const [pda] = this.findSanctionsPDA(target);
      const record = await this.program.account.sanctionsListRecord.fetch(pda);
      if (record.active) {
        return {
          sanctioned: true,
          severity:   record.severity.toString(),
          source:     record.listSource.toString(),
        };
      }
      return { sanctioned: false, severity: null, source: null };
    } catch {
      return { sanctioned: false, severity: null, source: null };
    }
  }

  // ── Risk Profile ─────────────────────────────────────────────
  async initRiskProfile(subject: PublicKey): Promise<string> {
    const [profilePDA] = this.findRiskProfilePDA(subject);
    return this.program.methods
      .initializeRiskProfile(subject)
      .accounts({ riskProfile: profilePDA })
      .rpc();
  }

  async updateRiskScore(params: RiskScoreParams): Promise<string> {
    const subject    = new PublicKey(params.subject);
    const [profilePDA] = this.findRiskProfilePDA(subject);
    return this.program.methods
      .updateRiskScore({
        ...params,
        evidenceHash: Array.from(
          Buffer.from(params.evidence_hash, 'hex')
        ),
      })
      .accounts({ riskProfile: profilePDA })
      .rpc();
  }

  async flagSubject(
    subject: PublicKey,
    params: FlagParams
  ): Promise<string> {
    const [profilePDA] = this.findRiskProfilePDA(subject);
    return this.program.methods
      .flagSubject(params)
      .accounts({ riskProfile: profilePDA })
      .rpc();
  }

  async getRiskProfile(subject: PublicKey) {
    const [pda] = this.findRiskProfilePDA(subject);
    return this.program.account.subjectRiskProfile.fetch(pda);
  }

  async getRiskTier(subject: PublicKey): Promise<RiskTier> {
    const profile = await this.getRiskProfile(subject);
    return profile.riskTier as RiskTier;
  }

  async isBlocked(subject: PublicKey): Promise<boolean> {
    try {
      const profile = await this.getRiskProfile(subject);
      return profile.flagStatus.blocked !== undefined;
    } catch {
      return false;
    }
  }

  // ── Transaction Screening ────────────────────────────────────
  async screenTransaction(params: TxScreenParams): Promise<string> {
    const [txRiskPDA] = PublicKey.findProgramAddressSync(
      [
        Buffer.from('tx_risk'),
        Buffer.from(params.tx_sig_hash, 'hex'),
      ],
      PROGRAM_ID
    );
    return this.program.methods
      .screenTransaction({
        ...params,
        txSigHash: Array.from(
          Buffer.from(params.tx_sig_hash, 'hex')
        ),
      })
      .accounts({ txRiskRecord: txRiskPDA })
      .rpc();
  }

  // ── Alerts ───────────────────────────────────────────────────
  async createAlert(params: AlertParams): Promise<string> {
    const subject  = new PublicKey(params.subject);
    const profile  = await this.getRiskProfile(subject);
    const nonce    = profile.openAlertCount;
    const [alertPDA] = this.findAlertPDA(subject, nonce);

    return this.program.methods
      .createAlert({
        ...params,
        evidenceHash: Array.from(
          Buffer.from(params.evidence_hash, 'hex')
        ),
      })
      .accounts({ alertRecord: alertPDA })
      .rpc();
  }

  async resolveAlert(
    alertPDA: PublicKey,
    resolution: AlertResolution
  ): Promise<string> {
    return this.program.methods
      .resolveAlert(resolution)
      .accounts({ alertRecord: alertPDA })
      .rpc();
  }

  async getOpenAlerts(subject: PublicKey) {
    return this.program.account.alertRecord.all([
      {
        memcmp: {
          offset: 8,
          bytes:  subject.toBase58(),
        },
      },
    ]);
  }

  // ── Cases ────────────────────────────────────────────────────
  async openCase(params: CaseParams): Promise<string> {
    const [casePDA] = this.findCasePDA(params.case_id);
    return this.program.methods
      .openCase(params)
      .accounts({ caseRecord: casePDA })
      .rpc();
  }

  async fileSar(
    caseId: string,
    sarHash: Buffer
  ): Promise<string> {
    const [casePDA] = this.findCasePDA(caseId);
    return this.program.methods
      .fileSar(Array.from(sarHash))
      .accounts({ caseRecord: casePDA })
      .rpc();
  }

  async closeCase(
    caseId: string,
    outcome: CaseOutcome
  ): Promise<string> {
    const [casePDA] = this.findCasePDA(caseId);
    return this.program.methods
      .closeCase(outcome)
      .accounts({ caseRecord: casePDA })
      .rpc();
  }

  // ── ZK Risk Proof ────────────────────────────────────────────
  async verifyRiskZk(
    subject: PublicKey,
    proof: Uint8Array,
    publicInputs: Uint8Array[]
  ): Promise<string> {
    const [proofPDA] = PublicKey.findProgramAddressSync(
      [
        Buffer.from('zk_risk_proof'),
        subject.toBuffer(),
        this.program.provider.publicKey!.toBuffer(),
      ],
      PROGRAM_ID
    );
    return this.program.methods
      .verifyRiskZk(
        Array.from(proof),
        publicInputs.map(i => Array.from(i))
      )
      .accounts({ zkRiskProof: proofPDA })
      .rpc();
  }
}
```

---

## 8. Workflows

### Automated Risk Scoring Pipeline

```
Screening Provider (Chainalysis / Elliptic / TRM Labs)
    │
    ├─ 1. Provider screens subject wallet
    │       → sanctions_score, behavioral_score,
    │         counterparty_score, geographic_score
    │
    ├─ 2. Off-chain Rule Engine computes composite_score:
    │       composite = (sanctions  * sanctions_weight
    │                  + behavioral * behavioral_weight
    │                  + counterparty * counterparty_weight
    │                  + geographic * geographic_weight) / 100
    │
    ├─ 3. SDK: updateRiskScore({ subject, composite_score, ... })
    ├─ 4. Program: update SubjectRiskProfile PDA
    ├─ 5. Program: assign RiskTier (Low/Medium/High/Critical)
    │
    ├─ If composite_score >= auto_flag_threshold (e.g. 80)
    │       └─ Program: auto flagSubject({ reason: HighRiskScore })
    │
    ├─ If composite_score >= alert_threshold (e.g. 70)
    │       └─ SDK: createAlert({ alert_type: ScoreThreshold, severity: High })
    │
    └─ If composite_score >= auto_block_threshold (e.g. 95)
            └─ Program: flagSubject({ reason: HighRiskScore, status: Blocked })
```

### Sanctions Hit Workflow

```
Sanctions Sync Job (runs every 15 min)
    │
    ├─ 1. Fetch updated OFAC/UN/EU SDN lists
    ├─ 2. Diff against current on-chain sanctions list
    │
    ├─ New entry detected:
    │       ├─ SDK: addSanctionsEntry({ target, source: OFAC, severity: Critical })
    │       └─ Check all active subjects against new entry
    │
    ├─ Existing subject matches new entry:
    │       ├─ SDK: flagSubject({ reason: SanctionsMatch, status: Blocked })
    │       ├─ SDK: createAlert({ alert_type: SanctionsHit, severity: Critical })
    │       └─ SDK: openCase({ case_type: SAR, priority: Critical })
    │
    └─ Removed entry detected:
            └─ SDK: removeSanctionsEntry({ reason: "Delisted from SDN" })
```

### Transaction Risk Screening

```
Transaction submitted to Solana
    │
    ├─ 1. Off-chain monitor detects new tx involving tracked wallets
    ├─ 2. Rule Engine evaluates tx against risk patterns:
    │
    │       TxRiskFlag checks:
    │       ├─ HIGH_VALUE:        amount_usd > $10,000
    │       ├─ STRUCTURING:       multiple txs just below threshold
    │       ├─ RAPID_MOVEMENT:    >5 hops in <24h
    │       ├─ SANCTIONED_CPTY:   counterparty in sanctions list
    │       ├─ HIGH_RISK_GEO:     origin/dest in blocked jurisdiction
    │       ├─ MIXER_INTERACTION: interaction with known mixer
    │       ├─ DARK_MARKET_LINK:  chain link to dark market
    │       └─ LAYERING_PATTERN:  structured layering detected
    │
    ├─ 3. SDK: screenTransaction({ tx_sig_hash, subject, risk_score, flags })
    ├─ 4. Program: create TransactionRiskRecord PDA
    │
    ├─ If SANCTIONED_CPTY flag set:
    │       └─ Immediate block + Critical alert
    │
    └─ If risk_score >= threshold:
            └─ createAlert({ alert_type: SuspiciousPattern })
```

### Case Investigation and SAR Filing

```
Alert Escalated to Case
    │
    ├─ 1. Compliance officer reviews open alerts for subject
    ├─ 2. SDK: openCase({
    │           case_id:    "CASE-2026-001",
    │           subject:    wallet,
    │           case_type:  SAR,
    │           priority:   High,
    │           alert_pdas: [alert1, alert2]
    │         })
    │
    ├─ 3. Officer investigates (off-chain):
    │       ├─ Pull screening history
    │       ├─ Review tx risk records
    │       ├─ Gather evidence bundle
    │
    ├─ 4. Decision: SAR required (score >= sar_required_threshold)
    │       ├─ Officer drafts SAR (off-chain, encrypted)
    │       ├─ Hash SAR document: sar_hash = sha256(sar_document)
    │       ├─ Submit to FinCEN / FIU (off-chain)
    │       └─ SDK: fileSar({ case_id, sar_hash })
    │               → Program: anchors sar_hash on-chain (immutable proof)
    │
    └─ 5. SDK: closeCase({ case_id, outcome: SARFiled })
```

### ZK Risk Proof for DeFi Protocol Access

```
DeFi Protocol requires: "risk_score < 70 AND not sanctioned"
    │
    ├─ 1. Subject requests access to DeFi protocol
    ├─ 2. Off-chain ZK prover generates proof:
    │       Public  inputs: subject_wallet, threshold (70), sanctioned (false)
    │       Private inputs: composite_score, sanctions_check, screening_evidence
    │       Statement: "score < 70 AND sanctions_clean = true"
    │
    ├─ 3. SDK: verifyRiskZk({ proof, public_inputs })
    ├─ 4. Program: verify Groth16 via alt_bn128 precompile
    ├─ 5. Program: create ZkRiskProofRecord
    └─ 6. DeFi protocol: reads ZkRiskProofRecord → grants access
         (never sees actual score or evidence)
```

---

## 9. Security Considerations

| Threat | Mitigation |
|---|---|
| Score manipulation | `update_risk_score` restricted to registered screeners; evidence hash required |
| False sanctions removal | `remove_sanctions_entry` requires multisig authority signature |
| Alert flooding | Alert nonce tracked in `SubjectRiskProfile`; rate limit enforced off-chain |
| SAR hash tampering | `sar_hash` written once; no update instruction; PDA is immutable after set |
| Proof replay | ZK proofs include subject + timestamp nullifier; double-verify rejected |
| Backdoor flag clearing | `clear_subject_flag` requires `REVIEW_QUEUE` permission + case resolution |
| Threshold manipulation | `RiskEngineConfig` authority is Squads 5/9 multisig |
| PII in evidence | Only `evidence_hash` on-chain; raw evidence stored encrypted off-chain |
| Screener impersonation | Screeners registered via `initialize_risk_profile`; pubkey checked on every write |
| Config pause bypass | All instructions check `config.paused` first |

---

## 10. Compliance Considerations

| Requirement | Implementation |
|---|---|
| FATF Recommendation 16 | Transaction screening flags counterparty risk; travel rule data hashed |
| Bank Secrecy Act (BSA) | SAR filing anchored on-chain via `fileSar`; audit trail immutable |
| 6AMLD (EU) | Predicate offences tracked via `AlertType`; case records timestamped |
| OFAC Compliance | Real-time sanctions sync via `addSanctionsEntry`; auto-block on `Critical` |
| FinCEN CTR | High-value tx flag (`HIGH_VALUE`) triggers alert for CTR filing workflow |
| Structuring Detection | `STRUCTURING` tx flag detects smurfing patterns |
| PEP Screening | `SanctionsListType::PEP` with mandatory review queue |
| Travel Rule | `counterparties` array in `TxScreenParams` enables originator/beneficiary tracking |
| Record Retention | On-chain records immutable; off-chain SQL retained per jurisdiction rules |
| SAR Confidentiality | SAR content never stored on-chain; only `sar_hash` anchored |
| Whistleblower Protection | Alert/case records reference hashed evidence; officer identity protected off-chain |

---

## 11. Integration Points

| Module | Integration |
|---|---|
| **solana-kyc-credential-engine** | AML `SubjectRiskProfile` gates KYC tier upgrades; `isAmlClean()` checked before KYC issuance |
| **solana-identity-registry** | Risk profile linked to `IdentityRecord` via subject wallet |
| **zk-Identity-masks** | ZK proofs use `SubjectRiskProfile` score as private witness |
| **Aurora-zk-cryptography-framework** | Groth16 circuits for ZK risk score proofs |
| **GitDigital Financial Core** | `isBlocked()` check required before all financial operations |
| **solana-governance-policy-engine** | `RiskTier::Critical` subjects excluded from governance participation |
| **ZK-5D-Badge-Authority-app** | Risk tier feeds into badge authority restrictions |
| **Chainalysis / Elliptic / TRM Labs** | External screening providers submit via `updateRiskScore` |
| **OFAC / UN / EU sanctions feeds** | Sanctions sync job calls `addSanctionsEntry` / `removeSanctionsEntry` |
| **FinCEN / FIU reporting systems** | SAR filed off-chain; hash anchored on-chain via `fileSar` |

---

## 12. Documentation

### Events

```rust
#[event]
pub struct RiskScoreUpdated {
    pub subject:         Pubkey,
    pub composite_score: u8,
    pub risk_tier:       RiskTier,
    pub provider:        String,
    pub timestamp:       i64,
}

#[event]
pub struct SubjectFlagged {
    pub subject:    Pubkey,
    pub reason:     FlagReason,
    pub status:     FlagStatus,
    pub timestamp:  i64,
}

#[event]
pub struct SanctionsEntryAdded {
    pub target:       Pubkey,
    pub list_source:  SanctionsSource,
    pub severity:     SanctionsSeverity,
    pub timestamp:    i64,
}

#[event]
pub struct AlertCreated {
    pub subject:    Pubkey,
    pub alert_type: AlertType,
    pub severity:   AlertSeverity,
    pub timestamp:  i64,
}

#[event]
pub struct AlertResolved {
    pub subject:    Pubkey,
    pub resolution: AlertResolution,
    pub timestamp:  i64,
}

#[event]
pub struct CaseOpened {
    pub case_id:   String,
    pub subject:   Pubkey,
    pub case_type: CaseType,
    pub priority:  CasePriority,
    pub timestamp: i64,
}

#[event]
pub struct SarFiled {
    pub case_id:   String,
    pub sar_hash:  [u8; 32],
    pub timestamp: i64,
}

#[event]
pub struct ZkRiskVerified {
    pub subject:   Pubkey,
    pub verifier:  Pubkey,
    pub timestamp: i64,
}
```

### Error Codes

```rust
#[error_code]
pub enum AmlError {
    #[msg("Engine is paused")]                    EnginePaused,
    #[msg("Subject is sanctioned")]               SubjectSanctioned,
    #[msg("Subject is blocked")]                  SubjectBlocked,
    #[msg("Risk score exceeds threshold")]        RiskThresholdExceeded,
    #[msg("Sanctions entry already exists")]      DuplicateSanctionsEntry,
    #[msg("Sanctions entry not found")]           SanctionsEntryNotFound,
    #[msg("Risk profile not initialized")]        ProfileNotInitialized,
    #[msg("Alert not found")]                     AlertNotFound,
    #[msg("Case not found")]                      CaseNotFound,
    #[msg("SAR already filed for this case")]     SarAlreadyFiled,
    #[msg("Invalid ZK proof")]                    InvalidZkProof,
    #[msg("Jurisdiction is blocked")]             JurisdictionBlocked,
    #[msg("Unauthorized screener")]               UnauthorizedScreener,
    #[msg("Unauthorized")]                        Unauthorized,
}
```

### Repository Structure

```
aml-risk-engine/
├── programs/aml_risk_engine/
│   └── src/
│       ├── lib.rs
│       ├── instructions/
│       │   ├── initialize_engine.rs
│       │   ├── update_config.rs
│       │   ├── add_sanctions_entry.rs
│       │   ├── remove_sanctions_entry.rs
│       │   ├── init_risk_profile.rs
│       │   ├── update_risk_score.rs
│       │   ├── flag_subject.rs
│       │   ├── clear_subject_flag.rs
│       │   ├── screen_transaction.rs
│       │   ├── flag_transaction.rs
│       │   ├── create_alert.rs
│       │   ├── escalate_alert.rs
│       │   ├── resolve_alert.rs
│       │   ├── open_case.rs
│       │   ├── update_case.rs
│       │   ├── close_case.rs
│       │   ├── file_sar.rs
│       │   └── verify_risk_zk.rs
│       ├── state/
│       │   ├── risk_engine_config.rs
│       │   ├── subject_risk_profile.rs
│       │   ├── screening_record.rs
│       │   ├── transaction_risk_record.rs
│       │   ├── sanctions_list_record.rs
│       │   ├── alert_record.rs
│       │   └── case_record.rs
│       ├── errors.rs
│       └── events.rs
├── sdk/
│   └── src/
│       ├── AmlRiskEngineSDK.ts
│       ├── types.ts
│       └── idl.json
├── services/
│   ├── indexer/
│   ├── api/
│   ├── rule-engine/        ← configurable risk rule processor
│   ├── sanctions-sync/     ← OFAC/UN/EU feed synchronizer
│   └── tx-monitor/         ← real-time transaction watcher
├── schemas/
│   └── postgres/
└── tests/
    ├── anchor/
    └── integration/
```

---

**Status:** Architecture complete. Ready for `anchor build`.
