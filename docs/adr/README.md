# Architecture Decision Records

ADRs capture *why* a significant, hard-to-reverse decision was made — not
what the code does. Use this format for a new ADR:

```markdown
# ADR NNNN: Title

## Status
Proposed | Accepted | Superseded by ADR NNNN

## Context
What problem forced this decision?

## Decision
What was decided?

## Consequences
What does this make easier or harder, and what did we give up?
```

Number ADRs sequentially and never renumber or delete one — supersede it
instead.

| ADR | Title |
| --- | --- |
| [0001](0001-why-stellar.md) | Why Stellar / Soroban |
| [0002](0002-off-chain-data-on-chain-attestation.md) | Off-chain data, on-chain attestation only |
| [0003](0003-attester-allowlist-governance.md) | Attester allowlist governed by committee |
| [0004](0004-nextjs-supabase-stack.md) | Next.js + Supabase for `lafiya-web` |
| [0005](0005-usdc-for-chw-incentives.md) | USDC on Stellar for CHW incentives |
| [0006](0006-verification-key-management.md) | Attester key management |
| [0007](0007-patient-account-recovery.md) | Patient account-recovery mechanism |
| [0008](0008-guardian-dependent-account-model.md) | Guardian-dependent account model |
| [0010](0010-contract-upgradeability-strategy.md) | Contract upgradeability strategy for `lafiya-contract` |
