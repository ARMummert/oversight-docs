# Oversight ICS: Technical Systems & Design Architecture Report

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

## Key Architectural Features

### Safety First Predictive Failover Engine 
  Unlike legacy **Heartbeat** monitors that only trigger after a hardware crash, OverSight operates as a supervisory
  reliability layer. It monitors internal telemetry such as scan-time jitter, memory leaks, and CPU thermal-spikes, to 
  orchestrate a graceful transition while the primary hardware is still operational, without ever replacing the primary
  control loop.

### Cognitive Digital Twin (NATS JetStream + Bytewax)
  The system maintains a real-time cognitive digital twin of the field instrumentation, control assets, 
  (PLC/RTU/DCS), and rotating equipment. Oversight executes continuous windowed $Z$-Score drift analysis by streaming
  telemetry through a single instance of NATS JetStream and Bytewax. This allows the system to detect logic drift 
  before it causes a physical hardware failure.
> **Strategic Orchestration**: Oversight doesn't just predict values; it plans a sequence of actions, validates them 
> against a safety schema, and executes a managed failover plan.

### Command Query Responsibility Segregation (CQRS) Architecture
  Oversight implements a strict **CQRS** architecture to separate the **Read Path** from the **Write Path**. The 
  **Read Path** is a high-velocity telemetry stream that is ingested into the system, processed, and stored in a 
  Redis Hot Cache and TimescaleDB historian. The **Write Path** is strictly controlled by the Level 1.5 Control 
  Authority Model, which requires cryptographic verification, dual-authorization, and hardware TPM signing before 
  executing any write commands to the Level 1 physical assets.

### Event-Driven Architecture (NATS JetStream)
  The system is built on an event-driven architecture that allows for real-time processing of telemetry data. It uses 
  NATS JetStream as the command spine for event streaming, allowing for high-throughput, low-latency communication. 
  This architecture enables the system to react to events as they occur.

### Remaining Useful Life (RUL) Prognostics
  By correlating thermal stress and vibration cycles, the agent provides a **Days To Failure** estimate. This switches 
  maintenance from scheduled maintenance to condition-based maintenance. This allows field assets to be replaced during 
  planned downtime rather than emergency shutdowns.

### Multi-Worker Scalability
  Oversight uses **NATS JetStream Consumer Groups** and **Bytewax data parallelism**, which allows the system to scale
  reasoning and stream processing across many workers (n-workers) in a lightweight cluster architecture.
  - **Hot Swapping**: Additional reasoning nodes can be added to the cluster without downtime as plant complexity 
    increases.
  - **Subject-Based Routing & Partitioning**: Telemetry streams are partitioned so that all messages for a specific
    asset are routed to the same worker. This prevents race conditions and maintains strict sequential state.

### Split Brain & Arbitration Strategy
  - **Distributed Locking (NATS KV)**: Oversight uses the **NATS Key-Value(KV) store** to manage state recovery and 
    distributed locking. This means that only one worker can be the primary decision-maker for a specific asset at any
    given time. This prevents conflicting failover commands.
  - **Physical Priority Arbitration**: The system **shed loads** (stops all AI commands) the instant a physical human 
    override is detected, ensuring the human operator always has a final authority.

### Closed-Loop Intent Verification
  The Southbound Driver does not assume success upon sending a command. It waits for **Process Feedback Validation**
  (e.g., a limit switch closure or a current-draw increase observed via the NATS stream) and writes the confirmation 
  back to the database to close the **Intent-to-Verification** pessimistic loop.

### Hardware Root of Trust 
  Oversight uses a **TPM 2.0 hardware module** to cryptographically sign every command before it can be executed. This 
  proves to the **Southbound Driver** that the payload met all safety audits and was not tampered with in transit.

### Zero-Trust Safety Boundary - Semantic Gatekeeper
  Before any intents are sent to the physical process assets, they must pass through a strict semantic firewall. The
  gatekeeper audits every command against physical bounds limits and hard-coded **Forbidden Register Lists**. This is
  critical so that the AI can never override safety critical **SIS** logic.

### Level 1.5 Control Authority Model & Two-Key Protocol
  For high-impact interventions, the system pauses at the arbitrator (Level 1.5), and stays paused in isolated memory
  buffers until Multi-Factor Authentication (MFA) is provided by both the primary operator and either the lead safety
  engineer or the plant manager. Multi-Factor Authentication is provided by two independent RS256-signed JWT tokens.

### IDMZ Purdue Model Alignment
  To prevent the AI from compromising the deterministic control loops, vector databases, all reasoning, and stream
  processing, Oversight is architected to be strictly air-gapped inside the Level 3.5 IDMZ zone of the Purdue Model.

### WORM Forensic Logging
  To comply with strict industrial compliance (e.g., IEC 62443, EPA), every $Z$-Score anomaly, reasoning trace, and 
  operator acknowledgement are permanently logged to a WORM (Write Once Read Many) storage system. If the WORM drive
  reaches 95% capacity, the system auto-triggers a safety lockout, enforcing the rule: **If an AI cannot securely log
  its reasoning, it cannot be trusted and cannot act**.

### Local RAG-Grounded Reasoning
  The Agent is grounded to a local, air-gapped pgVector database containing the OEM manuals and digitized Standard
  Operating Procedures (SOPs). Every failover proposal **MUST** cite the manufacturer's technical documentation.

### Pessimistic UI State Machine

**The dashboard provides a pessimistic confirmation to ensure it never confuses the operator 
  about the state of the system.**

- When a failover is initiated, the system will switch to a "Pending" state until the Southbound Driver confirms the 
  hardware has taken control. 
- The operator will only receive a "Success" confirmation after the Southbound Driver observes the new physical reality
  on the NATS telemetry stream.
- The dashboard's state machine is designed to provide a visual representation of the system's current physical state. 
  This allows the field assets to be in only one state at a time. This prevents sending commands to the 
  hardware while they are already running one. 

> **Example**: If a VFD is STOPPED, you can send a command to start it, but you cannot send a command to reverse it 
while it is still starting and the motor is still ramping up.

The state machine is based on the following states:

| State                         | Indicator     | Description                                                                                                                                                                    |
|:------------------------------|:--------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Nominal**                   | Green         | System is healthy; $Z$-scores are $< 1.5$. UI remains muted per ISA-101 standards to prevent cognitive overload.                                                               |
| **Analyzing**                 | Pulsing Amber | Bytewax has detected a $Z$-score drift ($1.5 - 3.0$). The LangGraph Agent is actively performing RAG reasoning to draft a failover intent.                                     |
| **Command Sent**              | Flashing Red  | The Southbound Driver has observed the physical state change (e.g., switch closure) on the NATS telemetry stream. Standby hardware is confirmed in control.                    |
| **Confirmed**                 | Solid Blue    | The Southbound Driver has verified the Standby hardware is in control.                                                                                                         |
| **Manual Reversion Required** | Static Red    | Failover complete. A 15-minute stability lock is applied to allow the physical process to settle within a new baseline before further human or AI interventions are permitted. |

## Purdue Model Reference Architecture

To understand the system architecture of Oversight ICS from a Software Engineering perspective, it is important to 
reference the Purdue Model for industrial network segmentation: **The Purdue Model for Cybersecurity (ISA/IEC 62443)**. 

The Purdue Model defines the hierarchy of control systems in industrial environments, separated into different levels.
Modern enterprise tech stacks (e.g., cloud services, databases) operate at Level 4 and above, Oversight ICS is
designed with strict layer separation and sits on Level 3.5 IDMZ. This is important for security and reliability, as 
the system must never cross-contaminate or block deterministic scan loops of the field assets. 

### Purdue Model Overview

The Purdue Model is used in industrial environments on six strict levels of control hierarchy (**Levels 0-5**). 

**Core Engineering Principles**: Data flows dynamically **upward** (**Read Path**), but command authority is strictly 
**downward** (**Write Path**). The write path must only drop down under strict, auditable boundaries.

