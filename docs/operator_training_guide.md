# OPERATOR TRAINING GUIDE 

### 1. Objective
This guide provides the necessary procedures to train human operators to safely interact with the Oversight ICS AI
reliability layer. The focus is on the "Human-In-The-Loop" (HITL) model, which guarantees that while the AI provides 
reasoning and setpoint suggestions, the human remains the ultimate authority for critical process changes.

### 2. Operational Modes
The system operates in two primary states. The current state is always visible in the top-right corner of the Operator
Databoard.

**Advisory Mode (Manual Commit)**:
 - The reasoning agent monitors telemetry and generates **Intents** in the `intent_outbox` with a `PENDING` status.
 - The operator must review the **Justification Hash** and the linked technical manual citation before clicking COMMIT.
 - This mode is mandatory for all high-stakes assets like primary effluent pumps.

**Autonomous Mode (Supervised)**:
- The agent writes directly to the `intent_outbox` for execution by the Southbound Driver.
- The operator acts as an auditor, observing real-time $Z$-Score trends to ensure the AI's actions are stablizing the 
  process.

### 3. Understanding the Health Score ($Z$-Score)
The AI uses statistical deviation to identify **Process Drift** before an alarm occurs.
- **$\pm 0.0$ to $1.5\sigma$ (Green)**: Process is stable; no action required.
- **$\pm 1.5$ to $2.5\sigma$ (Amber)**: "Drift Detected." The Reasoning Agent will begin a "Trace" to investigate 
  manuals. Expect a suggested intent shortly.
- **$> 3.0\sigma$ (Red)**: "Anomaly Detected." If in Advisory Mode, an emergency notification will trigger. If in 
  Autonomous Mode, the system may initiate a pre-approved failover.

### 4. Handling a Security Lockout (P1 Incident)
If the Semantic Gatekeeper detects an "Out-of-Bounds" command or a loss of a heartbeat, the system enters **LOCKOUT**
mode.
 
1. **Identify**: The dashboard will strobe **RED** to display `SECURITY_VIOLATION_LOCKOUT`.
2. **Verify**: Check the physical asset. It should have automatically transitioned to is **Safe State** 
   (e.g., pump stopped, valve closed) within 500ms.
3. **Recovery**: Switch the physical equipment to **Local/Hand** mode at the MCC (Motor Control Center) before 
   attempting to reboot the Edge Gateway.

### 5. Training Checklist Certification
- [ ] Demonstrate ability to locate a `trace_id` in the `audit_log`
- [ ] Sucessfully approve a pending intent in the `intent_outbox`
- [ ] Correctly identify a **Hallucination** (e.g., if the AI suggests a setpoint that contradicts the RAG citation).
- [ ] Execute a manual **AI LOCKOUT** using the dashboard E-Stop button.
