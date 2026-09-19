# Lafiya Threat Model

## Assets

- **Emergency subset data** (blood group, genotype, allergies, medications,
  conditions) — sensitive health data, but designed to be shown to an
  unauthenticated responder
- **Full patient profile** — sensitive health data, authenticated access only
- **Attestation private keys** (health worker) — compromise lets an attacker
  forge verifications
- **CHW payout funds** (USDC) — theft or double-claiming drains the incentive
  pool
- **Attester allowlist** — unauthorized additions let unverified people issue
  attestations

## Actors

| Actor | Trust level | Can do |
| --- | --- | --- |
| Patient / guardian | Owns their own data | Edit their profile, choose what's public, revoke consent |
| Health worker (attester) | Allowlisted, semi-trusted | Attest to a record's accuracy on-chain |
| Responder | Untrusted, anonymous | Read the public emergency page only |
| CHW | Semi-trusted, paid | Register patients, trigger attestation, receive payout |
| Governance / committee | Trusted | Manage attester allowlist, resolve disputes |
| Attacker | Untrusted, adversarial | Everything else |

## Trust boundaries

1. **Patient device ↔ `lafiya-web`** — standard web auth boundary (session,
   Row-Level Security in Supabase)
2. **`lafiya-web` ↔ Soroban** — service account boundary; only
   `lafiya-web`'s backend (or a designated relay) can submit attestations,
   using an allowlisted attester identity
3. **Public emergency page ↔ anyone** — no authentication; this is the point
   of the page, so it's the boundary that matters most

## STRIDE analysis (selected)

| Threat | Example | Mitigation |
| --- | --- | --- |
| **Spoofing** | Attacker impersonates a health worker to submit a false attestation | Attester allowlist enforced on-chain; attester identity is a Stellar keypair, not a password |
| **Tampering** | Attacker edits the emergency subset after attestation | Any edit invalidates the hash; the public page shows "unverified" until re-attested |
| **Repudiation** | Attester denies having verified a record | On-chain attestation is a public, timestamped, non-repudiable record |
| **Information disclosure** | Full patient profile leaks via the public page | Emergency subset and full profile are architecturally separate; the public page code path never has access to full-profile fields |
| **Denial of service** | Attacker floods the public page endpoint | Standard rate limiting at `lafiya-web`'s edge; QR pages are static/cacheable by design (see [outage communication plan](outage-communication-plan.md)) |
| **Elevation of privilege** | CHW account is used to add itself to the attester allowlist | Allowlist changes require governance/committee action, not CHW-level credentials |

## Audit-readiness

Every entry above is tracked in the
[audit-readiness checklist](audit-readiness-checklist.md) — each asset and
STRIDE threat is marked mitigated-with-evidence or open-with-owner-and-date
for funder/auditor consumption.

## Residual risk

- **Coerced or bribed attester**: an allowlisted health worker who is bribed
  to attest to false data is not detected by the cryptography — this is a
  governance and dispute-resolution problem, not a technical one. See the
  root README's [Attestation & Trust Model](../README.md#attestation--trust-model)
  notes and the equivalent process to be built in `lafiya-contract`.
- **Lost patient credentials**: account recovery mechanism is decided in
  [ADR 0007](adr/0007-patient-account-recovery.md) (recovery contacts,
  guardian-assisted recovery for dependents, and governance-mediated
  recovery as a fallback); implementation is pending `lafiya-web`.
- **Physical coercion of a patient at the scene**: out of scope for a
  technical threat model; noted for completeness.