| Level | Zone        | Firewall Boundary   | Description                                                                                                                                                                                                                                             |
|:------|:------------|:--------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 5     | Enterprise  | -                   | Directory Services, Enterprise SIEM, ERP Systems, Corporate Applications                                                                                                                                                                                |
| 4     | Business    | Northbound Firewall | IT network, Email Servers, and Secure Dashboard Access Point                                                                                                                                                                                            | 
| 3.5   | IDMZ        | -                   | Oversight ICS: Northbound Ingest, Southbound Driver, NATS JetStream, Bytewax, Redis Hot Cache, TimescaleDB, LangGraph Agent, pgVector DB, Nginx Perimeter Gateway, Grafana Observability Engine, Prometheus System Monitor, and the Semantic Gatekeeper |
| 3     | Operations  | Southbound Firewall | Operational Core: Plant-wide SCADA systems, MES, and local Operational Historians                                                                                                                                                                       |
| 2     | Supervisory | -                   | Plant Floor Supervisory Control: HMIs, local control room panels                                                                                                                                                                                        |                                                   
| 1.5   | Arbitration | -                   | Control Authority Layer: Manages token arbitration Two-Key verification, Cryptographic Notary Buffers, and TPM 2.0 Hardware-rooted signing                                                                                                              |
| 1     | Control     | -                   | Deterministic Field Logic: PLCs, RTUs, DSC Controllers, and Safety Logic Solvers                                                                                                                                                                        |                                          
| 0     | Physical    | -                   | Physical Field Assets: Pumps, valves, motors, actuators, sensors, SIS                                                                                                                                                                                   |                    

### Level 0: Physical Kinetics 

**Real-World Application**: Waste Water Reclamation & Treatment Plants, Chemical Processing, Food & Beverage, 
Power Generation, etc.

**Components**: Pumps, Actuators, Valves, Motors, Sensors, Safety Instrumented System (SIS),etc
  - **Safety Instrumented Systems (SIS)**: The physical safety layer of the plant. These are hardwired, deterministic 
    systems that monitor critical process parameters (e.g., pressure, temperature, flow rate) and trigger physical 
    safety mechanisms (e.g., emergency shutdown, pressure release valves) when unsafe conditions are detected. The SIS 
    sits adjacent to the primary process assets and operates independently of any software control layer.

**Software Engineering Context**: The real-world physical process equipment is the ultimate **Source of Truth**. Runtime
logic deviations, motor winding destruction, hardware cavitation failure, or pipeline over-pressurization are all
examples of how software state can impact mechanical failure modes. In traditional web or distributed cloud development,
an unhandled exception might cause a crash or a degraded user experience. In industrial environments, an unhandled
exception or logic deviation can cause physical destruction, structural degradation, or catastrophic safety events.
There is no `try/catch` block.

### Level 1: Local Control & Automation

**Real-World Application**: Wastewater Ingestion Stations, Chemical Mixing Vats, Food & Beverage Pasteurization Lines, 
Power Generation Turbine Control, etc.

**Components**: Primary PLC, Standby PLC, RTUs, DCS, I/O Modules, Safety Logic Solvers, and Local Control Panels

- **Programming Logic Controllers (PLCs)**: PLC controllers run strict, synchronous **Cyclic Scan Loops** (typically
  *5ms* and *10ms*). In a single loop, the CPU reads all inputs, executes the logic (Ladder Logic or Structured Text), 
  and writes new binary / integer commands out to the hardware tables. PLCs have no heap memory, no garbage collection,
  and no virtual runtime environments. 
- **Input / Output Modules**: The I/O modules are the hardware physical bus. It samples electrical currents or voltages
  from field assets and digitizes them into unsigned integers via input cards. Output cards execute the inverse, 
  modulating relays, variable frequency drives (VFDs), or actuators by converting digital commands into physical 
  voltage or current changes.
- **RTUs (Remote Terminal Units)**: RTUs are similar to PLCs, but they are designed for remote monitoring and control.
  They are often used in distributed environments. They gather field telemetry and maintain local deterministic control
  while they communicate back to a central SCADA system. 
- **DCS (Distributed Control Systems)**: The local controller nodes of massive, plant-wide distributed control systems.
  They are essentially large clusters of PLCs and RTUs that govern complex, continuous manufacturing process loops. 
  DCS systems integrate thousands of analog variables into a unified control strategy.
- **Safety Logic Solvers**: The computational engine of the **Safety Instrumented System** (SIS). These are
  microprocessors that run adjacent to the primary process asset. They monitor level 0 hardwired sensors and execute 
  hardcoded logic to trigger physical safety mechanisms (e.g., emergency shutdown, pressure release valves, etc.) when 
  unsafe conditions are detected.

**Software Engineering Context**: Level 1 is the **Deterministic Control Layer**. Process assets execute hardcoded 
logic loops with sub-second jitter targets. They are designed to be deterministic and fail-safe, with hardwired 
interlocks and safety limits.

**Example:** A PLC is the last barrier before physical hardware, and it must never be 
compromised by software logic deviations. The PLC is the ultimate authority for command execution, and it must be 
protected from any external interference. 

### Level 1.5: Control Authority Model (The Arbitration Layer)

**Real-World Application**: Safety Intercept Checkpoints, Cryptographic Signature Enforcement Gates, Multi-Factor
Command Validation points, AI-to-PLC Write-Access Token Arbitration, etc.

**Components**: Oversight Arbitrator, Hardware TPM 2.0 Notary Chip, Multi-Factor Authentication (MFA) Token 
Validation, Cryptographic Notary Buffers, Hardware enforced safety policies.
  - **Level 1.5 Arbitrator**: Responsible for managing the **Write-Access Token**, enforcing the **Bumpless Transfer 
    Protocol**, and arbitrating command authority between the AI Agent and Human Operators. It contains zero
    dependencies on neural networks, large language models, or dynamic cloud runtimes. The Arbitrator sits directly on
    the boundary between the asynchronous message streaming layer (Level 3.5 IDMZ) and the deterministic control layer 
    (Level 1). Its only responsibility is to act as a stateless gatekeeper that inspects every single downward payload
    before it can reach the physical memory maps.
  - **Cryptographic Notary Buffers**: This component manages the processing and state-checking of cryptographic keys.
    It interacts with the local endpoint's TPM 2.0 (Trusted Platform Module) to decode asymmetric cryptographic 
    payloads. This buffer drops any transaction frame whose public-key signature does not explicitly match a trusted
    identity profile or that has been modified or corrupted in transit.
  - **TPM 2.0 Hardware Notary Module**: Before any control directive can be executed, it must be signed by the TPM 2.0 
    hardware module. This provides a hardware root of trust showing that the command has not been tampered with in 
    transit and that it originates from a verified source. The Notary Buffer requests a signature from the TPM and 
    appends it to the payload. This signature is a critical part of the safety architecture because it guarantees the 
    integrity of the command before it can affect physical hardware. The TPM 2.0 then passes the hardware-notarized 
    block down to the Southbound Driver for execution.
  - **Two-Key Protocol Dual-Auth Validation Registers**: Is a stateful register layer that handles high-impact command
    orchestration boundaries. When an intervention or predictive failover request is flagged by the Level 3.5 AI Agent, 
    this component blocks immediate execution and triggers an immutable lock state. It requires **Dual-Cryptographic**
    signatures from two-independent RS256-signed JWT tokens. One token is from the primary field operator and the other
    is from the safety supervisor or plant manager.

**Software Engineering Context**: Level 1.5 is a **Zero-Trust Policy Enforcement Layer**. At Level 1.5, Oversight 
applies an absolute zero-trust model that says, **Never trust an AI model with direct, unmediated write access to 
physical infrastructure**. If a software engineer connects a cognitive agent directly to industrial registers, a person
exploiting an adversarial prompt injection vector or a corrupted vendor manual via **RAG Poisoning**, or a simple
runtime hallucination can cause an out-of-bounds parameter to traverse down to real-world pumps. The **Control 
Authority Model** at Level 1.5 solves this entirely. It treats the cognitive agent as an advisory engine rather than
an executive override. When the LangGraph Agent generates a predictive safety patch or register correction, the payload
is caught by the Arbitrator. By placing the Arbitrator at the boundary, it guarantees that even if the AI Agent is
compromised, the physical assets remain safely insulated from malicious or unpredicted software state changes.

### Level 2: Supervisory 

**Real-World Application**: Plant floor operator control room, central water district monitoring station, electrical
grid dispatch controls, etc.

**Components**: HMIs, Local Control Room Panels
  - **HMI (Human-Machine Interface)**: HMIs are local touchscreens or operator terminals running graphics applications. 
    They are a real-time window into Level 1 PLC states, and continuously read and map controller tag values to 
    graphical dashboards. HMIs also have the authority to issue commands to field assets.
  - **Local Control Panels**: Graphical interfaces with hardwired push buttons used by operators on the physical plant
    floor to monitor localized process health, acknowledge alarms, and execute manual state changes.

**Software Engineering Context**: Level 2 is the **Supervisory and State Visualization Layer**. At Level 2, state
changes do not happen in an isolated memory database, they alter active physical kinetics. Because of this, Level 2
requires strict security controls and strict **Read-Write Path Bifurcation (Separation)**. The **Read Path** performs 
the continuous high-frequency polling, but the **Write Path** is treated as a dangerous operation.  

