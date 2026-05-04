Policy & Authorization Engine

Governance and access control layer giving fine-grained answers to: who can do what, under what conditions, based on which credentials, with what risk thresholds.

This engine sits above the Compliance Supergraph and answers the policy question after compliance status is known. It translates business rules into on-chain verifiable decisions.

---

1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Policy & Authorization Engine                       │
│              Rule-Based Access Control, Delegation, and                 │
│              Risk-Aware Action Authorisation                           │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Policy Store (PDA)                           │  │
│  │          Rules, Templates, Access Definitions                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│           │                    │                    │                   │
│           ▼                    ▼                    ▼                   │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │ Risk Threshold  │  │ Credential        │  │ Role & Attribute    │  │
│  │ Evaluator       │  │ Validator         │  │ Resolver            │  │
│  └─────────────────┘  └──────────────────┘  └──────────────────────┘  │
│           │                    │                    │                   │
│           └────────────────────▼────────────────────┘                  │
│                      ┌──────────────────────┐                          │
│                      │  Policy Decision     │                          │
│                      │  Program (Anchor)    │                          │
│                      └──────────┬───────────┘                          │
│                                 │                                       │
│          ┌──────────────────────┼───────────────────────┐              │
│          ▼                      ▼                       ▼              │
│  ┌──────────────┐    ┌────────────────────┐   ┌──────────────────┐    │
│  │  Access      │    │  Delegation        │    │  Policy Cache    │    │
│  │  Guard CPI   │    │  Manager           │    │  (Off‑chain)     │    │
│  └──────────────┘    └────────────────────┘   └──────────────────┘    │
│          │                      │                       │              │
│          └──────────────────────▼───────────────────────┘              │
│                       ┌──────────────────┐                             │
│                       │  Compliance      │                             │
│                       │  Supergraph CPI  │                             │
│                       └──────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────┘
```

The engine consumes the unified SubjectComplianceProfile via CPI to the Compliance Supergraph and applies additional rules: roles, credentials, delegations, and risk thresholds.

---

2. Subsystem Breakdown

Subsystem Responsibility
Policy Store (on‑chain) Stores named policies as PDAs, defining required compliance tiers, roles, credentials, and risk caps
Policy Decision Program Evaluates a (subject, action, resource) tuple against stored policies and the current compliance profile
Risk Threshold Evaluator Maps business‑specific risk tiers (low/medium/high) to required AML scores and compliance tiers
Credential Validator Checks that the subject holds required credentials (e.g. author badges, licence NFTs)
Attribute Resolver Pulls on‑chain and off‑chain attributes (role, jurisdiction, organisation) for fine‑grained conditions
Delegation Manager Allows a principal to delegate a subset of their permissions to another wallet for a limited time
Policy Cache (off‑chain) Materialises active policies and recent decisions for high‑speed authorisation without extra CPI
Event Logger Records every policy evaluation and delegation event for audit

---

3. Solana Program Design

```rust
// programs/policy_engine/src/lib.rs

declare_id!("PolAuth1111111111111111111111111111111111111");

#[program]
pub mod policy_engine {
    use super::*;

    // ── Policy Lifecycle ──────────────────────────────────────
    pub fn create_policy(
        ctx: Context<CreatePolicy>,
        name: String,
        description: String,
        rules: PolicyRules,
    ) -> Result<()>;

    pub fn update_policy(
        ctx: Context<UpdatePolicy>,
        rules: PolicyRules,
    ) -> Result<()>;

    pub fn deactivate_policy(ctx: Context<DeactivatePolicy>) -> Result<()>;

    // ── Authorisation ─────────────────────────────────────────
    pub fn evaluate(
        ctx: Context<Evaluate>,
        action: Action,
        resource: Resource,
    ) -> Result<AuthDecision>;

    pub fn assert_authorized(
        ctx: Context<AssertAuthorized>,
        action: Action,
        resource: Resource,
    ) -> Result<()>;
    // Hard‑fail version: reverts if not authorized

    // ── Delegation ────────────────────────────────────────────
    pub fn create_delegation(
        ctx: Context<CreateDelegation>,
        delegate: Pubkey,
        permissions: Vec<Permission>,
        expiration: i64,
    ) -> Result<()>;

    pub fn revoke_delegation(
        ctx: Context<RevokeDelegation>,
        delegation_pda: Pubkey,
    ) -> Result<()>;

