# SOP-AI-02: Threshold Adaptation Review and Commitment

Document ID: SOP-AI-02
Version: 1.0
Associated Components: Semantic Gatekeeper & Consciousness Node

## 1. Purpose

The purpose of this SOP is to define the mandatory procedure for reviewing, approving, and committing changes to 
$Z$-Score anomaly thresholds proposed by the Consciousness Node. This guarantees that **logic drift** is governed by 
human oversight to prevent the masking of mechanical failures.

## 2. Scope
This SOP applies to all $Z$-Score anomaly thresholds proposed by the Consciousness Node. It covers the review and 
approval process for threshold adjustments, including the roles and responsibilities of operators, lead engineers, and 
safety leads/managers. The SOP also outlines the criteria for different levels of impact based on the percentage change 
in thresholds and the presence of restricted tags.

## 3. Roles & Responsibilities

- **Operator**: Reviews low-level and medium impact proposals and provides first-line validation of RAG justifications.
- **Lead Engineer**: Authorizes Medium impact proposals (5-15% delta).
- **Safety Lead / Manager**: Authorizes high impact proposals (>15% delta or restricted tags).
  - **Consciousness Node**: Generates proposals based on intervention quality and cannot commit changes >5% delta 
    without a valid `operator_id`.

## 4. Procedural Steps

### Step 1
**Proposal Triage**: When a new proposal is generated the HMI will flag an **Advisory** or **Action Required** alert.
The operator must open the **Threshold Adaptation Console** and review the proposal.

### Step 2
**Justification Review**: The operator must verify the **RAG reasoning trace** for every proposal.
  - Check the linked telemetry event (e.g., "frequent false positives during pump restart").
  - Verify that the proposed change does not conclift with manufacturer operating limits found in the attached 
    documentation citations.

### Step 3

**Approval Workflow**

| Proposal Impact | Delta ($\Delta$) | Action Required         | System Outcome                                            |
|:----------------|:-----------------|:------------------------|:----------------------------------------------------------|
| Low             | $\le 5\%$        | None (Acknowledge only) | Auto-patched by Gatekeeper; Logged to Audit Trail.        |
| Medium          | 5% – 15%         | Single-Key Approval     | Operator enters ID; Gatekeeper promotes status to ACTIVE. |
| High            | $> 15\%$         | Two-Key Override        | Requires Engineering CR-Form + Supervisor Bypass.         |

### Step 4
**Expiration & Purge**: All proposals have a **24-Hour Time-to-Live** (TTL). If a medium or high proposal is not acted 
upon within 24 hours, it is automatically purged from the `intent_outbox` to prevent **Stale Approvals**.

## 5. Critical Safety Guardrails
- **Desensitization Freeze**: If the system has issued more than three desensitization proposals (raising thresholds) in a 
  7-day period for the same asset, the Consciousness Node is locked out of making any further proposals until a 
  mandatory **Root-Cause Analysis** is performed.
- **Forbidden Tags**: Any proposal that contains a forbidden tag in the `forbidden_register_list` (SIS tags) must be
  rejected by the Semantic Gatekeeper, regardless of the delta percentage.

## 6. Audit & Records 
Every adaptation is logged in the `threshold_history` table. The following fields are mandatory:
- `timestamp`
- `old_value` vs. `new_value`
- `operator_id`(for changes >5% delta)
- `justification_uuid`(RAG trace)

## 7. Emergency Contacts
- **Primary**: OT LEAD ENGINEER
- **Secondary**: SAFETY MANAGER