**Example:**
If an operator repetitively clicks a button on the HMI, it can cause a **Command Storm** that overwhelms a PLC and 
the physical communication bus can experience severe thread starvation. The network saturation can block a microsecond 
controller heartbeat, causing the system watchdog to time out that will completely halt factory operations. 

Oversight addresses this by sitting adjacent to Level 2. It enforces write rate-limiting and captures incoming commands. 
In this case, the AI reasoning loops never saturate the real-time communication lanes tracking the physical hardware.

### Level 3: Operational Site Core

**Real-World Application**: Plant-Floor Server Rooms, Localized OT Computing Clusters, Water/Wastewater Operations Data 
Hubs, Chemical Facility Server Racks.

**Components**: Plant-wide SCADA Systems, Manufacturing Execution Systems (MES), and Operational Historians.
  - **Operational Truth (SCADA / MES / Historian)**: The core source of truth for the plant's current state. These 
    exist independently of Oversight so that traditional plant control and monitoring remain deterministic and available
    even if Oversight is offline. SCADA on Level 3 functions as a site-wide data aggregator, polling the Level 2/1 
    assets and and feeding the telemetry to the local historian. This allows a view of the entire plant and enables 
    MES (Manufacturing Execution Systems) integration and site-wide reporting.
  - **Role in Architecture**: Level 3 provides the validated telemetry data required for the AI Agent to perform 
    statistical analysis.  It is explicitly separated from the AI reasoning layer (Level 3.5) by a strict firewall 
    boundary.  This is to maintain a **Safety First, Zero-Trust** architecture. Oversight is supervisory and influences
    this level but does not replace control, monitoring, and safety logic by these site core systems.
  - **Security**: Level 3 serves as the facilities baseline truth, which must be protected from unauthorized access.
    Oversight interacts with Level 3 via standardized, read-only telemetry or secure audited conduits. In this case, the
    integrity of the site's primary operational monitoring is never compromised by Oversight's cognitive interventions.

**Software Engineering Context**: Level 3 is the **Operational Truth** for the site's baseline operations. Oversight 
treats level 3 as a unidirectional read-only telemetry source. The Agent is prohibited from performing any CRUD 
operations on any level 3 databases or network devices.

### Level 3.5 AI & IDMZ Zone

**Real-World Application**: Security Isolation, Multi-Agent Cognitive Decision Nodes, Vendor Knowledge Grounding Hubs, 
Cross-Domain Boundary Protection.