    // ── Admin ─────────────────────────────────────────────────
    pub fn set_risk_tiers(
        ctx: Context<SetRiskTiers>,
        tier_config: RiskTierConfig,
    ) -> Result<()>;

    pub fn set_role_mapping(
        ctx: Context<SetRoleMapping>,
        role: String,
        required_credentials: Vec<CredentialRequirement>,
    ) -> Result<()>;
}
```

---

4. PDA Map

```
Seed Pattern → Account Type → Description
──────────────────────────────────────────────────────────────────────
["policy", name]
  → PolicyAccount
  → Stored policy rules (name must be unique)

["authority_delegation", principal, delegate]
  → DelegationRecord
  → Delegation of a subset of permissions from principal to delegate

["policy_config"]
  → PolicyConfig
  → Global thresholds: risk tiers, default credential issuers

["policy_event", event_nonce]
  → PolicyEventRecord
  → Audit log of evaluations and delegation changes
```

---

5. API Schema

REST Endpoints

```
── Policy Management ──────────────────────────────────────────────────
POST   /v1/policy/create                → { tx_sig, policy_pda }
PUT    /v1/policy/{name}                → { tx_sig }
DELETE /v1/policy/{name}                → { tx_sig }
GET    /v1/policy/{name}                → PolicyAccount

── Authorization ──────────────────────────────────────────────────────
POST   /v1/auth/evaluate               → AuthDecision
POST   /v1/auth/assert                 → { tx_sig } or revert

── Delegation ──────────────────────────────────────────────────────────
POST   /v1/delegation/create            → { tx_sig, delegation_pda }
DELETE /v1/delegation/{pda}             → { tx_sig }
GET    /v1/delegation/{principal}       → DelegationRecord[]
GET    /v1/delegation/check/{principal}/{delegate} → ActiveDelegation[]

── Configuration ────────────────────────────────────────────────────────
POST   /v1/config/risk-tiers            → { tx_sig }
POST   /v1/config/role-mapping          → { tx_sig }
GET    /v1/config                        → PolicyConfig
```

Core Request Types

```typescript
interface PolicyRules {
  required_compliance?: AccessPolicy;  // from Compliance Supergraph
  required_credentials?: CredentialRequirement[];
  required_roles?: string[];           // e.g. "author", "validator"
  max_aml_risk?: number;               // AML score threshold
  jurisdiction_allowlist?: string[];
  custom_predicate?: string;           // WASM or off-chain reference
}

interface AuthDecision {
  authorized: boolean;
  reason: string;
  matched_policy: string | null;
  required_checks: RequiredCheck[];
  delegation?: {
    delegate: string;
    principal: string;
  };
}

interface RequiredCheck {
  type: 'compliance' | 'credential' | 'role' | 'risk' | 'jurisdiction';
  passed: boolean;
  detail: string;
}

interface Action {
  program_id: string;    // target program
  instruction: string;   // e.g. "transfer", "vote", "publish"
  params_hash?: string;
}

interface Resource {
  type: string;          // "token", "governance_proposal", "document"
  identifier: string;    // token mint, proposal PDA, etc.
  amount?: number;
}
```

---

6. Data Models

On‑chain (Rust)

```rust
#[account]
pub struct PolicyAccount {
    pub name:           String,      // max 32
    pub description:    String,      // max 256
    pub rules:          PolicyRules,
    pub active:         bool,
    pub created_at:     i64,
    pub updated_at:     i64,
    pub authority:      Pubkey,      // policy owner (DAO or multisig)
    pub bump:           u8,
}

#[account]
pub struct DelegationRecord {
    pub principal:       Pubkey,
    pub delegate:        Pubkey,
    pub permissions:     Vec<Permission>,  // max 8
    pub expiration:      i64,
    pub active:          bool,
    pub created_at:      i64,
    pub bump:            u8,
}

#[account]
pub struct PolicyConfig {
    pub authority:              Pubkey,
    pub risk_tiers:             RiskTierConfig,
    pub default_credential_issuers: Vec<Pubkey>,  // trusted badge programs
    pub bump:                   u8,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone)]
