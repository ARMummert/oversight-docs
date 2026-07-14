# Operational Playbook

## Revision History
| Version | Date       | Author       | Description of Change                                           | Approved By |
|:--------|:-----------|:-------------|:----------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | [User]       | Initial Baseline for Version 1.0 Release.                       | [Manager]   |

This document defines the Standard Operating Procedures (SOP) for plant operators and shift supervisors interacting with the OverSight ICS dashboard.

## 1. Alert Triage & Interpretation

OverSight categorizes asset health based on statistical drift ($Z$-Score). Use the following table to determine the severity of an incident.

| State        | $Z$-Score   | HMI Color | System Action            | Operator Requirement                 |
|:-------------|:------------|:----------|:-------------------------|:-------------------------------------|
| **Nominal**  | $< 1.5$     | Green     | Background Monitoring    | None. Process is stable.             |
| **Drift**    | $1.5 - 3.0$ | Amber     | Agentic Reasoning Active | Review drafted plan & RAG citations. |
| **Critical** | $\ge 3.0$   | Red       | **Autonomous Failover**  | Immediate physical verification.     |

## 2. Standard Operating Procedure (SOP): Red Alert

When a **Critical Failover** is initiated, the system follows the **Pessimistic Confirmation Loop**.

1. **Acknowledge Alarm**: Silence the audible siren on the HMI to confirm you are aware of the incident.
2. **Review Reasoning Trace**: Open the "Incident Card" on the dashboard. Review the RAG-sourced citations (e.g., *"Manual Pg. 45: Current spike indicates impeller blockage"*) to understand the AI's "Why."
3. **Validate Physical State**: 
   - Observe the **Process Variable (PV)** (e.g., Flow Rate, Pressure).
   - Confirm the Standby Asset is "Running" and the Primary Asset is "Stopped."
4. **Closing the Loop**: Once physical stability is confirmed, click **"Finalize Handover"**. This archives the LangGraph state and marks the incident as resolved.

## 3. Manual Override & Kill-Switch

In the event of logic drift or "Semantic Firewall" rejection, use the following tiered override procedures:

### Software Override (Level 1)
Toggle the **"Agent Authority"** switch to **OFF** on the Dashboard. This immediately kills the CDC Relay service, preventing the Agent from committing any new "Intents" to the Southbound Driver.

### Hardware Override (Level 2)
Physically move the **Hand-Off-Auto (HOA)** switch on the local motor starter cabinet to **HAND** or **OFF**. This physically breaks the control circuit from the PLC, rendering OverSight (and the PLC) unable to command the equipment.

# SOP-[ID]: [Process Name]
**Revision**: 1.0 | **Author**: Amy Mummert | **Criticality**: [High/Medium/Low]

## 1. Objective
A brief sentence explaining *why* this procedure exists.

## 2. Prerequisites & Safety
- **LOTO**: Lock-Out Tag-Out requirements.
- **PPE**: Personal Protective Equipment needed.
- **System State**: Does the machine need to be stopped first?

## 3. Step-by-Step Execution
1. **[Action Verb]** [Subject] [Details].
2. **[Action Verb]** [Subject] [Details].

## 4. Verification
How do I know I did it right? (e.g., "Verify green light on HMI").

## 5. Emergency Contacts
Who to call if this doesn't work.

Governance & Compliance
This playbook is governed by the broader **Operational Security Procedures** required for ISO 27001 and IEC 62443 compliance. 

- **Access Reviews**: Service account audits are performed quarterly per the [Access Review Schedule](operational_procedures.md#1-access-review-schedule).
- **Physical Security**: Cabinet tamper alerts are handled as `SECURITY_PRIORITY_1` events.
- **Asset Registry**: For a full list of data owners and classifications, see the [Information Asset Register](operational_procedures.md#2-information-asset-register).
---