**Components**: Northbound Ingest Service, NATS Telemetry Stream, Bytewax Streaming Engine, TimescaleDB Historian, 
Redis Hot Cache, LangGraph Agent, Local Vector Database (`pgvector`), NATS Command Queue, Nginx Reverse Proxy (Automated 
Threat Mitigation), Semantic Gatekeeper, the Southbound Write Driver, and the Observability Stack.
  - **Northbound Ingest Service**: Telemetry is streamed to the Northbound Ingest Service where protocol translation 
     (Modbus TCP, EtherNet/IP, OPC-UA) is performed and published. Normalized telemetry (Sparkplug B/MQTT) is published 
     directly into the NATS Telemetry Stream (the UNS backbone).
  - **NATS JetStream (UNS Backbone)**: NATS JetStream serves as the event-driven backbone for the entire system. It
    implements the systems Unified Namespace (UNS) using strict, text-indexed topic structures
    (e.g., `Enterprise/Site/Area/Line/Asset/Tag`). All telemetry from the Northbound Ingest is published to JetStream,
    and all downstream consumers (Bytewax, LangGraph Agent, etc.) subscribe to the relevant topics. Every normalized
    tag transformation is statefully committed to JetStream, creating an immutable, replayable, append-only log of all
    telemetry. This allows for message persistence, handles high-throughput with millisecond distribution windows, and 
    also allows independent consumer microservices to safely replay historical data states without affecting live 
    equipment queues.
  - **Bytewax Stream Processing Framework**: Bytewax operates as a real-time data streaming processor. It subscribes to
    the NATS JetStream topics and performs continuous windowed $Z$-Score analysis on the telemetry data. Bytewax uses
    microsecond rolling time windows to map statistical algorithms over live inputs, computing real-time metrics to 
    instantly catch industrial process drift or unaligned control behavior.
  - **Redis In-Memory Hot State Cache & TimescaleDB**: This represents the systems strict **CQRS (Command Query 
    Responsibility Segregation)** read engine. Bytewax splits its output into two branches. Calculated $Z$-Scores, 
    anomalies and the operational profile are pushed directly to the **Redis Hot Cache**, providing sub-millisecond
    key-value snapshots of the plant floor. The raw time-series measurements are sent to **TimescaleDB**, a time-series
    database acting as the localized plant historian to preserve an immutable record of all telemetry for long-term 
    tracking and analysis.
  - **Local pgvector Database (Knowledge Grounding)**: Stores high-dimensional vector embeddings of plant manuals, 
    historical **Root Cause Analysis (RCA)**, piping and instrumentation diagrams (P&IDs), and factory acceptance 
    testing rules. When the LangGraph Agent detects an anomaly or a logic deviation, it can query this local vector
    database to retrieve relevant contextual information to ground its reasoning and generate a more informed response.
  - **LangGraph Agent**: The LangGraph Agent represents the active thinking loop of the system. It is implemented as 
    a stateful, graph-based execution environment where nodes represent discrete reasoning steps and edges define 
    state-machine logic transitions. The LangGraph Agent continuously evaluates the live state of the plant by querying
    the Redis Hot Cache and TimescaleDB. If it detects an anomaly or a logic deviation, it can generate a predictive
    control directive or a safety patch. However, the agent does not have direct write access to the operational core.
    Instead, it routes all control intents through the **NATS Command Queue**.
  - **NATS Command Queue**: This critical message queue serves as the single source of truth for all control 
    intents generated by the LangGraph Agent or human operators. It is a strictly audited, append-only log that feeds
    into the [Level 1.5 Arbitrator](#level-15-control-authority-model-the-arbitration-layer). By routing all commands through this queue, every control intent is traceable,
    auditable, and subject to the same validation process. This prevents any command from bypassing the established
    safety protocols.
  - **Semantic Gatekeeper**: After the Level 1.5 Arbitrator validates the Multi-Factor Authentication signatures, the 
    payload is pushed to the Semantic Gatekeeper, which is a critical safety component that audits all AI-proposed 
    intents, performs an SOP compliance audit, RBAC enforcement, and forensic checkpointing. It validates the 
    AI-proposed intents against hardcoded **Forbidden Register List** and physical constraints derived from the plant's 
    P&ID diagrams. If any control directive violates these safety constraints it is blocked from proceeding back to 
    the Level 1.5 Arbitrator, preventing unsafe commands from reaching the physical assets. The payload is then destroyed
    by the gatekeeper before it can be signed by the hardware TPM module. It enforces the following constraints:
    - **Safety Constraints**: Cross-references $Z$-Scores from **Bytewax** to ensure commands don't violate physical
      thresholds.
    - **SOP Compliance**: Validates the proposed actions align with digitized SOPs.
    - **Role-Based Access Control (RBAC)**: Only authorized users can execute approved actions and checks if the AI Agent
      is authorized to interact with the specific process asset.
    - **Least Privilege Principles**: To prevent **Scope Creep**, the AI Agent makes sure the Southbound Driver possesses 
      the minimum required permissions to execute the intended action, and no more. An exploit in one area cannot compromise 
      the entire system.

    [!NOTE]: The Semantic Gatekeeper is a critical safety layer and does not execute any commands itself.

  - **Southbound Driver**: The final egress for write access. The Southbound Driver consumes the hardware-notarized
    block from the TPM module and the pushes the block through the Southbound Firewall, via Deep Packet Inspection for 
    safe writes. If the command exceeds safe operating limits, the firewall blocks it and triggers an alert. Once the 
    command is sent to the field asset, the Southbound Driver enters a **Pessimistic Confirmation Loop**. It then 
    subscribes to the NATS JetStream backbone and waits for the Northbound Ingest to publish a state change. The state 
    change confirms that the field asset has acknowledged the command. Only after the Northbound Ingest reports 
    the state change from the PLC does the system update the **Redis Hot Cache**. It handles the following tasks:
    - **Protocol Translation**: Converts AI-generated intents into hardware-specific register writes (e.g., PLC coil sets 
      or VFD speed references).
    - **Reliable Delivery**: Manages retries, acknowledgments, and state-verification loops to ensure the process asset 
      reached the intended state.
    - **Audit Logging**: Records the "Success/Failure" of every execution back to the Historian for full forensic 
      transparency.
  - **Stateless API Gateway & Nginx Reverse Proxy**: This component serves as the secure entry point for IT traffic. 
    The Nginx Reverse Proxy terminates TLS, enforces rate-limiting, and drops malformed frames at the perimeter. It acts
    as an **Automated Threat Prompt Mitigation** layer, scanning inbound API requests and RAG manual uploads for 
    adversarial injections. It enforces zero-trust, RS256-signed JWT authentication before any operational interaction 
    is allowed into the Level 3.5 IDMZ.
  - **Observability Stack (Prometheus & Grafana)**: 
    - **Prometheus**: Acts as the infrastructure collector within the Level 3.5 IDMZ. It scrapes high-frequency 
      technical metrics (CPU/RAM usage, NATS queue depth, container health, and Southbound Driver latency) from all 
      local services.
    - **Grafana**: Acts as the visualization engine that queries both TimescaleDB (for **Operational Truth**) and 
      Prometheus (for **System Health**) to create a unified dashboard for both system health and process performance.

**Software Engineering Context**: Level 3.5 is the **AI Reasoning & IDMZ Layer**. At Level 3.5 a severe security 
vulnerability like prompt injection or vector database poisoning could cause the AI Agent to generate malicious or 
unapproved commands. Oversight resolves this by maintaining absolute architectural isolation. The LangGraph Agent does 
not reside on the operational network, and it cannot open direct sockets to the field assets or databases below it. 
Instead, the agent updates the state models by consuming data **Out-of-Band** from the Redis Hot Cache. When a 
predictive control directive or safety patch is generated, the response must be explicitly treated as unverified and 
advisory. The intent then must go through schema validation by the gateway, cryptographic Two-Key Dual Authorization by
the Level 1.5 Arbitrator, and be audited against hardcoded physical constraints by the Semantic Gatekeeper. Even if a 
catastrophic exploit or runtime failure within the heavy cognitive compute zone occurs, the zone remains fully isolated 
from the physical kinetics of the plant, which remain fully protected by deterministic barriers.

[!Note] **Out-of-Band** means that the AI agent is not directly connected to the live telemetry stream. Instead, it queries the
Redis Hot Cache for real-time state snapshots and the TimescaleDB for historical data. 

### Level 4: Secure Dashboard & Business Logistics Network

**Real-World Application**: Corporate Operation Centers, Multi-Site Performance 
Analytics, Long-Term Data Lakes, Engineering Review Desktops, etc.

**Components**: Enterprise Operational Dashboards, Corporate LAN, Performance Tools (Distributed Fleet Health 
Aggregators).
- **Enterprise Operational Dashboard**: Deployed as a secure, air-gapped interface inside the Level 3.5 IDMZ
  for operators, plant managers, and safety engineers. This dashboard provides real-time read-only telemetry and acts
  as a secure, restricted portal for issuing intents which are then processed by the lower-level safety layers.
- **Corporate LAN**: The primary network for business operations.
- **Performance Tools**: These are cloud-based or on-premises tools used for long-term performance tracking, 
  fleet health monitoring, and cross-site analytics.

**Software Engineering Context**: Level 4 is the **Secure Interface & Business Logistics Layer**. The dashboard cannot
directly control or interact with Level 3 and below.  All data must be treated as read-only, and all commands must be 
treated as unverified intents until they are processed by the Control Authority Model (Level 1.5 Arbitrator). Data flows
upward from the OT/IDMZ to the Enterprise Zone, and all commands flow downward through strict, auditable boundaries
(Unidirectional Data Flow). All downward traffic must pass through the lower-level safety enforcements. Since 
Oversight treats the UI client at Level 4 as **Untrusted**, it prevents potential compromise of the corporate network 
from extending into the operational core. Data that is sent from the IDMZ to Level 4 must be stripped of low-level 
register information (Modbus TCP register addresses, EtherNet/IP CIP tag names, or OPC-UA node IDs) and instead be 
presented as high-level parameters with attached metadata. This prevents the UI from exposing the industrial network 
topology to unauthorized users.

### Level 4: Human-in-the-Loop Secure Dashboard Workflow

Because the Level 4 Secure Dashboard is an **Untrusted Client**, it has zero direct access to the OT layer. Instead, it
acts as a secure interface for operators to issue intents, receive read-only telemetry and review reasoning traces.

### Step-By-Step Operational Workflow

**The Intent Proposal Lifecycle**

When the Bytewax stream processor detects a process drift or an anomaly, it triggers the LangGraph Agent. The LangGraph 
Agent then performs root cause analysis (consults: pgVector for historical RCA, TimescaleDB for historical trends, and 
Redis Hot Cache for real-time state) and drafts a proposed intent.

- **Step 1 - Alert & Triage**: During Step 1 the dashboard will switch from a Nominal state (Grey Scale) to an Alert
  state (Red for Critical & Amber for a Warning). The operator can acknowledge the alarm to silence the alert audio. 
- **Step 2 - Review Reasoning Trace & RCA**: The dashboard will show a `PENDING` AI intent. The operator can click into 
  the alert to see the Agents **Statistical Correlation Score (SCC)**, which is a confidence score derived from the 
  $Z$-Score analysis. The dashboard also shows the operator the RAG citation from the vector database and the Agents
  reasoning trace.
- **Example**: 
    > **SCC**: "92% confidence"
    > 
    > **RAG Citation**: "Manual Pg. 45: Current spike combined with low flow indicates impeller blockage."
    > 
    > **Reasoning Trace**: Sanitized Summary of the historical context (e.g., "$Z$-Score deviated by 2.8σ over the 
      last 10 minutes, matching the failure signature from last month.").
- **Step 3 - Primary Authorization (First Key)**: If the operator agrees with the proposed intent, the field operator
  clicks the `APPROVE` button. The dashboard UI then enters a `SAFETY_DELAY_ACTIVE` state (Yellow Banner + Countdown 
  Timer) indicating that the intent is being processed by the safety layers. This locks out the operator from making 
  any more requests to prevent blind clicking (If an operator gets frustrated and clicks multiple times, it can cause 
  a command storm that overwhelms the field asset and the physical communication bus can experience severe thread 
  starvation). The safety delay and operator lockout prevent this anti-pattern. The dashboard then extracts the active 
  RS256 JWT token from the operator's session (The operator's RS256 JWT token is issued by the Level 5 Corporate 
  Identity Provider via SSO) and attaches it to the intent payload as a **Bearer Token** to prove the identity of the 
  requestor. 
- **Step 4 - Secondary Authorization (Second Key - if High Impact)**: If the proposed intent involves a high-impact 
  major state change, the dashboard will trigger a second supervisor credential. Either the Lead Safety Engineer or 
  the Plant Manager must authenticate to attach their RS256 JWT token to the intent payload. This creates and fulfills
  the **Two-Key Protocol** that requires dual authorization for high-impact commands. 
- **Step 5 - Nginx Perimeter Validation & Submission to Control Authority Model (Arbitrator)**: The intent payload with 
  the attached JWT token(s) is then pushed through the Northbound Firewall. The Northbound Firewall then sends the 
  request to the Nginx Reverse Proxy where the JWT token signatures get validated and the token is checked against the 
  system's access control policies. Nginx also enforces rate-limiting to prevent potential DDoS attack patterns and 
  rapid-fire clicking. The Nginx Reverse Proxy then forwards the request to the Level 1.5 Arbitrator. When the 
  Arbitrator verifies the human's identity is valid and authorized, the command is passed to the Semantic Gatekeeper.  
- **Step 6 - Physical Validation & Final Execution**: The operator monitors the dashboard telemetry. The UI remains in 
  a `PENDING` state until the Southbound Driver receives physical hardware confirmation from the field asset, routes
  the state change through the Northbound Ingest, and updates the Redis Hot Cache. Once the operator sees the UI update
  with the new state, they click `FINALIZE_HANDOVER`, which resolves the incident and archives the entire sequence into 
  the WORM audit log.

**Manual Adjustments & Operator Overrides**

If an operator needs to adjust a process or override the Agent, the dashboard provides a secure interface to do so while
still maintaining **Zero-Trust** compliance.

- **Adjusting a Parameter**: An operator inputs a new setpoint on the dashboard and clicks `SUBMIT`. The dashboard (UI)
  packages up this human-in-the-loop request and attaches the RS256 JWT token for the operator (and the supervisor if 
  it is a high-impact change) and pushes it through the Northbound Firewall. The request goes through the same Nginx
  validation, rate-limiting, arbitrator authorization, and semantic gatekeeper physical bounds checking as an 
  AI-generated intent. 
- **Changing Agent Authority**: If the Agent is generating false positives or behaving erratically, the 
  operator toggles the Agent's authority to **`MONITOR_ONLY`**. It then sends a command to kill the CDC Relay service, 
  immediately switching the AI to a read-only `MONITOR_ONLY` mode. Cutting the CDC Relay stops the Agent from sending
  commands to the Southbound Firewall. This keeps the Southbound Driver alive so the operator can have manual control 
  and still visually see what is happening. 
- **Agent Kill Switch**: In an emergency, the operator can click the `AI_LOCKOUT` button. This forces the Southbound
  Driver to drop the heartbeat bit, which triggers a physical safe-state in the field assets within 500ms. This bypasses 
  all software logic for the Agent.

**Dashboard Capability Matrix (RBAC)**

The Corporate Identity Provider (IdP) issues the RS256 JWT tokens that govern the access control policies. The dashboard 
enforces Role-Based Access Control (RBAC) based strictly on the token's embedded claims. The Corporate Identity Provider 
assigns these claims based on the users federated roles. This allows for seamless deletion of a user once, which cuts 
their access across the entire system. This includes access to the secure dashboard, which is instantly revoked.

### **Dashboard Capability Matrix (RBAC)**

| Role                | View                                                                                                                                                  | Action                                                                                                                                                                  | Override                                                                                                                                                                     | Configuration                                                                                                                                                          | Governance                                                                                                                                                                                                                                                      |
|:--------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Field Operator**  | Can view real-time telemetry, $Z$-Score trends, operational status, and active alarms.                                                                | Can acknowledge alarms, view RAG reasoning traces, and execute the Primary Authorization (First Key) on pending AI intents.                                             | Can toggle the Agent Authority down to `MONITOR_ONLY`, trigger the dashboard kill switch for an `AI_LOCKOUT`, or enforce physical `MAINTENANCE` mode via the HOA switch.     | -                                                                                                                                                                      | -                                                                                                                                                                                                                                                               |
| **Plant Manager**   | All Field Operator Views + High-level KPIs, Overall Equipment Effectiveness (OEE), read-only historical WORM logs, and compliance audit summaries.    | All Field Operator Actions + Provide Secondary Auth (Second Key) for facility-wide strategic overrides or parameter changes that significantly impact production yield. | All Field Operator Overrides + Holds the sole strategic override capability to transition the system into active `CONTROL_ENABLED` (Full Autonomous Production) mode.        | -                                                                                                                                                                      | Holds the final executive authorization to digitally sign off on transitioning the system into **Active Production Control**. Oversees Change Management governance, acts as the incident response liaison (e.g., to CISA), and audits the immutable WORM logs. |
| **Safety Engineer** | All Field Operator & Plant Manager Views + access to the Prometheus/Grafana dashboards, the WORM log spool capacity, and the Forbidden Register List. | All Field Operator & Plant Manager Actions + Provides secondary authorization (Second Key) for high-impact failovers and physical asset transitions.                    | All Field Operator & Plant Manager Overrides + ability to toggle the system into `COMMISSIONING_MODE` (Sandbox Testing) and authorize high-impact `ADVISORY_MODE` overrides. | Propose updates to the Semantic Gatekeeper boundaries, modify the **Forbidden Register List**, or ingest new RAG vendor manuals (subject to CI/CD staging validation). | Conducts mandatory **Alarm Rationalization Reviews** (ISA-18.2), performs FMEA Revision Reviews, acts as the witness for FAT-03 testing, and provides Dual-Auth approval for 'High Risk' System Change Requests.                                                |

### Level 5: Enterprise Cloud Hub

**Real-World Application**: Corporate-wide Data Warehousing, Enterprise-scale Security Monitoring, Compliance Auditing, 
and Central Observability for System Performance, Vendor Knowledge Distribution.

**Components**: Enterprise SIEM(Security Information and Event Management),Observability Stack, Cloud Data Lakes, Vendor
Knowledge Bases
- **Enterprise SIEM**: Aggregated security event logging
- **Observability Stack (Prometheus, Grafana)**:
  - **Prometheus**: Time-series database for monitoring system performance, latency, and error rates across the 
    distributed architecture.
  - **Grafana**: Acts as the visualization engine that queries both TimescaleDB (for **Operational Truth**) and 
    Prometheus (for **System Health**) to create a unified dashboard for both system health and process performance.
- **Cloud Data Lakes**: Long-term data storage regulatory compliance.
- **Vendor Knowledge Base (RAG Source)**: The central repository for downloading and hosting OEM PDF technical manuals, 
  maintenance schedules, and SOPs before they are distributed to the plant level.

**Software Engineering Context**: Level 5 is for monitoring both cognitive performance and infrastructure stability. 
Oversight strictly enforces this barrier. There is zero pathing, raw-socket access, or directly open conduits down to 
the control layer. It is entirely air-gapped from the lower automation networks by the Northbound and Southbound 
Firewalls, meaning corporate business logic can only ever interact with read-only data branches. 

Interactions with the Level 3.5 IDMZ are strictly controlled. Telemetry updates and WORM Audit Logs are pushed up (via ]
secure gateways) to the Enterprise SIEM, and security updates, IdP key rotations or PDF manual ingestion are pushed down.
When an OEM (vendor) releases a new PDF manual for a field asset, it is downloaded to the Level 5 Enterprise Cloud first. 
From here, it is securely pushed down through the Northbound Firewall into the IDMZ, where it is cryptographically 
verified and vectorized into the local RAG database (pgVector). This guarantees that AI Agent gets up-to-date vendor 
knowledge without ever exposing the operational plant floor to a direct internet connection. Any data arriving at Level 
5 is assumed to be **Untrusted** for control purposes until ingested, audited and verified within the IDMZ environment.

