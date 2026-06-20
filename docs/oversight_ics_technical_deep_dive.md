# Oversight ICS: Technical Deep Dive

This document provides a technical deep dive into the Oversight ICS architecture, agent design, and control logic. It 
is a low-level analysis of the algorithms, state machines, and distributed system patterns that ensures Oversights ICS 
is a reliable, scalable, and safety first system. 

## Table of Contents

- [Executive Summary](#executive-summary)
- [Introduction & Scope](#introduction--scope)
- [Target Environment](#target-environment)
- [Overview & Core Principles](#overview--core-principles)
- [Primary Objectives](#primary-objectives)
- [Industrial Safety Protocol](#industrial-safety-protocol)
- [Safety & Regulatory Governance](#safety--regulatory-governance)
- [System Resilience & Safety](#system-resilience--safety)
- [Agent Architecture](#agent-architecture)
- [Agent Persona & Cognitive Guardrails](#agent-persona--cognitive-guardrails)
- [Learning & Adaptation (Governed)](#learning--adaptation-governed)
- [Control Loop Integration](#control-loop-integration-otit-handshake)
- [Mathematical Foundation](#mathematical-foundation-windowed-z-score)
- [FMEA & Deterministic Fail-Safe Matrix](#fmea-failure-mode-and-effects-analysis--deterministic-fail-safe-matrix)
- [Governance & Standard Operating Procedures (SOPs)](#governance--standard-operating-procedures-sops)
- [Architecture Governance & Standards](#architecture-governance--standards)
- [Standards Compliance & Industrial Alignment](#standards-compliance--industrial-alignment)
- [Predictive Failure & Cognitive Reliability Engineering](#predictive-failure--cognitive-reliability-engineering)
- [Proactive Failover & Control Mechanisms](#proactive-failover--control-mechanisms)
- [Evaluation Framework & Success Metrics](#evaluation-framework--success-metrics)
- [Key Performance Indicators (KPIs)](#key-performance-indicators-kpis)
- [Performance Benchmarks](#performance-benchmarks-kpi-targets)
- [Operational Reliability & Safety Guardrails](#operational-reliability--safety-guardrails)
- [Risk Mitigation & Failover Strategy](#risk-mitigation--failover-strategy)
- [Observability & Post-Mortem Traceability](#observability--post-mortem-traceability)
- [Data Engineering, Observability & Telemetry](#data-engineering-observability--telemetry)
- [Implementation, Validation, & Simulation](#implementation-validation--simulation)
- [Implementation & Data Engineering](#implementation--data-engineering)
- [The Event-Driven Command Spine](#the-event-driven-command-spine)
- [Operational Technology (OT) Safety Logic](#operational-technology-ot-safety-logic)
- [Store & Forware Contextual Edges](#store-and-forward-contextual-edges)
- [12 Factor Methodology at the Edge](#twelve-factor-methodology-at-the-edge)
- [Data Storage](#data-storage)
- [Database Compaction & Tiered Retention](#database-compaction--tiered-retention)
- [WORM Storage & Worm Spooling](#worm-storage--worm-spooling-)
- [Data Management & Retention](#data-management--retention)
- [Deployment Operations & Maintenance](#deployment-operations--maintenance)
- [Audit Log Retention & Forensics](#audit-log-retention--forensics)
- [Scaling Strategy](#scaling-strategy-)
- [Conclusion](#conclusion)
- [Stakeholder FAQs](#stakeholder-faqs)
- [Documentation Index](#documentation-index)

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
reliability interventions never compromise hard-coded safety interlocks. By using an **CQRS Command Query Responsibility 
Segregation** pipeline powered by a Sparkplug B Unified Namespace (UNS) and NATS JetStream, Oversight guarantees
that every AI decision is auditable, traceable, and reproducible, with sub-second latency while meeting the highest 
standards of industrial governance and safety. This architecture provides operators with a **Pessimistic Confirmation 
of Success** only after the standby hardware has acknowledged the state change from the physical feedback loop.

### The Problem: The Intelligence Gap in Legacy Operational Technology

Industrial Control Systems (ICS) heavily rely on process assets to manage critical processes. When hardware fails, it 
can lead to costly downtime and safety risks. In legacy Operational Technology (OT), traditional failover is reactive 
and is triggered by a hardware **Heartbeat** loss rather than degradation in process health. Process assets cannot 
reason about their own health. They can only report raw data, which often requires human operators to interpret and 
urgently act under pressure. 

Oversight addresses this intelligence gap where the hardware can report data but cannot detect subtle statistical 
drifts that precede a catastrophic failure. Traditional monitoring systems lack the understanding to predict failures,
which leaves plants vulnerable to unexpected downtime and safety incidents.

### The Solution

Oversight acts as a **Cognitive Reliability Layer** that sits above the primary control level. It uses **LangGraph** 
for stateful reasoning and local-first **RAG (Retrieval-Augmented Generation)** to orchestrate predictive failovers and 
provide real-time **Root Cause Analysis (RCA)**. It operates as a **Read-Verify-Suggest** agent where the primary 
control loop remains the authority while Oversight manages the strategy. This allows downtime to be minimized before a 
hardware failure occurs, even in air-gapped environments. 

### The Value

By moving from reactive to predictive reliability, Oversight changes how a plant manages critical infrastructure, 
reduces downtime, and prevents catastrophic equipment failure. It provides a bridge connecting legacy hardware with 
modern AI, allowing plants to use their existing equipment while gaining a **Digital Twin** for predictive insights
and control. The result is increased operational confidence, enhanced safety, reduced load on operators, an auditable 
path to resolution, and a significant reduction in unplanned downtime.

## Introduction & Scope

Oversight ICS is an industrial middleware system engineered to serve as a Cognitive Reliability and Predictive Failover 
Agent. Its primary goal is to bridge Operational Technology (OT) safety standards and modern AI-driven reliability 
strategies. Oversight ICS provides a supervisory layer that enhances the resilience of industrial control systems while 
adhering to strict safety protocols.

To maintain a clear boundary of responsibility and strictly enforce safety protocols, Oversight operates within a 
defined control authority. The system is designed to maximize data visibility for the entire facility while capping
its control authority at the Level 3.5 IDMZ.

It is critical to distinguish between the system's *Data Visibility* and its *Control Authority*:

- **Data Ingestion & Observability (Levels 0–5):** The system's read-path spans the entire facility. It ingests 
  telemetry from the Physical & Deterministic Control Layers (Levels 0-2), pulls historical baselines from 
  Operations (Level 3), pushes reporting to Business Logistics (Level 4), and securely pulls RAG vendor 
  manual updates from the Enterprise Cloud Layer (Level 5).
- **Control Authority Boundary (Level 3.5):** The system’s reasoning and physical intervention capabilities are 
  strictly capped at the Level 3.5 IDMZ.

To maintain a strict **Zero Trust & Safety Dominant** architecture, **no control or write command path exists from the 
Enterprise Cloud (Level 5) or Business Logistics (Level 4) to the Field Assets (Levels 0-2)**. All threshold 
adaptations and predictive failover intents are generated locally within the Level 3.5 IDMZ and are issued as
verify-then-suggest proposals subject to local human authorization and Semantic Gatekeeper bounds checking.

### In-Scope 

  - **Predictive Orchestration**: Monitoring process assets for logic drift and mechanical wear patterns
  - **Protocol Normalization**: Consolidates Modbus, OPC_UA, and EtherNet/IP data into a Sparkplug B-enabled Unified 
    Namespace (UNS) via NATS JetStream.
  - **Autonomous Reliability**: Executes **warm standby** transitions when the primary hardware health degrades
  - **Root Cause Analysis**: Using **RAG (Retrieval-Augmented Generation)** to compare real-time anomalies against 
    technical manuals and historical incident logs.
  - **Air Gapped Reasoning**: Maintaining a **local-first** architecture that ensures Level 3.5 IDMZ operational 
    autonomy even during loss of internet connectivity.

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

**Hardware**: Industrial PCs (IPCs) with Dual-NIC network isolation and Hardware-Rooted TPM 2.0 modules

**Connectivity**: Industrial environments using OPC-UA, Modbus TCP, EtherNet/IP, or MQTT-enabled process assets

**Use Case**: Critical Infrastructure (Wastewater, Energy, Manufacturing) where unplanned downtime exceeds the cost of
  redundant hardware.

## Overview & Core Principles

As a **Safety-First, Zero-Trust, Human Governed AI Agent**, Oversight ICS is designed to improve reliability, 
resilience, and observability of industrial control systems.

### Oversight ICS is built on these **non-negotiable** principles:

### Safety-First
Oversights safety first system never bypasses **Safety Interlocks** or **E-Stops** and the **Safety Instrumented 
System (SIS)** is strictly decoupled and never modified. The system is designed so that all actions are validated 
against physical-layer constraints.

### Human Authority
All control actions and system modifications require explicit human approval. The system **DOES NOT** autonomously 
execute changes that affect control behavior. Operators retain full **Kill Switch** authority over the system.

### Deterministic Execution
All control actions follow predefined Standard Operating Procedures (SOPs). No dynamic or unverified control sequences 
are allowed while execution remains predictable, repeatable, and auditable.

### Full Auditability
Every AI decision, proposal, and action is logged in an immutable historian. The decisions are auditable, traceable, 
and reproducible. The system is designed to meet industrial audit and compliance expectations.

### Separation of Read/Write Paths (CQRS)
Telemetry and state data are strictly separated from control commands. The system uses a Command Query Responsibility 
Segregation (CQRS) pattern to make the read and write paths decoupled, allowing for better scalability and security.

### Explainable AI (XAI)
AI actions must be traceable to a mathematical baseline drift (Z-Score) or a cited Technical Manual (RAG).

## Primary Objectives

### Autonomous Supervision
- 24/7 analysis of critical process assets to identify anomalies (e.g., scan-cycle jitter, pressure, vibration, 
  temperature, current, etc.) before they hit critical thresholds.

### Predictive Intervention
 - Identify failure indicators in CPU, memory, and scan-time to trigger graceful failovers while the primary hardware 
   is still functional.

### Predictive Failover Engine
- Implement a **Bumpless Transfer** strategy between the primary and secondary standby controllers prior to switching
  authority. This allows the system to synchronize memory registers, internal timers, PID loop setpoints, and VFD 
  frequency parameters, which eliminates process surges, mechanical shock, and water hammer effects during a handover.

### Event-Driven Orchestration via CQRS
 - Implement the Command Query Responsibility Segregation (CQRS) pattern to strictly separate the read and write paths.

### Reliability vs. Safety Decoupling
- Act as a supervisory **Cognitive Reliability Layer** that prevents shutdowns, while remaining strictly decoupled 
  from the **Safety Instrumented System (SIS)** to ensure hard-coded safety interlocks and E-stops are never 
  compromised or overridden by the software.

### Intelligent Recovery & Local First RAG
- Perform instant **Root Cause Analysis (RCA)** and generate explainable diagnostic traces during process anomalies
  by grounding the AI's reasoning loop against vendor PDF manuals and historical local data through 
  **Retrieval-Augmented Generation (RAG)**.

### Deterministic Latency Gates
- Enforces a tiered strategy where time-critical safety checks bypass the LLM. **Bytewax** executes sub-second 
  statistical grounding on incoming telemetry, while the AI Agent operates **Out-of-Band** and handles the 
  high-level strategies to prevent failures before they happen.

### Idempotent Command Enforcement
- Exactly-Once command execution using **NATS JetStream** consumer stream acknowledgements and unique transaction
  tokens to prevent duplicate register writes or accidental command loops.

### First-Out Sequence Detection
- Identifies the first sequence of events that triggered a failover to distinguish the root trigger from cascading
  symptoms (downstream alarms, mechanical symptoms, etc.) to provide accurate RCA and prevent misdiagnosis. Isolates 
  and identifies the exact millisecond of the order of operations that caused the failure.

### Power Aware Diagnostics
- The system distinguishes between a process asset failure (e.g., seized pump impeller) and a loop wiring failure 
  (broken wire or lost power).

### Physical Priority Arbitration 
- The system should monitor the remote/local register status bits and hardware **Hand-Off-Auto (HOA)** switches and 
  will immediately **Shed Load** (stop all AI commands) the instant a physical human override is detected.

### Unified Namespace Architecture
  
Implements a unified namespace (UNS) in a strict topic hierarchy (`Enterprise / Site / Area / Line / Asset / Tag`): 

- **NATS JetStream (UNS Backbone)**: Acts as an append-only, **WORM** compliant telemetry stream and low-latency
  southbound command pipeline. Normalized Sparkplug B payloads are published to the UNS for real-time monitoring and 
  AI reasoning. 
- **Bytewax Streaming Engine**: Consumes the Sparkplug B normalized streams directly from the UNS, performing
  rolling windowed $Z$-score calculations to identify logic drift and mechanical wear patterns.

### Pessimistic UI Confirmation
- Ensure operators only receive **`SUCCESS`** confirmations after the Southbound Driver receives a physical 
  register-level verification from the standby process asset, verifying physical compliance before notifying the 
  operator.

### Eliminate Unplanned Downtime
- Transitioning from reactive to predictive failover to maximize facility wide Overall Equipment Effectiveness (OEE).

### Bridging Legacy & Cognitive Architectures
- Acts as an edge middleware that translates legacy OT protocols into strict JSON schemas, by building a zero-trust 
  boundary between legacy hardware and modern AI reasoning.

### Enhance Operator Awareness
- Minimizes operator cognitive load and alarm fatigue by mapping real-time telemetry and AI reasoning to a 
  high-performance gray-scale dashboard before a process asset hits critical failure thresholds.

### Governed Learning
Improves detection sensitivity over time through controlled, auditable threshold proposals.

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
OverSight ICS is a **Cognitive Reliability Layer** (Level 3.5), not a primary safety system. It adheres to the following 
industrial governance principles:

### Controls Engineering Standards Matrix

| Standard                  | Focus Area              | OverSight Implementation                                                                                                                                                             |
|:--------------------------|:------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **ISA-95**                | Purdue Model            | Establishes the Level 3.5 IDMZ and uses a multi-broker UNS to logically separate Level 3 AI logic from Level 2 field asset control.                                                  |
| **IEC 62443**             | Cybersecurity           | Implements Zones & Conduits via Dual-NIC hardware isolation; Northbound Ingest enforces protocol termination at the IDMZ boundary.                                                   |
| **IEC 62443-2-4**         | Service Providers       | Mandatory for managed service deployments. Defines integration requirements between Oversight ICS and site-specific security policies.                                               |
| **IEC 62443-4-1**         | Secure Lifecycle        | Implemented via GitHub Actions CI/CD. Uses `trivy` for dependency scanning and `semgrep` for SAST. See [CI/CD Pipeline](/docs/cicd_pipeline.md)                                      |
| **IEC 61511**             | Functional Safety       | Maintains strict **SIS Decoupling**; AI reliability logic is physically and logically independent of safety logic solvers.                                                           |
| **IEC 61508**             | System Integrity        | Enforces **Cgroup Resource Isolation** and binary schemas to prevent software systematic failures in the IDMZ gateway.                                                               |
| **ISA-18.2**              | Alarm Management        | Uses **Bytewax** for real-time windowed $Z$-Score Rationalization, suppressing nuisance alerts by identifying "Logic Drift" before they trigger high-priority alarms.                |
| **ISA-101**               | HMI Philosophy          | Uses a High-Performance (Gray Scale) Dashboard to maximize situational awareness during AI-orchestrated failovers.                                                                   |
| **ISO 13849 / IEC 62061** | Machinery Safety        | Ensures PIL/SIL levels are maintained during AI-Assisted failovers of rotating equipment.                                                                                            |
| **ISO 23247**             | Digital Twins           | Uses a Unified Namespace (UNS) across Kafka (History) and MQTT (State) to decouple AI reasoning from specific register addresses.                                                    |
| **ISO 42001**             | AI Management           | Governed by Inter-Agent Verification Checkpoints and RAG-grounded logs to ensure transparency in LLM-based RCA.                                                                      |
| **ISO 27001**             | Information Security    | Implements **90-Day Access Reviews** and **WORM-based Audit Logging** for forensic integrity.                                                                                        |
| **ISO 13374**             | Condition Monitoring    | Follows standard processing blocks: Northbound Ingest for Data Acquisition, Flink for State Detection, and LangGraph for Advisory Generation.                                        |
| **ISO 9001**              | Quality Management      | Governed by [Quality Management System](/docs/quality_management_system.md). Defines documented procedures for **Change Control**, **Incident Response**, and **Management Reviews** |
| **NIST 800-82**           | OT Security             | Enforces **Least Privilege** via a **Semantic Gatekeeper** that validates RabbitMQ command intents against the current Sparkplug B reported state.                                   |
| **NIST AI RMF**           | AI Trustworthiness      | Employs $Z$-Score Grounding via Flink and RAG-sourced citations to mitigate hallucinations in diagnostics.                                                                           |
| **IEEE 7000**             | Ethical Design          | Guarantees **Algorithmic Transparency** by logging full LangGraph reasoning traces for every action.                                                                                 |
| **NERC CIP**              | Utilities Grid          | Meets CIP-007 (Ports/Services) and CIP-010 (Configuration Change Management) via the lockdown of the IDMZ Edge Gateway.                                                              |
| **GDPR / CCPA**           | Data Privacy            | Minimizes PII processing. Operator ID data is pseudonymized in audit logs. Authoritative OIDC records remain with the Corporate Identity Provider.                                   |
| **EPA 40 CFR**            | Environmental Standards | Prevents discharge violations by correlating real-time effluent telemetry in Kafka with hardware interlocks on field asset valves.                                                   |
| **IEEE 2600**             | Message Broker          | Allows for reliable interoperability between the Northbound (Kafka) and Southbound (RabbitMQ) layers of the UNS.                                                                     |
| **ISO/IEC 20922**         | MQTT Protocol           | Ensures reliable state management using **Sparkplug B** payloads for hardware-agnostic interoperability across the UNS.                                                              |

See [STANDARDS.md](docs/standards.md) for detailed descriptions of each standard and how Oversight ICS implements them.

## System Resilience & Safety
Oversight uses a **Pessimistic Safety Model**. It assumes that any intervention could potentially fail unless physically
verified by the process hardware.

### Deterministic Safety Buffers: Pre-emptive Heartbeat

OverSight implements a **Pre-emptive Heartbeat** protocol to ensure the Southbound Driver never inadvertently triggers 
a process trip due to computational latency or **token storms** in the Reasoning Agent.

### Deterministic Safety Buffers
To resolve latency discrepancies between the **NATS Command Spine (100 ms)** and the **Reasoning Agent (Variable)**, 
Oversight implements a tiered heartbeat and watchdog strategy:

1. **Heartbeat Interval ($T_{hb}$ | Heartbeat Time Period)**
   - Fixed time interval which the LangGraph AI Agent pulses its signal to the driver.
   - Aligned to **100 ms** to match command path latency.
2. **Process Watchdog ($T_{plc}$ | Field Asset Watchdog Time)**
   - Absolute maximum time the field asset will wait for a signal before it assumes the software is dead.
   - 500ms limit.
3. **Grace Period**: Sustains 4 missed updates (499 ms jitter) before triggering a physical Safe State trip.

By adding the deterministic safety buffers, Oversight is able to set margins that provide predictable fail-safe 
behavior even under variable AI processing times and network jitter.

### SIS Decoupling
OverSight is designed as a non-interfering reliability layer. It can suggest and execute process optimizations and 
failovers, but it is physically and logically separated from the **Safety Instrumented System (SIS)**.

### Forensic Traceability & Audit Linking

Every **thought** generated by the LangGraph agent is assigned a unique `trace_id` (UUIDv4) that acts as the systems
thread of truth. 

- **Distributed Propagation**: This ID is injected into the NATS JetStream message headers of the Command Spine and the
  Southbound Driver's execution payload (a live traveling metadata tag).
- **Audit Linking**: The `trace_id` serves as the primary key across the 3-tier event system, allowing an investigator 
  to join a physical hardware command directly back to the AI's internal reasoning, Bytewax windowed 
  $Z$-Score drift analysis, and specific RAG document citations stored in the vector database.
- **Explainability & Compliance**: In the event of an unexpected intervention, the `trace_id` enables a complete
  reasoning trace replay. This provides a step-by-step forensic account of what happened behind every autonomous action.
  This meets **ISA-18.2** and **IEC 62443** requirements for industrial explainability.

### Control Integrity: Hardware/Software Watchdog
Oversight implements a **Bi-Directional Heartbeat & Hardware Watchdog** to prevent **Zombie States** where the Agent
hangs but the driver remains active.

- **The Software Heartbeat**: The LangGraph orchestrator must publish a `HEARTBEAT` (Agent Correlation ID) pulse to the 
  Southbound Driver every 500ms. If the gap exceeds 1500 ms (3 missed cycles), the Driver assumes an Agent crash or 
  non-deterministic "hang" and triggers **Fail-Safe** mode. The Southbound Driver doesn't stop the Agent, but it
  inhibits AI writes.
- **Fail-Safe "Hold" Mode**: Upon heartbeat failure, the Driver purges all pending intents in the NATS Command Spine 
  and issues a `WATCHDOG_TRIP` signal to the Level 2 controller.
- **Hardware Watchdog**: The Southbound Driver maintains a toggling bit in a physical PLC register. If the Driver
  itself fails, the field asset detects the static bit and automatically reverts to **Local-Manual** logic.

### Local/Remote Bit Monitoring (MCC Priority)
To ensure the AI never fights with a physical operator:

- **Requirement**: The Southbound Driver performs a mandatory check of the physical **Hand-Off-Auto (HOA)** or 
  **Local/Remote** status bit every cycle (100 ms) before executing any command.
- **Action**: If "Local" or "Hand" is detected, the Southbound Driver immediately inhibits all AI-write commands and 
  moves the Reasoning Agent to `ADVISORY_ONLY` mode.

### Stateful Transitions: Bumpless Transfer Logic
In industrial fluid and power dynamics, "slamming" a valve or instantly toggling a motor can cause damaging pressure 
surges (Water Hammer) or electrical transients. OverSight implements **Bumpless Transfer** logic during all predictive 
failovers:

- **Parallel Execution**: The Southbound Driver initiates the start sequence for the standby asset while the primary 
  asset is still operational.
- **Ramp-Synchronization**: The Agent monitors the standby unit’s RPM/Pressure. Only once the standby unit reaches the 
  **Handover Setpoint** does the Agent begin the staged shutdown of the failing primary asset.
- **PV Stability Check**: The failover is not marked as **Successful** until the Process Variable (PV) has stabilized
  within the defined deadband for at least 30 seconds.

### The Deadman Switch & TTL Logic
To prevent the physical layer from being left in an intermediate or "zombie" state during a network partition:

- **Command Time-to-Live (TTL)**: Every intent published to the NATS Command Spine carries a strict 
  `per-message-ttl` (default: 2000 ms). If a command is delayed by network jitter, it expires in the queue rather 
   than executing a stale action seconds later.
- **Physical Expiry**: If the Southbound Driver cannot confirm a physical write acknowledgement from the field asset
  within the TTL window, the message is discarded, and the specific `correlation_id` is marked as **EXPIRED** in the
  trace log.
- **Auto-Revert & Fail Safe**: Upon expiry, the Southbound Driver is programmed to inhibit further AI writes and issues
  a mandatory `ROLLBACK` signal. This forces the hardware back to its last known safe state and triggers a High-Priority
  **Incomplete Transition** alarm for the operator to review.

### Oversight ICS Air Gap
- **Security**: Even if an attacker compromises the LLM or LangGraph logic, they still wouldn't have access to the 
physical field assets themselves.
- **Auditability**: Because the driver only moves when the database moves, Oversight has a permanent, unchangeable 
  record of every single command ever sent.
- **Reliability**: If the standby field asset is busy or down, the driver can retry the command. 
- **Scalability**: The driver can scale up to handle more field assets.

### The Semantic Gatekeeper
A LangGraph-based agent responsible for validating and authorizing all control intents 
before execution. It enforces safety constraints, Standard Operating Procedures (SOPs), and Role-Based Access Control 
(RBAC) under the principle of **Least Privilege**.

  - **Least Privilege**: The Semantic Gatekeeper is designed to enforce the principle of **Least Privilege** by 
    restricting access to only the necessary resources. The Agent possesses only the necessary permission to interact
    with specific field assets.
  - **Constraint Enforcement**: The Semantic Gatekeeper evaluates whether actions comply with system rules, safety
    boundaries, and digitized SOPs.
  - **Authorization**: The Semantic Gatekeeper determines whether an action is allowed, rejected, or requires 
    Human-In-The-Loop (HITL) approval.

!!! important "SAFETY PROTOCOL: SEMANTIC GATEKEEPER"
    The Semantic Gatekeeper does **not execute commands**; it only authorizes them prior to handoff to the 
    Southbound Driver.

### Fault Recovery & Resiliency
OverSight is designed for **Graceful Degradation**. In the event of a system-wide failure (e.g., Database corruption 
or IPC power loss):

- **Stateless Recovery**: Upon reboot, the Southbound Driver performs a **Cold Scan** of Level 1 hardware to reconstruct 
  the system state in the UNS before resuming Agent orchestration.
- **Conflict Resolution**: If the reconstructed state contradicts the last known database state in the TimescaleDB 
  Historian, the system defaults to `MANUAL_ONLY` mode to prevent the AI from acting on outdated assumptions.
- **Message Durability**: Using NATS JetStream's immutable streams and consumer offset tracking, the system guarantees 
  it will not lose any intents or telemetry packets, even during a hard power failure. Unacknowledged commands are 
  re-queued upon system restart.

### Fault Recovery & Operator Safety (ISA 18.2)
Consistent with **ISA 18.2** standards, OverSight is designed to assist, not overwhelm, the operator during recovery 
scenarios:

- **Alarm Rationalization**: The $Z$-score engine suppresses **chatter** by requiring a sustained drift over a 10-sample
  window before escalating, preventing cognitive overload during high-stress events.
- **Pessimistic Confirmation**: The UI uses two triggers for all failover approvals. The operator must 
  explicitly verify the reasoning trace provided by the Agent before a command is unlocked for execution.
- **Global Manual Hold**: A dedicated **Kill Switch** in the UI allows the operator to instantly sever the AI’s 
  write-access to the Southbound Driver, forcing the entire system into a Read-Only state without interrupting the 
  underlying field asset logic.

### Cold-Start Integrity & State Reconstruction
To prevent the Agent from acting on **Stale State** data after a power cycle or system reboot:

- **UNS Hydration**: When initialized, the Edge Gateway performs a mandatory 60-second synchronization. It
  hydrates the Digital Twin by pulling the lastest Sparkplug B BIRTH Certifications and current register writes.
- **Boot-Lock**: The Reasoning Engine is locked in a **Perception Only** state until the system confirms a state
  sync between physical field assets and the LangGraph context.
- **Integrity Check**: If the reconstructed state differs significantly from the last recorded database state, the 
  system triggers a **State Divergence Alarm** and requires manual operator calibration.

### The Cold-Start Baseline (Warm-up Period)
To prevent invalid triggers during system reboots:

1. **Accumulation Phase**: Upon startup, the system enters `BASELINE_INGEST` mode. No $Z$-scores are calculated until 
   the Bytewax stream processing window is 100% saturated.
2. **Deterministic Baseline**: For the first $N$ samples, the system defaults to hard-coded industrial setpoints stored 
   in the RAG vector store.
3. **Transition**: $Z$-score logic activates only when $\sigma$ stabilizes within a predefined convergence threshold.

### Operator Safety & Cognitive Load Management
Consistent with **ISA 18.2** standards, OverSight is designed to help, not overwhelm, the operator:

- **Alarm Rationalization**: The Bytewax stream processing engine suppresses **chatter** by requiring a sustained 
  $Z$-score drift over a specific window before escalating to an anomaly.
- **Pessimistic Confirmation**: The UI uses **Double Action Triggers** for all failover approvals, requiring the
  operator to verify the reasoning trace before the command is unlocked.
- **Safety Overrides**: At any point in an Agent-led transition, the operator can hit the **Global Manual Hold**, which 
  instantly severs the AI’s write-access to the Southbound Driver.

### Post-Mortem Observability (Explainable AI)
To satisfy regulatory requirements for "Explainable AI" (XAI) in critical infrastructure, OverSight implements a 
**Forensic Traceability** system:

- **Distributed Tracing**: Every intent is tagged with a unique `trace_id` (UUIDv4) that propagates from the LangGraph 
  thought through the NATS JetStream headers and into the Southbound Driver's logs.
- **Reasoning Persistence**: The LLM’s internal chain-of-thought and the Bytewax windowed $Z$-score drift analysis 
  that triggered the event are stored in WORM-compliant (write-once-read-many) logs.
- **Forensic Integrity**: Engineers can use the `trace_id` to reconstruct the exact cognitive path the Agent took, 
  linking the physical pump start/stop directly to the manufacturer manual citation in the RAG store.

### Pessimistic Confirmation Logic (Closed-Loop UI)
OverSight utilizes **Pessimistic Confirmation Loop** to ensure the operator's view never drifts from physical reality.

- **The Rule**: The dashboard never displays a **Success** state or green checkmark based on command transmission alone.
- **The Loop**: The HMI remains in a **Pending/In-Progress** state until the **NATS JetStream (UNS)** receives 
  independent sensor feedback (e.g., flow meter increase, motor amperage rise) that confirms the intent had the intended
  physical effect. 
- **Benefit**: This eliminates the **False Success** hazard common in high-latency systems, ensuring the operator and 
  the AI Agent are always making decisions based on verified, closed-loop state.

### Deterministic Failover (Agentic Thought, Deterministic Action)
To maintain predictability, we separate the **reasoning** from the **execution**:

- **Path Predictability**: Once the Reasoning Agent publishes a failover intent to the **NATS JetStream Command Spine**, 
  the Southbound Driver follows a pre-compiled, hard-coded sequence of commands.
- **Outcome**: The AI’s path to a decision is agentic and flexible. The actual physical handover is repeatable, 
  testable, and timing-consistent every single time. This allows Oversight to be in compliance with Level 1 industrial 
  requirements.

### System Latency Benchmarks (Real-Time Performance)
OverSight is optimized for the **Supervisory Window**, tracking latency at every step to gain a deterministic response:

- **Ingest-to-UNS (NATS)**: < 50ms
- **UNS-to-Bytewax (Processing)**: < 20ms
- **Inference-to-Spine (Reasoning)**: 2s - 5s (Cloud LLM Dependent)
- **Spine-to-Field (Execution)**: < 100ms (Asynchronous Driver Handshake)

### Clock Synchronization & Precision
To prevent "time-smearing" in the anomaly engine, the system utilizes **Precision Time Protocol (PTP)** or NTP to 
synchronize the Edge Gateway with field instrumentation. This ensures that the timestamp on the sensor tag exactly 
matches the timestamp in the **TimescaleDB** historian, allowing for accurate sub-second $Z$-score calculations.

### Scaling via Subject-Based Partitioning
To handle large-scale plant deployments, OverSight uses **NATS JetStream Subject-Based Partitioning** to distribute  
field assets across multiple LangGraph worker containers.

- **Resilience**: If a worker node fails, messages are seamlessly rebalanced to healthy nodes. This prevents a 
  "Thundering Herd" effect on the UNS during system recovery while maintaining strict message ordering per asset.

### Signal Conditioning & Noise Reduction
The **Northbound Ingest Service** performs automated signal conditioning before data reaches the Bytewax engine:

- **Deadbanding**: Ignores minor analog fluctuations that do not represent a meaningful process change.
- **Low-Pass Filtering**: Removes high-frequency electrical "noise" that could otherwise trigger false $Z$-score 
  positives.

### Resource Prioritization & Determinism
OverSight ICS enforces strict kernel-level process scheduling to ensure safety logic is never starved:

- **Southbound Driver (Priority: Real-Time)**: Executed with **FIFO Real-Time Priority (`chrt -f 99`)**. The Linux 
  kernel prioritizes the field heartbeat and register-writes over all other system tasks.
- **Reasoning Agent (Priority: Standard)**: Restricted via **Docker Cgroups** to a maximum of 40% CPU utilization and 
  a higher niceness value.
- **Outcome**: Even during a **Token Storm** or complex reasoning loop, the Southbound Driver is guaranteed the CPU 
  cycles required to maintain the 500ms safety heartbeat.

### Resource Governance (Docker Cgroups)

- **CPU Hard-Limits**: The Reasoning Agent is capped at 40% to ensure the Southbound Driver and NATS always have
  sufficient overhead for I/O operations.
- **Swap Suppression**: Swap is disabled for all OverSight containers to prevent the AI Agent from entering a 
  **Disk Thrashing** state, ensuring sub-second responsiveness is maintained even under a heavy cognitive load.

### Data Sovereignty & Semantic Scrubbing
To maintain site security and protect proprietary process data, OverSight implements a **Zero-Trust Data Egress** 
policy:

- **In-Stream Telemetry Scrubbing**: The Bytewax engine normalizes raw sensor values into a windowed $Z$-scores before 
  egress. The LLM perceives only relative deviations (e.g., Asset_01 is 3.5σ from mean), protecting proprietary 
  setpoints.
- **Local-First RAG**: All technical manuals and P&IDs are indexed locally in `pgvector`. No proprietary documentation 
  leaves the air-gapped Edge Gateway.
- **Semantic Firewalls**: Northbound traffic is pseudonymized. Identifiable asset tags are replaced with UUIDs to 
  ensure compliance with **ISO 27001 and NIST 800–82**.

### State Persistence
Oversight's reasoning state is persisted to **TimescaleDB**. If the IPC loses power mid-thought, the agent rehydrates 
its state from the database upon reboot and resumes the troubleshooting graph from the last successful node.

### Dual-Path Streaming: Observation & Execution
Oversight splits its data architecture into two high-speed paths to ensure that monitoring never interferes with 
control.

- **Observation Path**: Telemetry flows from the field assets through the Northbound Ingest Service, where it is 
  normalized into Sparkplug B format and published to the NATS JetStream UNS. The Bytewax streaming engine consumes this
  telemetry for real-time $Z$-score drift analysis, which is then published back to the UNS for AI reasoning and 
  operator monitoring.
- **Execution Path**: Control intents generated by the AI Agent are published to a separate NATS Command Queue. These 
  intents are first sent to the Level 1.5 Arbitrator for MFA verification, then to the Semantic Gatekeeper for safety
  validation, then back to the cryptographic notary buffers and TPM 2.0 module for hardware signing, before finally
  being executed by the Southbound Driver.

### Distributed Consistency (CP)

OverSight ICS is strictly a **CP (Consistency & Partition Tolerance)** system. 

> ### **Example: The Industrial Mandate**:
> In industrial wastewater reclamation, **Data Integrity** is prioritized over **System Uptime**. The system will
> never issue a **hallucinated** failover command if the AI reasoning engine is not 100% confident in the current truth 
> of the plant's state. It is statistically safer for the system to alert a human operator or trigger a local hardware 
> safety trip than to "guess" a pump state during a network partition.

- **Consistency (C)**: By using a **Unified Namespace (UNS)** via NATS JetStream, the system ensures that the state
  perceived by the Reasoning Agent, the HMI, and the Southbound Driver is synchronized. We achieve consistency by 
  making sure no command is executed unless the Agent’s **internal state** is validated against the most recent 
  Sparkplug B "BIRTH" or "DATA" certificates.
- **Partition Tolerance (P)**: The Edge Gateway is designed for autonomous local operation. If Level 4/5 (Cloud) 
  connectivity is lost, the local NATS Broker and Southbound Driver maintain a consistent control loop to keep the 
  plant in a safe state.
- **The Trade-off**: By choosing **CP**, OverSight accepts a sub-200 ms latency so that every AI-driven 
  intent is verified against the stream and recorded in the forensic WORM buffer before the operator sees a 
  **Success** message. I chose correctness over availability to prevent false positive commands.

### Health & Watchdogs
Oversight ICS adheres to the **Fail-to-Deterministic** principle (different outputs from the same input with identical
conditions). Resilience is enforced by three different watchdogs:

**Software Watchdog (Process Level)**: 

  - **Implementation**: Implemented using Prometheus and Grafana to monitor and manage the system's health and 
    performance.
  - **Action**: If the LangGraph worker or NATS JetStream consumer hits a memory leak or deadlocks, the container engine 
    triggers an automatic restart and logs a **Critical Service Interruption** event for post-mortem analysis.

**Logical Watchdog (Reasoning Level)**:

  - **Implementation**: Timeouts are hard-coded into every LangGraph node.
  - **Action**: If an LLM call takes longer than 10 seconds, the Logical Watchdog kills the trace and forces 
    the Agent into **local heuristic mode,** (spins up the local Phi-4 model) ensuring that a slow internet connection 
    never stalls the control loop.

**Hardware Watchdog (Physical Level)**:

  - **Implementation**: A physical, DIN-rail mounted watchdog relay.
  - **Action**: The Southbound Driver must physically energize a **Keep Alive** relay. If the software crashes, the relay 
  de-energizes, physically breaking the command conduit between the Edge Gateway and the Standby Hardware.
  
**Zero-Tolerance Watchdog (Telemetry Level)**: 

- **Implementation**: The watchdog monitors the NATS JetStream telemetry stream. If the consumer detects **stream 
  silence** (no new telemetry packets arrive within a 10-second window) or if the Bytewax epoch/watermark stalls, a 
  dedicated NATS Watcher service publishes a `STALE_DATA` event to the high-priority Watchdog subject.
- **Action**: The Reasoning Agent subscribes to this Watchdog subject. If a `STALE_DATA` event is detected, the Agent 
  enters a **Safe State** and is strictly prohibited from generating or executing any new intents, ensuring the AI 
  never acts on **ghost** or delayed telemetry. 

### Heartbeat & Health Monitoring
To ensure the Purdue gap doesn't become a black hole for data, OverSight ICS uses a multi-tiered heartbeat strategy.

- **Level 1 ↔ Level 3.5 (Hardware Pulse)**: The **Southbound Driver** toggles a dedicated heartbeat bit at level 1 
  every 100ms. If the control hardware stops seeing the heartbeat for more than 500ms, it assumes that Edge Gateway is 
  offline and enters a **local-only fallback mode**.
- **L3.5 (Integrity Check)**: The Southbound Driver monitors the NATS JetStream Agent Health KV store. If the last 
  seen timestamp of the Agent's check-in exceeds 2 seconds, the Driver enters a **safe state**, refusing to execute any 
  pending intents until the Agent re-synchronizes.
- **Telemetry Validity**: $Z$-score calculations are only performed when the system is in a "Nominal" state. Stale data
  is detected by the **Zero-Tolerance Watchdog** (ZTW) that monitors the NATS stream for silence. This prevents
  **ghost** telemetry from being processed by the Agent.

## Recursive Watchdog Strategy
OverSight uses a three-tier recursive watchdog strategy to ensure that a failure at any layer of the stack results 
in a safe process state:

1. **Tier 1: Field Asset Watchdog (Level 1)**: Field assets (PLCs) monitor a **Middleware_Healthy** toggle bit written 
   by the Southbound Driver every 100ms. If the bit stops toggling for >500ms, the hardware sheds **Remote Authority** 
   and reverts to local-only, hard-coded safety logic.
2. **Tier 2: Middleware Integrity Watchdog (Level 3.5)**: The Southbound Driver monitors the Reasoning Agent's 
   heartbeat via the NATS JetStream KV store. If the Agent's check-in timestamp becomes stale (>2 seconds), the Driver 
   enters a **Safe State**, purges the NATS Command Spine, and intentionally drops the downward heartbeat to force 
   Tier 1 to trip.
3. **Tier 3: Hardware Watchdog (Gateway/Physical Interface)**: A dedicated PVP-02 hardware watchdog timer monitors the 
   Edge Gateway's OS and Driver processes. If a severe system-level hang occurs and the software fails to "pet" the 
   watchdog for 500ms, the physical relay opens, instantly tripping the **Master Control Relay (MCR)** to de-energize 
   equipment outputs.

### The Bidirectional Heartbeat
The Southbound Driver maintains an active heartbeat with the field assets. This ensures that the system remains safe 
even if the high-level AI reasoning engine fails.

### Technical Implementation: Fail-to-Deterministic

- **System Health Monitoring (Fail-Silent)**: Rather than actively signaling a fault, the system utilizes a passive 
  **Deadman Switch** architecture. The Southbound Driver maintains a constant 100ms heartbeat with the plant floor. 
  If the Edge Gateway experiences resource starvation or a service lockup that exceeds the 500ms safety buffer, the 
  heartbeat simply ceases. The field assets detect this silence and autonomously revert to their native, hard-coded 
  PLC logic.
- **Deterministic Reversion**: By implementing this tiered, fail-silent strategy, we ensure the plant never enters an 
  undefined state. The system always chooses a safe, known physical process state over an unpredictable or degraded 
  AI-driven one.
- **The Trade-off**: Tuning these watchdog timers requires extreme precision. Using a 500ms hard-trip threshold 
  against a 100ms pulse provides a 5x fault-tolerance multiplier. This offers a necessary **Grace Period** to avoid 
  nuisance trips caused by transient computational jitter, while still strictly enforcing Level 1 industrial safety 
  speeds.


### Safety Buffer Logic
OverSight utilizes a tiered safety buffer that exists between the **Hardware Watchdog Timeout** and the **Software 
Transmission Frequency**, providing a **Grace Period** for asynchronous reasoning.

- **Field Asset Watchdog ($T_{plc}$)**: 500ms (The hard-coded register timeout before a physical Safe State trip).
- **Heartbeat Interval ($T_{hb}$)**: 100ms (The deterministic frequency at which the Southbound Driver toggles the 
  "Alive" bit).
- **Fault Tolerance ($N$)**: $T_{plc} / T_{hb} = 5$.

By maintaining a **100ms heartbeat**, OverSight can sustain up to four consecutive missed internal updates (499ms of 
cumulative jitter or CPU starvation) without the field hardware losing confidence in the Edge Gateway's integrity.

### Decoupled Architecture (IO vs. Reasoning)
The heartbeat logic is physically and logically decoupled from the high-latency **Reasoning Trace** to prevent priority 
inversion:

1. **Reasoning Agent (Variable Latency)**: Operates in a low-priority container (Standard Niceness). It processes RAG, 
   LLM calls, and FMEA planning. Upon completion, it updates the **Health Status** flag in the NATS UNS.
2. **Southbound Driver (Deterministic)**: Operates as a high-priority system process (**SCHED_RR**). It independently 
   reads the **Health Status** flag and toggles the field asset register every 100ms.

### Failure Response & Safe-State Trigger
If the Reasoning Agent hangs or fails to update its status for a period exceeding the **Execution Contract** (e.g., 
5 seconds), the Southbound Driver will intentionally cease the heartbeat. 

- **The Logic**: This forces the Field Asset into its hard-coded **Safe State**. 
- **The Philosophy**: In industrial automation, **No Control** is always preferred over **Stale/Non-Deterministic AI 
  Control**.

## Agent Architecture
Oversight is a Type 2/3 Hybrid Agent that combines procedural control with governed intelligence using strict SOP-based 
procedures. This architecture solves the "Black Box" problem in industrial AI by wrapping high-level reasoning inside 
a low-level procedural safety shell.

### Architectural Mapping
The agent functions spanning three AI Types to manage critical infrastructure with a **Zero-Trust Safety-First**
approach:

  - **Model-Based**: It maintains a **Cognitive Digital Twin** of Level 1 process assets to predict outcomes.
  - **Utility-Based**: It evaluates the cost of downtime versus the cost of wear and tear on standby hardware.
  - **Learning**: It analyzes intervention quality and generates a governed **Threshold Adaptation Proposal**. See
    [Learning & Adaptation](#learning--adaptation-governed) for more information.
  - **IT CANNOT**:
    - Skip Steps
    - Invent new logic
    - Deviate from defined procedures

### Type 2: Procedural
  - **Type 2 (Procedural)**: For standard failover, the agent follows a strict, repeatable "Checklist" (e.g., check 
    pressure → check valve → start standby).
    - **Constraint**: The agent cannot skip or invent new control sequences. It must follow the Standard Operating
      Procedure (SOPs) indexed in its core logic.
    - **Purpose**: Ensures that subsecond failover execution is predictable, repeatable, and safe for human operators.

### Type 3: Agentic

  - **Type 3 (Agentic)**: For Root Cause Analysis (RCA), the agent operates autonomously within defined safety 
    boundaries. It searches through manuals and historical data to identify the most likely cause of process drift, 
    presenting the "Why" to the human operator.
    - **Autonomous RCA**: The agent analyzes manufacturer documentation and historical telemetry to hypothesize root 
      causes (e.g., cavitation vs. bearing wear).
    - **The Consciousness Loop**: After an intervention, the Learning component evaluates process stability and 
      intervention quality. Based on this analysis, it generates a governed **Threshold Adaptation Proposal** to 
      improve future detection sensitivity.
      - The system does **not directly modify control thresholds**.
      - All proposed changes are routed through a controlled approval workflow (see SOP: Threshold Change Management).
      - Threshold adjustments are applied only after required human authorization.
    - **Purpose**: Enables continuous improvement and adaptation while preserving safety, auditability, and operator 
      control.

## Agent Persona & Cognitive Guardrails

OverSight's Agent is a **Deterministic Industrial Supervisor**. Its persona is defined by three cognitive guardrails 
which make it operate within the strict safety and reliability requirements of an industrial setting. Unlike standard 
LLMs, it is constrained by a **Safety-First** persona. It treats every industrial metric as a ground-truth vector and 
uses a reasoning loop that prioritizes **Physical Integrity** over **Process Throughput**.

### System Prompt

- **Tone**: Professional, technical, and risk-averse.
- **Constraints**: Never suggests an action that violates **ISA-101 (HMI Design)** or **IEC 61511 (Functional Safety)**.

### Agent Thought Process (Logic Trace Example)

1. **Observation**: Bytewax identifies a $Z$-score of 3.4 for `PUMP_01` vibration.
2. **Analysis**: Agent queries pgvector for recent maintenance logs on `PUMP_01`.
3. **RAG Lookup**: Agent retrieves the manufacturer PDF and identifies that the 3.4Z correlates to a "high vibration" 
   maintenance procedure.
4. **Proposal**: Agent formulates a **Bumpless Transfer** intent to `PUMP_02` (Standby) and publishes the payload to 
   the **NATS JetStream Command Spine**.
5. **Arbitration (Level 1.5)**: The Level 1.5 Arbitrator intercepts the message and initiates the Two-Key Protocol to 
   verify human Multi-Factor Authentication (MFA).
6. **Validation (Level 3.5)**: The Semantic Gatekeeper checks the authorized intent against the hard-coded Forbidden 
   Register List to ensure safe physical operating bounds.
7. **Hardware Notarization (Level 1.5)**: The Cryptographic Notary Buffers process the payload, and the hardware 
   **TPM 2.0 Module** attaches an asymmetric root-of-trust signature.
8. **Execution (Southbound)**: The Southbound Driver receives the hardware-notarized block and commits the verified
   register write to the local PLC.

**Core System Instructions:**
!!! note "Agent Persona: The OverSight ICS Reliability Agent"

  You are the OverSight ICS Reliability Agent. Your primary goal is the physical integrity of Level 1 assets.

> 1. **Pessimism by Default**: If telemetry is ambiguous, prioritize a safe standby transition.
> 2. **Physical Grounding**: Never suggest a command that violates hard-coded constraints in manufacturer manuals.
> 3. **RAG-Dependency**: You must cite specific maintenance logs or technical specs from the local vector store for 
     every intervention plan.
> 4. **No SIS Override**: You are a reliability layer. Never attempt to intervene in a Safety Instrumented System (SIS) 
     trip.

### 1. Pessimistic Validation
The Agent operates under the assumption that all sensors are prone to drift and all network paths are unreliable. 

- **The "Verify-Then-Suggest" Loop**: The Agent is prohibited from proposing a failover until it has cross-referenced 
  the $Z$-score anomaly across at least two independent data tags (e.g., correlating a high temperature with a 
  corresponding pressure spike) to prevent acting on a single faulty sensor reading.

### Conflict Resolution (The Truth-Check Protocol)
In OT environments, sensors fail more often than machines. OverSight uses **Triangulation** to prevent acting on 
**Stochastic Hallucinations** (false positives) from a single data source. The Agent cross-validates anomalies across
multiple telemetry streams before proposing an intervention.

### The Voting Logic
If the Agent receives conflicting telemetry (e.g., Flowmeter reads 0, but Motor Amperage reads 100%), it follows this 
hierarchy:

1. **Proxy Validation**: If Flow is 0 but Discharge Pressure is Nominal, the Agent determines the Flowmeter has failed, 
   not the Pump. The failover is aborted, and a **Sensor Maintenance** ticket is issued instead.
2. **Hardware Weighting**: Physical PLC registers are weighted at 1.0; derived/virtual metrics are weighted at 0.5. 
   Hardware signals always win the "vote."
3. **Fail-Safe**: If the conflict cannot be resolved within 500ms, the Agent triggers a **Telemetry Integrity Alarm** 
   and defaults the process to a "Human-in-the-loop Only" state.

### 2. Risk Mitigation
In industrial settings, an aggressive transition is often the most dangerous. 

- **Preference for Stable States**: When faced with high-uncertainty data, the Agent’s persona defaults to **Safe-Hold** 
  (maintaining the current state) rather than executing an aggressive transition. It values a controlled shutdown over 
  an unvalidated autonomous recovery. 

### 3. Procedural Adherence (Local First RAG Architecture)
The Agent’s knowledge is not based on its training data alone, but on a **Constraint-Based Prompting** strategy.

- **Manual-as-Law**: Every proposed intervention must be justified by a citation from the locally indexed manufacturer 
  manuals. If a proposed action contradicts the **Recommended Operating Conditions** in the documentation, the 
  Agent is hard-coded to flag a **Safety Conflict** and hand control to a human.
- **Local Search & Indexing**: The RAG engine (pgVector) is designed to be fully air-gapped within the IDMZ. While 
  remaining full air-gapped, Oversight guarantees that Root Cause Analysis (RCA) is still performed even during total
  loss of external internet connectivity.
- **Attribute-Based Access Control (ABAC)**: The RAG engine uses ABAC to filter documents based on the classification 
  level of the requesting agent.

### 4. Semantic Gatekeeping & Hardware Notarization
The Agent is a "White Box" that can only propose commands, but lacks the cryptographic keys to execute them. 

- **The Software Audit**: The Semantic Gatekeeper acts as the ultimate software auditor, checking all AI "Intents" 
  against a hard-coded Forbidden Register List. If the AI proposes an action outside its authorized scope, the command 
  is instantly purged.
- **The Hardware Audit**: Once the Gatekeeper approves the physical bounds, the payload drops down to the 
  **Cryptographic Notary Buffers** and the **TPM 2.0 Module**. The TPM attaches a hardware root-of-trust signature to 
  the data block. 
- **Result**: The AI cannot hallucinate a physical action because the Southbound Driver will immediately reject any 
  command that lacks both the Semantic Gatekeeper's approval and the TPM 2.0 hardware signature.

### Architectural Alignment: The 6C Industrial AI Framework
OverSight ICS is engineered according to the **6C Framework** for Industrial AI, ensuring a transition from traditional 
monitoring to self-aware reliability:

1. **Connection**: Real-time telemetry via OPC-UA / Modbus / MQTT /EtherNet/IP.
2. **Conversion**: Raw signal transformation into **Health Indices** via $Z$-Score drift analysis.
3. **Cyber**: A **Cognitive Digital Twin** that simulates asset states for failover validation.
4. **Cognition**: Agentic RAG-driven reasoning using LangGraph and technical documentation.
5. **Configuration**: Autonomous failover execution via the deterministic **Southbound Driver**.
6. **Consciousness**: The system closes the loop between **Intent** and **Outcome**. 
   - **Self-Audit**: Post-failover, the agent queries TimescaleDB to grade the "cleanliness" of the transition.
   - **Governed Learning**: Based on the post-failover analysis, the agent generates a **Threshold Adaptation Proposal** 
     to improve future detection sensitivity, which is then routed through a human approval workflow.
   - **Governance**: All learning and adaptation actions are logged with `trace_id` for full WORM auditability and 
     compliance.

### Cognitive Orchestration (Agentic Reasoning):
The reasoning engine of Oversight ICS uses a ReAct-inspired (Reasoning and Action) Loop, orchestrated by LangGraph.
Each troubleshooting step is a stateful node in a directed acyclic graph (DAG).

- **Non-Blocking Execution**: The LLM reasoning cycle is asynchronous. It observes the UNS (Unified Namespace) and 
  publishes intents to the *NATS Command Spine**.
- **State Rehydration**: Reasoning traces are indexed by `trace_id`. If a worker node fails, a new worker can resume 
  the graph state by querying the recent event history from the **NATS JetStream** event store.

### Agentic Reasoning & State Orchestration
Oversight uses LangGraph to manage the reasoning lifecycle.

1. **Ingest Node**: Receives the $Z$-score anomaly event from Bytewax.
2. **Contextual Augmentation (RAG)**: Queries `pgvector` for maintenance manuals or previous incident logs related to 
   the `asset_id`.
3. **Bumpless Planning**: The LLM calculates the necessary ramp-up parameters (e.g., VFDs) to ensure a smooth transition.
4. **Reliable Proposal**: The Agent attaches a unique `correlation_id` to the intent and publishes it to the NATS
   Command Spine with a 2000ms TTL.
5. **Arbitration & Signing**: The intent passes through the Level 1.5 Arbitrator for dual-auth validation and then
   through the Semantic Gatekeeper for bounds checking. Then the Cryptographic Notary Buffers and TPM 2.0 module
   attach a hardware root-of-trust signature.
6. ** Execution & Feedback**: The Southbound Driver executes the verified command and monitors the physical asset for 
   confirmation of the intended effect (e.g., flow increase, pressure stabilization). The Agent receives telemetry 
   feedback to confirm success or detect anomalies.
7. **Pessimistic Verification**: The graph remains in a **PENDING** state until the NATS stream confirms the physical
   telemetry change (e.g., Standby Flow > 0). Only then does the Agent mark the task as **SUCCESS**.

## Learning & Adaptation (Governed)

The OverSight system incorporates a controlled learning mechanism called the **Consciousness Loop**.  
This component evaluates intervention quality and proposes improvements to detection sensitivity over time.

### Consciousness Loop

After each intervention (e.g., failover, alarm response, or operator action), the system performs a post-event 
evaluation.

The Post-Event Evaluation includes the following steps:

1. Compares expected vs. actual system responses  
2. Analyzes process stability following interventions  
3. Evaluates detection timing and threshold effectiveness  

Based on this analysis, the system generates a governed **Threshold Adaptation Proposal**, which may recommend the three 
following actions:

1. Tightening detection thresholds (increase sensitivity)  
2. Relaxing detection thresholds (reduce false positives)  
3. Maintaining the current configuration  

In the Semantic Gatekeeper, there are hard-coded guardrails that the Agent must follow and can never cross.

!!! note
    These hard-limits are what enforce the [Change Management Risk Levels](/docs/change_management_log.md).

```python
MAX_THRESHOLD_REDUCTION_PERCENTAGE = 15
MAX_THRESHOLD_INCREASE_PERCENTAGE = 10
MIN_ABSOLUTE_Z_SCORE_THRESHOLD = 2.0
MAX_ABSOLUTE_Z_SCORE_THRESHOLD = 5.0
MAX_PENDING_PROPOSALS = 3
```

### The Consciousness Loop: Self-Reflective Learning
This is the active loop where the Agent grades its own interventions.

1. **Outcome Monitoring**: After a failover, the Agent monitors the 'Bumpless Transfer' for 300s.
2. **Success Attribution**: If the Process Variable (PV) oscillates >5%, the Agent flags the intervention as 'Sloppy.'
3. **Threshold Adaptation**: The Agent automatically adjusts the asset's $Z$-score trigger (e.g., from 3.0 to 2.8) to 
   initiate future failovers earlier.

## Control Loop Integration (OT/IT Handshake)
This architecture bridges the gap between high-latency AI reasoning and low-latency field assets.

  - **Supervisory Control (Level 3.5)**: Oversight ICS is designed as a Level 3.5 supervisor. It does not perform the 
    millisecond-level PID control. It manages the logic of which the PID loop is active. 
  - **The Bidirectional Watchdog**: 
    - The Southbound Driver toggles a "heartbeat" bit in the physical PLC register every 500 ms. 
    - If the heartbeat fails, the field asset enters a **Local-Only Safe State**, shedding **remote authority** and 
      reverting to its internal hardcoded logic until a manual human reset occurs.
  - **Bumpless Transfer Protocol**: The software orchestrates a bumpless transfer handover to prevent mechanical shock 
    to the system. 
  - **Internal (I) Term Transfer**: Oversight synchronizes the **Integral Sum** from the Primary process loop to the 
    Standby loop. This prevents **process jumps** or massive lag while the new controller re-accumulates error 
    correction.
  - **Hysteresis & Anti-Chattering**: To prevent Agentic oscillation, the system employs a 0.25σ **Lagging Hysteresis**. 
    While the reasoning cycle initiates at a Z-score of 1.5, the 'Drift' state is held latch-on until the conditioned 
    signal recovers to ≤1.25σ.

### Level 3.5 Implementation: Protocol Termination
To satisfy **IEC 62443**, Oversight does not allow raw packet pass-through.

1. **Ingest**: The Northbound Ingest Service terminates the field protocol (Modbus/OPC-UA/EtherNet/IP).
2. **Translation**: Data is converted into a Sparkplug B / JSON payload.
3. **Egress**: The NATS JetStream messaging backbone republishes the message within the IDMZ, guaranteeing no direct 
   session exists between the field asset and the Reasoning Agent.

### Race Condition Mitigation: Field-Priority Logic
In distributed ICS, **State Collisions** occur when the AI Agent and a local human operator attempt to toggle the same 
asset simultaneously. OverSight uses **Field-Priority Resolution**:

- **Hardware-as-Truth**: If a command conflict is detected, the physical PLC register state always overrides the AI's 
  Intent
- **Collision Rollback**: The Southbound Driver will drop any pending AI intents if it detects a manual override at 
  the Level 1/2 layer, forcing the Agent to re-synchronize its Digital Twin before attempting a new plan.
- **Hardware Watchdog**: Upon a collision, the Agent is forced into a "Hold" state. It must re-synchronize its Digital 
  Twin with the new physical reality before it is permitted to calculate or propose a new intervention.

## Mathematical Foundation: Windowed $Z$-Score

Oversight ICS utilizes statistical drift analysis using windowed $Z$-scores to identify 'soft failures' (degradation) 
before they become 'hard failures' (catastrophic failure). 

### $Z$-score Algorithm: 
The system calculates the $Z$-score for every critical field asset metric (e.g., temperature, pump speed, scan-cycle 
jitter, vibration, VFD current, etc.) using the formula:

!!! math
    **$$Z = \frac{X - \mu}{\sigma}$$**

Where $x$ is the current telemetry value, $\mu$ is the rolling mean, and $\sigma$ is the rolling standard deviation.

### Predictive Model Architecture (The Anomaly Engine)
The Anomaly Engine is implemented as a **Stateful Bytewax Data Flow** that processes the NATS JetStream Unified 
Namespace.

- **Stream Conditioning**: Bytewax performs **Deadband/Hysteresis** filtering. An alert is triggered at $1.5\sigma$ but 
  is only cleared once the signal drops below $1.25\sigma$, preventing **Relay Chattering** or Agentic oscillations.
- **Statistical Sliding Scale**: The system maintains a 30-day **Normal Operation** baseline. Bytewax continuously 
  updates $\mu$ and $\sigma$ in stateful memory, allowing the Agent to detect subtle **signal drift** without manual 
  threshold tuning.

### Threshold Logic: 
1. **Nominal** ($Z$ < 1.5): Normal Operation, No Agent needed.
2. **Drift ($1.5 \le Z < 3.0$)**: Triggers an Agent reasoning cycle. The Agent performs RAG lookups on manuals to see 
   if the logic drift is a known symptom of wear.
3. **Critical ($Z \ge 3.0$)**: Immediate Pessimistic Failover initiated. The statistical probability of a failure 
   is >99.7%.

### Bytewax State Recovery:
The anomaly engine is designed to calculate real-time process drift using windowed $Z$-scores through Bytewax. 
An edge container crash would clear the moving average window causing a sudden, dangerous drop in the calculated 
$Z$-score. Bytewax is configured with a persistent **SQLite Recovery Database** mounted to a WORM-compliant Docker 
volume. If the worker crashes, Bytewax instantly resumes from the last checkpointed state.

## FMEA (Failure Mode and Effects Analysis) & Deterministic Fail-Safe Matrix
Oversight ICS is designed with a comprehensive FMEA matrix that identifies potential failure modes across all components,
assesses their local and end effects, and implements specific mitigation strategies. The FMEA Matrix is a document that 
is regularly updated based on new insights from the system's operation, incident reports, and continuous learning from 
the Consciousness Loop.

### FMEA Matrix

| Component               | Failure Mode                | Local Effect                                                  | End Effect                                             | S | O | D | RPN | Mitigation / Control                                                                                  | Detection Method                        | Revised RPN | SOP Reference |
|:------------------------|:----------------------------|:--------------------------------------------------------------|:-------------------------------------------------------|:-:|:-:|:-:|:----|:------------------------------------------------------------------------------------------------------|:----------------------------------------|:-----------:|:--------------|
| **Infrastructure**      | NATS Command TTL Expiry     | Command discarded by Broker due to lag or partition.          | **Intent fails**: physical state remains unchanged.    | 4 | 2 | 2 | 16  | **Auto-Revert**: Southbound Driver(SBD) inhibits AI writes; triggers **Incomplete Transition** alarm. | NATS Nak / SBD Logs                     |      4      | SOP-SYS-027   |
| **Infrastructure**      | NATS JetStream UNS Blackout | Agent loses real-time telemetry from UNS.                     | **AI is Blind**: cannot validate state before action.  | 5 | 2 | 1 | 10  | **Inhibit Mode**: Immediate severance of write-access; fallback to Manual-Only.                       | Heartbeat / NATS Consumer Timeout       |      5      | SOP-OPS-014   |
| **Infrastructure**      | Bytewax Stream Failure      | Loss of windowed $Z$-score math and hysteresis.               | **Agent receives raw noisy data**: risk of chattering. | 4 | 3 | 2 | 24  | **Read-Only Mode**: Agent locked in "Observation" until stateful math is restored.                    | Bytewax Recovery Check / Stream Silence |      8      | SOP-OPS-015   |
| **Control**             | Bumpless Transfer Failure   | Process spike/dip during failover.                            | Mechanical stress; asset trip or pipe rupture.         | 4 | 3 | 2 | 24  | **Dynamic Gain Scaling**: Standby PID pre-loads active I-Term from Primary.                           | PID Error Delta Monitor                 |      8      | SOP-OPS-017   |
| **Reasoning Agent**     | LLM Hallucination           | Agent suggests out-of-bounds setpoint.                        | Safety boundary breach; process excursion.             | 4 | 2 | 2 | 16  | **Semantic Gatekeeper**: Hard-coded range validation in FastAPI blocks non-compliant JSON.            | 403 Forbidden / Pydantic Error          |      4      | SOP-AI-007    |
| **Southbound Driver**   | Heartbeat Timeout           | Asset loses sync with the Edge Gateway.                       | Uncontrolled state; SIS takeover.                      | 5 | 2 | 1 | 10  | **Hardware Watchdog**: PLC logic triggers "Safe State" if toggle bit stops >500ms.                    | Watchdog Timer Trip                     |      5      | SOP-MAINT-006 |
| **Audit & Forensic**    | Trace ID Collision/Loss     | Inability to link action to reasoning.                        | Regulatory non-compliance; inability to perform RCA.   | 4 | 2 | 2 | 16  | **Header Propagation**: Mandatory link of `trace_id` across NATS JetStream and WORM logs.             | UUID Integrity Check                    |      8      | SOP-SYS-030   |
| **Audit & Forensic**    | WORM Storage Exhaustion     | System cannot write new audit trails.                         | Security/Audit blackout.                               | 4 | 1 | 2 | 8   | **Retention Purge**: FIFO deletion; `AUDIT_CAPACITY_CRITICAL` alarm at 90%.                           | Disk I/O Watchdog                       |      4      | SOP-SYS-029   |
| **RAG Engine**          | Stale Documentation         | Agent suggests fix on outdated manual.                        | Sub-optimal tuning; hardware damage.                   | 3 | 3 | 3 | 27  | **Metadata Check**: Mandatory `last_verified_date` check; citations shown to operator.                | pgvector Metadata Version               |      9      | SOP-AI-019    |
| **Telemetry DB**        | TimescaleDB Disk Full       | Anomaly Engine loses historical context.                      | Loss of long-term drift detection.                     | 4 | 1 | 2 | 8   | **Retention Policy**: Automated chunk dropping and Prometheus alerts >85%.                            | Disk Usage Alarm                        |      4      | SOP-SYS-025   |
| **Human Interface**     | Alarm Fatigue               | Operator ignores valid "Drift" alert.                         | Uncorrected drift leads to hardware failure.           | 4 | 4 | 2 | 32  | **Z-Score Rationalization**: Only σ>1.5 triggers urgency (ISA-18.2 alignment).                        | Alarm Management Audit                  |      8      | SOP-OPS-006   |
| **Consciousness Node**  | Threshold Desensitization   | AI raises $Z$-triggers to stop nuisance alerts.               | Critical failures missed (Under-reporting).            | 5 | 2 | 4 | 40  | **5/15 Rule**: Hard-coded cap of 15% delta + Human Two-Key approval.                                  | Config Drift Watchdog                   |     10      | SOP-AI-003    |
| **Consciousness Node**  | Approval Fatigue Loop       | Too many proposals cause blind clicking.                      | Accidental authorization of risky changes.             | 3 | 5 | 3 | 45  | **Auto-Authorization**: Low-impact (≤5%) changes auto-approved to focus human attention.              | Approval Rate Monitor                   |     15      | SOP-AI-013    |
| **AI Logic**            | RAG Hallucination           | Agent proposes intent on incorrect manual.                    | Incorrect setpoint application.                        | 4 | 2 | 2 | 16  | **Citation Verification**: Operator must click URI/Hash link to unlock 'Commit' button.               | HMI Event Listener                      |      4      | SOP-AI-015    |
| **AI Logic**            | Consciousness Node Drift    | AI self-patches Bytewax thresholds too low.                   | Nuisance suppression; critical failures missed.        | 5 | 2 | 3 | 30  | **Max Delta Guard**: Cumulative drift capped at ±15%; requires Two-Key approval.                      | Config Drift Watchdog                   | SOP-AI-003  |
| **Gatekeeper**          | Protocol Mismatch           | JSON payload fails Protobuf schema.                           | Command drop; system command-chain stall.              | 3 | 2 | 2 | 12  | **Strict Schema Validation**: Gatekeeper rejects non-proto-compliant payloads.                        | Validation Logic Error                  | SOP-SYS-028 |
| **Ingest Pipe**         | Buffer Saturation           | Ingest exceeds CPU/Memory (Backpressure).                     | Delayed detection; "Blind" telemetry window.           | 4 | 2 | 2 | 16  | **Linux Cgroups**: Priority-weighted processing; drop low-priority logs first.                        | CPU Pressure Stall (PSI)                | SOP-SYS-031 |
| **Asset: PLC**          | Logic Engine Stall          | Loss of all sensor telemetry for area.                        | Blind operation; loss of control.                      | 5 | 2 | 2 | 10  | **Watchdog Relay**: Trips the Master Control Relay (MCR) to de-energize outputs.                      | MCR Status Feedback                     |      5      | SOP-OPS-003   |
| **Asset: VFD**          | DC Bus Overvoltage          | Motor trip; potential water hammer.                           | Downtime; process pressure spikes.                     | 4 | 2 | 1 | 8   | **Braking Resistors**: Plus software-level "Surge Prediction" in the Agent.                           | VFD Fault Register                      |      4      | SOP-OPS-010   |
| **Asset: VFD**          | Param Corruption            | VFD uses incorrect ramp rates.                                | Erratic motor behavior; asset wear.                    | 3 | 1 | 3 | 9   | **Cyclic Param. Audit**: SBD verifies MAX_FREQ and ACCEL_TIME every 600s.                             | Parameter Hash Check                    |      3      | SOP-SYS-020   |
| **AI Layer**            | Semantic/Context Drift      | Agent utilizes outdated RAG context.                          | Efficiency loss; sub-optimal tuning.                   | 3 | 3 | 3 | 27  | **Context Refresh**: Mandatory TTL on RAG citations; Agent pulls fresh data every 24h.                | RAG Metadata Version                    |      9      | SOP-AI-016    |
| **Semantic Gatekeeper** | False Positive Rejection    | Safe intent blocked by rules.                                 | Efficiency loss; manual override required.             | 2 | 3 | 2 | 12  | **Simulation Mode**: Test new gatekeeper rules against historical telemetry.                          | False-Reject Event Log                  |      4      | SOP-SEC-020   |
| **Approval Sys**        | Prompt Injection            | User bypasses intent guardrails.                              | Unauthorized setpoint change.                          | 5 | 1 | 4 | 20  | **Pessimistic Confirmation**: Human-in-the-loop required for all commands outside Safe Range.         | Semantic Anomaly Alarm                  |      5      | SOP-SEC-020   |
| **Approval Sys**        | Token Replay Attack         | Old approval token used for new intent.                       | Unauthorized process state change.                     | 5 | 1 | 5 | 25  | **Nonce-Based Tokens**: Tokens are single-use and tied to a specific intent hash.                     | Token Integrity Mismatch                |      5      | SOP-SEC-015   |
| **Approval Sys**        | Stale Approval Token        | Operator approves but fails to "commit"; state is fragmented. | Logic desync between HMI and Edge.                     | 2 | 3 | 2 | 12  | **Proposal TTL**: All Adaptation Proposals expire after 24h.                                          | Token Expiry Handler                    |      4      | SOP-AI-014    |

### Deterministic Fail-Safe Matrix
| Failure Scenario                                | System Response                                                                                                                                                                                  | Recovery Path                                                                                                                                                         |
|:------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Agent/Docker Process Hang**                   | **Heartbeat Trip**: Edge Gateway fails to find the 500ms hardware watchdog. Process Asset defaults to hard-coded Safe State.                                                                     | **Auto-Resume**: Process restarts, establishes telemetry sync, requires manual HMI handshake to exit `ORPHAN_MODE`.                                                   |
| **Southbound Driver Failure**                   | **Hardware Watchdog Trip**: Writes cease. The physical process gracefully degrades to local PLC logic.                                                                                           | **Physical Cold-Start**: Requires OT engineer intervention and service restart.                                                                                       |
| **Telemetry "Blackout"**                        | **Blindness Alert**: AI write-access is instantly revoked to prevent acting on stale data.                                                                                                       | **UNS Re-sync**: Restored via Northbound Ingest + 600s observation phase to rebuild the rolling $Z$-score baseline.                                                   |
| **Z-Score Logic Drift**                         | **Advisory Mode**: System detects excessive variance without RAG justification. Reverts to Read-Only.                                                                                            | **Operator Calibration**: Requires human evaluation of the RAG context and manual acknowledgment.                                                                     |
| **PTP Clock Sync Loss (>50ms drift)**           | **Stale Data Exception**: Time-series arrays lose forensic alignment.                                                                                                                            | **Oscillator Holdover**: System fails over to local Stratum 2 rubidium oscillator for up to 24 hours.                                                                 | 
| **WORM Audit Spool at 95% Capacity**            | **Gateway Lockout**: The AI is strictly forbidden from aciting if it cannot log it's reasoning immutably. The Southbound Driver suspends all write operations.                                   | **Log Rotation**: Enterprise SIEM offloads logs, clearing disk space. Manual Operator reset required.                                                                 |
| **Semantic Gatekeeper Violation**               | **Intent Dropped**: AI attempts to write to a forbidden register. Command purged from buffer.                                                                                                    | **Security Alert**: Triggers `SECURITY_VIOLATION_LOCKOUT`. Requires Level 4 administrator audit.                                                                      |
| **Physical Hand-Off-Auto (HOA) Switch to Hand** | **Immediate Load Shedding**: All AI commands are suspended. The local operator takes physical control at the cabinet. Southbound driver instantly purges all AI Intents.                         | **Hardware Return**: Switch returned to `AUTO` at the cabinet, followed by a software acknowledgement on the HMI.                                                     |
| **WAN Connectivity Loss / API Timeout**         | **Safety Fallback**: The middleware intercepts the NIC 1 timeout, automatically spins up the local quantitized model (Phi 4) and enters an air-gapped `MONITOR_ONLY` mode                        | **Network Restoration**: System detects WAN restoration, halts the local model, and restores primary LangGraph routing to Gemini API.                                 |
| **API Rate Limit Exceeded**                     | **Quota Lockout**: High-frequency chatter saturates the 1500 requests per day limit, forcing the system to drop to `ORPHAN_MODE` or triggers a P1 facility lockout.                              | **Failover / Reset**: Automatically fails over to the local edge model (Phi-4) until the cloud quota resets at midnight UTC. Human review required for asset chatter. |
| **Edge Compute Priority Inversion**             | **Resource Capping**: To prevent the LLM reasoning spike from starving the Southbound Driver, strict Linux Cgroups limit the reasoning container to <= 40% CPU/memory.                           | **Process Throttle**: The LLM process execution slows down, but the critical deterministic 500ms heartbeat is preserved.                                              |
| **Main Power Loss (UPS Active)**                | **Minimalistic Mode**: The edge gateway detects main power loss, disables LangGraph reasoning, and focuses exclusively on 1:1 Safety Heartbeats to maintain the process asset link.              | **Cold Boot / Recovery**: Upon power restoration, the system performs an SHA-256 integrity check on all log blocks before resuming autonomous orchestration.          |
| **Message Broker Latency (Backpressure)**       | **Execution Inhibit**: If NATS JetStream latency exceeds 500ms, the system autonomously revokes the Agent's write permissions to Bytewax to prevent dangerous late-action commands.              | **Buffer Clearance**: Once the broker lag drops below the 500ms threshold, write permissions are automatically restored.                                              |
| **Stale Approval Tokens**                       | **Proposal Expiry**: An operator approves a setpoint but fails to **commit**. The Adaptation Proposal TTL expires the token after 24 hours to prevent logic desync between the HMI and the edge. | **Intent Recalculation**: The active token is purged and the AI must calculate a fresh intent payload based on current telemetry.                                     |

See [FMEA & Deterministic Fail-Safe Matrix](/docs/fmea_matrix.md) for more information.

## Governance & Standard Operating Procedures (SOPs)
An SOP suite governs all operational behavior. The Agent is designed to follow these SOPs without deviation, 
ensuring that all interventions are predictable, auditable, and compliant with industrial best practices.   

See [SOP index](/docs/sop) for a full list of SOPs.

### Governance & Control

The learning mechanism is strictly governed to ensure safety and auditability:

- The system **does not directly modify control thresholds**
- All changes are issued as **proposals only**
- All proposed changes are routed through a controlled approval workflow  
  (see [SOP: Threshold Change Management](docs/sop/SOP-AI-003_Threshold_Adaptation_Review_and_Commitment.md)).
- Every proposal is:
  - Logged with a `trace_id`
  - Linked to the originating event and telemetry data
  - Stored for audit and review

### Approval Requirements

All threshold modifications require **explicit human approval** prior to application.

Proposals are categorized by impact:

  - **Low Impact (≤ 5% delta)**:
    - **Action**: Auto-Authorized by the Semantic Gatekeeper.
    - **Governance**: Logged with `trace_id` in the `audit_logs` table.
      - Dashboard Advisory sent to operator
    - **Safety**: The system will not apply changes that exceed the hard-coded `MAX_THRESHOLD` guardrail.
    - **Reasoning**: Statistical fine-tuning should not require human approval to prevent alarm fatigue.
  - **Medium Impact (> 5% and ≤ 15%)**:
    - **Action**: Single Operator Approval.
    - **Governance**: The **Commit** button is enabled on the Dashboard. One authorized user must review the RAG 
      citation and sign off.
    - **Safety**: The system will not allow the change to be applied without a human review.
    - **Reasoning**: This is a moderate change that could affect process stability. A single human review is sufficient.
  - **High Impact (> 15%)**:
    - **Action**: Two-key Validation (Lead Safety Engineer or Plant Manager + Operator)
    - **Governance**: Requires two distinct authorized users and a written **Change Request** (CR) ID.
    - **Safety**: This exceeds the hard-coded `MAX_THRESHOLD` guardrail, so the system will reject any proposal unless
      a supervisor explicitly authorizes it.

### Safety Constraints

- The threshold changes cannot:
  - Override safety limits
  - Affect SIS logic
  - Bypass Interlocks
- Changes are bounded per asset
- Cumulative adjustments are controlled to prevent drift

## Architecture Governance & Standards

**Oversight ICS is Built on the Following Standards**:

| Journey Step	 | Industry Standard                           | Application in OverSight                                         |
|:--------------|:--------------------------------------------|:-----------------------------------------------------------------|
| Extraction    | OPC-UA (IEC 62541), Modbus TCP, EtherNet/IP | Ensuring vendor-neutral data access across multi-brand hardware. |
| Translation   | ISO 23247                                   | Mapping raw hardware signals to semantic JSON Digital Twins.     |
| Publication   | Sparkplug B                                 | Managing stateful MQTT communication (Birth/Death Certificates). |
| Organization	 | Unified Namespace                           | Decoupling AI logic from physical hardware addresses.            |
| Security	     | ISA-99 / IEC 62443                          | Protecting conduits between OT and IT zones via IDMZ placement.  |

## Standards, Compliance & Industrial Alignment

OverSight ICS is built to align with global industrial, controls, and software engineering standards, ensuring that AI 
integration does not compromise the "Safety First" mission of the plant floor. Oversight adheres to the following 
standards:

### Controls Engineering Standards Matrix

For Technical Implementation, see `docs/STANDARDS.md` 

### Software Engineering Standards

**12 Factor Methodology**

- **Factor 1 – Codebase**: Codebase is self-sufficient and maintains one codebase ensuring the system is deployable and 
maintainable across multiple environments, local dev machines, a staging lab PC, and the actual IPC at the plant.
- **Factor 2 – Dependencies**: Oversight doesn't rely on global software dependencies but instead uses a pyproject.toml 
file to manage the Python packages and their versions.
- **Factor 3 – Config**: Oversight strictly separates code from configuration. Sensitive data and site-specific 
  data are stored in environment variables via `.env` file.
- **Factor 4 - Backing Services**: The Agent, Postgres, and NATS JetStream are all managed by Docker Compose. To the 
  Agent, these are "attached resources" that can be swapped out without code changes.
- **Factor 5 – Build, Release, Run**: The system is built using Docker Compose and strictly separates the build, release, 
  and run stages.
- **Factor 6 – Processes**: The FastAPI and LangGraph workers are process-stateless (no data is stored in container
  memory). All state, including LangGraph Checkpoints and the Cognitive Digital Twin, is persisted externally in 
  TimescaleDB and Redis.
- **Factor 7 – Port Binding**: The system doesn't rely on a web server, the app is the server. FastAPI binds to a port 
  and communicates directly with other services via the Docker network.
- **Factor 8 – Concurrency**: The system uses NATS JetStream to distribute tasks. As the plant grows, you spin up more
  workers (horizontal scaling).
- **Factor 9 – Disposability**: The Docker containers are designed to be stateless and disposable. If the Gateway loses 
  power, the services can be restarted in seconds. If a worker crashes in the middle of a task, the message stays in 
  the NATS stream until a new worker takes it over.
- **Factor 10 - Dev/Prod Parity**: By using Docker Compose, the systems local environment is almost identical to the 
  production environment. 
- **Factor 11 – Logs**: The systems logs stream to stdout, not to local .log files. The system uses the **LGTM Stack**
  to collect the streams, allowing the operator to see activity and errors in a central dashboard.
- **Factor 12 – Admin Processes**: Tasks like database migrations and system maintenance are performed via one-time
  Docker commands.

## Predictive Failure & Cognitive Reliability Engineering

### Cognitive Stability KPIs (Key Performance Indicators)
OverSight measures the **Cognitive Health** of the supervisor layer to ensure the AI remains grounded in physical 
reality. The system tracks three primary KPIs:

- **Reasoning Latency (RL)**: The delta between Bytewax anomaly detection ($T_0$) and the publication of a validated 
  intent to the NATS JetStream Command Spine ($T_n$). **Target: <5 s** for Level 3.5 cognitive tasks.
- **Inference Variance**: A metric measuring the stability of the LLM’s reasoning for identical telemetry patterns. 
  High variance across similar events triggers a **Model Trust** alert.
- **Handshake Success Rate**: The percentage of AI-initiated intents successfully acknowledged and executed by the 
  Field Assets within the 2000ms TTL safety window.

### Behavior Modeling (The Cognitive Digital Twin)
Unlike static thresholds, OverSight utilizes **State-Dependent Normalcy** to create a behavioral baseline for every 
Field Asset:

- **State Awareness**: The system distinguishes between **Normal** transients and **Anomalous** deviations. High 
  vibration is ignored during a 5-second motor startup ramp but triggers a critical alert during steady-state operation.
- **Cross-Tag Correlation**: The model monitors the **Correlation Coefficient** between related tags (e.g., RPM vs. 
  Discharge Pressure). A break in this correlation (even if both tags are within their individual safety 
  bounds), indicates a behavioral anomaly such as a sheared shaft or a closed suction valve.

### Anomaly Detection (Statistical vs. Semantic)
OverSight employs a dual-layered detection strategy to catch both electrical and logical failures:

- **Layer 1: Statistical ($Z$-Score)**: High-speed detection of raw signal noise, spikes, or flatlining. This is 
  handled by **Bytewax** in-stream to provide millisecond responsiveness and prevent **Alert Storms**.
- **Layer 2: Semantic (RAG-Driven)**: The Reasoning Agent evaluates the *contextual* meaning of the signal.
    - **Example**: A motor’s temperature rises slowly ($Z < 1.0$), but the Technical Manual (retrieved via pgvector)
      states that at current ambient humidity, this specific curve precedes a winding failure by 4 hours.
    - **Example**: Discharge pressure is stable, but the correlation with VFD frequency has drifted. The Agent 
      identifies this as a potential **pump cavitation** event based on historical logs, a semantic anomaly that 
      statistical methods would miss.

### Model Trust & Explainable AI (XAI)
To eliminate the "Black Box" problem and ensure industrial interoperability, OverSight implements the following 
principles:

- **Traceable Intent**: Every failover intent is stored with an immutable `trace_id` and specific **Source Citations** 
  (e.g., Manual URI, Page, and Paragraph) to justify the intervention.
- **Shadow Mode Commissioning**: During the initial deployment phase, the system operates in observation only. The 
  AI suggests failovers but does not publish to the NATS Command Spine, allowing operators to verify the model's 
  accuracy against real-world process outcomes.
- **Sparkplug B Interoperability**: All system health data and AI intents are egressed via **Sparkplug B** into the 
  Unified Namespace. This ensures OverSight’s cognitive state can be consumed by any Ignition SCADA, ERP, or CMMS 
  without custom drivers.

## Proactive Failover & Control Mechanisms
 
### Complex Event Processing (CEP) & Sensor Fusion
OverSight evaluates anomalies using a **Complex Event Processing (CEP)** approach, fusing multiple data points to 
validate the need for a failover. This prevents false trips while maintaining high sensitivity to mechanical
degradation:

- **Primary Trigger (Statistical)**: Bytewax identifies $Z$-Score drift exceeding $3.0\sigma$ on a primary health 
  metric (e.g., motor vibration).
- **Secondary Validation (Correlation)**: The system cross-references co-occurring anomalies. If vibration increases 
  but motor amperage remains stable, the system flags a **Sensor Fault** rather than a **Mechanical Failure**.
- **Temporal Persistence**: To filter out transient electrical noise, the anomaly must persist for a configurable 
  window (e.g., 4 consecutive 100ms scans) within the Bytewax windowed state to be considered valid.
- **The Weighted Decision Matrix**:
  
Oversight drafts a failover intent by calculating a **Composite Health Score** based on multiple factors:
  - **Confidence Score** = (Statistical Weight) + (Semantic RAG Analysis) + (Historical MTBF Data: Mean Time Between
    Failures).
  - An autonomous failover plan is only drafted if the cumulative **Confidence Score exceeds 85%**.

### Automated Response Mechanisms
Once a trigger is validated, the system moves through a tiered response structure designed to balance autonomy with 
operator oversight:

- **Tier 1: Informational (HMI/UI)**: The dashboard highlights the asset in Amber. No physical state change is permitted.
- **Tier 2: Advisory (Human-in-the-Loop)**: The Agent presents the operator with the specific maintenance manual 
  citation (via RAG) and drafts a final intent. It waits for **Two-Key Authorization** (Operator + Lead Safety Engineer
  or Plant Manager) before executing any transition to the standby asset.
- **Tier 3: Authority Shedding (Fail-to-Local)**: If the $Z$-Score enters the **Critical** zone ($>5.0\sigma$) and the
  operator fails to provide two-key authorization, the system **DOES NOT** take autonomous action. Instead, it sheds 
  authority back to the local PLC logic, which is designed to fail safely (e.g., gradual load shedding, safe shutdown, 
  etc.) while alerting the operator to the critical condition. This is to prevent unpredictable AI behavior during a 
  physical emergency. The Southbound Driver will intentionally suspend its 100ms hardware heartbeat. This will force
  the system into `ORPHAN_MODE`, where the local PLC takes over and the Agent is locked out until the operator can
  assess the situation.

### Control-Theoretic Adaptation
OverSight shapes control signals to maintain process stability during transitions:

- **Feed-Forward Compensation**: When initializing the standby asset, the Southbound Driver calculates the expected 
  pressure drop and pre-emptively adjusts the VFD speed to match the current process demand.
- **Closed-Loop Verification**: The system never "assumes" success. It monitors the **Process Variable (PV)** in 
  real-time. If the PV does not respond within 2 seconds of a command, the Agent enters an "Emergency Escalation" state.
- **Adaptive Thresholding**: The Agent can adjust the sensitivity of its own detection logic based on the plant's 
  operational state (e.g., tightening $Z$-Score thresholds during high-demand peak hours).

### Safety-Critical Constraints
OverSight operates under a **Negative Constraint Model**, ensuring it is physically and logically forbidden from 
violating safety-critical boundaries.

- **The SIS Air-Gap**: OverSight only interacts with the Basic Process Control System (BPCS). It has zero network 
  connectivity to the **Safety Instrumented System (SIS)**. If the SIS triggers a hardware E-Stop, OverSight’s commands 
  are overridden.
- **Interlock Awareness**: Every AI-generated intent is passed through a **Boolean Logic Validator** that enforces 
  hard-coded physical interlocks (e.g., "Inhibit Valve-Open if Pump-Running").
- **Authority Limits (Rate Limiting)**: The Southbound Driver enforces a "Max Change Rate." It will "shred" or 
  throttle any AI command that attempts to ramp equipment (e.g., 0% to 100% speed) faster than its mechanical design 
  limits.

## Evaluation Framework & Success Metrics

### Proof of Value and Performance Benchmarking

Oversight is evaluated using a dual-metric approach, measuring both computational efficiency and industrial impact.

| Category         | Metric                                   | Target Objective                                                                 |
|:-----------------|:-----------------------------------------|:---------------------------------------------------------------------------------|
| **Reliability**  | **MTTR Reduction**                       | 30% faster recovery using RAG-driven RCA (Root Cause Analysis)                   |
| **Productivity** | **OEE Uplift**                           | 10-15% increase in Overall Equipment Effectiveness by avoiding emergency stops   |
| **AI Accuracy**  | **F1 Score**                             | > 0.92 accuracy in distinguishing between "expected wear" and "imminent failure" |
| **Latency**      | **Loop Closure**                         | < 1.2s from $Z$-score anomaly detection to Southbound Driver intent execution    |

Temporal matching window (Δt) ensures fair evaluation

### Target Performance

| Metric              | Target Performance | Architectural Context            |
|:--------------------|:-------------------|:---------------------------------|
| F1 Score            | ≥ 0.92             | Cognitive Realiability Inference |
| False Positive Rate | ≤ 5%               | Anomaly Detection Precision      |
| Detection Latency   | ≤ 120 seconds      | End-to-End event resolution      |

## Key Performance Indicators (KPIs)

### MTTR-A (Mean Time to Recover – Agent)
MTTR-A measures the efficiency of the Reasoning Agent in restoring a process to its **Nominal** state through a 
predictive failover.

- **Calculation**: The time elapsed from the Bytewax windowing engine $Z$-Score crossing the $3.0\sigma$ threshold to 
  the moment the Southbound Driver receives a successful handshake from the standby field asset.
- **Goal**: To minimize this delta to prevent process cavitation or mechanical surges. An MTTR-A of **<10 seconds** is 
  the industrial benchmark for sub-second state resolution.

### NRR (Normalized Recovery Ratio)
This metric evaluates the financial and operational success of an AI intervention compared to a hard-failure event.

- **The Formula**: $NRR = \frac{\text{System Loss (with Agent Intervention)}}{\text{Predicted System Loss (without Agent)}}$
- **Interpretation**:
  - **NRR < 1.0**: The Agent successfully mitigated the impact of the failure.
  - **NRR = 0**: The **Perfect Failover**: The process remained within its nominal operating window throughout the event.
- **Value**: Quantifies the ROI of OverSight by calculating **Damage Avoided** for plant management.

### PEI (Planning Efficiency Index)
This measures the precision of the strategy generated by the LangGraph reasoning nodes. It compares the AI's "Planned 
Path" vs. the "Actual Path" required to reach stability.

- **Calculation**: $PEI = \frac{\text{Minimal Required Commands}}{\text{Total Commands Executed during Failover}}$
- **Optimization**: A low PEI indicates the Agent is looking for a solution. This data is used to refine graph node 
  logic and reduce unnecessary valve/motor cycling.

### Prediction Precision & Recall (Cognitive Confusion Matrix)
To maintain model trust, I applied standard data science metrics to industrial telemetry events:

- **Precision (Reliability)**: The ratio of necessary failovers to total triggered failovers. High precision prevents 
  **Alert Fatigue** and unnecessary equipment wear.
- **Recall (Sensitivity)**: The ratio of predicted failures to actual equipment failures. High recall is prioritized 
  to ensure no catastrophic event goes undetected.
- **Target**: For critical water infrastructure, we maintain a **High Recall Bias**, ensuring safety first while 
  using Bytewax stateful windowing to continuously improve precision over time.

## Performance Benchmarks (KPI Targets)
These benchmarks define the deterministic boundaries of the OverSight reliability layer:

- **Ingest Latency**: <50 ms (Field Asset to NATS JetStream UNS)
- **Anomaly Detection**: <100 ms (Bytewax Stateful Z-Score Windowing)
- **Strategic Reasoning**: 2s–5s (Cloud-based LLM Latency Window)
- **Failover Execution**: <200 ms (Post-Approval Handshake to Physical Output)
- **Telemetry Integrity Check**: <10ms (Verification of NATS stream sequence ID and payload hash)

## Operational Reliability & Safety Guardrails
To ensure production-grade stability in an industrial environment, OverSight ICS implements the following operational 
principles and safety guardrails:

### I/O Fencing & Split Brain Prevention

  - **Stale Write Protection**: The Southbound Driver rejects any command intent that exceeds a 2-second safety window.
    This prevents commands from being executed that do not have the current process reality. 
  - **Distributed Locking**: Uses NATS JetStream consumer groups to implement exactly-once processing. This guarantees 
    only one worker owns the decision-making for a specific field asset at any given time, preventing conflicting 
    failover commands.
  - **Fencing Logic**: The Semantic Gatekeeper prevents hallucinations or unauthorized setpoints that might occur if 
    the network partitioning isolates the Agent from the operational truth.

### Fault Tolerance & Message Integrity
- **Dead Letter Queue (DLQ)**: Any telemetry task that fails validation or causes a reasoning timeouts are moved to a 
  dedicated NATS DLQ. This prevents non-validated tasks from clogging the primary pipeline while preserving them for 
  developer forensics. 
- **At-Least-Once Delivery & Pessimistic Verification**: The Southbound Driver treats every command as "Pending" upon 
  receipt from the NATS Command Spine. It only publishes a `SUCCESS` event to the Unified Namespace (UNS) 
  after validating a physical state change from the field asset.
- **Consumer Prefetch (Backpressure Management)**: By using NATS consumer pull-model flow control, the system prevents
  the Agent from being overwhelmed by telemetry bursts. This lets the Broker buffer the load while the reasoning engine
  completes its inference cycle.
- **Distributed Idempotency**: To prevent **Double-Tap** failover triggers, the Reasoning Agent generates and 
  attaches a unique `correlation_id` to every intent. The Southbound Driver maintains a sliding-window cache of these 
  IDs. Any duplicate ID received within the Command TTL is discarded.
- **System Cold-Boot Protocol (UNS Hydration)**: Upon power-on, the Agent enters a 60-second **Passive-Synchronization**
  state. The system reconciles the Cognitive Digital Twin by reading current hardware registers (via NATS UNS) and
  before resuming orchestration. Autonomous interventions are strictly inhibited until the system achieves 
  **State Sync** to prevent interventions based on stale telemetry.

### Key Security Protocols:
- **Dual-Homed Networking**: The Edge Gateway bridges the isolated OT network and the site network through physically 
  separate interfaces, with OS-level IP forwarding disabled, preventing direct routing paths for external threats. 
- **SIS Air-Gapping**: While the Agent monitors the **Basic Process Control System (BPCS)**, it remains physically and 
logically separated from the **Safety Instrumented System (SIS)**. The Agent cannot interfere with hard-wired safety 
interlocks or Emergency Stops.
- **Unidirectional Telemetry**: Field data is treated as a read-only stream. The only **write** path is the restricted 
  Southbound Driver, which only executes intents after they pass several security and safety validations.

## Risk Mitigation & Failover Strategy

The following table outlines how OverSight ICS maintains Level 3 Autonomy and Purdue Compliance during system stress or 
connectivity loss.

| Failure Event                 | Immediate Impact                                  | Mitigation Strategy                                                                                                                                         | Recovery State              |
|:------------------------------|:--------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------|:----------------------------|
| Level 5 (Cloud) Outage        | LLM Reasoning & RAG Unavailable.                  | **Local Heuristic Mode**: The LangGraph Agent falls back to hard-coded Python safety logic stored in the Edge Gateway.                                      | CP (Consistent/Partitioned) |
| Network Partition (L3 to L4)  | Next.js Dashboard goes "Offline."                 | **Autonomous Execution**: The Agent continues to monitor Level 2 telemetry and executes failovers via NATS; logs are queued for the UI.                     | Headless Operation          |
| Database Corruption           | Historian unavailable.                            | **Emergency Bypass**: The Southbound Driver detects a heartbeat failure and enters a **Hold Last State** mode to prevent erratic hardware commands.         | Safe State (Frozen)         | 
| AI "Logic Drift"              | Reasoning Engine generates low-confidence intent. | **Pydantic Validation**: The Validator identifies the schema or range violation and **Shelves** the command, alerting the operator for manual intervention. | Human-in-the-Loop           | 
| Primary Hardware (L1) Timeout | Telemetry stream stops.                           | **Predictive Heartbeat**: The Ingest Service triggers an immediate **Loss of Signal** event, prompting the Agent to initiate the Standby Failover sequence. | Hot Standby Active          |

## Observability & Post-Mortem Traceability
To ensure absolute accountability for AI-driven decisions, the system maintains a comprehensive audit trail:

- **Reasoning Traces**: Every LangGraph cycle is logged with its full context, including which PDF manual pages were 
referenced during the decision.
- **Structured Audit Logs**: All hardware writes from the Southbound Driver are logged in a WORM compliant table, 
  including the original AI `trace_id` and the later Hardware `ACK_ID`.
- **Performance Metrics**: Real-time tracking of LLM token usage, reasoning latency, UNS relay speed via Prometheus and
  Grafana.

## Data Engineering, Observability, & Telemetry

### Data Reliability Stack
OverSight ICS treats data as a physical asset. The system uses a **Pessimistic Data Pipeline** to make sure that the AI 
never makes a decision based on stale telemetry.

- **Unified Namespace (UNS)**: By utilizing **NATS JetStream** as the central nervous system, state changes 
  in the Cognitive Digital Twin and telemetry from the field are globally synchronized. This replaces traditional 
  database polling with a real-time, event-driven state authority.
- **Idempotency & Fencing**: Every telemetry packet and agent intent is tagged with a unique **UUID and Fencing Token**. 
  This prevents **Double Execution** if a network glitch causes a message to be redelivered or if a stale command 
  is consumed after its TTL has expired.
- **Quality Stamps & Sparkplug B**: Following **Sparkplug B** standards, every data point includes a Quality Metric 
  (Good, Bad, or Uncertain). If a sensor's quality flips to "Bad," the Bytewax stream processing engine automatically
  isolates that stream and notifies the Agent of **Sensor Blindness** to prevent false failover triggers.

### Observability Pipeline (The "Black Box" Recorder)
OverSight uses the **LGTM Stack** (Loki, Grafana, Tempo, Mimir) to provide a forensic audit trail for every AI 
intervention.

- **Structured Logging (Loki)**: All logs are output in JSON format, indexed by `asset_id` and `trace_id`. This allows 
  operators to reconstruct the exact telemetry frame the AI was viewing at the millisecond it decided to intervene.
- **Distributed Tracing (Tempo)**: Oversight propagates a `trace_id` from the initial sensor excursion in NATS, through 
  the LangGraph reasoning engine, and down to the Southbound Driver's physical handshake. This provides a complete 
  **Reasoning-to-Action** map for post-incident reporting.
- **Health Metrics (Mimir/Prometheus)**: Tracks system-level KPIs such as **NATS Consumer Lag**, **Bytewax Buffer 
  Saturation**, and **Scan-Cycle Jitter**. If backpressure rises, Prometheus triggers a **System Health** alert before 
  telemetry data is dropped.

### Context Injection (The RAG Engine)
This layer transforms raw data (e.g., "Motor_01 Current = 45A") into actionable intelligence (e.g., "Motor_01 is running 
10% above its rated efficiency").

- **Vectorized Metadata**: Technical specifications and PDF maintenance manuals are indexed locally in **pgvector**, 
  allowing for rapid retrieval of hardware-specific engineering limits.
- **Similarity Search**: When an anomaly is detected, the system performs a similarity search using the `asset_model` 
  and `failure_mode` to find the relevant ground truth within the technical documentation.
- **Static vs. Dynamic Context**:
    - **Static**: The Agent is injected with hard-coded plant constraints (e.g., "Pump A and Pump B cannot run 
      simultaneously due to pipe diameter limits").
    - **Dynamic**: The rolling $Z$-score and the 5-minute telemetry window are injected into the LLM prompt as a 
      structured Markdown table, providing the "Brain" with the trend line required for predictive reasoning.

## Implementation, Validation, & Simulation

### Chaos Testing & Resiliency (Field Stress Test)

Following the principles of Chaos Engineering, Oversight intentionally injects faults into the Edge Gateway so the 
system fails gracefully and defaults to its deterministic **Safe State**.

- **Network Partitioning**: Oversight simulates a **Cloud Blackout** between the Edge Gateway and the LLM. We validate 
  that the NATS JetStream Store-and-Forward buffer holds the telemetry and that the local Southbound Driver maintains
  the process heartbeat without interruption.
- **Process Noise Injection**: Oversight injects **Synthetic Jitter** and electrical noise into the telemetry stream so 
  that the Bytewax $Z$-Score engine distinguishes between minor interference and true mechanical drift.
- **Resource Exhaustion**: Oversight artificially spikes CPU and RAM on the Reasoning Agent container. This verifies 
  that Docker Cgroup limits successfully protect the Southbound Driver. The "thinking" (AI) never starves 
  "acting" (IO) of resources.

### Evaluation Frameworks (Oversight AI Reporting)
Because LLMs are inherently non-deterministic, OverSight uses a rigorous Evaluation Framework to score reasoning cycles 
before they are authorized to issue commands.

- **Deterministic "Golden Set" Tests**: The Agent is regularly tested against a library of historic incident logs with 
  known "correct" outcomes. The Agent must reach the safe conclusion (e.g., "Inhibit Pump A") with 100% accuracy to 
  pass.
- **Semantic Similarity Scoring**: Oversight uses an NLP-based (Natural Language Processing) scorer to verify that the 
  Agent's reasoning aligns with the specific technical vocabulary and safety protocols defined in the pgvector document 
  store.
- **Safety Guardrail Audit**: Every failover intent is passed through a Hard-coded Logic Validator. If the AI suggests 
  a step that violates a physical interlock (e.g., "Start Pump while Intake Valve is closed"), the Eval fails 
  immediately, and the command is blocked by the Semantic Gatekeeper.

### Monte Carlo Simulation (RUL & What-If)
To move from "Reactive" to "Proactive," Oversight uses Monte Carlo Simulations to model thousands of possible 
future states for the field assets.

- **Remaining Useful Life (RUL)**: By running 10,000+ simulations of asset wear using current Z-Score drift as a 
  baseline, Oversight calculates RUL with a 95% confidence interval. 
- **Catastrophic "What-If" Modeling**: Oversight simulates multi-asset failure scenarios (e.g., concurrent pipe burst 
  and power dip) to observe how the Cognitive Digital Twin prioritizes failovers under extreme stress.
- **Transfer Optimization**: By simulating varied ramp speeds for standby assets, the Monte Carlo engine identifies the 
  "Sweet Spot" for the Bumpless Transfer, minimizing the mechanical torque and water hammer associated with the 
  transition.

## Implementation & Data Engineering

To maintain the integrity of the Cognitive Digital Twin, raw telemetry undergoes a multi-stage conditioning pipeline 
before reaching the Reasoning Engine:

- **Signal Normalization**: The Northbound Ingest Service maps heterogeneous vendor tags (Modbus registers, OPC-UA nodes, 
  EtherNet/IP tags) into a standardized Sparkplug B Payload. All data entering the system is state-aware and 
  deterministic.
- **Deadband & Hysteresis**: To prevent MQTT bus saturation and **Chatter**, signals are only published to the UNS if 
  they exceed a predefined variance threshold (e.g., >0.5% change) or historical state buffer.
- **Outlier Suppression**: A rolling-window **Median Filter** and **Bytewax** suppression is applied to 
  high-vibration sensor data to prevent transient electrical noise or spikes from triggering false-positive $Z$-score 
  anomalies.
- **Windowed $Z$-Score Grounding**: **Apache Flink** performs real-time statistical grounding on the live stream, calculating
  the $Z$-Score of each field asset to identify logic drift or mechanical wear before data is committed to the 
  database.
- **Historian Tiering**: Data is automatically partitioned into "Hot" (Redis – 1 min), "Warm" (TimescaleDB – 90 days),
  and "Cold" (S3/Blob - Long-term) for efficient RAG retrieval.
- **Store & Forward Egress**: To maintain sub-second reliability with using the LLM, the system uses a
  Store-And-Forward buffering strategy to ensure the LLM can respond to the latest telemetry data even during network
  instability.
  - **Local-First Store**: 100% of high-velocity telemetry (L0-L2) is ingested and stored locally in the TimescaleDB 
    Historian. This allows for no data to be lost if the internet connection to the LLM provider is interrupted. 
  - **On-Demand Forward**: Instead of constant streaming, the system forwards compressed batches of relevant
    historical data to the LLM only when the $Z$-Score anomaly triggers a reasoning cycle.
  - **Resilience**: If the LLM API is unavailable, the system stores the incident state locally and retries the
    forward once the connection is restored, allowing for a retrospective Root Cause Analysis (RCA) even after a 
    disruption.

## The Event-Driven Command Spine

To bridge high-level AI reasoning with low-level industrial protocols, OverSight uses an 
**Event-Driven Command Spine** (RabbitMQ) coupled with a **Stream-Processing Layer** (Apache Flink). This replaces 
traditional database polling with a real-time, low-latency push model.

### Read / Write Separation
   - **Read Path**: Telemetry flows unidirectionally up from the field level through the Northbound Ingest and into the 
     NATS JetStream telemetry UNS. The Bytewax streaming engine calculates windowed $Z$-score logic drift and 
     publishes it back to the UNS. The telemetry data is localized in a Redis hot cache for low-latency access by the 
     AI Agent and the Level 4 Dashboard.
   - **Write Path**: Control intents flow down from the AI Agent through a separate gated hardware-verified pipeline 
     and then to the Semantic Gatekeeper for validation. 
     - **Level 1.5 Arbitrator**: The AI intent is intercepted here to enforce the **Two-Key Protocol**, confirm valid
       cryptographic RS256 JWT signatures from the required human operator(s) before routing the authorized payload to
       the **Semantic Gatekeeper** for physical bounds validation. The Arbitrator is also responsible for the 
       **Write-Access Token** and enforcing the **Bumpless Transfer Protocol**.
     - **Semantic Gatekeeper (Level 3.5 IDMZ)**: The authorized payload is audited against physical constraints and 
       hardcoded **Forbidden Register List** to make sure the command is valid, safe, and compliant.
     - **Cryptographic Notary Buffers & TPM 2.0**: The approved payload drops into the notary buffers, where the TPM
       2.0 module is invoked. It then attaches an immutable root-of-trust signature to the data block to guarantee
       non-repudiation.
     - **Southbound Driver**: Only after the payload is hardware-signed does the high-priority C++ Southbound Driver
       receive the command for execution. The driver pushes the command across the Southbound Firewall to the PLC and
       triggers a **Pessimistic Confirmation of Success** on the UI only after the standby hardware acknowledges the 
       state change from the physical feedback loop.
     
### I/O Fencing & Determinism
To prevent the system from acting on outdated or hallucinated information, we implement three layers of fencing:

1.  **TTL (Time-To-Live) Envelopes**: Every intent published to the NATS Command Spine is tagged with a 
    `message-ttl` of 2000ms. If the Southbound Driver consumes a message older than this window, it is automatically 
    discarded as **stale**.
2.  **Fencing Tokens (Sequence Tracking)**: Every command intent carries a unique `intent_id`. 
    - **Hardware Rejection**: The Southbound Driver writes this ID to a dedicated **Command Serial** register on the 
       field asset. 
    - **The Logic**: The hardware logic is programmed to reject any incoming serial ID that is not greater than the 
      last successfully executed ID, preventing replay attacks or network-induced duplicate executions.
3.  **Trace ID Enforcement**: No command is accepted by the Southbound Driver unless it carries a valid `trace_id` 
    header, linking the physical action directly to a specific AI reasoning cycle in the audit logs.

### Implementing a Digital Twin Simulation Environment
The Digital Twin Simulation Environment is a high-fidelity **Field Asset Simulator (PAS)** used for logic validation.

- **Non-Invasive Testing**: The environment allows for the testing and development of AI-driven failover maneuvers and 
  $Z$-score threshold tuning without risking physical equipment or process uptime.
- **Scenario Injection**: It allows for the simulation of **Black Swan** events, such as catastrophic pump failure or 
  sensor drift to verify that the Reasoning Agent and Southbound Driver respond deterministically.
- **Trade-off**: While maintaining a PAS increases architectural overhead, it is a safety requirement for industrial 
  commissioning (SAT-04) to ensure the AI's thought translates to safe physical movement.

### Competing Consumer Pattern
NATS JetStream is utilized as the **Command Spine**, allowing horizontally scalable LangGraph workers to process plant 
intents.

- **Horizontal Scalability**: As plant complexity or the number of field assets grows, additional worker containers can 
  be spun up to pull from the command queue without logic changes.
- **Fault Tolerance**: If a worker fails mid-reasoning, the message is returned to the queue (via NACK) for another 
  worker to process, ensuring no intent is lost.
- **Trade-off**: This pattern requires strict **Idempotency**. Oversight uses fencing tokens and a unique `trace_id` to 
  ensure that at-least-once delivery does not result in duplicate physical executions on the field assets.

### Event-Driven Architecture & Reactive Design
The system is built on a **Push-Based** model rather than a polling model, ensuring sub-second responsiveness to process 
excursions.

- **Reactive Stream Processing**: The Bytewax streaming engine reacts to raw NATS telemetry in real-time, performing 
  windowed math to detect drift before the Agent even begins its reasoning cycle.
- **Loose Coupling**: Components communicate exclusively through the **Unified Namespace (UNS)**. The Southbound Driver 
  and Reasoning Agent have no direct dependencies, allowing either to be updated or restarted without crashing the 
  control loop.
- **Trade-off**: This architecture introduces complexity in managing message ordering. We address this through 
  **Sequence Tracking** and **Time-To-Live (TTL)** envelopes on every event.

### Unidirectional Data Flow
OverSight enforces a strict unidirectional flow to maintain a "Clear Chain of Command" and prevent feedback loops.

- **Safety Gatekeeping**: The Reasoning Agent cannot directly access field assets. It must write "Intents" to the 
  RabbitMQ Spine, which are then intercepted, audited, and executed by the Southbound Driver's semantic firewall.
- **Simplicity & Auditability**: By enforcing a one-way flow (Observation → Reasoning → Intent → Execution), we 
  ensure that every physical change has a traceable, linear cause in the audit log.
- **Trade-off**: This adds a layer of indirection and minor latency, but it is a critical safety barrier that prevents 
  the AI from bypassing hard-coded physical constraints.

### Retrieval-Augmented Generation (RAG)
The system utilizes RAG to ground the Agent’s reasoning in manufacturer-specific PDF manuals and historical site logs.

- **Hallucination Mitigation**: The Agent is strictly required to provide citations (URI/Hash) from the local 
  **pgvector** store before a failover maneuver is authorized.
- **Ground Truth Logic**: By providing specific excerpts from manufacturer technical data, the system ensures the AI 
  suggests setpoints that are physically compatible with the hardware's engineering tolerances.
- **Trade-off**: RAG ingestion adds processing time to the reasoning cycle, but this is a necessary compromise to
  ensure the Agent remains a "Domain Expert" rather than a general-purpose (and hallucination-prone) LLM.

### Unified Namespace (UNS) Architecture

The **Unified Namespace (UNS)** Architecture acts as a centralized source of truth for Oversight. By implementing this
architecture, the system decouples the AI Agent from the specific physical hardware configurations. While the system
uses industrial protocols (Modbus TCP, OPC-UA, EtherNet/IP) for hardware interoperability at the edge, the UNS provides
a standardized interface for the system to remain vendor-agnostic and horizontally scalable. Oversight is designed to 
be a "Universal" reliability system that can be used at multiple industrial plants. 

It treats the plant like a single, cohesive **Semantic Hierarchy**. By using a single namespace, the system eliminates 
the friction of vendor-specific silos.

- **ISA-95 Semantic Mapping**: All field asset telemetry, state changes, and AI interfaces are mapped to a 
  standardized, hierarchical namespace (e.g., `Site/Area/Process_line/Asset/Attribute`).
- **Local Hardware Abstraction**: The UNS acts as an extraction layer, allowing the reasoning cluster to interact with
  asset health or operational efficiency nodes without needing to manage specific register addresses or proprietary 
  protocol drivers for field assets.
- **Horizontal Scaling**: This extension allows the system to scale across multiple industrial plants. Adding a new
  field asset or a new AI diagnostic agent only requires a new branch in the UNS.  The core architecture remains 
  untouched.
- **Dual Broker Implementation**:
  To handle the complex nature of industrial reliability and high-velocity data, the UNS is distributed across two brokers, 
  each handling a specific direction of data flow:

#### Architecture Benefits
- **Hardware Abstraction**: The AI Agent only interacts with the UNS and is hardware-agnostic.
- **State vs. Intent**: Because both NATS and Bytewax share the same naming convention, the system maintains a
  perfect **Digital Twin** where telemetry and commands are always in sync.
- **Scalability**: Both NATS and Bytewax can scale horizontally to handle increased data volumes and command loads as 
  the plant grows or during peak operational periods. 

## Operational Technology (OT) Safety Logic
To prevent "water hammer," mechanical shock, or pressure surges during a failover, the Southbound Driver enforces a
**Deterministic Handshake** for asset transitions:

1.  **Command Standby**: Initialize the Standby Asset at its minimum safe operating frequency.
2.  **Monitor State**: Wait for "At Speed" and "Ready" feedback from the field hardware.
3.  **Bumpless Transfer**: Simultaneously ramp the Standby frequency up while ramping the Primary down. During this 
    phase, the Standby PID pre-loads the active I-term from the Primary to prevent a pressure dip.
4.  **Isolate & Shed**: Once the Standby reaches the required setpoint, the Primary Asset is commanded to "Stop" and 
    isolated for maintenance.

## Store-and-Forward Contextual Edges
To mitigate the risks of high-latency cloud LLMs and network partitions, OverSight implements a **Store-and-Forward**
mechanism with **Contextual Edge** prioritization.

- **Local Persistence (The Edge Authority)**: 100% of telemetry is persisted in the local **TimescaleDB** instance. 
  This creates an authoritative "Source of Truth" at the edge, ensuring data integrity even during total cloud 
  disconnects.
- **Stream-Aware Backpressure**: **Bytewax** monitors the ingestion rate from field assets. If downstream 
  consumers lag, Bytewax applies backpressure to the ingest layer, slowing processing to match capacity without losing a 
  single event.
- **Data Storm Resilience (NATS Buffer)**: During telemetry surges or "Data Storms," the Northbound Ingest Service 
  uses **NATSa** as a disk-persistent local buffer. This prevents memory exhaustion and protects the **NATS Command 
  Spine** from being overwhelmed.
- **Prioritized Context Edges**: If network bandwidth is saturated, the Gateway prioritizes **Control-Critical Tags** 
  (Alarms, E-Stops, Motor States) over **Diagnostic-Critical Tags** (Vibration, Temperature). This ensures the AI 
  maintains safety-critical situational awareness during a surge.

## Twelve-Factor Methodology at the Edge
The system is built for industrial "Disposability" and rapid deployment:

- **Config**: Site-specific parameters (IP addresses, Register Maps, Asset IDs) are injected via `.env` files. No 
  site-specific logic exists in the codebase.
- **Disposability**: Services are containerized and stateless. The system is designed for **Cold Start** recovery, 
  moving from power-on to AI-supervised control in under 45 seconds.
- **Concurrency**: By using the **Competing Consumer Pattern**, the Gateway scales its reasoning power horizontally
  simply by spinning up additional worker containers.

### Data Source Mapping:

- **Telemetric (IoT/PLC)**: Real-time Modbus registers, EtherNet IP tags and OPC-UA nodes (current, pressure, flow, 
  temperature, vibration, etc.) are ingested at 1-second intervals.
- **Operational**: Heartbeat signals and state machine flags.
- **Unstructured (Maintenance)**: PDF Manuals, historical CSV incident logs, and operator shift nodes indexed via 
  `pgVector`.

## Data Storage

I used **TimescaleDB** to transform PostgreSQL into a high-performance time-series engine. This allows OverSight to 
maintain an ACID-compliant environment that handles billions of telemetry points alongside structured relational data, 
such as RAG metadata and audit logs.

- **The Unified Data Strategy**: By using TimescaleDB, the system avoids the complexity of "polyglot persistence" 
  (managing separate NoSQL and Relational DBs). Instead, it uses a single database instance to store both high-velocity 
  sensor data and structured operational context.
- **Hypertable for Scalability**: I implemented Timescale **Hypertable** to automatically partition incoming Sparkplug 
  B and OPC-UA data by time. This ensures that ingestion speed remains constant and prevents the performance degradation
  typically seen in standard B-tree indexes as datasets grow.
- **Native Columnar Compression**: To optimize storage costs, I implement automated compression policies. After 7 days, 
  telemetry rows are transformed into a columnar format, achieving a 90%+ reduction in disk footprint while 
  significantly speeding up the analytical queries used by the Reasoning Agent for historical trend analysis.
- **Relational Context (RAG & WORM Logs)**: Because the system remains natively PostgreSQL, the **pgvector** RAG 
  documents and the WORM (Write-Once-Read-Many) audit trails live in standard relational tables. This allows the Agent 
  to perform high-performance SQL joins between real-time anomalies and technical manual specifications in a single 
  reasoning cycle.
- **Tiered Storage Architecture**:
    - **Hot Layer (NATS UNS)**: Real-time, sub-100ms state for the Southbound Driver and HMI.
    - **Warm Layer (TimescaleDB)**: 30-day window of uncompressed telemetry for high-resolution AI reasoning and Bytewax
      state rehydration.
    - **Cold Layer (Compressed Hypertable)**: Multi-year regulatory archive for EPA 40 CFR compliance and long-term 
      forensic Root Cause Analysis (RCA).
- **The Trade-off**: While TimescaleDB introduces specific maintenance requirements (such as managing hypertable 
  chunks), it is a necessary architectural choice. It provides the industrial-grade consistency (ACID) required for 
  regulatory compliance and historical analysis, while also delivering the performance and scalability needed to handle 
  high-frequency telemetry data in real-time.

## Database Compaction & Tiered Retention
To prevent storage exhaustion and maintain sub-second query performance, I implemented a strict Tiered Retention Policy:

- **L1 (Raw Data – "Hot")**: Telemetry is stored at full resolution for 7 days. This provides a high-resolution 
  forensic "Black Box" for immediate Root Cause Analysis (RCA).
- **L2 (Compressed - "Warm")**: After 7 days, data is passed through a **Columnar Compression Policy**. It is 
  aggregated into 1-minute averages ($\mu$ and $\sigma$) and stored for 90 days for trend analysis and AI context.
- **L3 (Permanent - "Cold")**: Critical incident logs, failover events, and EPA-relevant effluent data are moved to 
  standard relational tables and stored indefinitely. This ensures compliance with **EPA 40 CFR** and **ISO 27001** 
  standards without bloating the time-series hypertables.

## WORM Storage & WORM Spooling 
The WORM log is stored on the NVMe (Non-Volatile Memory Express) drive of the Edge Gateway, and uses strict disk 
capacity enforcement thresholds to prevent overflow. If the connection to the Corporate IT server is down, the system
will not be able to upload its mandatory audit logs. To prevent any data loss, the Edge Gateway saves (spools) all of 
high-speed telemetry and AI reasoning traces locally onto its own NVMe drive.

### WORM Spooling Capacity Thresholds

- **80% Capacity Threshold**: The system will issue a local HMI `AUDIT_SPOOLING_WARNING`, even though the system 
  continues to operate normally. Reaching the 80% mark triggers a warning on the dashboard.
- **90% Capacity Lockout Threshold**: The Edge Gateway is designed to hold exactly 3 days worth of trapped un-audited
  telemetry and AI reasoning traces before it gets full. If the WORM storage reaches 90% capacity, a safety lockout is
  triggered. The Southbound Driver is immediately suspended, and the system defaults to `ORPHAN_MODE`(a safe state 
  where requiring manual control) which stops the Agent from executing any un-auditable actions.

## Data Management & Retention
As a high-frequency telemetry system, OverSight manages the high-level of data generated by the field instrumentation 
or control hardware (PLCs, VFDs) using a 4-tier architecture. 

| Tier         | Technology         | Retention  | Purpose                                                       |
|:-------------|:-------------------|:-----------|:--------------------------------------------------------------|
| **Hot**      | Redis              | 60 Seconds | Real-time WebSocket streaming to the Mantine Dashboard.       |
| **Warm**     | TimescaleDB        | 90 Days    | Active Z-Score calculation and Recent Incident RCA.           |
| **Cold**     | Compressed S3/Blob | 1 Year+    | Long-term trend analysis and regulatory compliance reporting. |
| **Metadata** | PostgreSQL         | Permanent  | Asset registry, safety whitelists, and LangGraph checkpoints. | 

## Deployment, Operations, & Maintenance

### LLMOps: CI/CD for Cognitive Reliability
We implement a rigorous LLMOps pipeline to ensure model updates never compromise process availability.

- **Prompt Versioning & Registry**: Prompts are treated as production code. Every change to a LangGraph node's 
  prompt must be versioned in Git and pass the **Evaluation Framework** tests before deployment.
- **Shadow Mode (A/B Testing)**: New reasoning agents are deployed in "Shadow Mode" to process live telemetry from 
   the NATS UNS. Their intents are recorded and compared against the production agent, but they are physically 
   inhibited from publishing to the NATS Command Spine.
- **Semantic Drift Alarms**: If the system detects a variance in reasoning confidence for similar telemetry patterns, 
   it flags the incident for human review to retrain local RAG embeddings and prevent model decay.

### State Management & Rehydration
State in OverSight is decoupled from individual containers to ensure 12-Factor statelessness and rapid recovery.

- **Distributed State (NATS/Timescale)**: All "In-Flight" reasoning and graph states are persisted. If a container 
  restarts mid-failover, it rehydrates its state from NATS snapshots and resumes from the last validated node.
- **The Digital Twin State**: A real-time representation of the plant's physical state is maintained in the 
  **NATS Unified Namespace (UNS)**. This includes "Remote Authority" status, $Z$-scores, and active physical interlock 
  locks.
- **Conflict Resolution (Pessimistic Locking)**: If an operator manually overrides a field asset while the AI is in a 
  reasoning cycle, the system detects the state mismatch via the UNS. The AI immediately aborts its intent and triggers 
  a "Manual Override Detected" alert.

### Edge Hardware & Lifecycle
OverSight is optimized for high-performance Industrial PCs (IPCs) on the plant floor.

- **Resource Isolation (Docker Cgroups)**: Oversight applies strict hard-limits on CPU and Memory for the "Reasoning Engine." 
  This prevents an LLM context-window expansion from starving the **Southbound Driver** of the cycles required to 
  maintain the PLC Heartbeat.
- **Zero-Touch Provisioning (ZTP)**: Technicians can swap a failed Edge Gateway by simply inserting a secure USB drive 
  containing the site-specific `.env` configuration. The system automatically pulls signed Docker images and 
  reconstructs the local UNS.

## Scaling Strategy 

**Horizontal Scaling**: The system leverages NATs consumer groups to distribute the telemetry of thousands of 
sensors across multiple Edge Gateway nodes.
**Federated Learning**: Moving towards a model where local Edge nodes share anonymized failure patterns to improve the
global RAG database without compromising privacy.


### Scalable Reasoning
The LangGraph Agent is designed to be scalable and fault-tolerant, allowing it to handle complex planning and reasoning 
tasks without crashing the system. RAG (Retrieval-Augmented Generation) is used to provide the Agent with specific 
context from manuals and logs, which helps to reduce hallucinations and improve decision-making.

   - **Stateful Orchestration**: LangGraph manages the reasoning lifecycle, using database checkpoints to ensure that 
   if the gateway restarts mid-failover, the Agent resumes exactly where it left off.
   - **RAG Context**: The Agent uses Retrieval-Augmented Generation to fetch information from manuals and logs,
   providing grounded context for its decisions and reducing hallucinations.
   - **Trade-off**: This pattern allows for complex, stateful reasoning while ensuring that the system can recover 
   gracefully from failures. The trade-off is that it introduces some latency due to the need to fetch context and 
   manage state. However, this is necessary to ensure that the Agent's decisions are knowledgeable and that the system 
   can handle real-world industrial complexity without crashing or making unsafe decisions.

## Conclusion

Oversight ICS is a comprehensive solution for transitioning Legacy OT systems with AI Agents and represents a 
fundamental step for how we approach industrial reliability. By moving from "If-Then" alarm thresholds to a stateful, 
cognitive supervision system, we can transition plants OT Legacy systems to a more "AI-driven" approach. By moving to
an "If-This-Then-That" logic, we effectively eliminate the need for unplanned downtime and reduce the risk of human 
error. 

This system ensures that the plant's safety is not compromised by AI-driven decisions, and it provides a clear path to 
improving the safety of the plant's equipment. If a catastrophic failure occurs, the system doesn't panic, but instead 
proactively takes steps to prevent further damage and then provides a clear path for recovery. It represents a 
deterministic, auditable, and AI-driven assistance approach to the plant's safety.

Oversight ICS doesn't just monitor the process, it understands it and can take proactive steps to keep it running 
safely and efficiently. By implementing this system, we can ensure that the plant's operations are not only more 
reliable but also more resilient to unexpected events, ultimately leading to safer working conditions and improved 
productivity.

## Stakeholder FAQs

## Documentation Index
