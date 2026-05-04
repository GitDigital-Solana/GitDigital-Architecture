GitDigital Reputation Graph

The social trust layer — tracks identity, credentials, contributions, risk, governance participation, and a computed reputation score, creating a Web of Trust that can be used for sybil resistance, access control, and reputation‑based privileges.

This module builds a directed, weighted graph from on‑chain and off‑chain signals. The on‑chain component stores a lightweight SubjectReputation PDA, while a powerful off‑chain Reputation Engine computes a global trustScore from the graph. Users can endorse each other, stake reputation, and prove a minimum reputation in ZK.

---

1. High‑Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       GitDigital Reputation Graph                              │
│                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                     Off‑Chain Reputation Engine                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐│   │
│  │  │ Graph Builder│  │ Trust Rank   │  │ Weight Calc  │  │ Reputation ││   │
│  │  │ (Neo4j)      │  │ (PageRank)   │  │ (ML model)   │  │ Score API  ││   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬─────┘│   │
│  │         └─────────────────┴─────────────────┴─────────────────┘      │   │
│  └──────────────────────────┬────────────────────────────────────────────┘   │
│                              │  (reads / writes)                              │
│  ┌───────────────────────────▼──────────────────────────────────────────┐    │
│  │                     On‑Chain Reputation Program (Anchor)               │    │
│  │                                                                        │    │
│  │  • SubjectReputation PDA (score, endorsers, disputes)                  │    │
│  │  • endorse / dispute instructions                                      │    │
│  │  • ZK reputation proof verifier (prove score ≥ X)                      │    │
│  │  • governance‑weighted reputation boost (auto‑distributed)             │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 │  Data Sources
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Event Bus + Indexer Supernode                         │
│   • Identity registrations, credential issuances, contribution proofs         │
│   • Governance votes, risk updates, transaction history                       │
│   • Cross‑chain attestations                                                  │
└──────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Other GitDigital Modules                                    │
│  • Identity Registry (DIDs)                                                   │
│  • ZK‑5D‑Badge Authority (credentials)                                        │
│  • Proof‑of‑Contribution Protocol (contributions)                             │
│  • Governance (voting, participation)                                         │
│  • AI Risk Engine (risk scores)                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

2. Subsystem Breakdown

Subsystem Responsibility
Reputation Engine (Off‑Chain) Builds and maintains the graph in Neo4j; runs periodic PageRank to compute a global trustScore; serves scores via API; detects sybil attacks and reputation manipulation.
On‑Chain Reputation Program Anchors the latest trustScore and endorsements; verifies ZK reputation proofs; allows staking and slashing of reputation.
Endorsement Mechanism Users can endorse other DIDs with a confidence weight. This creates a directed edge in the graph and contributes to trust propagation.
Contribution Weight Proofs of contribution (from the Proof‑of‑Contribution Protocol) are fed into the graph as evidence of positive activity.
Governance Participation Voting, proposal creation, and participation in DAOs add reputation. The graph uses this to identify active community members.
Risk De‑boosting AML risk score and sanctions flags decrease reputation, ensuring high‑risk entities cannot accumulate undue trust.
ZK Reputation Proof Enables a user to prove “my trustScore ≥ X” without disclosing the exact score, using a ZK circuit similar to the identity mask (but potentially simpler).
API / SDK REST and WebSocket for score queries, endorsement submissions, and graph insights.

---

3. Solana Program Design

