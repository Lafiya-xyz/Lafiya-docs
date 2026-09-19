# Outage Communication Plan: Patient and Responder Incident Notification

## Overview & Objective

Lafiya is designed for safety-critical emergency healthcare contexts where seconds matter and network connectivity is frequently unstable or absent (such as 2G/EDGE networks, cellular tower power outages, or rural healthcare facilities). Because emergency responders rely on patient cards to ascertain blood group, genotype, severe allergies, and chronic conditions, any system disruption poses real clinical risk.

This runbook outlines the operational plan for communicating system outages, planned maintenance, and service degradations to patients, community health workers (CHWs), and frontline responders across low-connectivity environments.

---

## Low-Bandwidth Communication Channels

Traditional status pages (e.g., heavy JavaScript single-page applications) fail in low-connectivity settings. Lafiya utilizes four low-bandwidth-appropriate communication channels designed to operate under strict bandwidth constraints:

### 1. SMS Broadcasts & Flash Alerts (GSM-7 Standard)

- **Target Audience:** Patients, guardians, and enrolled frontline responders.
- **Constraints:** Must adhere to single-message GSM-7 encoding (maximum 160 characters per segment) to avoid multi-part concatenation failures on basic feature phones.
- **Delivery Mechanism:** Automated trigger via multi-carrier SMS gateway (e.g., Africa's Talking / local telecom aggregators) with carrier redundancy.

### 2. USSD Status Menu (`*384*LAFIYA#`)

- **Target Audience:** Responders and patients using 2G feature phones without data bundles.
- **Constraints:** Zero internet data required; operates over cellular signaling channels. Response payloads must fit standard 182-character USSD screen limits.
- **Functionality:** Responders can enter a patient's short ID or card code to verify basic status or check system availability during web outages.

### 3. Ultra-Lightweight Plaintext Status Endpoint (`/status.txt`)

- **Target Audience:** Web clients, ambulance dispatch terminals, and clinics with constrained bandwidth.
- **Constraints:** Plain text or raw static HTML under 2 KB total size, zero external scripts, zero web fonts, and zero tracking pixels.
- **Delivery Mechanism:** Served directly from geographically distributed edge nodes (Cloudflare/Fastly) with an aggressive 30-second TTL to ensure immediate visibility over 2G networks.

### 4. CHW & Health Facility Broadcast Cascade (WhatsApp / VHF Radio)

- **Target Audience:** Hospital emergency rooms, Primary Health Centres (PHCs), and active CHWs.
- **Constraints:** Relies on local emergency coordinator hierarchy when telecom infrastructure faces regional disruption.
- **Delivery Mechanism:** Hierarchical cascade through designated regional health liaisons via low-data messaging channels and local VHF facility radio nets.

---

## Interaction with Offline Caching & Threat Model

This communication plan explicitly integrates with the architectural principles and threat model documented in [`docs/threat-model.md`](threat-model.md):

### 1. Alignment with STRIDE DoS Mitigations

Under the STRIDE analysis in `docs/threat-model.md`, Denial of Service attacks against the public emergency page are mitigated because:
> *"Standard rate limiting at `lafiya-web`'s edge; QR pages are static/cacheable by design."*

During an outage or edge degradation:

- **Service Worker Local Caching (PWA):** Responders running the mobile web interface retain previously loaded emergency profiles in client-side `CacheStorage` (`stale-while-revalidate` policy).
- **Physical Card Redundancy:** Patients carry a physical card containing both a high-contrast QR code and human-readable emergency essentials (blood group, genotype, emergency contacts) stamped during registration.

### 2. Cryptographic Integrity & Attestation Indicators

When the network is unreachable or degraded:

- **Tamper-Evidence Preserved:** A cached emergency profile includes the SHA-256 hash digest of the emergency subset. The client verifies this locally against the signed record cached from previous sessions.
- **Clear Degradation Badge:** If the Soroban attestation registry cannot be reached to verify the latest on-chain block height, the UI switches from the green **"Verified"** indicator to a high-visibility yellow badge:

```text
[!] OFFLINE MODE: Showing cached record.
Attested on: 2026-06-15 by Dr. A. Bello (PHC Kaduna).
Live blockchain verification currently offline.
```

- **Responder Protocol:** Emergency personnel are trained to treat cached records as valid historical baselines while taking standard clinical precautions for unconfirmed current medications.

---

## Outage Scenarios & Concrete Message Templates

### Scenario 1: Soroban RPC / Attestation Network Degraded

*Condition:* The web app and profile database are fully operational, but the Stellar/Soroban RPC endpoint is unreachable or experiencing block ingestion delays. Attestation verification cannot query the ledger in real time.

#### A. Patient SMS (136 characters)

```text
Lafiya Alert: System maintenance active. Your emergency card QR works normally. Verification badge will show cached status. No action needed.
```

#### B. Responder SMS / Alert (154 characters)

```text
Lafiya Ops: Stellar attestation lookup delayed. Emergency pages remain accessible. Rely on cached verification timestamps and physical patient cards.
```

#### C. Web UI Offline Banner (`lafiya-web`)

```text
[NOTICE] Real-time blockchain attestation verification is temporarily unavailable. All emergency medical data displayed below reflects the latest cryptographically attested record saved on device.
```

---

### Scenario 2: Total Web Edge / Database Unreachable (P1 Critical)

*Condition:* Supabase database cluster or web edge hosting is completely offline due to upstream infrastructure outage. Scanning QR links results in HTTP 500/503 or DNS timeouts.

#### A. Patient SMS (148 characters)

```text
Lafiya Emergency: Web system temporarily down. If seeking care, please present your physical Lafiya card or tell responders your emergency contact.
```

#### B. Responder SMS (158 characters)

```text
CRITICAL: Lafiya emergency portal is temporarily unreachable. Dial *384*LAFIYA# on mobile to query emergency card numbers, or refer to physical patient card.
```

#### C. Lightweight Status Endpoint (`/status.txt`)

```text
LAFIYA EMERGENCY STATUS: MAJOR OUTAGE
TIMESTAMP: 2026-09-19T10:00:00Z
STATUS: Public QR resolution currently unavailable due to database maintenance.
ACTIONS FOR RESPONDERS:
1. Use USSD backup: Dial *384*LAFIYA# (accessible across all GSM networks).
2. Inspect physical card for human-readable blood group, genotype, and contact.
3. Call Emergency Desk: +234-800-LAFIYA-HELP.
Next update in 30 minutes.
```

---

### Scenario 3: Scheduled Maintenance Window

*Condition:* Planned 60-minute maintenance window announced 48 hours in advance for smart contract migrations or core schema updates.

#### A. Advance Notification SMS (48 Hours Prior — 149 characters)

```text
Lafiya Notice: Scheduled maintenance on Sunday 02:00-03:00 WAT. Emergency cards will operate in offline cached mode during this hour. Thank you.
```

#### B. Active Maintenance SMS (128 characters)

```text
Lafiya Ops: Scheduled maintenance is now underway until 03:00 WAT. Cached cards and physical cards remain fully accessible.
```

---

## Operational Runbook: Incident Escalation & Response Flow

```text
+-------------------+      +-------------------------+      +--------------------------+
| 1. Outage Detect  | ---> | 2. Triage & Severity    | ---> | 3. Channel Activation    |
| Edge monitor ping |      | P1: Total Service Down  |      | Dispatch SMS within 15m  |
| 3 consecutive 5xx |      | P2: Soroban RPC Failure |      | Update /status.txt       |
+-------------------+      +-------------------------+      +--------------------------+
                                                                         |
+-------------------+      +-------------------------+                   v
| 6. Post-Mortem    | <--- | 5. Resolution Broadcast | <--- +--------------------------+
| Published within  |      | "All systems restored"  |      | 4. Hourly Updates        |
| 72 hours          |      | SMS & USSD banner clear |      | Broadcast until recovery |
+-------------------+      +-------------------------+      +--------------------------+
```

### Response Time Targets (SLAs)

- **Initial Notification:** Dispatched via SMS and `/status.txt` within **15 minutes** of confirmed P1 incident.
- **Cadence of Updates:** Status updates broadcast every **45 minutes** until resolution.
- **All-Clear Notification:** Dispatched within **10 minutes** of service restoration.
- **Post-Mortem Publication:** Documented in repository within **72 hours** following standard blameless post-mortem guidelines.

---

## Audit & Governance Alignment

- **Digital Public Goods (DPG) Standard:** Adheres to Indicator 6 (Data Privacy) and Indicator 7 (Adherence to Standards & Resilient Architecture) by ensuring patient safety is maintained during digital downtime.
- **Nigeria Data Protection Act (NDPA 2023):** Ensures availability and resilience of systems processing personal data without leaking sensitive patient diagnostic histories over unencrypted channels.
