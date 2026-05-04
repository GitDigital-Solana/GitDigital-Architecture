Cross‑Chain Compliance Bridge

Export identity, KYC, AML proofs and risk scores to any chain — making GitDigital chain‑agnostic.

This bridge enables a single GitDigital compliance credential (the ZK MetaProof or a trusted attestation) to be consumed on Solana, Ethereum, zkSync, Bitcoin L2s, and future expansion chains. Two modes are supported:

1. Privacy‑Preserving ZK Export – the user submits their existing Groth16 MetaProof directly to the target chain’s verifier.
2. Automated State Sync – a trusted bridge relay (using Wormhole) pushes compliance status summaries (FullyCompliant, risk tier, etc.) to destination chain registries, enabling low‑latency on‑chain checks.

---

1. High‑Level Architecture

```
                          GitDigital Compliance Hub (Solana)
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  Compliance Supergraph │ Policy Engine │ ZK MetaProof Engine              │
  └───────────────────┬──────────────────────────────────────────────────────┘
                      │
          ┌───────────▼───────────┐
          │  Compliance Bridge    │   On‑chain emitter program (Anchor)
          │  Program              │   ◇ Requests Wormhole message
          │  • emit_attestation   │   ◇ Records attestation PDA
          └───────────┬───────────┘
                      │  (Wormhole message via Core Bridge)
          ┌───────────▼───────────┐
          │  Wormhole Guardians   │
          └───────────┬───────────┘
                      │
    ┌─────────────────┼─────────────────┬─────────────────┐
    ▼                 ▼                 ▼                 ▼
┌────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│Ethereum│    │   zkSync    │    │  (Other EVM) │    │  Bitcoin L2 │
│Smart   │    │   Era       │    │  Chains      │    │  (e.g., Stacks)
│Contract│    │  Contract   │    │              │    │             │
└────────┘    └─────────────┘    └─────────────┘    └─────────────┘
     ▲                 ▲                 ▲                 ▲
     └───────────┬─────┴─────────────────┴─────────────────┘
                 │  Direct ZK MetaProof submission
                 │  (no relay required)
```

---

2. Subsystem Breakdown

Subsystem Responsibility
Compliance Bridge Program (Solana) Anchor program that emits a Wormhole message containing a compliance attestation (profile summary or meta‑proof reference).
Bridge Attestation PDA On‑chain record of exported attestations, including a digest of the exported data and nonce.
Wormhole Core Bridge Trusted message passing between Solana and target chains.
Attestation Service (off‑chain) Listens for AttestationRequested events and initiates the Wormhole message emission (or can be called directly by user).
EVM Verifier Contracts Solidity contracts deployed on Ethereum, zkSync, etc. They can either: (a) verify a Groth16 MetaProof against a stored verifying key, or (b) process a Wormhole VAA and update a local compliance registry mapping.
Bitcoin Integration Module A Stacks clarity contract (or an Ordinals‑based registry) that stores compliance attestations. For purely on‑chain Bitcoin, a federated multi‑sig can enforce compliance checks via presigned transactions (out of scope but noted).
Policy Sync Oracle Keeps compliance policies (AccessPolicy definitions) consistent across chains.

---

3. Solana Program Design (compliance_bridge)

```rust
// programs/compliance_bridge/src/lib.rs

declare_id!("CompBr111111111111111111111111111111111111");

#[program]
pub mod compliance_bridge {
    use super::*;

    pub fn emit_attestation(
        ctx: Context<EmitAttestation>,
        target_chain: ChainId,
        attestation_type: AttestationType,
        data_hash: [u8; 32],
    ) -> Result<()>;
    // Creates a BridgeAttestation PDA, emits Wormhole message with the attestation payload

    pub fn verify_and_record_vaa(
        ctx: Context<VerifyAndRecordVaa>,
        vaa: Vec<u8>,
    ) -> Result<()>;
    // (For incoming messages from other chains — e.g., if a user migrates compliance from Ethereum
    //  back to Solana. Phase 2.)

    pub fn set_policy_mapping(
        ctx: Context<SetPolicyMapping>,
        policy_name: String,
        remote_policy_hash: [u8; 32],
        remote_chain: ChainId,
    ) -> Result<()>;
    // Maps a Solana policy to its equivalent on a remote chain

    // Admin
    pub fn initialize_config(
        ctx: Context<InitConfig>,
        wormhole_program: Pubkey,
        authority: Pubkey,
    ) -> Result<()>;
}
```

