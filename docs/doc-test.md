# Oversight ICS: Comprehensive Glossary

This glossary defines all critical bolded terminology found across the entire Oversight ICS project documentation.


## A
* **Active Control**: The operational state where the system transitions from a passive monitoring role to autonomously managing predictive failovers and process interventions.
* **Alarm Triage & Interpretation Table**: A matrix used by operators to correlate system states, Z-Scores, and HMI colors to determine the appropriate response to anomalies.
* **Anomaly Engine**: The Bytewax-driven stream processing component responsible for calculating real-time rolling means and Z-scores to detect statistical drift in telemetry.
* **API & Message Schema Specification**: The core governance document dictating the structural contracts, authentication (JWT), and rate-limiting rules for all inbound and outbound IDMZ traffic.
* **Asset Registry**: The authoritative database table maintaining the unique UUIDs, MAC addresses, and firmware versions of all approved physical hardware spares.

## B
* **Bytewax**: The Python-native stream processing framework used within the IDMZ to calculate rolling windows and statistical anomalies from the NATS JetStream.

## C
* **CI/CD Pipeline**: Continuous Integration/Continuous Deployment infrastructure utilizing GitHub Actions, trivy, and semgrep to enforce IEC 62443-4-1 secure lifecycle requirements.
* **CIP (Common Industrial Protocol)**: An industrial protocol used over EtherNet/IP for tag-based communication with modern Allen-Bradley controllers.
* **CMMS (Centralized Maintenance Management System)**: The site-wide software used to track work orders, maintenance schedules, and historical failure data.
* **Condition-Based Maintenance**: A maintenance strategy that relies on the AI's Remaining Useful Life (RUL) prognostics to replace assets based on actual wear rather than fixed schedules.
* **Conduits**: Logical or physical communication paths between different
* **Consciousness Loop**: The continuous asynchronous evaluation cycle where the LangGraph Agent processes new telemetry, references historical patterns, and drafts predictive intents.
* **CQRS (Command Query Responsibility Segregation)**: An architectural pattern separating the high-velocity telemetry read path (Redis/Timescale) from the highly governed command write path (Arbitrator/Southbound Driver).

## D
* **Degraded Mode**: A resilient fallback state triggered by component failures (e.g., loss of cloud connectivity), ensuring the system sheds AI capabilities but maintains deterministic local safety.

## E
* **Edge IPC (Industrial PC)**: The fanless, dual-NIC physical hardware appliance deployed on the plant floor to host the Oversight ICS middleware containers.
* **EtherNet/IP**: An industrial Ethernet network protocol widely used for real-time control and data acquisition from PLCs and field devices.
* **Execution Token**: The cryptographic permission object managed by the Level 1.5 Arbitrator, granting an entity (AI or Human) the temporary right to commit writes to physical registers.
* **Explainable AI (XAI)**: The systemic capability to provide a fully transparent, auditable reasoning trace (via UUIDs and RAG citations) for every AI-generated decision.

## F
* **Factory Acceptance Test (FAT)**: The rigorous validation protocol conducted prior to deployment to verify hardware, logic, and safety compliance (e.g., FAT-01, FAT-02, FAT-03).
* **Failure Mode and Effects Analysis (FMEA)**: A living matrix that identifies potential component failures, assesses their operational impact, and maps deterministic fail-safe responses.
* **FastAPI**: The Python framework used to build the high-performance backend API serving the HMI and processing RESTful requests within the IDMZ.
* **Field Assets**: The physical rotating equipment, valves, and sensors located in Level 0 that execute the actual industrial process.
* **Functional Safety**: The engineering discipline and compliance requirement (IEC 61511) ensuring that safety systems operate correctly in response to inputs, overriding all AI directives during critical events.

## G
* **Grafana**: The Level 4 observability visualization engine used to query the Prometheus system metrics and TimescaleDB operational truth.

## H
* **HMI (Human-Machine Interface)**: The interactive dashboard (often synonymous with UI) utilized by Level 3 operators to monitor OEE, respond to alarms, and approve intents.
* **Human-in-the-Loop (HITL)**: The operational requirement demanding a human operator physically verify and authorize high-impact AI intents before execution.

