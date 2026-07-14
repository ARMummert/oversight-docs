# Oversight ICS Quality Management System (QMS)

## 1. Quality Policy & Objectives
The OverSight Quality Management System is designed to ensure that AI-driven control intents are **deterministic, safe,
and verifiable**. The core goal is zero unintended motion caused by software logic failure.

## 2. Document & Version Control (ISO 9001:2015)

- **Single Source of Truth**: All technical contracts and schemas are maintained in the `/docs` folder.
- **Change Management**: Any modification to the **Semantic Gatekeeper** or **Forbidden List** requires a Pull Request
  (PR) with at least one **Safety Lead** approval and a successful run of the [FAT 02 Test Suite] 

## 3. Secure Development Lifecycle (IEC 62443-4-1)
The following automated gates are enforced in the CI/CD pipeline:
- **SAST (Static Application Security Testing)**: Scans for hardcoded credentials and unsafe memory handling. 
- **SCA (Software Composition Analysis)**: Blocks builds containing dependencies with CVSS scores $\ge 6.0$.
- **SBOM (Software Bill of Materials) Generation**: Automatically generates an SPDX manifest for each release.

## 4. Management Review & Audit Trail 

- **Monthly Review**: Engineering Leads must review the `incident_log` and `anomaly_report` tables to identify drift in
  the $R_p$ (Performance Ratio) calculations.
- **WORM Storage**: The `intent_outbox` and `reasoning_trace` logs are written to a WORM-compliant storage system (
  Write Once, Read Many) to prevent forensic tampering.

## 5. Media & Sanitization Policy (NIST SP 800-88)
When decommissioning the Edge Gateway, all NVMe (non-volatile memory) drives and SSDs must undergo a **purge** or 
**physical destruction** to prevent the recovery of site-specific memory maps or proprietary reasoning traces.