pub struct PolicyRules {
    pub required_compliance:    Option<AccessPolicy>,   // 0 = none
    pub required_credentials:   Vec<CredentialRequirement>, // max 5
    pub required_roles:         Vec<String>,            // max 5, length 32 each
    pub max_aml_risk:           Option<u8>,              // 0‑100
    pub jurisdiction_allowlist: Vec<String>,            // ISO codes
    pub custom_predicate_hash:  Option<[u8; 32]>,       // hash of WASM module
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone)]
pub struct CredentialRequirement {
    pub issuer:         Pubkey,      // badge authority program
    pub badge_name:     String,      // e.g. "AuthorBadge", "ValidatorLicense"
    pub minimum_tier:   u8,
    pub required_metadata: Vec<(String, String)>, // key-value pairs
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone)]
pub struct RiskTierConfig {
    pub low_threshold:    u8,   // e.g. < 30
    pub medium_threshold: u8,   // 30‑69
    // scores >= medium_threshold treated as high risk
}
```

Off‑chain SQL (Policy Cache)

```sql
CREATE TABLE policies (
    name                   TEXT PRIMARY KEY,
    description            TEXT,
    required_compliance    TEXT,
    required_credentials   JSONB,
    required_roles         TEXT[],
    max_aml_risk           SMALLINT,
    jurisdiction_allowlist TEXT[],
    active                 BOOLEAN DEFAULT TRUE,
    created_at             TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE delegation_cache (
    pda          TEXT PRIMARY KEY,
    principal    TEXT NOT NULL,
    delegate     TEXT NOT NULL,
    permissions  TEXT[],
    expiration   TIMESTAMPTZ,
    active       BOOLEAN
);

CREATE TABLE authorization_events (
    subject       TEXT NOT NULL,
    action        JSONB,
    resource      JSONB,
    decision      BOOLEAN,
    reason        TEXT,
    matched_policy TEXT,
    delegation_pda TEXT,
    timestamp     TIMESTAMPTZ DEFAULT NOW()
);
```

---

7. Client SDK

```typescript
// sdk/src/PolicyEngineSDK.ts

export class PolicyEngineSDK {
  program: Program<PolicyEngine>;

  // ── Policy CRUD ──────────────────────────────────────────────
  async createPolicy(
    name: string,
    description: string,
    rules: PolicyRules
  ): Promise<string> {
    const [pda] = PublicKey.findProgramAddressSync(
      [Buffer.from('policy'), Buffer.from(name)],
      PROGRAM_ID
    );
    return this.program.methods
      .createPolicy(name, description, rules)
      .accounts({ policy: pda })
      .rpc();
  }

  // ── Authorization ────────────────────────────────────────────
  async evaluate(
    subject: PublicKey,
    action: Action,
    resource: Resource
  ): Promise<AuthDecision> {
    const [policyPDA] = this.findActivePolicyPDA(action, resource);
    const [complianceProfilePDA] = new ComplianceSupergraphSDK(...)
      .findProfilePDA(subject);
    return this.program.methods
      .evaluate(action, resource)
      .accounts({
        policy: policyPDA,
        complianceProfile: complianceProfilePDA,
        // credentials, delegations resolved internally
      })
      .view();
  }

  async assertAuthorized(
    subject: PublicKey,
    action: Action,
    resource: Resource
  ): Promise<string> {
    // Will revert entire transaction if not authorized
    return this.program.methods
      .assertAuthorized(action, resource)
      .accounts({ ... })
      .rpc();
  }

  // ── Delegation Helpers ───────────────────────────────────────
  async delegate(
    principal: PublicKey,
    delegate: PublicKey,
    permissions: Permission[],
    expiration: number
  ): Promise<string> {
    const [pda] = PublicKey.findProgramAddressSync(
      [Buffer.from('authority_delegation'), principal.toBuffer(), delegate.toBuffer()],
      PROGRAM_ID
    );
    return this.program.methods
      .createDelegation(delegate, permissions, new BN(expiration))
      .accounts({ delegation: pda })
      .rpc();
  }

  async getActiveDelegations(
    principal: PublicKey
  ): Promise<DelegationRecord[]> {
    return this.program.account.delegationRecord.all([
      { memcmp: { offset: 8 + 32, bytes: principal.toBase58() } },
      { memcmp: { offset: 8 + 32 + 32 + 8 + 8, bytes: bs58.encode([1]) } } // active = true
    ]);
  }
}
```

---

8. Workflows

Policy Evaluation Flow (On‑chain)

```
Caller program (e.g. GitDigital Core) wants to know if subject can publish a manuscript
    │
    ├─ 1. CPI: policy_engine.assert_authorized(subject, Action::Publish, Resource::Manuscript)
    │
    ├─ 2. Policy Engine loads the relevant policy PDA for the action/resource
    │       (via action→policy mapping stored in config)
    │
    ├─ 3. Simultaneously loads:
    │       ├─ SubjectComplianceProfile PDA (via supergraph CPI)
    │       ├─ Credential holdings (via badge‑authority CPI or token accounts)
    │       └─ Active delegations (own delegation PDA)
    │
    ├─ 4. Checks in order (short‑circuit on first failure):
    │
    │    a. Risk Threshold:
    │        if rules.max_aml_risk && profile.aml_composite_score > max_aml_risk
    │            → FAIL "Risk score exceeds policy limit"
    │
    │    b. Compliance Requirement:
    │        CPI into supergraph: check_access(subject, rules.required_compliance)
    │            → FAIL "Insufficient compliance – KYC Enhanced required"
    │
    │    c. Jurisdiction Allowlist:
    │        if rules.jurisdiction_allowlist is set and
    │           profile.kyc_jurisdiction ∉ allowlist
    │            → FAIL "Jurisdiction not allowed"
    │
    │    d. Required Credentials:
    │        for each CredentialRequirement, verify ownership
    │        (via badge authority or metadata program)
    │            → FAIL "Missing author badge"
    │
    │    e. Required Roles:
    │        check if subject has been granted the role on‑chain
    │        (roles are stored in Role Registry PDA)
    │            → FAIL "Role 'editor' required"
    │
    │    f. Delegation: if subject != signer, check active delegation
    │        from signer → subject with required permissions
    │            → FAIL "No active delegation for this action"
    │
    └─ 5. If all passed: write authorization event (if configurable) and return OK
         Else: return error with detailed reason
```

Delegation Workflow

```
Author (“principal”) wants to let delegate (“assistant”) publish on their behalf.
    │
    ├─ 1. Principal calls create_delegation(delegate, [Permission::Publish], expires)
    │
    ├─ 2. Program validates:
    │        - Principal owns the author role / badge
    │        - Expiration is within a maximum (e.g. 30 days)
    │
    ├─ 3. Writes DelegationRecord with active=true
    │
    ├─ 4. Delegate now calls assert_authorized(subject=principal, action=Publish, ...)
    │        The engine sees subject != signer, finds delegation principal→delegate,
    │        and if it includes required permission, passes.
    │
    └─ 5. revoke_delegation can be called at any time by principal.
```

Risk‑Aware Access Example

```
Policy for high‑value transfer (value > $10k):
    required_compliance: KycEnhanced
    max_aml_risk: 30
    required_roles: ["verified_account"]

Authorization steps:
    1. Check that AML composite_score ≤ 30
    2. Check KYC Enhanced compliance via supergraph
    3. Check subject holds "verified_account" role (granted after additional steps)
    4. If delegate is acting, ensure delegation includes "transfer" permission
```

Policy Deployment & Update

```
Governance (DAO) votes to change the “publish” policy, tightening risk threshold.
    ├─ 1. Governance Proposal approves updatePolicy(tx)
    ├─ 2. Multisig (or DAO‑controlled PDA) calls update_policy(name, new_rules)
    ├─ 3. Program updates PolicyAccount under the predefined authority
    └─ 4. Next evaluate() uses the updated rules immediately (no migration needed)
```

---

9. Security Considerations

Threat Mitigation
Policy tampering Only the designated authority (multisig/DAO) can update policies; all changes logged
Bypassing compliance via delegation Delegation grants only specified permissions; subject still evaluated under original compliance profile
Stale credential snapshot Credential checks are live CPIs to badge programs or NFT ownership, never cached on‑chain
Risk score manipulation AML score sourced directly from supergraph CPI (signed by AML engine)
Policy‑action mismatch Action‑to‑policy mapping stored on‑chain in PolicyConfig, validated on every evaluation
Replay of delegation Delegation PDA derived from principal+delegate; only one active at a time; expiration forced
Missing role registry updates Role revocations immediately affect authorization; roles are checked via live CPI

---

10. Compliance Considerations

Requirement Implementation
Attribute‑based access (ABAC) Policies can combine compliance, credentials, roles, jurisdiction, and custom predicates
Risk‑based access Every policy includes an optional max_aml_risk threshold derived from AML engine
Delegation of authority Delegation records are auditable and include scope, expiration, and revocation history
Audit trails On‑chain PolicyEventRecord captures each evaluation and delegation change
Separation of concerns Policy engine consumes compliance status; it does not modify it
Privacy Risk scores and credentials are never revealed in public logs; only binary decision output

---

11. Integration Points

Module Integration
Compliance Supergraph CPI for check_access(subject, required_compliance) and to read AML score
ZK‑5D‑Badge Authority CPI to verify that subject holds a badge of required tier/type
Role Registry (DAO) On‑chain role assignments consumed during role checks
Anchor IDL programs External programs use policy_engine::assert_authorized as a guard before their own logic
Off‑chain services API layer serves policy decisions to non‑Anchor clients
Governance Policies are governed via DAO proposals; authority is a multisig PDA
Event Bus Authorization events are streamed for monitoring and alerting

---

12. Documentation

Events

```rust
#[event]
pub struct PolicyCreated {
    pub name:   String,
    pub rules:  PolicyRules,
    pub timestamp: i64,
}

#[event]
pub struct PolicyUpdated {
    pub name:   String,
    pub old_rules: PolicyRules,
    pub new_rules: PolicyRules,
    pub timestamp: i64,
}

#[event]
pub struct AuthorizationEvaluated {
    pub subject:      Pubkey,
    pub action:       Action,
    pub resource:     Resource,
    pub authorized:   bool,
    pub reason:       String,
    pub matched_policy: String,
    pub timestamp:    i64,
}

#[event]
pub struct DelegationCreated {
    pub principal: Pubkey,
    pub delegate:  Pubkey,
    pub permissions: Vec<Permission>,
    pub expiration: i64,
    pub timestamp: i64,
}

#[event]
pub struct DelegationRevoked {
    pub delegation_pda: Pubkey,
    pub principal: Pubkey,
    pub delegate: Pubkey,
    pub timestamp: i64,
}
```

Error Codes

```rust
#[error_code]
pub enum PolicyEngineError {
    #[msg("Policy not found")]                      PolicyNotFound,
    #[msg("Policy is inactive")]                    PolicyInactive,
    #[msg("Compliance level insufficient")]         ComplianceInsufficient,
    #[msg("Risk score exceeds policy limit")]       RiskScoreExceeded,
    #[msg("Missing required credential")]           MissingCredential,
    #[msg("Missing required role")]                 MissingRole,
    #[msg("Jurisdiction not allowed")]              JurisdictionNotAllowed,
    #[msg("No active delegation for this action")]  DelegationNotFound,
    #[msg("Delegation expired")]                    DelegationExpired,
    #[msg("Delegation permission missing")]         DelegationPermissionMissing,
    #[msg("Unauthorized to modify policy")]         Unauthorized,
}
```

Repository Structure (Addition to compliance-supergraph/)

```
policy-authorization-engine/
├── programs/policy_engine/
│   └── src/
│       ├── lib.rs
│       ├── instructions/
│       │   ├── create_policy.rs
│       │   ├── update_policy.rs
│       │   ├── evaluate.rs
│       │   ├── assert_authorized.rs
│       │   ├── create_delegation.rs
│       │   ├── revoke_delegation.rs
│       │   └── admin.rs
│       ├── state/
│       │   ├── policy.rs
│       │   ├── delegation.rs
│       │   └── config.rs
│       ├── cpi/
│       │   ├── compliance_supergraph.rs
│       │   └── badge_authority.rs
│       ├── evaluator/
│       │   ├── risk.rs
│       │   ├── credentials.rs
│       │   ├── roles.rs
│       │   └── rules_engine.rs
│       ├── errors.rs
│       └── events.rs
├── sdk/policy-engine-sdk/
│   └── src/
│       ├── PolicyEngineSDK.ts
│       └── types.ts
├── services/
│   └── policy-cache/     # Off‑chain policy decision cache
└── tests/
```

---

Status: Policy & Authorization Engine architecture complete.
Provides the “who can do what” layer on top of the unified compliance profile.
Integrates compliance tiers, risk thresholds, credentials, roles, and safe delegation.
Ready for implementation alongside Compliance Supergraph.
