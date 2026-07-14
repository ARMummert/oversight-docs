## Installation & System Setup
Oversight ICS is designed to operate as a Dual-Homed System (Edge Gateway + IDMZ). The Edge Gateway and IDMZ provide
an air-gap between the OT Layer and the IT Layer.

### Prerequisites
The following prerequisites are required to run Oversight ICS:

- **Operating System**: Linux (Debian/Ubuntu)
- **Docker**: For container orchestration and virtual networking
- **Python 3.12+**: Managed via uv for dependencies
- **Container Engine**: Docker and Docker Compose for service isolation and virtual networking
- **Database & Caching**:
  - **PostgreSQL**: With TimescaleDB for telemetry and pgVector for local RAG extensions
  - **Redis**: For sub-second "Hot" UI Caching
- **Message Broker**: RabbitMQ for asynchronous task queues and reliable messaging. Configured with Consistent Hashing
  for ordered asset processing

### Field Installation Requirements
- **Physical Infrastructure**
  - **Industrial PC (IPC)**: A fanless, DIN-rail mounted ruggedized server (e.g., OnLogic, Advantech, or Siemens)
  - **Dual-NIC Requirement**: The IPC must have at least two physical, independent network interface cards (NICs) 
    to maintain the dual-homed system.
    - **NIC 1 (Southbound Driver)**: Connected to isolated OT Network (Level 1, 2). No gateway no internet access.
    - **NIC 2 (Northbound Driver)**: Connected to the Plant Business Network (Level 4) for UI access and Level 5
      LLM API calls.
  - **Hardware Watchdog**: A physical relay that monitors a software 'heartbeat' signal 
     from the Edge Gateway. If the software service hangs, the relay physically opens, forcing the Control Hardware 
     into a local-only 'Safe State.'
  - **Power Supply**: A 24V DC industrial power input to match standard control cabinet rails.

### Connectors & Protocols
- **OPC-UA (IEC 62541)**: The primary ingest method for telemetry data from the process assets.
- **Modbus TCP (IEC 62468)**: For legacy process assets, power meters, and older VFDs.
- **MQTT (Sparkplug B)**: Used if the plant already has a Unified Namespace (UNS) for telemetry.

### Network & Security (Purdue Compliance)
The environment must be segmented to follow ISA-99/IEC 62443 standard for "Zones and Conduits." 

- **Level 1-2 (OT Subnet)**: An isolated network where the field instrumentation and control hardware live, with no 
  direct path to the internet.
- **Level 3 (Reasoning Cluster)**: A configured LangGraph cluster to run the AI reasoning. This is a "thick" edge, 
  ensuring that the system is highly available and resilient to failures.
- **Level 3.5 (IDMZ Subnet)**: A configured Industrial DMZ (DMZ) to host the Southbound Driver and Nginx Proxy, 
  preventing Level 5 Cloud traffic from reaching Level 1 hardware.
- **Level 4–5 (IT Subnet)**: The network hosting the Next.js HMI/Dashboard and the Level 5 LLM API.
- **Firewall Rules**: Strictly defined "pull-only" access for the Southbound Driver and mTLS for all encrypted
  inter-container traffic.
- **SSL/TLS Certificates**: Required for the Northbound Proxy to ensure operator credentials and telemetry are never 
  sent in plaintext.

### Thick Edge
The Thick Edge is used for Level 3 autonomy. All reasoning (LangGraph), the database (TimescaleDB), and RAG (pgVector) 
all live on-site, ensuring that the system is highly available and resilient to failures.

### Security & Multi-Tenancy
  - **Tunneling**: Use mTLS (Mutual TLS) or a Reverse Proxy to secure the communication between the UI and the Edge 
    Gateway. 
  - **Data Masking**: Before telemetry leaves the plant for Level 5 (LLM API), Pydantic validators strip sensitive 
    metadata, sending only normalized sensor values.

### Step-By-Step Field Setup Guide
  1. Phase 1: OT Integration
     - **Mounting**: Install the IPC in the control cabinet, powered by the 24V DC rail.
     - **Addressing**: Assign NIC 1 a static IP on the process asset subnet (e.g., 192.168.1.100).
     - **Tag Discovery**: Use the ingest service to browse the OPC-UA tree and map tags (Pressure, Temp, VFD frequency)
       to the Unified Namespace (UNS)
  2. Phase 2: Configuration & Ground Truth
     - **Manual Ingest**: Upload PDF manuals for the process assets into the Local RAG.
     - **Baseline Training**: Run the system in Passive Monitor mode for 48-72 hours. This allows TimescaleDB to 
       calculate the baseline Z-scores for "Nominal" operation.
     - **Safety Whitelist**: Hard-code the "Safe Operating Envelopes" into the Southbound Driver (e.g., "AI can 
       never command a pump to start if pressure is > 100 PSI")
  3. Phase 3: Commissioning the "Closed Loop"
     - **Validation**: Perform a "dry run" where the AI triggers a failover, but the Southbound Driver is in 
       "simulation mode" to verify the logic flow in the Pessimistic UI.
     - **Live Handover**: Switch the Southbound Driver to "Active: Test the Physical Priority Arbitration" by turning
       the process assets physical switch to "manual" to ensure the AI "sheds load" immediately.

### Virtual Setup 
For development and testing purposes, Oversight ICS can be run in a virtual environment using **Docker's Network 
Bridging** to simulate a dual-homed system.

### Docker Container Segmentation

OverSight ICS enforces network isolation through three distinct Docker bridges. This ensures that a compromise in the 
UI (Level 4) cannot reach the field instrumentation and control hardware (PLCs, VFDs) (Level 1).

| Zone                                        | Docker Bridge   | Purdue Level | Services Included                             |
|:--------------------------------------------|:----------------|:-------------|:----------------------------------------------|
| **Field Zone (OT)**                         | `ot_bridge`     | 1 & 2        | **Ingest Service, Hardware Simulators**       |
| **IDMZ (South)**                            | `idmz_south`    | 3.5          | **Southbound Driver** (The only bridge to OT) |
| **Edge Gateway**                            | `edge_internal` | 3            | **LangGraph, TimescaleDB, RabbitMQ, FastAPI** |
| **IDMZ (North)**                            | `idmz_north`    | 3.5          | **Nginx Proxy** (The only bridge to IT)       |
| **Operations Zone (IT - External LLM, UI)** | `it_bridge`     | 4 & 5        | **Next.js Dashboard**                         |

### **Installation: Creating the Segmented Infrastructure**

To deploy with full IDMZ isolation, run the following commands to initialize the virtual hardware layers:

```bash
# Create the isolated OT and IT bridges
docker network create --subnet=192.168.1.0/24 ot_bridge
docker network create --subnet=10.5.0.0/24 it_bridge

# Create the internal reasoning and IDMZ bridges
docker network create --internal edge_internal
docker network create idmz_north
docker network create idmz_south
```
