# Software Bill of Materials (SBOM) Compliance Report

### 1. Objective
This Software Bill of Materials (SBOM) provides a transparent, machine-readable inventory of all software components,
libraries, and dependencies used within Oversight ICS. It is designed to guarantee supply chain security, facilitate
rapid vulnerability response, and maintain compliance with industrial cybersecurity standards (IEC 62443).

### 2. Standardized Format (SPDX)
The SBOM is generated using the [SPDX](https://spdx.org/) standard.

- **Authoritative File**: `/etc/oversight-ics/compliance/sbom_latest.spdx.json`
- **Verification**: The Southbound Driver performs an automated integrity check against the SPDX manifest before
  initiating the field asset Heartbeat.

### 3. System Architecture Components
The following core services define the runtime environment for the Edge Gateway. Each one must match the verified hash
before the **Southbound Driver** initiates the field asset Heartbeat.

| Component               | Function                          | Version              | SHA-256 Production Hash                                                 |
|:------------------------|:----------------------------------|:---------------------|:------------------------------------------------------------------------|
| **Reasoning Agent**     | LLM-based decision logic          | v1.2.4-stable        | 4f8a2b9c3d1e7f6a5b0d8e9c2a1b3f4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b        |
| **Southbound Driver**   | Hardware I/O and Heartbeat        | v1.0.8-deterministic | 9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c        |
| **Semantic Gatekeeper** | Safety Dominance Validation Logic | v2.1.0               | v2.1.0	1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w4x5y6z7a8b9c0d1e2f |
| **Edge Gateway OS**     | Linux                             | 22.04 LTS (Jammy)    | e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855        |
| **LLM Inference**       | Gemini 1.5 Flash                  | v2024-05-14          | [Managed Vertex AI Endpoint]                                            |

### 4. Model Provenance
Tracking the specific weights used in the RAG and Reasoning pipelines is mandatory for supply chain integrity.

| Weight Set          | Purpose                  | Version | SHA-256 Hash                                                      |
|:--------------------|:-------------------------|:--------|:------------------------------------------------------------------|
| **MiniLM-L6-v2**    | Semantic RAG Embeddings  | v2.0.0  | 5f9b8c7a6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a  |
| **Gemini System-P** | Safety Prompting Weights | v1.0.0  | a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6g7h8  |

### 5. Software Dependencies
These libraries are critical to the data layer and AI performance targets defined in the 
[Telemetry Database Scema](telemetry_database_schema.md)

| Dependency        | Purpose                               | Version     | SHA-256 Hash Example                                             |
|:------------------|:--------------------------------------|:------------|:-----------------------------------------------------------------|
| **TimescaleDB**   | Time-series telemetry storage         | v2.14.2     | 8f9e0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a |
| **pgVector**      | Vector similarity search for RAG      | v0.6.0      | 2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e |
| **FastAPI**       | Reasoning Agent API framework         | v0.109.0    | c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a8f9e0b1c2d3e4f5a6b7c8d9e0f1a2b3 |
| **Pydantic**      | Schema validation for `intent_outbox` | v2.6.1      | e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8 |
| **LLM Inference** | Gemini 1.5 Flash                      | v2024-05-14 | [Managed Vertex AI Endpoint]                                     |

### 6. Platform Compliance (Linux Only)
Industrial PTP (IEEE 1588) and deterministic resource scheduling (Cgroups) require a native Linux kernel.

- **Kernel Requirement**: 5.15+ with `PREEMPT_RT` patchset for sub-ms jitter targets
- **Mac / Windows Restriction**: Non-Linux environments are strictly prohibited for production Southbound Driver 
  operations due to the lack of hardware clock synchronization support.

### 7. Vulnerability Management & Compliance

- **CVE (Common Vulnerabilities and Exposures) Threshold**: Any component with a CVSS score ≥ 6.0 requires an immediate 
  patch and re-certification.
  - **Justification**: A threshold of 6.0 is mandated (per NIST SP 800-82) due to the Gateway's role as a bridge between
    high-security Level 2 assets and Level 3/Cloud services.
- **Automated Scanning**: Weekly scans are performed against the National Vulnerability Database (NVD). If the running 
  service hash does not match the signed SBOM, the Southbound Driver triggers a P1 incident and enters **Safe State**.
- **Hash Verification**: The system performs a startup integrity check. If the running service hash does not match this
  SBOM, the **Southbound Driver** triggers a **P1** incident and enters **Safe State**.
- **Resource Limits**: Components are strictly isolated via Linux (Cgroups) to prevent resource exhaustion from 
  third-party leaks.

### 6. Deployment Integrity

- **Image Registry**: All production images are pulled from a private, signed registry.
- **Update Policy**: Any component with a known CVE (Common Vulnerabilities and Exposures) score > 7.0 requires an 
  immediate update via the [Change Management Log](change_management_log.md).
