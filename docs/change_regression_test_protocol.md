# Change Regression Test Protocol

**Version 1.2** | **Date:** 2026-05-14 | **Author:** armummert

## 1. Overview
The **Change Regression Test Protocol (CRT)** is a mandatory post-production validation suite for the Oversight ICS 
environment. These tests are designed to be executed against the live Edge Gateway (in `MAINTENANCE_MODE`)
to validate the functionality of the Edge Gateway. These tests prevent software updates, configuration changes, or RAG 
ingestions from degrading system determinism or violating safety boundaries.

## 2. Execution Triggers
CRTs are not meant to be run daily. They must be executed under the following conditions:
* **Major/Minor Version Updates**: Any updates to the LangGraph Agent, Southbound Driver, or Semantic Gatekeeper.
* **RAG Database Updates**: Following the ingestion of new technical manuals or vendor specs.
* **Infrastructure Patches**: OS-level patching, Docker engine updates, or TimescaleDB migrations.
* **Post-Incident**: After recovering from an `ORPHAN_MODE` or `SAFETY_LOCKOUT` event.

## 3. Regression Testing Matrix 

| Protocol | Scope                  | Focus Area                                              | Verification Logic                                                                                                                           |
|:---------|:-----------------------|:--------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------|
| CRT-01   | Safety and Determinism | Semantic Gateway, Forbidden Registers.                  | Attempt unauthorized register writes; verify instant rejection by Gatekeeper.                                                                |
| CRT-02   | Cognitive & RAG        | Vector DB, Reasoning Traces, $Z$-Score Accuracy.        | Validate RAG citation accuracy; verify $Z$-score calculation against mock telemetry.                                                         |
| CRT-03   | Security & Audit       | JWT, Service Accounts, LGTM Stack, WORM Integrity.      | Rotate RSA-4096 keys; verify SHA-256 hash consistency between Shadow Auditor and SIEM.                                                       |
| CRT-04   | Hardware & Driver      | Southbound Driver, PVP-02 Watchdog, Physical I/O.       | Trigger forced heartbeat failure; verify safe-state transition and watchdog reset.                                                           |
| CRT-05   | Traffic Control        | Level 3.5 IDMZ Gateway, API Rate Limits, DDoS.          | Execute `429` rate limit test; verify HMI transitions to **SAFETY DELAY ACTIVE** state.                                                      |
| CRT-06   | Disaster Recovery      | State Persistence, Cold Boot, Backup Restore.           | Restore from 24hr-old snapshot; verify zero loss of WORM logs.                                                                               |
| CRT-07   | Human-In-The-Loop      | Operator Veto, HMI interlock, Dual-Auth Flow.           | Force an AI intent that requires human sign-off. Verify Secondary Auth prompt.                                                               |
| CRT-08   | Semantic Drift         | AI Model Versioning, Prompt Template, Stability.        | Compare AI output for identical inputs across version updates for LLM consistency.                                                           |
| CRT-09   | Data Aging             | Retention Policies, TimescaleDB Compression.            | Verify 3-year data accessibility and test automatic pruning of lowest priority logs.                                                         |
| CRT-10   | Fail-to-Wire           | Network Partition, Southbound Timeout.                  | Sever Level 3.5 connection and verify edge agent maintains local safety logic.                                                               |
| CRT-11   | Time Synchronization   | PTP Grandmaster Failover, Rubidium Oscillator Holdover. | Sever connection to primary Grandmaster; verify seamless failover to local oscillator with drift $< 1\mu s$ and no loss of system heartbeat. |

## 4. Sign-off and Authorization
Upon completion of a CRT run, a formal sign-off must be generated and appended to the 
[**Change Management Log**](/docs/change_management_log.md).

**Lead Engineer Signature:** ___________________________ **Date:** _______________  
**Safety Lead Signature:** _____________________________ **Date:** _______________  

## Revision History
| Version | Date       | Author    | Description of Change                                                                 | Approved By |
|:--------|:-----------|:----------|:--------------------------------------------------------------------------------------|:------------|
| 1.0     | 2026-04-14 | armummert | Initial Baseline for Version 1.0 Release.                                             | armummert   |
| 1.1     | 2026-05-14 | armummert | Added CRT-11 (Time Sync Recovery), Execution Triggers, Sign-off block, and history.   | armummert   |026-05-14 | armummert | Added CRT-11 (Time Sync Recovery), Execution Triggers, Sign-off block, and history.   | armummert   |