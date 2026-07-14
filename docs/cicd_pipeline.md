# OverSight ICS: CI/CD Pipeline & Secure Development Lifecycle

**Security Level**: `Level 3.5 IDMZ`  
**Framework Alignment**: IEC 62443-4-1 (Secure Product Development Lifecycle)

## 1. Overview
The Continuous Integration and Continuous Deployment (CI/CD) pipeline for OverSight ICS is designed to guarantee that 
any software update (a patch to the **LangGraph Agent**, an update to the **Semantic Gatekeeper**, or a modification 
of the **Southbound Driver**) maintains absolute safety, determinism, and cybersecurity resilience. Code cannot reach 
the physical edge without passing strict automated and manual deployment gates.

## 2. Automated Security Scanning (Pipeline Gates)
Before any code can be merged into the `main` branch or compiled into a Docker artifact, it must pass the following 
automated security checks. Failure at any of these gates halts the build pipeline immediately.

### 2.1 Static Application Security Testing (SAST)
- **Objective**: Identify vulnerabilities in the source code before compilation.
- **Scope**: Scans the Python (LangGraph) and C++ (Southbound Driver) codebases for hardcoded credentials, buffer 
  overflows, and unsafe memory handling.
- **Gate Requirement**: Zero (0) High/Critical vulnerabilities.

### 2.2 Software Composition Analysis (SCA)
- **Objective**: Identify vulnerabilities in third-party libraries and dependencies.
- **Scope**: Scans `uv.lock` / `pyproject.toml` files, Dockerfile base images, and C++ linked libraries against the 
    National Vulnerability Database (NVD).
- **Gate Requirement**: Blocks any build containing dependencies with a CVSS score $\ge 6.0$. Generates the initial 
  **SPDX manifest** for the [SBOM Compliance Report](sbom_compliance_report.md).

### 2.3 Dynamic Application Security Testing (DAST)
- **Objective**: Identify vulnerabilities in the running application from an external attacker's perspective.
- **Scope**: Executes against a containerized ephemeral instance of the API Gateway, probing for OWASP Top 10 
  vulnerabilities (e.g., JWT validation bypass, API rate-limit exhaustion).
- **Gate Requirement**: Must pass the **DDoS Rate-Limiting Check** (verifying the `ERROR_SAFETY_DELAY_ACTIVE` triggers 
  correctly under load).

## 3. The Three-Tier Deployment Process
In alignment with our Operational Security Procedures (ISO 27001), all updates to the **Southbound Driver** or 
**Semantic Gatekeeper** must navigate a strict, phased rollout to physical assets.

### Phase 1: Virtual Twin Integration
- **Environment**: Fully simulated Level 1 and Level 2 environments (no physical hardware).
- **Action**: The new build is subjected to the full **Change Regression Test (CRT)** suite (CRT-01 through CRT-11). 
  $Z$-score algorithms are verified against simulated, high-variance process noise to ensure no false positives occur.
- **Exit Criteria**: 100% pass rate on all automated regression tests.

### Phase 2: Staging (Monitor-Only Mode)
- **Environment**: A live, but non-critical, physical asset in the plant (e.g., a secondary utility pump).
- **Action**: The software is deployed to the Edge Gateway but is hard-locked into `PASSIVE_MONITOR` mode. It can 
  ingest telemetry and propose intents, but the Write-Access Token is disabled.
- **Exit Criteria**: Minimum of 24 hours of stable operation with zero unhandled exceptions, zero logic drift alarms, 
  and zero memory leaks.

### Phase 3: Production (Active Control)
- **Environment**: The live production Control Zone.
- **Action**: The deployment is executed during a scheduled maintenance window.
- **Exit Criteria**: Requires a signed approval in the **Change Management Log** by both the **Lead OT Engineer** and 
  the **Safety Lead**. Following deployment, the system undergoes a live Verification Checkpoint to ensure the TPM 2.0 
  signatures are functioning correctly.

## 4. Hardware Root of Trust Attestation
Upon successful completion of Phase 3, the CI/CD pipeline calculates the final SHA-256 hashes of the compiled binaries. 
These hashes are injected into the **WORM Writable WORM Storage** and used by the TPM 2.0 module to verify the system's 
integrity during cold boots.

## Revision History
| Version | Date       | Author    | Description of Change                                                                 | Approved By |
|:--------|:-----------|:----------|:--------------------------------------------------------------------------------------|:------------|
| 1.0     | 2026-05-14 | armummert | Initial Baseline. Added SAST/DAST/SCA and Three-Tier Deployment Process.              | armummert   |
