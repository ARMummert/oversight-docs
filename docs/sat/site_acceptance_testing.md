# OverSight ICS: Site Acceptance Test (SAT) Protocol (v1)

**Project:** OverSight ICS Field Commissioning  
**Security Level:** Level 3.5 IDMZ  
**Reference Document:** [COMMISSIONING.md](/docs/COMMISSIONING.md)

## 1. Scope of Validation
This SAT Protocol serves as the final gateway before the system is permitted to enter **Active Control** mode on production field assets. It verifies that the Edge Gateway's theoretical safety models (RAG, Z-Scores) align with the physical realities of the plant floor.

## 2. Infrastructure & Time-Sync (SAT-01)
*Prerequisite: PTP Grandmaster must be active on Level 2 OT Network.*

| Step | Action          | Expected Result                                    | Pass/Fail                                                          |
|:-----|:----------------|:---------------------------------------------------|:-------------------------------------------------------------------|
| 1.1  | Verify PTP Lock | `ptp4l` reports a steady lock with < 500ns offset. | [ ]                                                                |
| 1.2  | Failover Test   | Disconnect Primary PTP Clock.                      | System fails over to Local Rubidium Oscillator; no heartbeat loss. | [ ] |
| 1.3  | WORM Check      | Attempt to delete a log entry via CLI.             | System returns "Operation Not Permitted" (WORM Integrity).         | [ ] |

## 3. Operational Safety & Control (SAT-02)
*Prerequisite: Field Assets must be in a steady "Nominal" state.*

| Step | Action            | Expected Result                              | Pass/Fail                                                      |
|:-----|:------------------|:---------------------------------------------|:---------------------------------------------------------------|
| 2.1  | Bumpless Transfer | Toggle from **Passive** to **Active** mode.  | PID Integral initialized to current output; no process bump.   | [ ] |
| 2.2  | Forbidden Write   | Attempt to write to a "Speed-Lock" register. | `403 FORBIDDEN_SOFTWARE_TAGOUT` returned; PLC value unchanged. | [ ] |
| 2.3  | Z-Score Drift     | Simulate a 10% flow transient.               | HMI triggers "Real-Time Drift" alert within < 2 seconds.       | [ ] |

## 4. Security & Dual-Auth (SAT-03)
*Prerequisite: Primary and Secondary operator accounts must be provisioned.*

| Step | Action              | Expected Result                                                        | Pass/Fail                                  |
|:-----|:--------------------|:-----------------------------------------------------------------------|:-------------------------------------------|
| 3.1  | Self-Approval Block | Primary Operator attempts to approve their own High-Impact CR.         | API returns `401 ERROR_INVALID_DUAL_AUTH`. | [ ] |
| 3.2  | LOTO Hardware       | Flip the physical "Hand/Local" switch at the PLC cabinet.              | API returns `403 FORBIDDEN_HARDWARE_LOTO`. | [ ] |
| 3.3  | IDMZ Isolation      | Attempt to `ping` a NIC 2 (OT) address from the Business Network (IT). | Request times out; no bridging detected.   | [ ] |

## 5. Final Acceptance & Evidence 
The following evidence must be attached to this protocol for final sign-off:
1. **PTP Sync Logs** (60-minute stable window).
2. **SHA-256 Integrity Report** from the Auditor Agent.
3. **FMEA Alignment Report** signed by the Safety Lead.

### Authorization for Live Production
I, the undersigned, certify that the OverSight ICS system has been verified against the physical constraints of this site and is authorized for **Active Control** mode.

**Safety Lead Signature:** ____________________ **Date:** ________  
**Plant Manager Signature:** __________________ **Date:** ________