# Networking & Clock Topology

## 1. Objective
This document defines the physical and logical network topology of the Oversight ICS Edge Gateway. It guarantees strict
isolation between the IDMZ and the Control Zone while also maintaining nanosecond time synchronization. NATS JetStream
is used for both deterministic high-availability telemetry ingestion and low-latency command orchestration. NATS acts
as the Unified Namespace spine, while Bytewax performs rolling windowed $Z$-Score anomaly detection. 

## 2. Full System Topology

The following diagram illustrates the complete network topology of the Oversight ICS system.

See [Network Topology Diagram](/docs/assets/oversight-network-and-hardware-topology.svg) for a visual representation 
of the network architecture.

### Network Architecture 

Oversight strictly separates decision-making, validation, and physical execution so that the AI-driven orchestration
never compromises plant safety. It is designed as a **Dual-Homed Hierarchical Gateway** (physically separating traffic
through dual NICs) that strictly enforces network segmentation based on the Purdue Model. Oversight uses a **Logical
Data Diode** approach which eliminates IP forwarding to maintain a strict air-gapped architecture between the IT and OT
networks.

Oversight uses a **Command Query Responsibility Segregation (CQRS)** architecture to separate reads from writes. 
- **Read Path**: Raw OT telemetry is normalized into a Sparkplug B unified namespace and streamed into NATS JetStream
  for anomaly detection.
- **Write Path**: All AI-generated control intents are submitted to the NATS Command Queue. The control intents 
  undergo dual-authorization, cryptographic notary, semantic validation, and TPM 2.0 signing by the gatekeeper before
  the Southbound Driver executes a hardware-notarized command to the control assets.

See [Oversight Purdue Model Diagram](/docs/assets/oversight-purdue-model.svg), 
[Oversight Purdue Network Architecture Diagram](/docs/assets/oversight-purdue-network-architecture.svg) and
[Oversight Network & Hardware Topology](/docs/assets/oversight-network-and-hardware-topology.d2) for a visual
representation of the network architecture.

### Logical Network Segmentation

The Oversight system operates as a **Star Topology** where the Edge Gateway is in the center of the network. 

- **Northbound Ingest Service**: The Northbound Ingest Service connects up to the Enterprise Zone for UI dashboards
  and NTP time sync. 
- **Southbound Driver**: The Southbound Driver is the central point pushing hardware-notarized commands down to the 
  OT environment. It is the only component allowed to communicate with the control assets.

### Industrial Protocols & Control Assets

The following table lists the supported industrial protocols and their associated control assets. The Edge Gateway
supports multiple protocols to ensure compatibility with a wide range of industrial equipment.  

| Protocol                        | Control Asset Type | Description                             |
|:--------------------------------|:-------------------|:----------------------------------------|
| Modbus TCP                      | PLCs, RTUs         | Legacy Asset Control                    |
| OPC-UA TCP                      | PLCs, RTUs         | Secure Telemetry Ingest (Sign/Encrypt)  |
| OPC-UA Pub/Sub                  | PLCs, RTUs         | Publish/Subscribe Telemetry             |
| OPC-UA Client-Server            | PLCs, RTUs         | Request/Response Telemetry              |
| EtherNet/IP                     | PLCs, RTUs         | CIP Explicit / Implicit Messaging       |
| MQTT (with Sparkplug B)         | Sensors, Gateway   | Stateful Unified Namespace (UNS) Ingest |
| PTP (IEEE 1588v2)               | All OT Assets      | Nanosecond Clock Synchronization        |
| Producer-Consumer CIP Messaging | Sensors, Actuators | Asynchronous Data Exchange              |
| PROFINET*                       | PLCs, RTUs         | Siemens Industrial Protocol             |
| IO-Link*                        | Sensors, Actuators | Device-Level Diagnostic Communication   |
| EtherCAT*                       | Motion Controllers | High-Speed Deterministic Control        |

*\* PROFINET, IO-Link, & EtherCAT are slated for future testing phases*

