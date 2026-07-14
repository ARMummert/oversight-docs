# Industrial Standards & Regulatory Compliance

## Revision History
| Version | Date       | Author       | Description of Change                                           | Approved By |
|:--------|:-----------|:-------------|:----------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | [User]       | Initial Baseline for Version 1.0 Release.                       | [Manager]   |
| 1.1     | 2026-04-17 | OverSight AI | Added ISO 9001 and IEC 62443-4-1 Secure Lifecycle requirements. | [User]      |

---

This document outlines the international engineering standards and regulatory frameworks governing the **OverSight ICS** architecture. Adherence to these standards ensures that the AI-driven reliability layer remains deterministic, secure, and physically safe within critical infrastructure environments.

## Controls Engineering Standards Matrix

| Standard                  | Focus Area              | OverSight Implementation                                                                                                                                                             |
|:--------------------------|:------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **ISA-95**                | Purdue Model            | Establishes the Level 3.5 IDMZ and uses a multi-broker UNS to logically separate Level 3 AI logic from Level 2 field asset control.                                                |
| **IEC 62443**             | Cybersecurity           | Implements Zones & Conduits via Dual-NIC hardware isolation; Northbound Ingest enforces protocol termination at the IDMZ boundary.                                                   |
| **IEC 62443-2-4**         | Service Providers       | Mandatory for managed service deployments. Defines integration requirements between Oversight ICS and site-specific security policies.                                               |
| **IEC 62443-4-1**         | Secure Lifecycle        | Implemented via GitHub Actions CI/CD. Uses `trivy` for dependency scanning and `semgrep` for SAST. See [CI/CD Pipeline](/docs/cicd_pipeline.md)                                      |
| **IEC 61511**             | Functional Safety       | Maintains strict **SIS Decoupling**; AI reliability logic is physically and logically independent of safety logic solvers.                                                           |
| **IEC 61508**             | System Integrity        | Enforces **Cgroup Resource Isolation** and binary schemas to prevent software systematic failures in the IDMZ gateway.                                                               |
| **ISA-18.2**              | Alarm Management        | Uses **Apache Flink** for real-time $Z$-Score Rationalization, suppressing nuisance alerts by identifying "Logic Drift" before they trigger high-priority alarms.                    |
| **ISA-101**               | HMI Philosophy          | Uses a High-Performance (Gray Scale) Dashboard to maximize situational awareness during AI-orchestrated failovers.                                                                   |
| **ISO 13849 / IEC 62061** | Machinery Safety        | Ensures PIL/SIL levels are maintained during AI-Assisted failovers of rotating equipment.                                                                                            |
| **ISO 23247**             | Digital Twins           | Uses a Unified Namespace (UNS) across Kafka (History) and MQTT (State) to decouple AI reasoning from specific register addresses.                                                    |
| **ISO 42001**             | AI Management           | Governed by Inter-Agent Verification Checkpoints and RAG-grounded logs to ensure transparency in LLM-based RCA.                                                                      |
| **ISO 27001**             | Information Security    | Implements ** 90-Day Access Reviews** and **WORM-based Audit Logging** for forensic integrity.                                                                                       |
| **ISO 13374**             | Condition Monitoring    | Follows standard processing blocks: Northbound Ingest for Data Acquisition, Flink for State Detection, and LangGraph for Advisory Generation.                                        |
| **ISO 9001**              | Quality Management      | Governed by [Quality Management System](/docs/quality_management_system.md). Defines documented procedures for **Change Control**, **Incident Response**, and **Management Reviews** |
| **NIST 800-82**           | OT Security             | Enforces **Least Privilege** via a **Semantic Gatekeeper** that validates RabbitMQ command intents against the current Sparkplug B reported state.                                   |
| **NIST AI RMF**           | AI Trustworthiness      | Employs $Z$-Score Grounding via Flink and RAG-sourced citations to mitigate hallucinations in diagnostics.                                                                           |
| **IEEE 7000**             | Ethical Design          | Guarantees **Algorithmic Transparency** by logging full LangGraph reasoning traces for every action.                                                                                 |
| **NERC CIP**              | Utilities Grid          | Meets CIP-007 (Ports/Services) and CIP-010 (Configuration Change Management) via the lockdown of the IDMZ Edge Gateway.                                                              |
| **GDPR / CCPA**           | Data Privacy            | Minimizes PII processing. Operator ID data is pseudonymized in audit logs. Authoritative OIDC records remain with the Corporate Identity Provider.                                   |
| **EPA 40 CFR**            | Environmental Standards | Prevents discharge violations by correlating real-time effluent telemetry in Kafka with hardware interlocks on field asset valves.                                                 |
| **IEEE 2600**             | Message Broker          | Allows for reliable interoperability between the Northbound (Kafka) and Southbound (RabbitMQ) layers of the UNS.                                                                     |
| **ISO/IEC 20922**         | MQTT Protocol           | Ensures reliable state management using **Sparkplug B** payloads for hardware-agnostic interoperability across the UNS.                                                              |
---

## Technical Implementations