Grafana dashboards are strictly for visualization and have no executive authority. Although the Grafana dashboard is 
accessible from users in Level 4, the Prometheus and Grafana engines themselves operate strictly within the Level 3.5 
IDMZ. The Grafana dashboard has read-only access to the internal service metrics and the operational telemetry. If the 
enterprise layer is ever compromised, the attacker would not be able to use these tools to alter field assets or 
bypass safety controls.

## Firewall & Security Boundary Summary
| Firewall                | Location & Role        | Security Controls                                                                                                                | What it Does                                                                                                                                         | Read Path                                                                                                                                                                              | Write Path                                                                                                                                                                    |
|:------------------------|:-----------------------|:---------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Northbound Firewall** | IT/Level 4 Barrier     | Deep packet inspection, Nginx reverse proxy termination, RS256 JWT Zero-Trust authentication, safety-critical API rate-limiting. | Acts as the upper perimeter for the IDMZ (Level 3.5), strictly separating Enterprise/Business clients (Levels 4 & 5) from the AI and messaging core. | Upward: Pushes read-only WORM summaries from TimescaleDB to the Enterprise Cloud (Level 5) and allows the Secure Dashboard (Level 4) to subscribe to real-time Redis telemetry updates | Downward: Strictly gates inbound API requests and Two-Key Auth intents from the UI. Allows the Enterprise Cloud to securely push OEM PDF manuals down for RAG vectorization.  |
| **Southbound Firewall** | IDMZ / Level 3 Barrier | Stateful inspection, rigid conduit restriction, strict industrial protocol validation (e.g., Modbus, OPC-UA, EtherNet/IP).       | Establishes the critical safety gate protecting the OT Process Assets (Levels 0-2) and Operational Core (Level 3) from direct IDMZ access.           | Upward: Allows raw industrial telemetry to continuously stream upward from field assets into the Northbound Read Ingest (Level 3.5) for Sparkplug B normalization.                     | Downward: Only permits fully vetted, TPM 2.0 hardware-notarized command blocks from the Southbound Write Driver to cross downward and commit register writes to field assets. |

## Step-By-Step Data Flow Execution

### The Inbound Ingestion Stream (Read Path)
*Oversight is completely hardware-agnostic because the Northbound Ingest microservice isolates protocol complexity 
right at the Level 3.5 IDMZ boundary. It can process raw Modbus TCP byte streams for legacy hardware, manage 
asynchronous Ethernet/IP CIP (Common Industrial Protocol) tag sessions for Rockwell environments, and maintains secure, 
certificate-authenticated OPC-UA node subscriptions for modern edge controllers.*

**The Telemetry Lifecycle and Read Path Execution**:
1. Level 0 → Level 1: The physical kinetics of the plant are continuously sampled by field sensors and digitized into 
   unsigned integers by the I/O modules. The primary PLC (`PLC_P`) executes a cyclic scan loop, pulling the digitized 
   inputs into its memory tables and executing hardcoded logic to update the state of the physical assets.
2. Level 1 → Level 2: The raw tag registers are exposed to the local network via industrial protocols. The SCADA 
   system and HMIs continuously poll or subscribe to these registers to visualize the plant state and allow operators 
   to issue commands.