---

4. PDA Map (Solana side)

```
Seed Pattern → Account Type → Description
──────────────────────────────────────────────────────────────────────
["bridge_config"]
  → BridgeConfig
  → Wormhole core bridge program ID, authority

["bridge_attestation", subject, nonce]
  → BridgeAttestation
  → Record of an exported attestation: chain, type, hash, timestamp

["policy_mapping", policy_name, remote_chain]
  → PolicyMapping
  → Maps a local AccessPolicy name to the equivalent hash on a remote chain
```

```rust
// PDA derivations
let (config_pda, _) = Pubkey::find_program_address(
    &[b"bridge_config"],
    &program_id
);

let (attest_pda, _) = Pubkey::find_program_address(
    &[b"bridge_attestation", subject.as_ref(), &nonce.to_le_bytes()],
    &program_id
);
```

---

5. API Schema

REST Endpoints

```
── Attestation Export ─────────────────────────────────────────────────
POST   /v1/bridge/attestation/request
  → { tx_sig, attestation_pda, nonce }
  Body: { subject, target_chain, attestation_type, data_hash? }

GET    /v1/bridge/attestation/{subject}/{nonce}
  → BridgeAttestation

── Policy Mappings ───────────────────────────────────────────────────
POST   /v1/bridge/policy-mapping        → { tx_sig }
GET    /v1/bridge/policy-mapping/{policy}/{chain} → PolicyMapping

── Destination Chain Verification (via relay API) ────────────────────
POST   /v1/bridge/relay/submit-vaa      → (relays VAA to target chain)
```

Core Request Types

```typescript
interface EmitAttestationRequest {
  subject:          string;
  target_chain:     ChainId;           // e.g., "ethereum", "zksync", "stacks"
  attestation_type: "FULL_PROFILE" | "COMPLIANCE_STATUS" | "RISK_SCORE" | "ZK_METAPROOF_DIGEST";
  data_hash?:       string;            // for ZK_METAPROOF_DIGEST, the proof hash
}

interface BridgeAttestation {
  pda:              string;
  subject:          string;
  target_chain:     ChainId;
  attestation_type: AttestationType;
  data_hash:        string;
  nonce:            number;
  emitted_at:       number;
}
```

---

6. Data Models

On‑chain (Solana)

```rust
#[account]
pub struct BridgeConfig {
    pub wormhole_program: Pubkey,   // Wormhole core bridge program ID
    pub authority:        Pubkey,   // multisig
    pub bump:             u8,
}

#[account]
pub struct BridgeAttestation {
    pub subject:          Pubkey,
    pub target_chain:     ChainId,      // u8 enum
    pub attestation_type: AttestationType,
    pub data_hash:        [u8; 32],     // digest of exported data (or zero)
    pub nonce:            u64,
    pub emitted_at:       i64,
    pub bump:             u8,
}

#[account]
pub struct PolicyMapping {
    pub policy_name:       String,       // "KycEnhanced"
    pub remote_policy_hash: [u8; 32],    // Poseidon hash of remote policy definition
    pub remote_chain:      ChainId,
    pub bump:              u8,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum ChainId {
    Solana = 1,
    Ethereum = 2,
    ZkSync = 3,
    Stacks = 4,
    // ... other EVM chains
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum AttestationType {
    FullProfile,          // entire compliance profile
    ComplianceStatus,     // overall_status + blocking factors
    RiskScore,            // AML composite score only
    ZkMetaProofDigest,    // hash of a ZK MetaProof + expiry
}
```

Off‑chain SQL (Attestation Indexer)

```sql
CREATE TABLE attestation_exports (
    pda               TEXT PRIMARY KEY,
    subject           TEXT NOT NULL,
    target_chain      TEXT NOT NULL,
    attestation_type  TEXT NOT NULL,
    data_hash         TEXT,
    nonce             BIGINT NOT NULL,
    emitted_at        TIMESTAMPTZ NOT NULL
);
```

