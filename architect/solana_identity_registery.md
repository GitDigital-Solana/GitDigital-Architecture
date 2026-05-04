# architect solana-identity-registry

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Solana Identity Registry                     │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│  │  Client  │───▶│   SDK    │───▶│   Anchor Program     │  │
│  │  (React/ │    │  (TS/RS) │    │   (On-chain State)   │  │
│  │   CLI)   │    └──────────┘    └──────────────────────┘  │
│  └──────────┘                             │                 │
│                                           ▼                 │
│                              ┌────────────────────────┐     │
│                              │   PDA Accounts          │     │
│                              │  - IdentityRecord       │     │
│                              │  - CredentialRecord     │     │
│                              │  - DelegationRecord     │     │
│                              │  - RegistryConfig       │     │
│                              └────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
  ┌─────────────┐                   ┌──────────────────┐
  │  Off-chain  │                   │   ZK Layer       │
  │  Indexer    │                   │  (zk-Identity-   │
  │  (SQL/IPFS) │                   │   masks)         │
  └─────────────┘                   └──────────────────┘
```

---

## 2. Subsystem Breakdown

| Subsystem | Responsibility |
|---|---|
| **Identity Core** | Register, update, revoke identity records on-chain |
| **Credential Engine** | Issue and verify attestations linked to an identity |
| **Delegation Layer** | Grant scoped permissions to sub-keys or agents |
| **ZK Bridge** | Generate/verify proofs without revealing raw identity |
| **Registry Config** | Governance-controlled global parameters |
| **Off-chain Indexer** | Event-driven SQL index for fast queries |
| **SDK** | TypeScript + Rust client libraries |

---

## 3. Solana Program Design

```rust
// programs/solana_identity_registry/src/lib.rs

declare_id!("IDReg1111111111111111111111111111111111111111");

#[program]
pub mod solana_identity_registry {
    use super::*;

    pub fn initialize_registry(ctx: Context<InitRegistry>, config: RegistryConfig) -> Result<()>;
    pub fn register_identity(ctx: Context<RegisterIdentity>, params: RegisterParams) -> Result<()>;
    pub fn update_identity(ctx: Context<UpdateIdentity>, metadata_uri: String) -> Result<()>;
    pub fn revoke_identity(ctx: Context<RevokeIdentity>) -> Result<()>;
    pub fn issue_credential(ctx: Context<IssueCredential>, params: CredentialParams) -> Result<()>;
    pub fn revoke_credential(ctx: Context<RevokeCredential>) -> Result<()>;
    pub fn delegate_authority(ctx: Context<DelegateAuthority>, params: DelegationParams) -> Result<()>;
    pub fn revoke_delegation(ctx: Context<RevokeDelegation>) -> Result<()>;
    pub fn verify_zk_identity(ctx: Context<VerifyZk>, proof: Vec<u8>, public_inputs: Vec<[u8;32]>) -> Result<()>;
}
```

---

## 4. PDA Map

```
Seed Pattern → Account Type → Description
────────────────────────────────────────────────────────
["registry"]
  → RegistryConfig
  → Global config: authority, fees, allowed issuers

["identity", wallet_pubkey]
  → IdentityRecord
  → One per wallet: DID, metadata URI, status, version

["credential", identity_pda, credential_id]
  → CredentialRecord
  → Issued attestation linked to an identity

["delegation", identity_pda, delegate_pubkey]
  → DelegationRecord
  → Scoped permission grant from identity to agent

["zk_verification", identity_pda, verifier_pubkey]
  → ZkVerificationRecord
  → Proof result anchored on-chain
```

```rust
// PDA derivation examples
let (identity_pda, bump) = Pubkey::find_program_address(
    &[b"identity", wallet.key().as_ref()],
    &program_id
);

let (credential_pda, bump) = Pubkey::find_program_address(
    &[b"credential", identity_pda.as_ref(), credential_id.as_bytes()],
    &program_id
);
```

---

## 5. API Schema

### REST (Off-chain Indexer)

```
GET    /v1/identity/{wallet}              → IdentityRecord
POST   /v1/identity/register              → { tx_sig, identity_pda }
PUT    /v1/identity/{wallet}/update       → { tx_sig }
DELETE /v1/identity/{wallet}/revoke       → { tx_sig }

GET    /v1/credential/{identity_pda}      → CredentialRecord[]
POST   /v1/credential/issue               → { tx_sig, credential_pda }
DELETE /v1/credential/{id}/revoke         → { tx_sig }