3. The Northbound Ingest receives raw tags (e.g., Modbus TCP, OPC_UA, & EtherNet/IP CIP). It processes and normalizes 
   all the protocols into a standardized Sparkplug B MQTT message schema.
   - This schema attaches critical metadata, unified data types, and mandatory real-time quality bits, before mapping
     the variable data into the Unified Namespace (UNS) topic structure. The payload is then standardized into JSON
     format (Enterprise/Site/Area/Line/Asset/Tag) annd coupled with its Integer Quality Bit. 
4. The normalized packet is published to NATS JetStream. NATS JetStream guarantees that all telemetry is stored in an 
   immutable, append-only log.
5. Because the normalized packet is published to NATS JetStream, the Bytewax stream processor and the LangGraph Agent 
   will only see the sanitized, text-indexed event stream. 
6. Bytewax stream workers consume the telemetry from the UNS stream, to calculate windowed rolling means $\mu$) and 
   $Z$-Scores in real-time to detect process drift.
7. Bytewax outputs the continuous, windowed $Z$-Score analysis on two branches to update the systems read models.
   - **Redis Hot State Cache:** High-frequency, windowed $Z$-scores and current asset states are pushed directly into 
     the cache. This provides a sub-millisecond, low-latency look-up layer for active variables.
   - **TimescaleDB:** The raw normalized Sparkplug B parameters are appended at the same time to update the time-series 
     historian database. The processed telemetry is stored in the TimescaleDB Hypertable in the Level 3.5 IDMZ. 
8. In the Level 3.5 IDMZ, the **LangGraph Agent** executes a continuous, asynchronous state evaluation loop. It 
   statefully queries the Redis Hot State Cache and references historical patters in TimescaleDB to match live 
   operational profiles against historical process rules.
9. If a calculation is outside statistical anomaly thresholds, the LangGraph Agent pulls localized repair boundaries 
   and asset contexts out of the embedded **pgvector local RAG database**. It does this to plan a predictive control 
   directive. This brings the read path to a close and gets the write path ready for intervention.

### The Outbound Command Intercept (Write Path)

The system enforces a **Zero-Trust Execution** model. The AI Agent and UI never possess direct write access to field
assets. All modifications must go through a multi-stage validation and arbitration process.
1. When the intent is finalized by the LangGraph Agent, it issues a `Control Directive Proposal`. Instead of executing 
   it directly, the Agent routes the proposal down to the **NATS JetStream Logical Command Queue** inside Level 3.5 
   (IDMZ).
2. At the same time, if a human operator issues a manual override request from the Secure Dashboard
   (Level 4), the request hits the **Nginx Proxy Gate** (Level 3.5), passes through schema validation, and is dropped 
   onto the exact same **NATS Command Queue** as a prioritized proposal.
3. The Level 1.5 Arbitrator consumes the proposal from the **NATS Command Queue**. The Arbitrator then immediately 
   places the command into a stateful isolation buffer and demands a **Dual-Key Multi-Factor Authentication (Two-Key
   Protocol)** handshake.
4. The Level 1.5 Arbitrator sends an authenticated challenge back to the Secure Dashboard. Execution will 
   remain frozen until two independent, cryptographically signed RS256 JWTs are provided. One token must be signed by 
   the active plant field operator, and the other one must be signed by Lead Safety Enginner or Plant Manager. The 
   Arbitrator then passes the authenticated payload to the **Semantic Gatekeeper** inside Level 3.5. 
5. The Semantic Gatekeeper performs a final safety audit. It checks the proposed command against the **Forbidden 
   Register List** and the **Physical Bounds Constraints** derived from the plant's P&ID diagrams. If the command 
   violates any of these hard safety constraints, it is immediately blocked from proceeding to the TPM hardware signer, 
   preventing unsafe commands from reaching the physical assets.
6. If the command passes the safety audit, it is then forwarded to the **Cryptographic Notary Buffers**. 
7. The Notary Buffers work with the TPM 2.0 hardware signer to attach a hardware root-of-trust signature to the command
   payload (proving it passed all IDMZ checks) and returns the notarized signature back to the buffers.
8. The fully signed packet is then dropped down to the **Southbound Driver (Write Relay)**. The Driver transmits the 
   hardware notarized command through the Southbound Firewall to the target PLC. The Southbound Firewall acts as a 
   stateful inspection boundary between Level 3.5 IDMZ and the OT network. This firewall will only allow vetted 
   deterministic commands to target field devices.
   - **Primary Path**: Primary Path: The register write is routed through the Southbound Firewall directly to the active
     field equipment controller (`PLC_P`).
   - **Failover Path**: If `PLC_P` fails to respond or reports process drift, the Southbound Driver instantly redirects
   the execution target through the firewall to the mirrored warm standby controller (`PLC_S`).
   - **Pessimistic Confirmation of Success Loop**: The Southbound Driver does not immediately report success. It waits 
     for a hardware-level register verification from the target PLC by subscribing to the NATS Telemetry Stream (which
     watches the Northbound Ingest). Only after the physical feedback loop confirms the state does the Southbound Driver
     updated the Redis Hot Cache, which then updates the UI dashboard with a `SUCCESS` status.

## Architectural Tradeoffs, Design Considerations, & Justifications

### Identity and Access Management (IAM): Federated OIDC vs. Local Active Directory (LDAP) 

I placed the Identity Provide (IdP) in the enterprise zone (Level 5) using Federated OIDC to issue RS256 JWTs, rather 
than hosting an LDAP (Lightweight Directory Access Protocol) server within the IDMZ. This is because 
the enterprise zone is already strictly separated from the operational core by Firewall 1, so the risk of a 
compromise extending from the enterprise network into the operational core is minimal. The Nginx Reverse Proxy at the 
Level 3.5 boundary will then validate the JWTs issued by the enterprise. I chose RS256 JWTs because they use asymmetric 
cryptography, which provides a higher level of security. Even if the public key is exposed, it cannot be used to forge 
tokens, whereas with a shared secret key (HS256), if the key is compromised, all tokens can be forged.

**Tradeoff**: Relying on an Enterprise IdP requires a functional northbound connection for initial 
authentication, meaning an internet outage could prevent new operator logins.

**Justification**: OT environments require strict architectural isolation and high security controls. By using a 
Federated OIDC provider, I can use existing corporate identity management systems, enforce single sign-on (SSO), and 
maintain a centralized user directory without having to manage a separate local directory service within the IDMZ. This 
minimizes introduction of additional attack surfaces within the IDMZ. The use of RS256 JWTs provides a secure, 
zero-trust authentication that is well-suited for the strict and critical nature of the operational environment.

| Feature            | RS256 JWT (OIDC)                                                                                                                                                            | Local Active Directory (LDAP)                                                                                                          | HS256 (HMAC) JWT                                                                                 |
|:-------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------|
| **Key Type**       | Asymmetric signing with public/private key pairs.                                                                                                                           | If compromised, the entire directory can be at risk. This would require additional security measures to protect the directory service. | Symmetric signing with a shared secret key. If the key is compromised, all tokens can be forged. |
| **Security Level** | High                                                                                                                                                                        | Moderate (requires robust key management and network security)                                                                         | Moderate (requires secure key management)                                                        |
| **Trust Model**    | Zero-Trust, The private key is strictly kept by the IdP, and the public key is used for verification. Even if the public key is exposed, it cannot be used to forge tokens. | The directory service must be protected with strong access controls and network security to prevent unauthorized access.               | Anyone with the secret key can sign and verify tokens                                            |
| **Use Case**       | Distributed Systems, multi-app architectures, APIs                                                                                                                          | Internal enterprise applications, legacy systems                                                                                       | Simple applications with low security requirements or where performance is a concern             | Closed, Monolithic systems with one internal server |

### Time-Series Database: TimescaleDB vs. Alternate Time-Series Databases (InfluxDB, MongoDB, or standard PostgreSQL)

I chose to use TimescaleDB as the time-series database for storing the raw telemetry data, rather than using a standard
PostgreSQL instance or other time-series databases. TimescaleDB builds right on top of PostgreSQL. This allows the
real-time telemetry, the intent audit logs, and the vector embeddings to all be stored within a single, unified database 
environment.  While TimescaleDB introduces specific maintenance requirements (such as managing hypertable chunks), it 
is a necessary for Oversight because it is ACID complaint, which is required for regulatory compliance and historical 
analysis. It also delivers the performance and scalability needed to handle high-frequency telemetry data in 
real-time.

**Tradeoff**:  InfluxDB and MongoDB are also time-series databases, but they do not offer the same level of 
relational integrity or ACID compliance as TimescaleDB. A standard PostgreSQL instance could be used, but it may not 
perform as well with high-frequency time-series data without the optimizations provided by TimescaleDB's hypertables 
and indexing. InfluxDB is designed for time-series data, but it forces the system into fragmented data storage where 
the vector embeddings and relational data are stored separately, which adds complexity to the architecture.