Destination‑Chain Contracts (Ethereum Example)

```solidity
// ComplianceRegistry.sol
contract ComplianceRegistry {
    mapping(address => bytes32) public userStatus;  // e.g., hash of (overall_status, expiry)
    mapping(address => uint256) public lastUpdated;

    event StatusUpdated(address indexed user, bytes32 statusHash, uint256 timestamp);

    function updateStatusFromVAA(
        bytes calldata vaa,
        bytes32[] calldata signatures
    ) external {
        // Verify Wormhole VAA signatures, parse payload, update userStatus
    }
}
```

Alternatively, for ZK MetaProof verification:

```solidity
// ZKComplianceVerifier.sol
contract ZKComplianceVerifier {
    using Pairing for *;
    VerifyingKey vk;

    function verifyProof(
        uint[2] memory a, uint[2][2] memory b, uint[2] memory c,
        uint[3] memory input   // [policyHash, nullifier, expiry]
    ) public returns (bool) {
        // Groth16 verify + nullifier check + expiry check,
        // then allow interaction
    }
}
```

---

7. Client SDK

```typescript
// sdk/src/CrossChainBridgeSDK.ts

export class CrossChainBridgeSDK {
  program: Program<ComplianceBridge>;

  // Emit an attestation on Solana (triggering Wormhole message)
  async emitAttestation(
    subject: PublicKey,
    targetChain: ChainId,
    attestationType: AttestationType,
    dataHash?: Uint8Array
  ): Promise<{ txSig: string; attestationPDA: PublicKey; nonce: number }> {
    const nonce = await this.getNextNonce(subject);
    const [pda] = PublicKey.findProgramAddressSync(
      [Buffer.from('bridge_attestation'), subject.toBuffer(), new BN(nonce).toArrayLike(Buffer, 'le', 8)],
      PROGRAM_ID
    );
    const txSig = await this.program.methods
      .emitAttestation(targetChain, attestationType, dataHash || Buffer.alloc(32))
      .accounts({ bridgeAttestation: pda, ... })
      .rpc();
    return { txSig, attestationPDA: pda, nonce };
  }

  // Helpers
  private async getNextNonce(subject: PublicKey): Promise<number> {
    // could be derived from a counter PDA
  }
}
```

For the destination chains, separate SDKs (ethers.js, etc.) would integrate the corresponding contracts.

---

8. Workflows

Privacy‑Preserving ZK Export (Direct)

```
1. User possesses a valid ZK MetaProof from the MetaProof Engine (Solana).
2. User calls the Ethereum contract directly:
   zkVerifier.verifyProof(a, b, c, [policyHash, nullifier, expiry])
3. Contract verifies the Groth16 proof (using pre‑stored VK), checks that
   policyHash matches a supported policy, nullifier is unused, expiry is future.
4. Contract records the nullifier and grants access / permissions.
   No bridge service required. No on‑chain compliance data revealed.
```

Automated State Sync via Wormhole

```
1. User or a keeper triggers `emit_attestation` on Solana for target chain Ethereum.
2. Compliance Bridge Program:
   a. Reads the latest SubjectComplianceProfile PDA.
   b. Constructs an attestation payload:
        { subject, overall_status, risk_score, kyc_tier, expiry, policy_hash, timestamp }
   c. Calls Wormhole `post_message` with the payload.
   d. Creates a BridgeAttestation PDA with the message hash.
3. Wormhole guardians produce a VAA.
4. A relayer (off‑chain service or keeper) submits the VAA to the Ethereum
   ComplianceRegistry contract.
5. Ethereumm contract verifies the VAA signatures, parses payload, updates the
   mapping: complianceStatus[subject_eth_address] = payloadHash (with timestamp).
6. Dapps on Ethereum can now call registry.getStatus(user) to read compliance.
```

Risk Score Export

```
- attestion_type = RiskScore, data_hash = keccak256(score + timestamp)
- Ethereum registry stores the score directly (or as a hash).
- Consumer contracts can enforce risk thresholds.
```