## I
* **IACS (Industrial Automation and Control Systems)**: The overarching term for the collection of personnel, hardware, and software that affects the safe operation of an industrial process.
* **IEC 62443-4-1**: The cybersecurity standard specifying the secure product development lifecycle requirements, satisfied via automated SBOMs and dependency scanning.
* **Incident Response Plan**: The governed Standard Operating Procedure (SOP) detailing how operators and management must react to cybersecurity threats or catastrophic physical trips.
* **Industrial Demilitarized Zone (IDMZ)**: The strictly guarded Level 3.5 network enclave housing the cognitive agent, vector database, and gatekeeper, isolated from both IT and direct OT.
* **Intent Outbox**: The authoritative PostgreSQL table acting as the transactional contract where all AI-generated intents must be persisted before physical execution.
* **ISA**: 18.2 : The international standard for the management of alarm systems
* **ISO 9001**: The international standard for quality management systems, incorporated into the system's Change Management and Document Control indexes.

## J
* **JWT (JSON Web Token)**: The RS256-signed cryptographic token used to verify identity and Role-Based Access Control (RBAC) across the Oversight API.

## L
* **LangGraph**: The core cognitive orchestration framework that drives the Reasoning Agent's multi-step decision-making, tool-calling, and intent generation.
* **Level 0 (Physical Process)**: The lowest tier of the Purdue Model containing the raw physical assets like pumps, actuators, and hardwired SIS interlocks.
* **Level 1 (Deterministic Control)**: The Purdue tier containing PLCs and VFDs that execute microsecond-level logic loops based on physical inputs or gateway commands.
* **Level 1.5 (Arbitration)**: A conceptual Zero-Trust tier sitting above Level 1, enforcing the Two-Key Protocol and hardware TPM signatures before allowing write access.
* **Level 2 (Supervisory Layer)**: The tier traditionally housing localized HMIs and control panels that provide operators with direct line-of-sight command over specific plant areas.
* **Level 3 (Operational Core)**: The tier housing central SCADA, MES, and Operational Historians acting as the facility's baseline 'Operational Truth'.
* **Level 3.5 (IDMZ)**: The security boundary where the AI reasoning and IT/OT protocol termination occurs, preventing direct connections between the cloud and the plant floor.
* **Level 4 (Enterprise/UI)**: The corporate network tier housing the primary web-based Dashboards and higher-level plant management interfaces.
* **Level 5 (Enterprise Cloud Hub)**: The external cloud layer responsible for data lakes, SIEM aggregation, and external LLM inference routing.
* **LGTM Stack**: An observability suite (Loki, Grafana, Tempo, Mimir) used for centralized logging and telemetry visualization.

## M
* **MES (Manufacturing Execution System)**: The Level 3 system that tracks and documents the transformation of raw materials into finished goods.
* **MiniLM-L6-v2**: The local, lightweight embedding model used to vectorize and query the OEM manuals within the pgvector RAG database.
* **Modbus TCP**: A legacy industrial communication protocol used primarily for power meters and secondary instrumentation.

## N
* **NATS JetStream**: The deterministic, append-only message broker utilized as the high-throughput Command Spine and Unified Namespace (UNS) backbone.
* **Nginx Perimeter Gateway**: The secure reverse proxy handling TLS termination, JWT validation, and rate-limiting for all traffic entering the Level 3.5 IDMZ.
* **Node Subscriptions**: The architectural pattern where the Northbound Ingest maintains authenticated connections to specific OPC-UA endpoints to read telemetry.
* **Northbound Firewall**: The logical and physical barrier preventing uninitiated traffic from entering the IDMZ from the Level 4 Enterprise network.
* **Northbound Ingest**: The microservice operating at the IDMZ boundary that translates raw Modbus/CIP/OPC-UA byte streams into normalized Sparkplug B payloads.

## O
* **OIDC (OpenID Connect)**: The federated identity framework used by the corporate IT network to authenticate users without storing passwords locally on the edge gateway.
* **OPC-UA**: A modern, secure industrial communication architecture widely used as the primary telemetry ingest method.
* **Operational Historians**: The Level 3 databases that store massive volumes of long-term, raw time-series data for the existing SCADA systems.
* **Operational Playbook**: The digitized technical manuals and Standard Operating Procedures (SOPs) utilized by the RAG system to generate context-aware intents.
* **Operator Databoard**: The unified visualization panel where real-time anomalies, Z-scores, and AI reasoning traces are presented to the floor team.