```rust
// programs/reputation_graph/src/lib.rs

declare_id!("RepGrph11111111111111111111111111111111111");

#[program]
pub mod reputation_graph {
    use super::*;

    /// Updates a subject's on‑chain reputation record with a new trustScore.
    pub fn update_reputation(
        ctx: Context<UpdateReputation>,
        new_score: u64,       // scaled integer (e.g., 0‑10000)
        proof: Option<Vec<u8>>, // if submitted off‑chain, a signature from oracle
    ) -> Result<()>;

    /// User endorses another subject.
    pub fn endorse(
        ctx: Context<Endorse>,
        target: Pubkey,
        confidence_weight: u8, // 1‑100
    ) -> Result<()>;

    /// User retracts an endorsement.
    pub fn remove_endorsement(
        ctx: Context<RemoveEndorsement>,
        target: Pubkey,
    ) -> Result<()>;

    /// Stake reputation on a claim (for future games / curation)
    pub fn stake_reputation(
        ctx: Context<StakeReputation>,
        amount: u64,
        claim_id: Pubkey,
    ) -> Result<()>;

    /// Slash staked reputation (invoked by governance or supervisory program)
    pub fn slash_reputation(
        ctx: Context<SlashReputation>,
        stake_record: Pubkey,
    ) -> Result<()>;

    /// Verify a ZK proof that the subject’s trustScore is ≥ a threshold.
    pub fn verify_reputation_zk(
        ctx: Context<VerifyReputationZk>,
        proof: Vec<u8>,
        public_inputs: Vec<[u8; 32]>,
    ) -> Result<()>;

    pub fn initialize_config(
        ctx: Context<InitConfig>,
        oracle_authority: Pubkey,
        governance_program: Pubkey,
    ) -> Result<()>;

    pub fn update_config(
        ctx: Context<UpdateConfig>,
        oracle_authority: Option<Pubkey>,
    ) -> Result<()>;
}
```

---

4. PDA Map

```
Seed Pattern → Account Type → Description
──────────────────────────────────────────────────────────────────────
["reputation_config"]
  → ReputationConfig
  → Oracle authority, governance program, allowed score range

["subject_reputation", subject]
  → SubjectReputation
  → Current trustScore, endorsement count, dispute count, last updated

["endorsement", endorser, target]
  → EndorsementRecord
  → Directed endorsement edge: endorser → target with weight

["reputation_stake", subject, claim_id]
  → ReputationStake
  → Amount of reputation staked on a claim

["reputation_zk_proof", subject, threshold]
  → ZkReputationProofRecord
  → Anchors a verified ZK proof of minimum reputation
```

---

5. API Schema

REST Endpoints

```
── Reputation Scores ──────────────────────────────────────────────
GET    /v1/reputation/{subject}              → SubjectReputation
POST   /v1/reputation/batch                  → SubjectReputation[]
GET    /v1/reputation/leaderboard?limit=100  → LeaderboardEntry[]

── Endorsements ──────────────────────────────────────────────────
POST   /v1/reputation/endorse               → { tx_sig }
POST   /v1/reputation/remove-endorsement    → { tx_sig }
GET    /v1/reputation/endorsements/{subject}→ EndorsementRecord[]

── ZK Proofs ─────────────────────────────────────────────────────
POST   /v1/reputation/zk/generate           → { proof, public_inputs }
POST   /v1/reputation/zk/verify             → { tx_sig, valid }

── Graph Insights ────────────────────────────────────────────────
GET    /v1/reputation/graph/{subject}        → ReputationGraphView
GET    /v1/reputation/clusters               → ClusterAnalysis[]
```

Core Request Types

```typescript
interface SubjectReputation {
  subject: string;
  trust_score: number;           // 0‑10000
  endorsement_in: number;        // count of endorsements received
  endorsement_out: number;       // endorsements given
  contribution_points: number;   // from Proof‑of‑Contribution
  governance_weight: number;     // derived from voting power
  risk_penalty: number;          // reduction due to AML risk
  last_updated: number;
}

interface EndorseRequest {
  target: string;
  confidence_weight: number;     // 1‑100
}

interface GenerateReputationZkRequest {
  subject: string;
  min_score: number;
}
```

---

6. Data Models

On‑chain (Rust)