**Justification**: Because TimescaleDB sits on top of PostgreSQL it allows the LangGraph Agent to perform complex joins
between the live $Z$-Score anomaly and the vectorized PDF manuals in milliseconds.

### Nginx Reverse Proxy vs. Dedicated API Gateway 

I chose to use Nginx as the reverse proxy and API gateway, rather than deploying a dedicated API gateway because it is
a stateless C-binary that supports RS256 JWT authentication and strict rate-limiting. Nginx provides and air-gapped,
zero-dependency perimeter that drops malformed API requests and traps command storms without requiring a cloud 
connection or heavy JVM-based infrastructure. It is also widely used, well-supported, and known for its strict security
and performance. 

**Tradeoff**: Dedicated API gateways would offer me more advanced analytics and developer tools, but they would also
introduce heavy JVM/Lua overhead or require active tethering to an enterprise cloud, which violates the air-gap
requirement. Nginx provides a more lightweight, secure, and self-contained solution that is better suited for the
strict security requirements of the IDMZ.

**Justification**: The Nginx Reverse Proxy serves as the critical perimeter of defense for the IDMZ. It validates all
incoming API requests, enforces rate-limiting to prevent DDoS attack patterns, and only allows authenticated requests 
with valid RS256 JWTs to reach the command queue. By using Nginx, I can maintain a secure, high-performance gateway 
with strict security requirements for the operational environment without violating the air-gap or introducing 
unnecessary complexity.

### Prometheus vs. Traditional RDBMS for System Health Monitoring

I chose to use Prometheus as the monitoring solution for system health and performance metrics. Although, traditional
RDBMS solutions could be used, Prometheus' is specifically designed for monitoring and alerting, with built-in support
for time-series data, flexible queries, and Grafana for visualization. 

**Tradeoff**: Other system health monitors, like push-based monitors (Datadog), would violate the Purdue Model by
requiring an open conduit from the IDMZ to the cloud. 

**Justification**: Prometheus uses a local, pull-based scraping model. It will natively sit inside the IDMZ and actively
scrape the local containers for internal health metrics. This allows for the observability stack to be completely 
contained within the IDMZ. 

### Redis vs. Memcached for Hot State Cache

I chose to use Redis as the hot state cache for real-time telemetry and $Z$-Score lookups, rather than Memcached, 
because heavy LLM queries to TimescaleDB could lock tables and delay the write-path of critical field sensor data.

**Tradeoff**: Memcached is a simpler and can be faster for simple caching use cases. However, it does not support 
data persistence or advanced data structures like Redis does. Redis provides more features and flexibility, 
but it may require more resources to run. If I rely solely on TimescaleDB for both historical telemetry and reading
live state it could cause intense I/O contention between the LangGraph Agent's LLM queries and the Bytewax 
stream processor's high-frequency writes.

**Justification**: Redis provides a high-performance, in-memory data store that can handle the low-latency requirements 
of real-time telemetry lookups. I implemented it to finalize the CQRS architecture by separating the read and write 
models. By using Redis as a hot state cache, the LangGraph Agent and the UI can instantly pull context without touching 
TimescaleDB. Redis acts as a high-speed snapshot of the plant floor and prevents the relational historian and AI 
reasoning loop from a bottleneck.

### Vector Database: pgVector vs. Dedicated Vector Database (Pinecone, Milvus, etc.)

I chose to use a local `pgVector` extension within the PostgreSQL (TimescaleDB) instance for vector storage and 
similarity search, rather than deploying a separate dedicated vector database like Pinecone or Milvus. This decision 
was made to maintain strict architectural isolation and minimize the attack surface within the IDMZ. By using 
`pgVector`, I can keep all data storage within a single, self-contained database environment that is already protected 
by the IDMZ's security controls. By using `pgVector` I am able to reduce the complexity of the architecture and 
eliminate the need for additional network communication between separate services.  If using a dedicated vector 
database, it could introduce latency and potential vulnerabilities. Additionally, `pgVector` provides sufficient 
performance for our use case, as the vector search operations are not expected to be high-frequency or latency-sensitive 
in this context.

**Tradeoff**: The dedicated `pgVector` database does not offer the same level of performance or scalability as 
a dedicated vector database service. As the volume of vector data grows or if the system requires high-frequency 
similarity searches, the performance of `pgVector` may degrade compared to a dedicated vector database.

**Justification**: The security and isolation of the IDMZ is required in an OT environment. By using `pgVector`, 
all data remains within the protected boundaries of the IDMZ, reducing the risk of data breaches or unauthorized access. 
The performance of `pgVector` is sufficient for the expected workload, and the benefits of maintaining a simplified 
architecture and strict security outweigh the potential performance advantages of a dedicated vector database.

**Evaluation**: 
Oversight ICS prioritizes strict local security and a reduced attack surface. Pinecone requires outbound internet 
access (violating the air-gap). Milvus and Weaviate, for example, require deploying massive Java/Go ecosystems into 
the constrained IDMZ. By utilizing `pgvector`, I can keep time-series telemetry, relational safety constraints, and
vector embeddings within a single, ACID-compliant, hardened PostgreSQL engine. This simplifies WORM backup protocols 
and limits the attack surface.

### Message Broker: NATS JetStream vs. Kafka vs. RabbitMQ

I chose NATS JetStream as the message broker for the telemetry stream and command queue, instead of RabbitMQ or a
standard MQTT Broker (Mosquitto) or Kafka. NATS JetStream provides a high-performance, lightweight messaging system 
that is appropriate for the high-velocity telemetry data in an OT environment. NATS JetStream offers lightweight
connectivity, low latency, and built-in support for message persistence and replay. These are all critical for the 
real-time monitoring and control requirements of the system. 

**Tradeoff**: NATS JetStream may not offer the same level of features or ecosystem support as message brokers 
like RabbitMQ or Kafka. RabbitMQ offers a wide range of plugins and integrations, while Kafka is designed for 
high-throughput, distributed log processing. RabbitMQ or Kafka might be more suitable choices for larger, more complex
systems with more messaging requirements. However, for this specific use case, the simplicity and performance of 
NATS JetStream make it the best fit for the telemetry and command messaging needs of the system.

I specifically chose not to use Kafka / Flink because as a solo developer building a complex system, I wanted to 
minimize the complexity of the architecture. Kafka and Flink are powerful tools, but they require more setup, 
maintenance, and operational expertise compared to NATS JetStream. NATS JetStream provides me a simpler, more 
lightweight solution that still meets the performance and reliability requirements of the Oversight system without 
introducing unnecessary complexity.

**Justification**: The CQRS architecture of the Oversight system requires a message broker that can handle 
high-throughout and low-latency telemetry, but also must provide strong message persistence and replay capabilities 
for the command queue. While MQTT is used to ingest telemetry *upward* to the gateway, the internal command spine 
requires an immutable, replayable, append-only log. RabbitMQ is a traditional message queue and messages are deleted 
once consumed. NATS JetStream acts as a distributed log that guarantees WORM (Write Once, Read Many) compliance. 
If the Southbound Driver crashes, it can reboot and request the exact stream sequence number it missed, ensuring 
zero command loss. WORM compliance is critical for the command queue because every control intent must be traceable,
auditable, and replayable for safety and compliance reasons. NATS JetStream provides this capability, while RabbitMQ
and MQTT do not. Kafka does provide a distributed log, but it is designed for big data pipelines and may introduce 
unnecessary complexity for this use case. 

| Feature                    | NATS JetStream                               | RabbitMQ                                                                     | Kafka                                                |
|:---------------------------|:---------------------------------------------|:-----------------------------------------------------------------------------|:-----------------------------------------------------|
| **Primary Architecture**   | Lightweight connectivity and Distributed Log | Append-Only distributed commit log                                           | Traditional message broker                           |
| **Footprint & Operations** | Single binary, easy to deploy and manage     | Heavy JVM (Jave Virtual Machine) clusters, which requires KRaft or Zookeeper | Moderate Erlang runtime, requires plugin management. |
| **Consumption Model**      | Push or Pull (highly flexible filtering)     | Pull-Based (through partitions or consumer groups)                           | Push-Based (competing consumers)                     |
| **Message Persistence**    | Yes, with JetStream                          | Yes, with durable queues                                                     | Yes, with topic partitions                           |
| **Performance**            | High throughput with low latency             | Moderate throughput, higher latency due to JVM overhead                      | High throughput, designed for big data pipelines     |
| **Routing Model**          | Subject-based wildcard routing               | Hardcoded topics and Partition Keys                                          | Complex exchanges (Direct, Fanout, Topic, Headers)   |