!!! note Conduit Security Rules
*Ingress traffic on NIC 1 is strictly limited to HTTPS (443) and NTP (123). Egress traffic on NIC 2 is strictly
limited to pre-approved industrial protocols (Modbus 502, OPC-UA 4840) to validated PLC IP addresses.*

## 3. Communication Conduits & Ports

| Source            | Destination        | Protocol    | Port            | Description                             |
|:------------------|:-------------------|:------------|:----------------|:----------------------------------------|
| Enterprise Zone   | NIC 1 (IDMZ)       | HTTPS       | 443 (TCP)       | HMI Access & API Calls                  |
| Enterprise Zone   | NIC 1 (IDMZ)       | NTP         | 123 (UDP)       | Master Clock Sync (Stratum 2)           |
| NIC 1 (IDMZ)      | Vertex AI Endpoint | HTTPS       | 443 (TCP)       | LLM Inference Egress (Gemini 1.5 Flash) |
| NIC 1 (IDMZ)      | Internal Bus       | TCP         | 4222 (TCP)      | NATS JetStream Telemetry Backbone (UNS) |
| NIC 1 (IDMZ)      | Internal Bus       | TCP         | 4222 (TCP)      | NATS JetStream Command Spine            |
| NIC 1 (IDMZ)      | Internal Bus       | RESP        | 6379 (TCP)      | Redis Hot Cache                         |
| Southbound Driver | TPM 2.0 Module     | SPI / IPC   | N/A             | Hardware-Notarized Command Signing      |
| NIC 1 (IDMZ)      | NIC 2 (L2)         | PTP         | 319/320 (UDP)   | Precise Time Protocol (Event Timing)    |
| NIC 2 (L2)        | Field Assets       | Modbus TCP  | 502 (TCP)       | Legacy Asset Control                    |
| NIC 2 (L2)        | Field Assets       | OPC-UA TCP  | 4840 (TCP)      | Secure Telemetry Ingest (Sign/Encrypt)  |
| NIC 2 (L2)        | Field Assets       | EtherNet/IP | 44818 (TCP/UDP) | CIP Explicit / Implicit Messaging       |

## Hardware & Network Requirements

### Physical Hardware Requirements
The Edge Gateway is designed for deployment in harsh OT environments (Class I, Div 2). To ensure high availability, 
the following specifications are required:

- **Form Factor**: **Industrial PC (IPC)** with **Fanless Passive Cooling** to prevent ingress of dust/particulates.
- **Mounting**: Standard 35mm **DIN-Rail mount** for control cabinet integration.
- **Dual NIC Requirement**: The IPC must have two independent physical network interface cards.
  - **OT NIC**: Connected to the isolated process network (Levels 1–2)
  - **IT NIC**: Connected to the business network (Level 4) for HMI and Level 5 API access.
- **Hardware Watchdog**: Physical safety monitoring the Edge Gateway heartbeat. Failure forces the plant into
  a local-only **Safe State**.
- **Environmental**: 
    * Operating Temperature: -20°C to 60°C.
    * Ingress Protection: **IP40** (minimum for a cabinet), **IP67** (if external).
- **EMC Compliance**: Shielded RJ45 ports and surge protection to maintain NIC integrity in high-EMI environments 
  (adjacent to VFDs/Motors).
- **Power**: Dual 24V DC power inputs for redundancy.

### Hardware Simulation & Development
For development and testing, without requiring physical field instrumentation and control hardware (PLCs, 
VFDs), OverSight ICS includes a **virtual field asset cluster** using `python-opcua` which simulates real-world logic 
drift and scan-cycle jitter. The simulation environment is designed to be fully containerized, allowing the 
**Field Level** to be spun up alongside the Gateway. It also uses **fault injection** to simulate memory leaks and 
thermal stress to verify the Agent's predictive response.

### Virtual Network & Container Segmentation
Oversight ICS uses Docker for containerization to simplify the deployment and management of the system. 
The Docker containers sit on top of the Edge Gateway, and each component of the system (Ingest Service, 
Southbound Driver, Agent, etc.) runs in its own container. This allows for easy scaling, isolation, and management 
for the different components of the system.