```rust
#[account]
pub struct ReputationConfig {
    pub oracle_authority:   Pubkey,      // authorised to push score updates
    pub governance_program: Pubkey,      // to verify governance participation
    pub bump:               u8,
}

#[account]
pub struct SubjectReputation {
    pub subject:                Pubkey,
    pub trust_score:            u64,      // scaled integer
    pub endorsement_count_in:   u32,
    pub endorsement_count_out:  u32,
    pub contribution_points:    u64,
    pub governance_weight:      u64,
    pub risk_penalty:           u64,      // subtracted from score
    pub last_updated:           i64,
    pub bump:                   u8,
}

#[account]
pub struct EndorsementRecord {
    pub endorser:   Pubkey,
    pub target:     Pubkey,
    pub weight:     u8,
    pub created_at: i64,
    pub bump:       u8,
}
```

Off‑chain Graph (Neo4j — Node & Edge Schemas)

```
Node: Subject
  - pubkey (string)
  - trustScore (float)
  - did (string)
  - riskScore (float)

Edge: ENDORSES
  - weight (float)
  - timestamp

Edge: DELEGATED_TO
Edge: VOTED_ON (proposal)
Edge: CONTRIBUTED (proof hash)
```

---

7. Client SDK

```typescript
// sdk/src/ReputationGraphSDK.ts
export class ReputationGraphSDK {
  program: Program<ReputationGraph>;

  async getReputation(subject: PublicKey): Promise<SubjectReputation> {
    const [pda] = this.findSubjectReputationPDA(subject);
    return this.program.account.subjectReputation.fetch(pda);
  }

  async endorse(
    endorser: PublicKey,
    target: PublicKey,
    weight: number
  ): Promise<string> {
    const [endorsementPda] = this.findEndorsementPDA(endorser, target);
    return this.program.methods
      .endorse(target, weight)
      .accounts({ endorsement: endorsementPda, ... })
      .rpc();
  }

  async verifyReputationZk(
    subject: PublicKey,
    proof: Uint8Array,
    publicInputs: Uint8Array[]
  ): Promise<string> {
    return this.program.methods
      .verifyReputationZk(Array.from(proof), publicInputs.map(i => Array.from(i)))
      .accounts({
        subjectReputation: this.findSubjectReputationPDA(subject)[0],
        ...,
      })
      .rpc();
  }
}
```

---

8. Workflows

Reputation Score Computation

```
1. Batch job runs periodically (e.g., daily) on the Graph Builder:
   - Pulls all identity registrations, KYC tiers, credential issuances,
     contribution proofs, governance votes, and endorsement events
     from ClickHouse/PostgreSQL.
   - Builds/updates the Neo4j graph.
2. Reputation Engine runs weighted PageRank:
   - Seeds: high‑KYC‑tier subjects, governance participants.
   - Edges weighted by endorsement confidence, delegation, co‑contribution.
   - Negative edges from risk scores (reduce reputation of risky nodes).
3. Each subject’s raw PageRank is normalised and combined with:
   - contribution_points (direct from Proof‑of‑Contribution) * weight
   - governance_weight (voting power / participation) * weight
   - minus risk_penalty (if AML flagged)
   producing a final `trustScore` (0‑10000).
4. The engine pushes updated `trustScore` to the on‑chain SubjectReputation PDA
   via the `update_reputation` instruction (signed by the oracle authority).
5. Events emitted; dashboard reflects new scores.
```

Endorsement Flow

```
1. Alice trusts Bob’s identity and wants to boost Bob’s reputation.
2. Alice calls `endorse(Bob, confidence_weight=80)` on‑chain.
3. Program creates/updates an EndorsementRecord PDA for Alice→Bob.
4. The off‑chain engine listens to this event and adds/updates the edge in the graph.
5. Next score recomputation will reflect this new endorsement.
6. Alice can later call `remove_endorsement(Bob)` to delete the edge.
```

ZK Reputation Proof