### 1. ISA-95 & The Purdue Model
OverSight sits at **Level 3.5 (Industrial DMZ)**, while the Southbound Driver creates a secure connection to 
Level 0–2 for field instrumentation and control hardware (PLCs, VFDs). This ensures that the "Brain" of the system is 
never directly exposed to the field level. 

### 2. IEC 62443: Cybersecurity & Semantic Gatekeeping (Security for IACS)
Oversight adheres to the **Zones & Conduits** security model.
- **Zones**: Physical hardware is isolated in a "Trusted Zone" (IDMZ). The trusted zone is the Level 1-2 network, while
  the untrusted zone is the Level 4-5 network. The Edge Gateway acts as an IDMZ zone that mediates between the two zones.
- **Conduits**: All AI intents are treated as untrusted until validated by the Southbound Driver.
- **Integrity**: The driver audits the intent against a hard-coded allowed list of registers, preventing the AI from 
  performing illegal physical operations.

### 3. IEC 61511: Functional Safety (SIS Decoupling)
Consistent with the **Independence of Protection Layers (IPL)**, the system is designed as a "Non-Interfering" 
reliability layer. 
- **Safety Dominance**: The Safety Instrumented System (SIS) holds absolute priority.
- **Non-Invasive Monitoring**: OverSight reads telemetry via a one-way mirror (UNS) and only writes to 
  non-safety-interlocked registers.

### 4. ISA-18.2: Alarm Management & Z-Scores
The reasoning engine's alerting logic follows the "Alarm Philosophy" to prevent operator fatigue. It rationalizes 
alerts to prevent "Alarm Floods."
- **Predictive vs. Reactive**: Alerts are generated based on statistical drift ($Z$ > 3.0) rather than simple boolean 
  thresholds, providing operators with earlier intervention windows. It suppresses nuisance alerts that prioritize 
  critical process anomalies to prevent operator fatigue.

### 5. ISO 23247: Digital Twins
The system is designed to be **Decoupled from Hardware** and **Decoupled from Software**.
- **Unified Namespace**: The AI interacts with a Unified Namespace (UNS) rather than specific hardware registers, 
  allowing for seamless integration across different equipment and vendors without requiring custom code changes.

### 6. ISO 42001: Explainable AI (XAI)
To meet regulatory requirements for autonomous systems:
- **Reasoning Trace**: Every intervention is logged with the "Internal Monologue" of the agent.
- **Grounding**: Decisions are mathematically anchored to $Z$-score drift and textually anchored to manufacturer 
  manuals via RAG.

### 7. ISA-101: HMI Design Philosophy
The dashboard is designed to maximize situational awareness during failover conditions:
- **High Performance (Gray Scale)**: The interface uses a high-contrast gray scale to ensure that critical alerts and 
  process deviations are immediately visible, even in high-stress situations. This design choice minimizes cognitive 
  load and allows operators to quickly assess the situation without being overwhelmed by color-coded information.

### 8. ISO 13374: Condition Monitoring
The system follows the standard processing blocks for condition monitoring:
- **Data Acquisition**: Collects real-time telemetry from field devices via the Southbound Driver.
- **State Detection**: Uses statistical analysis (e.g., $Z$-scores) to identify deviations from normal operating 
  conditions.
- **Advisory Generation**: Generates actionable insights and recommendations for operators based on detected anomalies, 
  prioritizing critical issues to prevent operator fatigue.

### 9. NIST 800–82: OT Security
The system uses the following security principles to ensure that the AI's influence on the control system is secure 
and auditable:
- **Least Privilege**: The AI's access to the control system is strictly limited to read-only access for telemetry and 
  a predefined set of non-critical registers for writing, ensuring that it cannot perform unauthorized actions. The 
  Southbound Driver serves as a "Semantic Gatekeeper," validating all AI-generated intents against a hard-coded allowed 
  list of registers and operations, preventing any illegal physical actions, and ensuring that the AI's influence on 
  the control system is always mediated and secure.
- **Transactional Outbox**: All AI-generated intents are first written to a secure "Outbox" within the IDMZ. The 
  Southbound Driver then validates and executes these intents, ensuring that all actions are logged and auditable, and 
  that the AI cannot directly interact with the control system without supervision.
- **Federated Identity and RBAC**: The system offloads human authentication to a centralized Corporate Identity Provider
  using OpenID Connect (OIDC). This ensures that access is governed by enterprise-level Role-Based Access Control (RBAC).
  - **Just-in-Time Revocation**: If an operator's corporate account is disabled, their access to Oversight is instantly
    revoked. This prevents stale or local credentials from being used to access the system after an employee leaves the
    company.
  - **Audit Attribution**: Every acknowledged alarm and executed failover intent is cryptographically linked to a 
    specific federated identity, ensuring 100% accountability in the event of an incident.

### 10. IEEE 1588: Precision Time Protocol (PTP)
To ensure that all components of the system are synchronized to a common time source, the system implements IEEE 1588 
PTP. This allows for precise timestamping of telemetry data, AI intents, and operator actions, which is critical for 
accurate reasoning traces, alarm rationalization, and forensic analysis in the event of an incident.
