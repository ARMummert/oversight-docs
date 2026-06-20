# Welcome to Oversight ICS documentation!

## Cognitive Reliability & Predictive Failover Agent for Critical Infrastructure
### Level 3.5 Industrial Middleware | AI-Driven Asset Orchestration
### Project Status: *In Development*

## Table of Contents
- [Executive Summary](#executive-summary)
- [Field Engineer Quick-Start](#field-engineer-quick-start)
- [Software Engineering & Tech Stack](#software-engineering--tech-stack)
- [Installation & Local Development](#installation--local-development)
- [Introduction & Scope](#introduction--scope)
- [Target Environment](#target-environment)
- [Overview & Core Principles](#overview--core-principles)
- [Core Architecture](#core-architecture-)
- [Agent Architecture](#agent-architecture)
- [Authority Mode Overview](#authority-mode-overview)
- [Degraded Mode Behavior](#degraded-mode-behavior)
- [Industrial Safety Protocol](#industrial-safety-protocol)
- [Safety & Regulatory Governance](#safety--regulatory-governance)
- [Governance & Standard Operating Procedures (SOPs)](#governance--standard-operating-procedures-sops)
- [FMEA (Failure Mode and Effects Analysis) & Deterministic Fail-Safe Matrix](#fmea-failure-mode-and-effects-analysis--deterministic-fail-safe-matrix)
- [Documentation Index](#documentation-index)
- [SOP Documentation Index](#sop-documentation-index)

## Executive Summary

Oversight ICS is a scalable, real-time, and event-driven distributed system that leverages AI to orchestrate complex 
human-in-the-loop failover scenarios. It is implemented as a **Safety-First, Human Governed** AI-Driven Cognitive 
Reliability Layer for **Distributed Control Systems (DCS), Programmable Logic Controllers (PLCs), Remote Terminal Units 
(RTUs), and Rotating Equipment.** The system doesn't replace the primary control hardware, it just supervises it. It 
maintains a **Cognitive Digital Twin** to monitor critical metrics, by streaming asset telemetry through 
**NATS JetStream** and **Bytewax**.  By streaming asset telemetry through the message broker and stream processor,
Oversight has continuous windowed $Z$-score drift analysis to detect logic deviations before they occur and trigger 
automated failovers before physical hardware fails.

Oversight implements strict decoupling from **Safety Instrumented Systems (SIS)** to guarantee that AI-driven 
reliability interventions never compromise hard-coded safety interlocks. The system relies on a **Command Query 
Responsibility Segregation** streaming architecture that uses a single instance of NATS JetStream with Bytewax for 
stream processing. Oversight guarantees that every AI decision is auditable, traceable, and reproducible, with 
sub-second latency while meeting the highest standards of industrial governance and safety. This architecture provides 
operators with a **Pessimistic Confirmation of Success** only after the standby hardware has acknowledged the state 
change from the physical feedback loop.

## Field Engineer Quick-Start

For immediate field deployment, testing, and validation, please refer to the core protocols below:

- **[FAT-01: Hardware & Communication Protocol](/docs/fat/FAT-01-hardware_and_communication_protocol.md)**
- **[FAT-02: Functional & Logic Protocol](/docs/fat/FAT-02-functional_and_logic_protocol.md)**
- **[FAT-03: Safety & Security Protocol](/docs/fat/FAT-03-safety_and_security_protocol.md)**
- **[Site Commissioning Guide](/docs/commissioning.md)**

## Software Engineering & Tech Stack
Oversight ICS is built using a modern, event-driven microservices architecture optimized for edge deployments in 
physical industrial environments.
- **Stream Processing:** Bytewax
- **UNS & Command Spine:** NATS JetStream
- **Orchestration:** LangGraph, local Phi-4 / Gemini 3.5 Flash
- **Database:** TimescaleDB
- **Vector Database:** pgVector
- **In-Memory Cache:** Redis
- **Backend/API:** FastAPI (Python)
- **Security & Hardware:** TPM 2.0 Hardware Notary, RS256 JWTs, Dual-NIC Edge IPC, Level 1.5 Arbitrator, Nginx Perimeter
  Gateway, Cgroup Resource Isolation, Cryptographic Notary Buffers
- **Containerization & Orchestration:** Docker Compose, GitHub Actions CI/CD, Linux Cgroups, Systemd Watchdogs
- **Monitoring & Observability:** Prometheus, Grafana, Loki (for air-gapped safety fallback engine), WORM-based Audit 
  Logs, Immutable Historian

## Installation & Local Development

### Prerequisites
- **Operating System**: Linux-based OS is required for compatibility with hardware interfaces and systemd watchdogs. 
    Windows users must use a fully configured WSL2 environment.
- **Container Engine**: Docker Engine 24.0+ & Docker Compose to spin up the core local gateway services (FastAPI 
    endpoints, TimescaleDB, Redis Hot Cache, NATS JetStream Command Spine, Bytewax Stream Workers).
- **NATS Server (w/ JetStream enabled)**: Required locally for stream processing and command orchestration.
- **Bytewax Streaming Runtime**: Required to execute the stateful stream processing jobs locally for windowed Z-score 
    analysis and logic drift detection.
- **Python 3.12+**: Managed via uv 
- **Node.js 20+**: Required for local building and testing of the frontend dashboard.
- **C++ Build Tools**: Requires full toolchain suite including `build-essential`, `cmake`, and a standard compiler to 
    anchor the native C++ Southbound Driver.
- **LLM Inference Credentials**: API access keys or tokens to an authorized low-latency LLM (e.g., Phi-4, Gemini 3.5
    Flash) for local reasoning and RAG capabilities.
- **(Optional) TPM 2.0 Emulator**: For local hardware testing
- **Storage Permissions**: Hard directory controls must be configured for the WORM audit spools so that local WORM 
    storage directories have strict write-only permissions configured before launching the stack.

### Physical Hardware Requirements
The Edge Gateway is designed for deployment in harsh OT environments (Class I, Div 2). To guarantee high availability, 
strict zero-trust network segregation, and deterministic safety interlocks, production deployments must meet or exceed
the following specifications:

- **Form Factor**: **Industrial PC (IPC)** with **Fanless Passive Cooling** to prevent ingress of dust/particulates.
- **Mounting**: Heavy-duty integrated chassis tailored for standard 35mm DIN-Rail mounting inside field control
    cabinets.
- **Memory & Storage**: Minimum 16GB Error-Correcting Code (ECC) RAM to prevent memory corruption during high-frequency
    telemetry tracking. Storage must be a minimum of 256GB industrial-grade SSD (SLC/pSLC enclaves) supporting
    unalterable log writes.
- **Hardware Secure Root (TPM 2.0)**: An active, provisioned physical TPM 2.0 module is **strictly required**. The
    private signing keys for the outbox intents are pinned inside this chip and cannot leave the hardware boundary.
    This satisfies the secure boot validation and prevents command spooling.
- **Dual NIC Requirement**: The IPC must have two independent physical 1GbE network interface cards (Intel i210/i211 
    preferred for native hardware-level PTP timestamping support):
  - **Southbound OT NIC**: Connected to the isolated process network (Levels 1–2) to interface with process assets.
  - **Northbound IT NIC**: Connected strictly to the **Level 3.5 Industrial DMZ (IDMZ)** infrastructure. 
    - **CRITICAL**: IP 
        Forwarding must be disabled at the host OS level to maintain a logical air-gap and prevent lateral traffic 
        routing between zones.
- **Hardware Watchdog (PVP-02)**:An onboard physical watchdog timer (PVP-02) must be mapped to the system. The C++ 
    Southbound Driver process must check this watchdog loop continuously; if the loop misses a heartbeat for >500ms, 
    the physical DO-0 relay drops open, forcing the system to shed AI control and instantly fall back to local safety 
    logic solvers.
- **Environmental & Enclosure**: Operating Temperature: Ruggedized tolerance across a -20°C to 60°C without thermal 
    throttling or clock jitter degradation. 
  - **Ingress Protection**: Minimum IP40 if mounted inside a certified plant control cabinet; IP67 weatherproofing if 
      deployed externally.
- **EMC Compliance**: Enforced shielding on all active RJ45 ports paired with industrial surge protection arrays. 
    Electrical resistance between the gateway chassis and plant common ground must check out at < 1 Ohm to mitigate 
    electromagnetic interference generated by adjacent heavy machinery (VFDs, high-horsepower motors).
- **Power Framework**: Integrated dual 24V DC terminal blocks supporting fully redundant power input lines backed up 
    by an external industrial UPS deployment.
[!NOTE] For local development and testing, you can use a standard PC or laptop. The hardware requirements are 
    specific to production deployments in industrial environments.

### Quick Start Guide

1. **Clone the repository:**
   ```bash
   git clone [https://github.com/org/oversight-ics.git](https://github.com/ARMummert/oversight-ics.git)
   cd oversight-ics
   ```
2. **Environment Configuration:**
   Copy the example env file and configure your local networking, LLM API keys, the Unified Namespace paths, and 
   database credentials:
    ```bash
    cp .env.example .env
    ```
3. **Run Preflight Compliance Verification**:
   Before attaching active interfaces, validate the local Cgroups, chrony configurations, and PTP interfaces meet 
   security constraints.
    ```bash
    python scripts/monitoring/compliance_watcher.py
    ```
4. **Launch the Core Infrastructure using Docker Compose**:
   Start the core services including NATS JetStream, Redis Hot Cache, TimescaleDB, and the FastAPI backend:
   ```bash
   docker compose up -d nats redis timescale backend
   ```
5. **Compile and Provision the Southbound Driver**:
  Build the native, high-priority C++ binary directly on the host to bridge NATS JetStream events to physical control 
  registers:
   ```bash
   mkdir build && cd build
   cmake ..
   make
   sudo ./oversight_sb_driver
   ```
6. **Initialize Bytewax Stream Workers**:
   Spin up the stateful Python dataflows to handle windowed $Z$-score analysis and logic drift detection.
   ```bash
   docker compose up -d bytewax-workers
   ```
7. Start the Frontend Secure Dashboard:
   ```bash
   cd ui
   npm ci
   npm run dev
   ```
8. Verify System Heartbeat
   - Navigate to http://localhost:3000/system/health to confirm the watchdog handshake is active and the tracking 
     process variables.

## Introduction & Scope

The project scope strictly follows the **ISA/IEC 62443 Purdue Model** for network segmentation, spanning from the 
Physical & Deterministic Control Layers (Levels 0-2) up through the Operations and the Business Logistics Layers (Levels
3-4), and finally reaches up to the Enterprise Cloud Layer (Level 5). 

The Industrial Demilitarized Zone (Level 3.5 IDMZ) acts as the boundary and security checkpoint, where the Northbound 
Ingest Service and Streaming Engine sanitizes and normalize OT data before it reaches the reasoning core. This is also
where the Nginx Perimeter Gateway intercepts enterprise data (e.g., cryptographically signed OEM manuals pushed from
the Level 5 for RAG ingestion) and sanitizes it before it reaches the AI Agent.

### In-Scope 

  - **Predictive Orchestration**: Monitoring process assets for logic drift and mechanical wear patterns
  - **Protocol Normalization**: Consolidates Modbus, OPC_UA, and EtherNet/IP data into a Sparkplug B-enabled Unified 
    Namespace (UNS)
  - **Autonomous Reliability**: Executes **warm standby** transitions when the primary hardware health degrades
  - **Root Cause Analysis**: Using **RAG (Retrieval-Augmented Generation)** to compare real-time anomalies against 
    technical manuals and historical incident logs.
  - **Air Gapped Reasoning**: Maintaining a **local-first** architecture that ensures Level 3 operational autonomy even 
    during loss of internet connectivity.

### Out-of-Scope 

  - **Primary Safety (SIS)**: Oversight operates under strict safety boundaries to prevent unauthorized interventions.
    Oversight does ***NOT*** manage Emergency Stops or high-pressure relief interlocks. These
    remain hard-wired at Level 1.
  - **Hard Real-Time Control (<10 ms)**: The system is a supervisory reliability layer. While the **NATS Command 
    Queue** and **Bytewax Streaming Engine** execution path is sub-100 ms, it is designed to operate within a 
    500 ms supervisory safety watchdog window and does not replace Level 1 deterministic control loops.
  - **Direct Cloud Control**: No control or write command path exists from the Cloud (Level 5) to the Field (Level 0-2). 
    All commands must be validated, authenticated and executed locally at the IDMZ/Edge (Level 3.5).
  - **Commands & System Modifications**: Oversight **DOES NOT** execute unapproved control actions or perform 
    autonomous system modifications

## Target Environment

**Hardware**: Industrial PCs (IPCs) with Dual-NIC network isolation Hardware-Rooted TPM 2.0 modules

**Connectivity**: Industrial environments using OPC-UA, Modbus TCP, EtherNet/IP, or MQTT-enabled process assets

**Use Case**: Critical Infrastructure (Wastewater, Energy, Manufacturing) where unplanned downtime exceeds the cost of
  redundant hardware.

## Overview & Core Principles

As a **Safety-First, Human Governed AI Agent**, Oversight ICS is designed to improve reliability, resilience, and 
observability of industrial control systems.

#### Oversight ICS is built on four **non-negotiable** principles:

### 1. Safety-First
Oversights safety first system never bypasses **Safety Interlocks** or **E-Stops** and the **Safety Instrumented 
System (SIS)** is strictly decoupled and never modified. The system is designed so that all actions are validated 
against physical-layer constraints.

### 2. Human Authority
All control actions and system modifications require explicit human approval. The system **DOES NOT** autonomously 
execute changes that affect control behavior. Operators retain full **Kill Switch** authority over the system.

### 3. Deterministic Execution
All control actions follow predefined Standard Operating Procedures (SOPs). No dynamic or unverified control sequences 
are allowed while execution remains predictable, repeatable, and auditable.

### 4. Full Auditability
Every AI decision, proposal, and action is logged in an immutable historian. The decisions are auditable, traceable, 
and reproducible. The system is designed to meet industrial audit and compliance expectations.

## Core Architecture 
Oversight strictly separates decision-making, validation, and physical execution so that the AI-driven orchestration
never compromises plant safety.

### Northbound Ingest (Protocol Isolation)
Absorbs all protocol complexity (Modbus TCP, OPC-UA, EtherNet/IP CIP) at the Level 3.5 IDMZ boundary, normalizing it 
into a unified data stream.
### Level 1.5 Arbitrator (The Gatekeeper of Authority)
A stateless routing checkpoint that physically halts any proposed command until it verifies cryptographically signed 
authorization (the Two-Key Protocol).
### Semantic Gatekeeper (Safety Dominance)
A strict validation ruleset that cross-references all AI intents against physical constraints, digitized SOPs, and the 
hardcoded Forbidden Register List.
### TPM 2.0 & Notary Buffers (Hardware Root-of-Trust)
Attaches an immutable, cryptographic hardware signature proving the command passed all safety audits before dropping to 
the plant floor.
### Southbound Driver (Deterministic Execution)
The only layer with write-access to the field assets. It executes vetted intents and monitors the physical hardware 
loop for a Pessimistic Confirmation of Success.

See [System Architecture Overview](/docs/system-architecture-overview-and-tradeoffs.md),
[Oversight C4 Context View Diagram](/docs/assets/oversight-c4-context-view.svg),
[Oversight C4 Container View Diagram](/docs/assets/oversight-c4-container-view.svg) and 
[Oversight ICS Technical Deep Dive](/docs/oversight_ics_technical_deep_dive.md) for detailed information.

## Agent Architecture
Oversight is a Type 2/3 Hybrid Agent that combines procedural control with governed intelligence using strict SOP-based 
sequences. This architecture solves the **Black Box** problem in industrial AI by wrapping high-level reasoning inside 
a low-level procedural safety shell.

For more detailed information on Oversight's Agent architecture, see 
[Agent Architecture](/docs/oversight_ics_technical_deep_dive.md#agent-architecture).

## Authority Mode Overview

The platform operates under the following defined authority modes, strictly enforced at Level 1.5 (The Arbitration 
Layer):

1. `CONTROL_ENABLED`: Full autonomous control with active write token held by the AI Agent.
2. `ADVISORY_MODE`: Write token is parked. AI can generate intents but requires human approval for execution.
3. `MONITOR_ONLY`: AI is strictly read-only. All control intents are dropped at the Arbitrator.
4. `AI_LOCKOUT`: AI is completely locked out. No intents can pass the Arbitrator. Control is manual via SCADA/HMI.
5. `MAINTENANCE`: All remote commands are blocked. Control is local-only via physical interfaces.
6. `ORPHAN_MODE`: Triggered by comm-loss. Software components are detached; local PLC reverts to hardcoded logic.
7. `SAFETY_LOCKOUT`: Triggered by safety violation. All software control is destroyed. Physical reset required.
8. `COMMISSIONING_MODE`: Shared control for testing. Intents must pass through the full validation loop with a testing key.

## Degraded Mode Behavior

> The system strictly defaults to non-actuating modes (`ORPHAN_MODE` or `MONITOR_ONLY`) in degraded infrastructure 
> conditions, ensuring that system safety and operational determinism are never compromised.

See [Control Authority Model](/docs/control_authority_model.md) for more information.

## Industrial Safety Protocol

!!! important "SAFETY PROTOCOL: READ CAREFULLY"
    OverSight ICS is a Cognitive Reliability Agent, ***NOT*** a Safety Agent.

    The system is strictly decoupled from the **Safety Instrumented System (SIS)**. Hard-coded safety interlocks and 
    physical E-Stops always maintain primary authority. OverSight operates as a **Read-Verify-Suggest** layer to prevent 
    those safety limits from ever being reached.

    Safety Decoupling is a critical design principle in OverSight ICS.

!!! CAUTION
    **Logic Integrity Guardrail**: The Southbound Driver is strictly prohibited from modifying PLC ladder logic or function blocks. It is limited to writing to a predefined **Command Global Data** block. In this instance, the AI Agent can only request state changes that have been pre-approved and hard-coded by the site's control engineers. 

## Safety & Regulatory Governance
OverSight ICS is a **Cognitive Reliability Layer** (Level 3.5), not a primary safety system. See [STANDARDS.md](docs/standards.md) 
for detailed descriptions of each standard and how Oversight ICS implements them.

## Governance & Standard Operating Procedures (SOPs)
An SOP suite governs all operational behavior. The agent is designed to follow these SOPs without deviation, 
ensuring that all interventions are predictable, auditable, and compliant with industrial best practices.   

See [SOP index](/docs/sop) for a full list of SOPs.

## FMEA (Failure Mode and Effects Analysis) & Deterministic Fail-Safe Matrix
Oversight ICS is designed with a comprehensive FMEA matrix that identifies potential failure modes across all components,
assesses their local and end effects, and implements specific mitigation strategies. The FMEA Matrix is a document that 
is regularly updated based on new insights from the system's operation, incident reports, and continuous learning from 
the Consciousness Loop.

For a detailed breakdown of the FMEA Matrix and the Deterministic Fail-Safe Matrix, 
see [FMEA Matrix](/docs/fmea_matrix.md).

For the full Technical Deep Dive see [Oversight ICS Technical Deep Dive](/docs/oversight_ics_technical_deep_dive.md) 
and for the specific safety protocols see [FAT-03: Safety & Security Protocol](/docs/fat/FAT-03-safety_and_security_protocol.md).

## Documentation Index
- [Alarm Philosophy](/docs/alarm_philosophy.md)
- [API Spec](/docs/api_spec.md)
- [Audit Logging](/docs/audit_logging.md)
- [Change Management Log](/docs/change_management_log.md)
- [Change Regression Test Protocol](/docs/change_regression_test_protocol.md)
- [CI / CD Pipeline](/docs/cicd_pipeline.md)
- [Commissioning](/docs/commissioning.md)
- [Control Authority Model](/docs/control_authority_model.md)
- [Control Logic](/docs/control_logic.md)
- [Cybersecurity Management](/docs/cybersecurity_management.md)
- [Deep Dive](/docs/deep_dive.md)
- [Document Control Index](/docs/document_control_index.md)
- [Evaluation Framework](/docs/evaluation_framework.md)
- [FAT-01: Hardware & Communication Protocol](/docs/fat/FAT-01-hardware_and_communication_protocol.md)
- [FAT-02: Functional & Logic Protocol](/docs/fat/FAT-02-functional_and_logic_protocol.md)
- [FAT-03: Safety & Security Protocols](/docs/fat/FAT-03-safety_and_security_protocol.md)
- [FAQs](/docs/FAQ.md)
- [FMEA Matrix](/docs/fmea_matrix.md)
- [Functional Safety Policy](/docs/functional_safety_policy.md)
- [Glossary](/docs/glossary.md)
- [Incident Response Plan](/docs/incident_response_plan.md)
- [Infrastructure](/docs/infrastructure.md)
- [Install](/docs/install.md)
- [Media Sanitization Decommissioning Guide](/docs/media_sanitization_decommissioning_guide.md)
- [Network Topology](/docs/network_topology.md)
- [Operation](/docs/operation.md)
- [Operational Procedures](/docs/operational_procedures.md)
- [Operator Training Guide](/docs/operator_training_guide.md)
- [Quality Management System](/docs/quality_management_system.md)
- [RAG Knowledge Versioning](/docs/RAG_knowledge_versioning.md)
- [References](/docs/references.md)
- [Requirements Traceability Matrix](/docs/requirements_traceability_matrix.md)
- [Roadmap](/docs/roadmap.md)
- [SBOM Compliance Report](/docs/sbom_compliance_report.md)
- [Site Acceptance Testing](/docs/sat/site_acceptance_testing.md)
- [Standards](/docs/standards.md)
- [Tag Mapping Master](/docs/tag_mapping_master.md)
- [Technical Contracts](/docs/technical_contracts.md)
- [Telemetry Database Schema](/docs/telemetry_database_schema.md)
- [Time Sync Policy](/docs/time_sync_policy.md)
- [Troubleshooting](/docs/troubleshooting.md)
- [UNS Schema](/docs/uns_schema.md)
- [Validation Protocols](/docs/validation_protocols.md)
- [Versioning Policy](/docs/versioning_policy.md)

## SOP Documentation Index