```
1. Carol needs to prove to a Protocol that she has a trustScore ≥ 3000
   without revealing her exact score.
2. Carol fetches her latest signed `SubjectReputation` PDA data.
3. She generates a ZK proof using a circuit similar to the mask circuit:
   inputs: private (actual score), public (min_score, nullifier, timestamp).
4. The Protocol calls `verify_reputation_zk(proof, [min_score_hash, nullifier, timestamp])`.
5. The on‑chain program verifies the Groth16 proof, checks the nullifier, and records it.
6. Carol is granted access based on proven reputation threshold.
```

Governance & Contribution Boosts

```
- Governance: When a user votes on a proposal or creates one, the governance program
  CPIs into the Reputation program to increment `governance_weight`.
- Contribution: The Proof‑of‑Contribution protocol also updates `contribution_points`
  on the SubjectReputation PDA (requires oracle authority).
```

---

9. Security Considerations

Threat Mitigation
Sybil endorsements (fake accounts) Endorsements are weighted by the endorser’s own trustScore (e.g., low‑score endorsers have minimal effect). Graph algorithms reduce the influence of isolated clusters.
Manipulation of score by oracle The oracle authority is a multisig; multiple independent replicas can be run; a ZK proof could be required for each update.
Privacy leakage of score ZK reputation proofs allow users to keep their exact score private.
Stake slashing misuse Slashing only possible via governance vote or predetermined conditions coded in the program.

---

10. Compliance Considerations

Requirement Implementation
Right to be forgotten Reputation scores are derived from publicly available on‑chain data; but a user can request removal of off‑chain stored data (graph edges) according to policy.
Fairness & bias The reputation algorithm is open‑source and its parameters are governed by the DAO. Oversight via dashboard.
Data minimisation On‑chain only stores scores and counts, no personal data. Graph database uses pubkeys.

---

11. Integration Points

Module Integration
Identity Registry Base node for each subject.
ZK‑5D‑Badge Authority Credentials feed into reputation.
Proof‑of‑Contribution Protocol Contribution points update.
Governance / DAO Voting participation boosts.
AI Risk Engine Risk scores de‑boost reputation.
zk‑Identity‑Masks ZK reputation proofs use the same proving infrastructure.
Financial Core Integration Could require a minimum reputation for certain high‑value operations.

---

12. Documentation

Events

```rust
#[event]
pub struct ReputationUpdated {
    pub subject:     Pubkey,
    pub new_score:   u64,
    pub old_score:   u64,
    pub timestamp:   i64,
}

#[event]
pub struct EndorsementCreated {
    pub endorser: Pubkey,
    pub target:   Pubkey,
    pub weight:   u8,
}
```

Error Codes

```rust
#[error_code]
pub enum ReputationError {
    #[msg("Invalid score update proof")]       InvalidUpdateProof,
    #[msg("Endorsement already exists")]       AlreadyEndorsed,
    #[msg("Insufficient reputation to endorse")] NotEnoughReputation,
    #[msg("Unauthorized oracle")]              Unauthorized,
}
```

Repository Structure (Addition)

```
gitdigital-reputation-graph/
├── programs/reputation_graph/
│   └── src/
│       ├── lib.rs
│       ├── instructions/
│       │   ├── update_reputation.rs
│       │   ├── endorse.rs
│       │   └── verify_zk.rs
│       ├── state/
│       │   ├── config.rs
│       │   ├── reputation.rs
│       │   └── endorsement.rs
│       ├── cpi/
│       │   └── governance.rs
│       ├── errors.rs
│       └── events.rs
├── engine/                     # Off‑chain reputation engine (Python/TS)
│   ├── graph/
│   ├── scoring/
│   └── api/
├── sdk/reputation-sdk/
│   └── src/
│       ├── ReputationGraphSDK.ts
│       └── types.ts
└── tests/
```

---

Status: GitDigital Reputation Graph architecture complete.
The social trust layer that aggregates identity, credentials, contributions, risk, and governance into a verifiable reputation score — with endorsement edges, ZK proofs, and sybil resistance.
Ready for implementation of the on‑chain program and off‑chain graph engine.