## P
* **P&ID (Piping and Instrumentation Diagram)**: The detailed engineering drawings that show the physical piping, valves, and control assets of the plant.
* **Passive Monitor**: The non-actuating operational state where the system observes telemetry and generates Z-scores and advisories, but lacks the authority to execute writes.
* **pgvector**: The PostgreSQL extension enabling semantic similarity search to support the Retrieval-Augmented Generation (RAG) capabilities.
* **Phi-4 / Gemini 3.5 Flash**: The large language models providing the cognitive reasoning capabilities, capable of running locally or via a secured cloud endpoint.
* **Platform Configuration Register (PCR)**: Hardware-secured memory slots within the TPM 2.0 used to store the cryptographic hashes (measurements) of the boot environment and critical software.
* **Prometheus**: The time-series database utilized specifically for scraping and storing internal system metrics, container health, and latency data (Technical Truth).
* **Protocol Buffers (Protobuf)**: The highly efficient binary serialization format used to format telemetry streams on the NATS JetStream backbone.

## Q
* **Quality Bit**: An integer tag appended to industrial telemetry indicating the health of the sensor read (e.g., must be >= 192 for the AI to consider the data valid).

## R
* **RAG Collisions**: An error state where the RAG pipeline retrieves contradictory instructions because a manual was updated without preserving the required semantic version history.
* **RBAC (Role-Based Access Control)**: The security framework enforcing that operators, safety leads, and plant managers can only execute actions permitted by their specific assigned privileges.
* **Redis Hot Cache**: The in-memory data store providing sub-millisecond retrieval of active asset states and Z-scores for the high-speed UI and reasoning loops.
* **Requirements Traceability Matrix (RTM)**: A compliance document that links high-level system requirements directly to the specific Factory Acceptance Tests (FAT) that validate them.
* **Root Cause Analysis (RCA)**: The automated diagnostic process where the AI correlates statistical drift with technical manuals to identify the underlying source of a mechanical anomaly.

## S
* **Safe Operating Envelope (SOE)**: The hardcoded minimum and maximum physical thresholds (e.g., max pump RPM) loaded into the Semantic Gatekeeper that the AI can never violate.
* **Semantic Hash**: A unique cryptographic signature assigned to a specific version of a technical document to ensure the AI always references the exact manual version active at the time of an incident.
* **Shift Reports**: Automated summaries generated by the AI detailing the Z-score anomalies, executed intents, and maintenance advisories compiled over an operational period.
* **Site Acceptance Testing (SAT)**: The final phase of commissioning where the hardware and logic are validated live in their actual operational environment.
* **Software Bill of Materials (SBOM)**: The comprehensive manifest detailing every software dependency, model weight, and library version to secure the software supply chain.
* **Southbound Firewall**: The rigid network boundary enforcing that only TPM-signed, hardware-notarized command blocks can exit the IDMZ into the Control Zone.
* **Sparkplug B**: The industrial IoT specification governing how MQTT edge payloads are structured to ensure birth/death certificates and state awareness.
* **State Alignment**: The mandatory pre-condition for Bumpless Transfer where the AI synchronizes its target setpoint with the physical asset's current state to prevent surges.
* **System Operating Modes**: The governed states (e.g., Active Control, Advisory Mode, Orphan Mode) dictating the system's level of autonomy and safety fallback behavior.

## T
* **TimescaleDB**: The PostgreSQL-based time-series historian acting as the immutable WORM audit log and long-term analytical storage for the Operational Truth.
* **TPM 2.0 (Trusted Platform Module)**: The discrete hardware cryptoprocessor responsible for establishing the Hardware Root of Trust and signing all physical command intents.
* **Trace ID**: A UUIDv4 tag appended to every agent thought and NATS message, providing the primary key for the 'Explainable AI' forensic audit trail.

## U
* **UI (User Interface)**: The visual dashboard layer (synonymous with HMI) built with Next.js, serving as the interactive gateway for human oversight.

## V
* **VFD (Variable Frequency Drive)**: The critical hardware asset that controls the rotational speed of electric motors, frequently managed by the system's predictive failover intents.

## W
* **Write-Access Token**: The highly restricted software permission managed by the Arbitrator that dictates whether the Human or the AI currently holds the right to modify physical equipment.

## Z
* **Zero Trust Industrial Architecture**: A security framework that eliminates
* **Zero-Trust Architecture**: The overarching security philosophy mandating that no actor, container, or request is trusted by default, requiring continuous cryptographic verification across all Purdue boundaries.