GET    /v1/delegation/{identity_pda}      → DelegationRecord[]
POST   /v1/delegation/grant               → { tx_sig, delegation_pda }
DELETE /v1/delegation/{id}/revoke         → { tx_sig }

POST   /v1/zk/prove                       → { proof, public_inputs }
POST   /v1/zk/verify                      → { valid: bool }
```

### Request Bodies

```typescript
interface RegisterParams {
  wallet: string;           // base58 pubkey
  did: string;              // e.g. "did:gitdigital:solana:..."
  metadata_uri: string;     // IPFS CID or URL
  identity_type: IdentityType;
}

interface CredentialParams {
  identity_pda: string;
  credential_id: string;
  credential_type: CredentialType;
  issuer_did: string;
  expiry_ts: number | null;
  metadata_uri: string;
}

interface DelegationParams {
  identity_pda: string;
  delegate: string;         // agent pubkey
  scope: ScopeFlags;        // bitmask of allowed actions
  expiry_ts: number | null;
}
```

---

## 6. Data Models

### On-chain (Rust)

```rust
#[account]
pub struct RegistryConfig {
    pub authority: Pubkey,
    pub fee_mint: Option<Pubkey>,
    pub registration_fee: u64,
    pub allowed_issuers: Vec<Pubkey>,   // max 32
    pub paused: bool,
    pub version: u8,
}

#[account]
pub struct IdentityRecord {
    pub owner: Pubkey,
    pub did: String,                    // max 128 chars
    pub metadata_uri: String,           // max 256 chars
    pub identity_type: IdentityType,
    pub status: IdentityStatus,
    pub version: u32,
    pub created_at: i64,
    pub updated_at: i64,
    pub bump: u8,
}

#[account]
pub struct CredentialRecord {
    pub identity: Pubkey,
    pub issuer: Pubkey,
    pub credential_id: String,          // max 64 chars
    pub credential_type: CredentialType,
    pub metadata_uri: String,
    pub status: CredentialStatus,
    pub issued_at: i64,
    pub expires_at: Option<i64>,
    pub bump: u8,
}

#[account]
pub struct DelegationRecord {
    pub identity: Pubkey,
    pub delegate: Pubkey,
    pub scope: u64,                     // bitmask
    pub granted_at: i64,
    pub expires_at: Option<i64>,
    pub revoked: bool,
    pub bump: u8,
}

// Enums
#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum IdentityType   { Individual, Organization, Agent, Device }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum IdentityStatus { Active, Suspended, Revoked }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum CredentialType { KYC, AML, Accreditation, Role, Qualification, Custom }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq)]
pub enum CredentialStatus { Active, Expired, Revoked }
```

### Off-chain (SQL)

```sql
CREATE TABLE identity_records (
    pda           TEXT PRIMARY KEY,
    owner         TEXT NOT NULL,
    did           TEXT NOT NULL UNIQUE,
    metadata_uri  TEXT,
    identity_type TEXT NOT NULL,
    status        TEXT NOT NULL DEFAULT 'Active',
    version       INTEGER DEFAULT 1,
    created_at    TIMESTAMPTZ,
    updated_at    TIMESTAMPTZ,
    raw_data      JSONB
);

CREATE TABLE credential_records (
    pda              TEXT PRIMARY KEY,
    identity_pda     TEXT REFERENCES identity_records(pda),
    issuer           TEXT NOT NULL,
    credential_id    TEXT NOT NULL,
    credential_type  TEXT NOT NULL,
    status           TEXT NOT NULL DEFAULT 'Active',
    issued_at        TIMESTAMPTZ,
    expires_at       TIMESTAMPTZ
);

