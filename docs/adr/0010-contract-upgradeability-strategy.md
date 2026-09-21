# ADR 0010: Contract upgradeability strategy for lafiya-contract

## Status

Proposed

## Context

Lafiya anchors patient health profile attestations and attester allowlists on-chain via Soroban smart contracts (`lafiya-contract`). As the protocol evolves from testnet through mainnet deployment ([roadmap.md](../roadmap.md)), contract improvements, security patches, and data model refinements will inevitably be required.

In an emergency medical attestation system, contract upgrades carry unique systemic risks:

1. **Attestation Integrity**: On-chain cryptographic attestations verifying emergency health profiles must not be invalidated, corrupted, or desynchronized during code transitions.
2. **Client Endpoint Stability**: Hospital emergency responders, Community Health Workers (CHWs), and third-party healthcare providers interact with deployed contract IDs; breaking or rotating contract addresses introduces severe operational friction and downtime risk during critical emergency lookups.
3. **Governance and Trust**: Unrestricted or centralized upgrade powers violate the decentralized trust model and introduce a single point of failure (an adverse administrator could tamper with attester validation or attestation registries).
4. **Storage Evolution**: Soroban contracts use typed persistent and instance storage. Introducing new features or migrating data representations requires a deterministic data migration path that guarantees forward and backward schema compatibility.

This ADR evaluates three alternative upgrade architectures and establishes the formal upgrade strategy for `lafiya-contract`.

---

## Considered Strategies

### 1. Native Soroban In-Place WASM Upgrades (`env.deployer().update_current_contract_wasm`)

Soroban provides built-in protocol-level contract upgradeability through `env.deployer().update_current_contract_wasm(new_wasm_hash)`. This replaces the executable bytecode of the contract while preserving the contract address, instance storage, and persistent storage entries intact.

- **Pros**:
  - Zero address churn: The contract ID remains constant across upgrades, preserving client configurations across `lafiya-web`, CHW interfaces, and emergency lookup portals.
  - Zero storage replication cost: Existing attestation records and allowlist entries remain directly accessible at zero migration gas cost.
  - Lowest execution overhead: No proxy redirection hops or inter-contract call overhead.
- **Cons / Risks**:
  - Requires strict storage layout discipline to avoid deserialization panics on preexisting storage entries.
  - Administrative upgrade authority must be hardened against compromise.

### 2. Router / Registry Proxy Pattern

A lightweight router contract maintains a registry pointer to active implementation contracts, forwarding client calls to the current implementation.

- **Pros**:
  - Modular separation between registry routing and domain logic.
  - Enables gradual rollout or routing of specific sub-modules to different contracts.
- **Cons / Risks**:
  - Redundant on Soroban: Reinvents mechanics natively supported by Soroban's WASM replacement API.
  - Increased invocation cost and CPU resource meter consumption on every cross-contract call.
  - Does not solve storage migration: Each new implementation contract has distinct contract storage, requiring either complex cross-contract storage access or costly bulk record copying.

### 3. Fully Immutable Contracts with Deprecation & Client-Side Migration

Contracts are deployed without upgrade mechanisms. Bug fixes or feature iterations deploy brand new contract instances; the governance committee deprecates the old contract and directs clients to the new contract address.

- **Pros**:
  - Absolute code immutability: Verifiers know bytecode can never mutate after deployment.
  - Eliminates administrative upgrade privilege escalation vectors.
- **Cons / Risks**:
  - Extremely disruptive for emergency responders: requires all clients, mobile apps, and hospital integrations to coordinate address migrations simultaneously.
  - On-chain attestation migration cost: Transferring thousands of existing health profile attestations from old contracts to new contracts incurs massive transaction fees and risks ledger inconsistency during transition windows.
  - Unacceptable in production emergency health infrastructure where zero-downtime availability is critical.

---

## Decision

Adopt **Native Soroban In-Place WASM Upgrades (`env.deployer().update_current_contract_wasm`)** governed strictly by the **Governance Committee Multisig with a Mandatory Timelock**:

1. **Native In-Place Code Replacement**:
   - `lafiya-contract` will expose an explicit `upgrade(new_wasm_hash: BytesN<32>)` entry point.
   - The contract invokes `env.deployer().update_current_contract_wasm(new_wasm_hash)` to update its execution code while maintaining its canonical contract ID and storage.

2. **Multi-Signature Governance Enforcement**:
   - Upgrade authority is restricted exclusively to the Governance Committee ([ADR 0003](0003-attester-allowlist-governance.md)) via multi-signature authorization (`M-of-N` consensus).
   - No single individual, developer, or automated CI credential holds unilateral upgrade authority.

3. **Timelock Delay for Non-Emergency Upgrades**:
   - Upgrades follow a two-step pattern: `propose_upgrade(wasm_hash)` followed by an enforced delay window (minimum 48 hours) before `execute_upgrade()` can be triggered.
   - This delay allows auditors, community stakeholders, and healthcare partners to verify the proposed WASM hash against reproducible open-source source code before activation.
   - An emergency fast-track path requires unanimous committee consensus (N-of-N) and is restricted to addressing audited, critical security vulnerabilities.

---

## Consequences & Data Migration Plan

### Positive Consequences

- **Zero Client Interruption**: Emergency lookup endpoints, mobile apps, and hospital systems maintain continuous integration using the fixed contract ID.
- **Gas & Rent Efficiency**: Avoids mass-transfer transaction fees for moving attestation records across contracts.
- **Clean Audit Trail**: Upgrade proposals, approvals, and WASM hashes are immutably emitted as contract events on the Stellar ledger.

### Data Migration & Schema Evolution Strategy

1. **Additive Schema Rule**: Existing `DataKey` enum variants and stored struct fields must never be removed, reordered, or redefined in backward-incompatible ways.
2. **Optional Field Extension**: New data fields added to stored structs must use `Option<T>` with default fallbacks for records created under earlier versions.
3. **Storage Version Keying**: Storage structures must maintain an explicit schema version counter (e.g. `SCHEMA_VERSION: u32`). When reading historical records, the contract verifies the schema version and conditionally adapts the record to the current internal memory representation.
4. **Automated Migration Invariants in CI**: Any PR introducing a contract upgrade must include automated regression tests in `lafiya-contract` that instantiate a state snapshot from previous contract releases and verify that all read/write paths operate without errors under the new WASM binary.