## Evaluation of Cognitive Frameworks and Orchestration Engines

When designing the AI reasoning layer, I evaluated both the cognitive frameworks and orchestration engines
available for building the Oversight system. Because this system has to handle high-velocity industrial telemetry and
interfaces with physical hardware on the Level 3.5 IDMZ boundary, I needed a solution that could provide strict schema 
enforcement, advanced reasoning capabilities, and strong security controls. I decided to prioritize deterministic 
constraints, data sovereignty, network topology boundaries, and safety interlocks.

### Cognitive Frameworks

#### Gemini 3.5 Flash (Cloud Based LLM API)
**Considerations**: When moving data from the Level 3.5 IDMZ to a third-party LLM API (over public HTTPS - 
Port 443 -NIC 1), there are network security risks, data privacy concerns, and potential latency issues involved. Under 
IEC 62443, any outbound conduit increases the potential attack surface of the gateway. If the LLM API is compromised, 
an attacker could manipulate the AI Agent's responses or inject malicious commands. Sending sensitive operational data
to an external cloud service also raises concerns about data privacy and compliance. Strict schema-enforced JSON is 
critical for generating reliable payloads that the Semantic Gatekeeper can immediately audit against the Forbidden
Register List and Constraint Boundaries without throwing a `SCHEMA_TYPE_MISMATCH` error. In addition to this, the 
advanced internal reasoning steps of Gemini 3.5 Flash provides a self-correction mechanism that supports the target
F1 score of $\ge 0.92$. 

**F1 Score**: Measures the harmonic mean of precision and recall. A low F1 score means that the model is generating a 
high number of false positives or nuisance alarms. A high F1 score means that the model is generating accurate 
predictions with minimal false positives and nuisance alarms.

**Self-Correction Mechanism**: Gemini 3.5 Flash's advanced reasoning capabilities allow it to perform internal 
self-correction steps. The system will filter out its own bad ideas before the human operator ever sees them. This is 
critical for maintaining a high F1 score and by the time an intent hits the operational dashboard, it has already 
highly-accurate and viable.

**Pros**:
  - Strict structural and schema enforcement.
  - Free Tier available for development and testing.
  - Advanced Reasoning capabilities with support for complex graph-based execution.
  - Strong security and compliance controls provided by Google Cloud.

**Cons**:
  - Requires outbound internet access, which increases the attack surface.
  - Potential latency issues due to network round-trip times
  - Data privacy concerns
  - At 1500 requests per day, the free tier may introduce rate-limit jitter.

**Tradeoff & Risks Summary**:  While Gemini 3.5 Flash offers advanced reasoning capabilities and strict schema
  enforcement, the requirement for outbound internet access and potential latency issues introduces some risks.
  However, the benefits of using a powerful, cloud-based LLM with strong security controls and compliance may outweigh
  these risks. Additionally, using the free-tier (quota limit of 1500 Requests Per Day) for development and testing 
  can help lower my costs while building the system. When the system goes into production, I can evaluate the cost
  of using Gemini 3.5 Pro for higher request volumes.

**Justification**: I selected Gemini 3.5 Flash for the AI reasoning layer because of its advanced reasoning capabilities,
and for how it handles massive context windows with strict schema enforcement. The necessity to have advanced agentic
performance justifies creating an isolated Northbound Ingest to bridge cloud data safely into the Level 3.5 IDMZ.

#### Open-Source High-throughput Third Party LLMs (Llama 3, Falcon 40B, etc.)
**Considerations**: Deploying an open-source LLM within the IDMZ would require significant computational resources,
which may not be doable for the constraints of the operational environment. Managing and maintaining
an open-source LLM would also require additional expertise and resources. While it would allow for complete control over 
the model and data, the operational needs and potential security risks of hosting an LLM within the IDMZ outweighs the 
benefits.

**Trade-offs & Risks Summary**: These types of LLMs also provide smaller context windows. The technical manual RAG
queries require extensive context windows to pull relevant information out of the `pgVector` database. If a context
window is too small, the models retrieval grounding capabilities will be limited, which increases AI hallucinations and
reduces the F1 score of the system.

**Justification**: I rejected open-source LLMs for the AI reasoning later because of the computational resources and
small context windows. The OT environment is already resource-constrained and latency-sensitive, so deploying a large 
open-source LLM would not be practical. 

#### Local On-Device Edge Compute (Self-hosted Phi-4 / Gemma 3)
**Considerations**: Deploying a local on-device edge compute solution would provide complete control over the model 
and data, on our own IPC edge hardware and entirely inside the IDMZ. The performance and capabilities of these models 
may not meet the requirements of the system but could provide a good option for a failover or backup reasoning engine.

**Trade-offs & Risks Summary**: Running an advanced model on the edge would require substantial local GPU/CPU resources.
If the LLM reasoning cycle overlaps with a high-priority control directive from the Southbound Driver, a resource race
condition could occur, which means the system would have to make a choice between processing the telemetry and 
generating a control directive. This could lead to increased latency or even missed critical control actions, which is
unacceptable in an OT environment where real-time performance is crucial.

On the other hand, using a local on-device edge compute solution as a failover reasoning engine could provide the 
facility with a backup in case the primary cloud-based LLM is unavailable or the network connection is lost. This would
allow the system to continue functioning, with potentially reduced capabilities, until the primary LLM is back online.

**Justification**: I chose the local-edge compute option to be implemented as a mandatory local failover reasoning 
engine, using Phi-4-mini locally. Plant protection requires continuous monitoring and a hard fallback. If NIC 1 drops
global internet access, the system detects the timeout, and automatically starts the local on-device LLM, and enters a 
lower-priority, air-gapped `MONITOR_ONLY` mode. In this mode, the system continues to process telemetry and update the 
dashboard, but it will not generate any control directives until the primary LLM is back online. 

**Phi-4**: I chose Phi-4-mini as the local on-device edge compute solution because it is optimized for 
memory-constrained environments, making it an efficient choice for Oversight. It will also run smoothly on most IPCs, 
but you must make sure your system can meet the needs of the model. Phi-4-mini is free to use, but consumes a large
amount of CPU, RAM, and GPU cycles. It is not suitable as a primary reasoning engine, but it provides a good backup
option in case of network connection issues. In order to make manage the models resource consumption, Cgroups are 
mandatory to partition the resources.  As noted in [FAT-03 Safety and Security Protocol](/docs/fat/FAT-03-safety_and_security_protocol.md)
and [Infrastructure Design Document](/docs/infrastructure.md), I will cap the usage of the CPU to 40% to prevent it 
from starving the critical real-time processes of the system. In addition to this, the IPC must have sufficient RAM
(at least 16GB) and a compatible GPU to run Phi-4-mini effectively.

### Graph-Based Orchestration Engines vs. Linear Pipeline / Autonomous Agents

I chose LangGraph as the orchestration engine for the AI reasoning layer because of its support for complex graph-based
execution and its ability to handle large context windows with strict schema enforcement. These are all critical for the
real-time monitoring and control requirements of the system. 

**Tradeoff**: Other orchestration engines, such as AutoGPT, allow an AI to dynamically write its own execution plan,
which could be catastrophic for Industrial Safety. LangChain is highly deterministic but it lacks the cyclic routing
required to handle complex errors or fallback loops.

**Justification**: The LangGraph Agent provides a structured, deterministic execution environment that is essential for
the safety-critical nature of the system. Nodes represent discrete reasoning tasks (e.g., "Check Bounds," "Query RAG"),
and edges define exactly how the agent can route between states. This means that the AI cannot invent a new execution
path.

### NTP vs PTP for Time Synchronization

I chose to use PTP (Precision Time Protocol - **IEEE 1588v2**) instead of NTP (Network Time Protocol) because PTP provides a much higher
level of precision and accuracy, which is critical for the real-time monitoring and control requirements.

**Tradeoff**: NTP is simpler and easy to configure, however it only guarantees millisecond accuracy. This introduces
packet jitter that the AI interprets as process drift. To prevent this, I implemented PTP, which provides 
sub-microsecond synchronization. PTP requires specialized network switches and hardware time-stamping NICs. This 
requires Linux `ptp4l` and hardware timestamping NICS on the Southbound Driver to prevent false-positive anomalies
caused by network latency.

**Justification**: Oversight relies on highly sensitive microsecond rolling time windows in Bytewax. If the network
uses NTP, standard packet jitter will artificially warp the timestamp of a sensor reading. The AI will interpret this as
a process drift, triggering a false positive anomaly. Using PTP guarantees sub-microsecond rolling time windows in 
Bytewax. 