CREATE TABLE delegation_records (
    pda           TEXT PRIMARY KEY,
    identity_pda  TEXT REFERENCES identity_records(pda),
    delegate      TEXT NOT NULL,
    scope         BIGINT NOT NULL,
    granted_at    TIMESTAMPTZ,
    expires_at    TIMESTAMPTZ,
    revoked       BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_identity_owner    ON identity_records(owner);
CREATE INDEX idx_credential_issuer ON credential_records(issuer);
CREATE INDEX idx_delegation_agent  ON delegation_records(delegate);
```

---

## 7. Client SDK

```typescript
// sdk/src/IdentityRegistrySDK.ts

import { Connection, PublicKey } from '@solana/web3.js';
import { AnchorProvider, Program } from '@coral-xyz/anchor';
import type { SolanaIdentityRegistry } from './types';
import { IDL } from './idl';

export const PROGRAM_ID = new PublicKey('IDReg1111111111111111111111111111111111111111');

export class IdentityRegistrySDK {
  program: Program<SolanaIdentityRegistry>;

  constructor(connection: Connection, wallet: any) {
    const provider = new AnchorProvider(connection, wallet, {});
    this.program = new Program(IDL, PROGRAM_ID, provider);
  }

  // ── PDAs ──────────────────────────────────────────────────
  findIdentityPDA(wallet: PublicKey): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('identity'), wallet.toBuffer()],
      PROGRAM_ID
    );
  }

  findCredentialPDA(identityPDA: PublicKey, credentialId: string): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('credential'), identityPDA.toBuffer(), Buffer.from(credentialId)],
      PROGRAM_ID
    );
  }

  findDelegationPDA(identityPDA: PublicKey, delegate: PublicKey): [PublicKey, number] {
    return PublicKey.findProgramAddressSync(
      [Buffer.from('delegation'), identityPDA.toBuffer(), delegate.toBuffer()],
      PROGRAM_ID
    );
  }

  // ── Instructions ──────────────────────────────────────────
  async registerIdentity(params: RegisterParams): Promise<string> {
    const [identityPDA] = this.findIdentityPDA(this.program.provider.publicKey!);
    return this.program.methods
      .registerIdentity(params)
      .accounts({ identityRecord: identityPDA })
      .rpc();
  }

  async issueCredential(params: CredentialParams): Promise<string> {
    const [identityPDA] = this.findIdentityPDA(new PublicKey(params.identity_pda));
    const [credentialPDA] = this.findCredentialPDA(identityPDA, params.credential_id);
    return this.program.methods
      .issueCredential(params)
      .accounts({ identityRecord: identityPDA, credentialRecord: credentialPDA })
      .rpc();
  }

  async delegateAuthority(params: DelegationParams): Promise<string> {
    const [identityPDA] = this.findIdentityPDA(this.program.provider.publicKey!);
    const [delegationPDA] = this.findDelegationPDA(identityPDA, new PublicKey(params.delegate));
    return this.program.methods
      .delegateAuthority(params)
      .accounts({ identityRecord: identityPDA, delegationRecord: delegationPDA })
      .rpc();
  }

  // ── Queries ───────────────────────────────────────────────
  async getIdentity(wallet: PublicKey) {
    const [pda] = this.findIdentityPDA(wallet);
    return this.program.account.identityRecord.fetch(pda);
  }

  async getCredentials(identityPDA: PublicKey) {
    return this.program.account.credentialRecord.all([
      { memcmp: { offset: 8, bytes: identityPDA.toBase58() } }
    ]);
  }
}
```

---

## 8. Workflows

### Register Identity

```
User Wallet
    │
    ├─ 1. SDK: findIdentityPDA(wallet)
    ├─ 2. SDK: registerIdentity({ did, metadata_uri, identity_type })
    ├─ 3. Program: validate inputs, check registry not paused
    ├─ 4. Program: create IdentityRecord PDA
    ├─ 5. Program: emit IdentityRegistered event
    └─ 6. Indexer: picks up event → writes to SQL
```

### Issue Credential (KYC)

```
Issuer (trusted org)
    │
    ├─ 1. Verify off-chain KYC data
    ├─ 2. SDK: issueCredential({ identity_pda, credential_type: KYC, expiry })
    ├─ 3. Program: check issuer is in allowed_issuers list
    ├─ 4. Program: create CredentialRecord PDA
    ├─ 5. Program: emit CredentialIssued event
    └─ 6. Subject can now prove KYC without re-submitting documents
```

### ZK Verification Flow

```
Verifier requests proof
    │
    ├─ 1. Subject: generate ZK proof off-chain (snarkjs/circom)
    │       Public inputs: identity_pda, credential_type
    │       Private inputs: credential details, private key
    │
    ├─ 2. SDK: verifyZkIdentity({ proof, public_inputs })
    ├─ 3. Program: call alt_bn128 precompile → verify Groth16 proof
    ├─ 4. Program: create ZkVerificationRecord
    └─ 5. Verifier: reads ZkVerificationRecord → confirmed without seeing raw data