1. **The Field Network**: (`ot-bridge`) 
    - Hardware Simulators / Physical Control Assets
    - *Only accessible by the Southbound Driver via NIC 2 (Level 1/2)*
2. **The Industrial DMZ (`idmz_south` & `idmz_north`**)
    - **Perimeter Gateways**: 
      - **Nginx Proxy** (Northbound Driver - `idmz_north`) - Terminates Northbound IT traffic.
      - **Southbound Driver** (Southbound Driver - `idmz_south`) - Terminates Southbound OT traffic.
    - **The Reasoning Cluster** (`edge_gateway`) 
      - **Cognitive**: LangGraph, Ollama Engine (Phi-4), FastAPI
      - **Data Services**: TimescaleDB (Historian), PostgreSQL, Redis
      - **Messaging**: NATS JetStream (UNS)
      - **Streaming**: Bytewax (Anomaly Detection)
      - **Security Guardrail**: Semantic Gatekeeper
3. **The IT Network** (`it_bridge` | Levels 4–5) 
    - Dashboard - Read only visualization on HMI
   
## 4. Security Controls

- **Logical Star Topology**: The network is designed to be a star topology, while all traffic is forced through the Edge 
  Gateway.
- **Dual-NIC Isolation**: There is no IP routing enabled between the two NICs and there is no direct path between the 
  field zone and the enterprise zone.
- **Broker Segmentation**: High-volume telemetry is logically separated from high-priority control commands via **NATS 
  Subject Segmentation**. This prevents **Network Flooding** from delaying critical safety interventions or E-stops.
- **Clock Drift Mitigation**: The system monitors PTP sync continuously. If the drift between the Southbound Driver and
  the Level 2 field assets exceeds 50ms, the **Bytewax** layer triggers a **Stale Data** exception, forcing the AI into 
  `ADVISORY_MODE` and locking out automated control writes.
- **Command Validation**: 
  - All command proposals first go through the **Level 1.5 Arbitrator**, which validates the identity (**Two-Key 
    Protocol**) and authority of the operator.
  - If valid, the command is passed to the **Semantic Gatekeeper** for the final safety audit against the **Forbidden
    Register List** and physical hardware limits.
  - **Cryptographic Notary Buffers**: Only if the semantic gatekeeper approves the command, it is sent to the 
    cryptographic notary buffers to invoke the TPM 2.0 hardware signer.
  - **Deterministic Execution**: The hardware-notarized command block is passed to the **Southbound Driver** for 
    deterministic execution on the control assets. The Southbound Driver must first pass through the **Southbound
    Firewall**.
- **Semantic Integrity**: Commands lacking a valid `justification_hash` or failing register-range validation are 
  dropped and logged as a security event.

## 5. Protocol & Payload Standards 
The following data standards are enforced to maintain interoperability across the Unified Namespace (UNS):

- **Ingest Normalization**: All raw industrial protocols are normalized into **Sparkplug B** payloads by the Northbound 
  Ingest Service.
- **Payload Serialization**: Telemetry streams within the NATS JetStream backbone use **Protocol Buffers (Protobuf)** to 
  minimize latency and maintain strict schema enforcement for the AI Reasoning Agent.
- **State Representation**: The **Redis Hot Cache** maintains a **Pessimistic State** where it only updates the 
  **current value** of an asset once the Southbound Driver receives mechanical feedback from the field, closing the 
  loop.

## 6. Firewall Boundary Policies
- **Northbound Boundary (Level 3.5 to Level 4):** Default Deny Inbound. Only specific, authenticated proxy routes via 
  NGINX are exposed to the Enterprise zone.
- **Southbound Boundary (Level 3.5 to Level 1/2):** Strict IP and Protocol Binding. Only the verified Southbound Driver 
  process is authorized to establish outbound connections to the physical field controllers.