---

9. Security Considerations

Threat Mitigation
Fake attestation injection Only the bridge program (controlled by authority) can emit valid Wormhole messages; VAA signatures verified on destination chain.
Replay of VAA Each attestation includes a unique nonce; destination contract tracks processed nonces.
Stale compliance data Attestation payload includes an expiry timestamp; consumers must check it.
Policy mismatch across chains PolicyMapping PDA maps Solana policies to remote hashes; enforced by governance.
Unauthorized updates on destination Only the Wormhole‑verified VAA path or ZK proof verification can update state.
ZK proof replay across chains Nullifier is chain‑specific (or global). We recommend incorporating chainId into the nullifier to allow per‑chain usage tracking.

---

10. Compliance Considerations

Requirement Implementation
Cross‑border data transfer For ZK proofs, no personal data leaves the user’s possession. For status attestations, only minimal status codes are transmitted.
Right to deletion On destination chains, attestations can be made time‑bound; revocation lists can propagate via the bridge.
Audit trail Every export recorded on Solana via BridgeAttestation PDA; imported verifications logged on destination.
Selective disclosure The bridge supports risk‑score‑only or compliance‑status‑only exports, not the full profile.

---

11. Integration Points

Module Integration
Wormhole Core Bridge Message emission and VAA verification.
Compliance Supergraph Source of the unified compliance profile.
ZK MetaProof Engine Direct verification of MetaProofs on any chain with a deployed verifier.
Policy Engine Policies referenced in attestations; mappings ensure equivalent access rules.
Ethereum / EVM chains Solidity verifier contracts and compliance registries.
Stacks / Bitcoin L2 Clarity contracts for on‑chain attestation storage.
Off‑chain relayers Services that submit VAAs to destination chains.

---

12. Documentation

Events (Solana)

```rust
#[event]
pub struct AttestationEmitted {
    pub subject:          Pubkey,
    pub target_chain:     ChainId,
    pub attestation_type: AttestationType,
    pub data_hash:        [u8; 32],
    pub nonce:            u64,
    pub timestamp:        i64,
}

#[event]
pub struct PolicyMappingSet {
    pub policy_name:       String,
    pub remote_chain:      ChainId,
    pub remote_policy_hash: [u8; 32],
}
```

Error Codes

```rust
#[error_code]
pub enum BridgeError {
    #[msg("Invalid target chain")]          InvalidChain,
    #[msg("Profile not initialized")]       ProfileNotInitialized,
    #[msg("Wormhole message emission failed")] EmissionFailed,
    #[msg("Policy mapping already exists")] MappingAlreadyExists,
    #[msg("Unauthorized")]                  Unauthorized,
}
```

Repository Structure (Addition)

```
cross-chain-compliance-bridge/
├── programs/compliance_bridge/
│   └── src/
│       ├── lib.rs
│       ├── instructions/
│       │   ├── emit_attestation.rs
│       │   ├── verify_and_record_vaa.rs
│       │   ├── set_policy_mapping.rs
│       │   └── admin.rs
│       ├── state/
│       │   ├── config.rs
│       │   ├── attestation.rs
│       │   └── policy_mapping.rs
│       ├── cpi/
│       │   └── wormhole.rs
│       ├── errors.rs
│       └── events.rs
├── solidity/
│   ├── ComplianceRegistry.sol
│   └── ZKComplianceVerifier.sol
├── sdk/cross-chain-sdk/
│   └── src/
│       ├── solana/
│       │   └── CrossChainBridgeSDK.ts
│       ├── evm/
│       │   └── ComplianceRegistryClient.ts
│       └── types.ts
├── services/
│   ├── relayer/          # VAA submission service
│   └── attestation-keeper/  # Periodically emits attestations for registered users
└── tests/
    ├── anchor/
    └── integration/
```

---

Status: Cross‑Chain Compliance Bridge architecture complete.
GitDigital compliance credentials now portable to Ethereum, zkSync, Bitcoin L2, and beyond.
Supports zero‑knowledge privacy‑preserving export and automated state sync.
Ready for implementation alongside Wormhole integration and Solidity contract development.