```

---

## 9. Security Considerations

| Threat | Mitigation |
|---|---|
| Unauthorized identity registration | PDA seeded by wallet; only wallet owner can sign |
| Credential forgery | Issuer must be in `allowed_issuers` registry config |
| Replay attacks on ZK proofs | Proof includes nonce + timestamp; nullifier checked |
| Registry takeover | Authority is a multisig (Squads); upgrade authority frozen post-audit |
| PDA collision | Seeds include unique wallet + type; bump stored in account |
| Stale credentials | `expires_at` enforced on-chain; indexer flags expired records |
| Agent abuse via delegation | Scope bitmask limits actions; expiry enforced; revocable anytime |
| Private key exposure | ZK proofs never require private key on-chain |

---

## 10. Compliance Considerations

| Requirement | Implementation |
|---|---|
| KYC/AML gating | `CredentialType::KYC` required before certain program interactions |
| GDPR / data minimization | No PII on-chain; only hashes and IPFS CIDs |
| Accredited investor checks | `CredentialType::Accreditation` + expiry enforced |
| Audit trail | All state changes emit on-chain events; immutable log |
| Revocation | Identity and credential revocation propagates immediately on-chain |
| Jurisdiction flags | `metadata_uri` points to off-chain jurisdiction metadata |
| RWA compliance | `allowed_issuers` controls which orgs can issue accreditation credentials |

---

## 11. Integration Points

| Module | Integration |
|---|---|
| **zk-Identity-masks** | ZK proof generation/verification for privacy-preserving identity |
| **solana-kyc-compliance-sdk** | Issues KYC/AML `CredentialRecord` linked to identity |
| **zk-authorship-license** | `IdentityRecord` serves as root DID for authorship proofs |
| **Aurora-zk-cryptography-framework** | Proof circuits for credential verification |
| **solana-governance-policy-engine** | Identity + credential status gates governance participation |
| **GitDigital Financial Core** | KYC credential required for financial operations |
| **ZK-5D-Badge-Authority-app** | Badge issuance uses `CredentialRecord` as source of truth |
| **Proof-of-Contribution-Protocol** | Contribution proofs reference `IdentityRecord` as prover DID |

---

## 12. Documentation

### Events

```rust
#[event] pub struct IdentityRegistered  { pub owner: Pubkey, pub did: String, pub timestamp: i64 }
#[event] pub struct IdentityRevoked     { pub owner: Pubkey, pub timestamp: i64 }
#[event] pub struct CredentialIssued    { pub identity: Pubkey, pub issuer: Pubkey, pub credential_type: CredentialType }
#[event] pub struct CredentialRevoked   { pub identity: Pubkey, pub credential_id: String }
#[event] pub struct DelegationGranted   { pub identity: Pubkey, pub delegate: Pubkey, pub scope: u64 }
#[event] pub struct ZkProofVerified     { pub identity: Pubkey, pub verifier: Pubkey }
```

### Error Codes

```rust
#[error_code]
pub enum IdentityError {
    #[msg("Registry is paused")]           RegistryPaused,
    #[msg("Issuer not authorized")]        UnauthorizedIssuer,
    #[msg("Identity already exists")]      DuplicateIdentity,
    #[msg("Identity is revoked")]          IdentityRevoked,
    #[msg("Credential expired")]           CredentialExpired,
    #[msg("Delegation scope exceeded")]    ScopeViolation,
    #[msg("Invalid ZK proof")]             InvalidProof,
    #[msg("Unauthorized")]                 Unauthorized,
}
```

### Repository Structure

```
solana-identity-registry/
├── programs/solana_identity_registry/
│   └── src/
│       ├── lib.rs
│       ├── instructions/
│       │   ├── register_identity.rs
│       │   ├── update_identity.rs
│       │   ├── revoke_identity.rs
│       │   ├── issue_credential.rs
│       │   ├── revoke_credential.rs
│       │   ├── delegate_authority.rs
│       │   └── verify_zk_identity.rs
│       ├── state/
│       │   ├── identity_record.rs
│       │   ├── credential_record.rs
│       │   ├── delegation_record.rs
│       │   └── registry_config.rs
│       ├── errors.rs
│       └── events.rs
├── sdk/
│   └── src/
│       ├── IdentityRegistrySDK.ts
│       ├── types.ts
│       └── idl.json
├── services/
│   ├── indexer/
│   └── api/
├── schemas/
│   └── postgres/
└── tests/
    ├── anchor/
    └── integration/
```

---

**Status:** Architecture complete. Ready for `anchor build`.
