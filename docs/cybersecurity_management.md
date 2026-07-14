# Cybersecurity Management Plan (CSMP)

## Table of Contents

- [1. Security Philosophy](#1-security-philosophy)
- [2. Network Segmentation](#2-network-segmentation-zones--conduits)
  - [2.1 Dual-NIC Isolation](#21-dual-nic-isolation)
  - [2.2 Data Loss Prevention (DLP)](#22-data-loss-prevention-dlp)
- [3. The Semantic Gatekeeper](#3-the-semantic-gatekeeper)
- [4. Least Privilege & Access Control](#4-least-privilege--access-control)
  - [4.1 Identity & Access Management (IAM)](#41-identity--access-management-iam)
- [5. Incident Response](#5-incident-response)
- [6. Data Governance & Classification](#6-data-governance--classification)
  - [6.1 Data Sensitivity Levels](#61-data-sensitivity-levels)
  - [6.2 Cross-Agent Handling Rules (Data Downgrading)](#62-cross-agent-handling-rules-data-downgrading)
- [7. Clock Synchronization & Forensic Alignment](#7-clock-synchronization--forensic-alignment)
  - [7.1 Tiered Synchronization Strategy](#71-tiered-synchronization-strategy)
  - [7.2 The Edge Gateway as "Time Bridge"](#72-the-edge-gateway-as-time-bridge)
  - [7.3 Security & Availability](#73-security--availability)

## 1. Security Philosophy
OverSight ICS adopts a **Zero-Trust Industrial Architecture**. The system assumes that any network-connected component, 
including the AI Reasoning Agent, could be compromised. It enforces strict access controls and data classification
to prevent unauthorized access to sensitive data.

## 2. Network Segmentation (Zones & Conduits)
In accordance with IEC 62443-3-3 for network segmentation, the architecture is divided into distinct **Security Level** 
(SL) targets for each zone. 

| Zone ID | Description       | Target Security Level (SL) | Primary Controls                                    |
|:--------|:------------------|:---------------------------|:----------------------------------------------------|
| **Z1**  | **Business (L4)** | SL-1 (Baseline)            | Standard Firewall, DLP.                             |
| **Z2**  | **IDMZ (L3.5)**   | **SL-3 (High)**            | Dual-NIC Isolation, Dual-Auth JWT, WORM Audit Logs. |
| **Z3**  | **Control (L2)**  | **SL-3 (High)**            | PTP Sync, Intent-Signing, Semantic Gatekeeper.      |
| **Z4**  | **Process (L1)**  | SL-2 (Functional)          | Hardware Watchdog, Modbus Allow-listing.            |

### 2.1 Dual-NIC Isolation
The Edge Gateway is equipped with two physical Network Interface Cards (NICs):
- **NIC 1 (Northbound)**: Connected to the IDMZ for MQTT/Dashboard traffic.
- **NIC 2 (Southbound)**: Connected to the Level 2 Control Network via a non-routable private subnet.
- **Routing**: IP Forwarding is disabled at the OS level to prevent bridge-traffic between zones.

### 2.2 Data Loss Prevention (DLP)
- **Unidirectional Command Logic**: The Southbound Driver is a "Stateful Proxy." It only accepts valid 'Intents' from 
  the Edge Gateway and is physically incapable of initiating outbound requests to the Enterprise Zone.
- **Northbound Egress Filtering**: The Northbound NIC is restricted via `iptables` to permit ONLY MQTTS (Port 8883) 
  and HTTPS (Port 443). All other outbound traffic is dropped.
- **Southbound Inbound Filtering**: The Southbound NIC permits only established industrial protocols from a hard-coded 
  "Allow-list" of Level 2 Controller IP addresses.
- **Encryption in Transit**: 
    - **Northbound**: All MQTT traffic uses MQTTS with TLS 1.3 certificates.
    - **Southbound**: Industrial traffic is encapsulated in a secure tunnel (e.g., WireGuard) to provide TLS 1.3 
      equivalent protection for legacy protocols.
- **Deep Packet Inspection (DPI)**: The Southbound Driver validates the structure of the industrial packets, not just
  the port numbers.

## 3. The Semantic Gatekeeper
The Southbound Driver enforces all write-commands against a hard-coded allowlist:

- **Logic Interlocking**: The driver maintains a hard-coded "Allowlist" of Modbus/OPC-UA registers.
- **Validation Schema**: Every intent is checked for Range and Rate-of-Change limits
    - **Range Checks**: Is the value within the physical limits of the machine?
    - **Rate-of-Change Checks**: Is the AI trying to ramp a motor faster than the mechanical spec allows?
- **Rejection Protocol**: Any intent failing validation is dropped, logged as a "Security Conflict," and triggers a 
  Critical Alarm.

## 4. Least Privilege & Access Control
- **Service Accounts**: The LangGraph Agent operates under a "Read-Only" service account for the Unified Namespace.
- **Write Authority**: Write access is restricted to the Southbound Driver process, which requires a cryptographic 
  handshake with the Edge Gateway.
- **OIDC/SSO**: The dashboard is secured with OpenID Connect (OIDC) with MFA requirements.
- **MFA Requirements**: MFA is mandatory for high-priority actions and must be verified.

### 4.1 Identity & Access Management (IAM)
- **OIDC**: The dashboard is secured with OpenID Connect (OIDC) for federated identity with the plant's existing 
  identity provider.
- **Single Sign-On (SSO)**: Users authenticate via an SSO provider, which enforces MFA and role-based
  access control. 
  - If a user leaves the company and their corporate account is deactivated, their access to the dashboard is 
    automatically revoked.
- **Role-Based Access Control (RBAC)**: 
  - **Viewer**: Can see the $Z$-Score trends but cannot acknowledge alarms.
  - **Operator**: Can acknowledge alarms and validate AI failover intents.
  - **Admin/Engineer**: Can modify deadbands, delay times, and $Z$-Score thresholds.
- **Multi-Factor Authentication (MFA)**: MFA is mandatory for all **High-Risk** actions (e.g., modifying the Forbidden 
   Register List). 
   - **Requirement**: MFA status is verified at **Gate 1** of the verification pipeline.
    - **Validation**: This requirement must be formally verified during **FAT-03 (ST-11/ST-12)** to ensure no bypass 
      exists.
- **Audit Trail**: All actions taken on the dashboard are logged for auditing purposes.
- **Security Alerts**: Any suspicious activity is logged and triggers a Critical Alarm.
- **Security Incidents**: Any detected breach or anomalous AI behavior is logged and triggers a Critical Alarm.
- **Security Notifications**: All security-related events are sent to the user's email address.
- **Security Dashboard**: The dashboard provides real-time insights into security-related events.

## 5. Incident Response & Air-Gap Fallback
In the event of a detected breach or anomalous AI behavior:

1. **Incident Protocol**: System severs NIC 1 connection and reverts to local **Safe-State** logic.
2. **Orphan Mode**: If PTP sync is lost, the system enters **Orphan Mode**.
3. **Notification**: An HMI critical alarm must be issued immediately upon entering Orphan Mode.
4. **Downgrade**: If sync is not restored within 24 hours, the AI Agent downgrades to **`MONITOR_ONLY`** mode.

## 6. Data Governance & Classification
To guarantee cross-agent data integrity and prevent information leakage across security zones, OverSight ICS enforces 
a four-tier data classification model.

### 6.1 Data Sensitivity Levels
Every data point within the system is assigned a sensitivity level which dictates its storage, visibility, and 
transmission rules.

| Level  | Classification   | Description                                                                  | Access Control                                                                             |
|:-------|:-----------------|:-----------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------|
| **L4** | **Restricted**   | PII, Credentials, API Keys, and raw manufacturer Intellectual Property (IP). | Restricted to **Agent Consciousness** layer; never transmitted to UI or Southbound Driver. |
| **L3** | **Confidential** | Internal $Z$-Score logic, Reasoning Traces, and Alarm Rationalization logic. | Accessible by authorized **Operators** via HMI; encrypted at rest.                         |
| **L2** | **Internal**     | Real-time telemetry, asset health states, and UNS topics.                    | Broadly available within the **Industrial DMZ** for monitoring and history.                |
| **L1** | **Public**       | System uptime and high-level dashboard heartbeat.                            | Available for general system health status.                                                |

### 6.2 Cross-Agent Handling Rules (Data Downgrading)
As data moves across security conduits, it must be "downgraded" to the lowest required classification for the target
zone to maintain the **Least Privilege** principle.

- **Intent Filtering (L4 → L2)**: When the Reasoning Agent sends an intent to the Southbound Driver, it must strip 
  away all proprietary reasoning logic and L4 metadata. Only the raw register command (L2) is transmitted to the
  Control Zone.
- **Trace Sanitization (L4/L3 → L3)**: Before a Reasoning Trace is transmitted to the Next.js HMI, the system performs 
  an automated sanitization pass. Any L4 data (such as specific database file paths, internal IP addresses, or 
  system credentials) is redacted to ensure the operator only sees L3 clinical logic.
- **Encryption at Transit**: All L3 and L4 data moving between agents must be encapsulated in TLS 1.3 or higher to 
  prevent man-in-the-middle (MITM) sniffing within the IDMZ.

### 7. Clock Synchronization & Forensic Alignment
The system uses the **Precision Time Protocol** to synchronize the clocks of all agents within the Industrial DMZ. This 
guarantees that all logs, reasoning traces, and telemetry data are timestamped accurately for forensic analysis and 
correlation during incident response. The Edge Gateway serves as the PTP Grandmaster clock. It provides a consistent 
time source for the Southbound Driver and any connected Level 2 controllers to prevent **Time-Shift** attacks and 
ensures non-repudiation (Denial of Service) of audit logs.

### 7.1 Tiered Synchronization Strategy
| Network Zone            | Protocol            | Accuracy | Purpose                                                            |
|:------------------------|:--------------------|:---------|:-------------------------------------------------------------------|
| **Enterprise (L4)**     | **NTP**             | ±50ms    | Global dashboard and user event alignment.                         |
| **Industrial DMZ (L3)** | **NTP**             | ±10ms    | Syncing the Edge Gateway to the Corporate Stratum 2/3 source.      |
| **Control Zone (L2)**   | **PTP (IEEE 1588)** | **<1μs** | Microsecond-precision for Z-Score telemetry and hardware commands. |

### 7.2 The Edge Gateway as "Time Bridge"
The Edge Gateway is configured as a **PTP Grandmaster** for the Southbound network (NIC 2). 
- It pulls UTC time from the Northbound NTP source and translates it to a high-precision hardware clock on the 
  Southbound NIC.
- **Hardware Timestamping**: Where supported by the NIC hardware, PTP packets are timestamped at the physical layer to 
  eliminate OS-level jitter.

### 7.3 Security & Availability
- **Time-Drift Monitoring**: The system monitors the offset between the AI Reasoning Agent and the Southbound Driver. 
  If the clock offset exceeds **100 ms**, the system triggers a `SECURITY_ALARM` and enters a 
  "Pessimistic Safety State," as forensic correlation can no longer be guaranteed.
- **Traffic Filtering**: Firewalls are configured to permit only authenticated time traffic:
    - **NTP**: UDP Port 123 (Inbound on NIC 1).
    - **PTP**: UDP Ports 319, 320 (Outbound on NIC 2).
- **Air-Gap Fallback**: In the event of a corporate network loss, the Edge Gateway maintains a local "Orphan Clock" to 
  ensure internal plant consistency until the Stratum source is restored.
  - **Maximum Allowable Drift**: The system is permitted to run in Orphan Mode for a maximum of **24 hours**. 
  - **Degraded State**: If the external Stratum source is not restored within 24 hours, the AI agent must automatically 
    downgrade to **"Monitor Only"** mode. All active intents are suspended to prevent misalignment between the AI's 
    logic and the field asset's execution.

### 7.4 Air-Gap Fallback (Orphan Mode)
In the event of a PTP source loss (Air-Gap), the system enters **Orphan Mode**.

- **Notification**: Immediately upon entering Orphan Mode, the system triggers a **`CRITICAL ALARM: ORPHAN_MODE_ACTIVE`**
  on the HMI and pushes a notification to the level 4 SIEM.
- **Degradation**: After 24 hours in Orphan Mode without re-sync, the system automatically downgrades to **`MONITOR_ONLY`**
  mode.

## 8. The Verification Pipeline
### 8.1 Inter-Agent Compliance Verification
To prevent **Agent Hallucination** or **Unauthorized Escalation**, every message moving between agents must undergo a 
**Handshake Verification**.

- **Protocol**: Standardized on **JSON over TLS 1.3** for all inter-agent and API communication.

*To ensure consistency across the Edge Gateway, **JSON** is the mandated protocol for all internal and external message 
exchanges.*

### 8.2 Multi-Stage Verification Table
| Stage  | Gate      | Verification Logic                    | Success Response | Error Code (Failure)            |
|:-------|:----------|:--------------------------------------|:-----------------|:--------------------------------|
| Gate 1 | Identity  | MFA Token & Role-Based Access Control | `200 OK`         | `401 MFA_REQUIRED`              |
| Gate 2 | Integrity | TPM Signature & Justification Hash    | `200 OK`         | `401 SECURITY_TAMPER`           |
| Gate 3 | Semantic  | Forbidden Register List Check         | `200 OK`         | `403 FORBIDDEN_SAFETY_REGISTER` |

### 8.3 The "Two-Key" Protocol
For any critical command (Level 4 → Level 2), the system requires two independent signatures:
1. **The Proposer (Reasoning Agent)**: Signs the intent with its Service Account JWT.
2. **The Validator (Semantic Gatekeeper)**: Re-calculates the $Z$-Score independently. If the Gatekeeper's math does 
   not match the Agent's math, the command is intercepted as an "Incoherent Intent."

### 8.4 Protocol Buffers & Schema Enforcement
Inter-agent communication is strictly enforced via **Protobuf**. 
- Any messages containing fields not defined in `UNS_SCHEMA.md` are dropped immediately.
- This prevents "Payload Injection" where an attacker might try to smuggle extra commands inside a valid-looking 
  JSON block.

### 8.4.1 Intent Verification Message
The Intent Verification message is used to validate the AI-generated intents.

```protobuf
syntax = "proto3";

message IntentVerification {
  string trace_id = 1;      // UUID from LangGraph
  int32 asset_id = 2;       // Target PLC Register
  float proposed_value = 3; // The Z-Score driven intent
  bool sis_override = 4;    // Safety check bit
}
```

## 9. Continuous Compliance Monitoring (CCM)
Oversight ICS employs an automated **Watcher** pattern to make sure the system never drifts from its security and 
performance baselines. 

### 9.1 Real-Time Policy Testing
The system runs automated **Synthetic Heartbeats** every 60 seconds to test the following:
- **Latent Intent Test**: The Watcher injects a non-executable **Ping Intent** to measure the round-trip time from the 
  Reasoning Agent to the Southbound Driver. If RTT exceeds **200 ms**, a performance compliance alert is raised.
- **Clock Integrity Test**: The Watcher compares the PTP Hardware Clock against the System OS Clock. 
- **Cgroup Enforcement**: A background script verifies that the Docker `cpu_period` and `cpu_quota` remain locked at 
  the defined 40% limit.

### 9.2 Automated Regression Testing (The "Chaos Agent")
In the staging environment, a **Chaos Agent** periodically attempts to send **Malformed Protobufs** or 
**Out-of-Bounds Set Points** to the Semantic Gatekeeper.
- **Compliance Goal**: 100% rejection rate for non-compliant payloads.
- **Logging**: Every rejection is logged in the SIEM to verify that the Inter-Agent Checkpoints are active and haven't 
  been bypassed by a software update.

### 10 Physical & Personnel Security (IEC 62443-2-1)
### 10.1 Physical Port Security
- **Hardening**: Unused physical ports (USB, RJ45, Serial) on the Edge Gateway IPC are physically disabled or locked.
- **Media Protection**: The use of removable media (USB drives) for logic updates is strictly prohibited. All updates 
  must flow through the IDMZ staging volume.

### 10.2 Patch & Update Management
- **Security Patching**: OS-level security patches are applied monthly after validation in the Virtual Twin environment.
- **AI Model Updates**: Changes to model weights or RAG vector embeddings require a full "Logic Drift Assessment" before 
  deployment.
- **Decommissioning**: Failed hardware must be sanitized (7-pass wipe) before leaving the facility to protect 
  proprietary RAG data and network topologies.

### 10.3 Decommissioning & Media Santization
In accordance with **NIST 800-88 Guidelines for Media Sanitization**, all decommissioned hardware must undergo a 
**Cryptographic Erasure (CE)** to remove all proprietary data followed by physical destruction if CE cannot be verified.
Legacy `7-pass wipe` procedures are not permitted for SSD/Flash storage.

## 11. Hardware-Backed Intent Signing (TPM 2.0)
To prevent "Man-in-the-Middle" or "Injection" attacks at the Edge Gateway OS level, the system uses the hardware 
**Trusted Platform Module (TPM 2.0)** to secure the `justification_hash`.

- **TPM 2.0**: Private keys for signing intents are stored in the TPM chip.
- **Private Key Storage**: The private key used to sign the `intent_outbox` entries never leaves the IPC’s TPM chip.
- **Key Versioning Grace Period**: The Southbound Driver maintains the *previous* public key for 60 seconds to allow
  pending intents in the `intent_outbox` to clear.
- **Verification**: The Southbound Driver performs a `Signature_Verify` using the TPM public key before converting any 
  JSON payload into a PLC command.
- **Key Rotation**: Mandatory rotation every 180 days
  - **SOP Requirement**: All rotation procedures must follow 
    [SOP-TPM-KEY-01: TPM Key Management & Air-Gapped Rotation](/docs/sop/SOP-SEC-003_TPM-Key-Rotation.md), which defines the use of a physical "Key Loading 
    Device" to maintain the air-gap.
- **Sanitization**: All decommissioned hardware must undergo **NIST 800-88 Cyptographic Erasure**.

## 12. Secret Rotation Policy 
TODO
- **Secret Rotation**: All secrets (e.g., API Keys, Service Accounts) are rotated every 90 days.
- **Secret Rotation SOP**: All secret rotation must follow 
  [SOP-SECRET-01: Secret Rotation & Key Management](/docs/sop/SOP-09-Secret-Rotation.md), which defines the use of a 
  secure vault and audit logging for all rotation events.

## Revision History
| Version | Date       | Author    | Description of Change                       | Approved By |
|:--------|:-----------|:----------|:--------------------------------------------|:------------|
| 1.0     | 2026-04-12 | armummert | Initial Baseline for Version 1.0 Release.   | armummert   |
| 1.1     | 2026-04-27 | armummert | Added SOP-TPM-KEY reference, SL's, SL table | armummert   |
