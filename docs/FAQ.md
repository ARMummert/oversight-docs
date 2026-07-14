# Failure Mode and Effects Analysis (FMEA) Matrix

### Project:
OverSight ICS | Scope: AI Reasoning Agent & Control Handshake

## 1. Objective
The purpose of this document is to identify potential failure modes within the Oversight AI reliability layer and 
define the technical mitigations required to maintain process safety and system availability. 

This FMEA Matrix focuses on the interaction between the Reasoning Agent and the Southbound Driver, as well as the 
integrity of the intent outbox, the safety interlock logic, and the deterministic execution of the Southbound 
heartbeat.

## 2. Risk Scoring Criteria
### Severity (S): 1 (Notice) to 5 (Catastrophic/Safety Trip)

| Score |   Ranking    |                                          Control Asset (PLC / Gateway)                                           |                                         Field Asset (VFD / Motor / Valve)                                          | 
|:-----:|:------------:|:----------------------------------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------------------------------------:|
|   5   | Catastrophic |   **Total System Blindness**: Logic engine stall and loss of all telemetry and safety interlocks for the area.   |    Uncontrolled Energy Release. Immediate mechanical failure (e.g., pipe burst) resulting from a command error.    |
|   4   |     High     | Degraded Control. Loss of redundancy or high-latency in the reasoning agent; system reverts to local-only logic. |    Asset Trip. Localized hardware fault (e.g., VFD Overvoltage) causing an immediate stop of one process unit.     | 
|   3   |   Moderate   |    Data Inconsistency. RAG Hallucination or stale manual data leading to sub-optimal tuning across the site.     | Process Excursion. Asset remains operational but drifts outside of $Z$-score efficiency baselines (e.g., hunting). |
|   2   |     Low      |   Minor Logic Desync. Non-critical UI glitch or a stale approval token that does not impact the active outbox.   |     Instrument Drift. Minor calibration error in a single sensor that is easily filtered by anomaly detection.     | 
|   1   |    Notice    |            Log Overflow. Forensic WORM buffer reaching 90% capacity; no impact on real-time control.             |         Nuisance Alarm. Transient vibration or temperature spike that self-corrects without intervention.          |

### Occurrence (O): 1 (Rare) to 5 (Frequent)
The likelihood that a specific failure mode will occur during a standard operating window.

| Score | Ranking    | Proability of Failure                                      | Frequency Example                                                                     |
|:------|:-----------|:-----------------------------------------------------------|:--------------------------------------------------------------------------------------|
| 5     | Frequent   | Constant / Expected                                        | Approval Fatigue: Human operators blind-clicking "Approve" after high alert volume.   |
| 4     | Probable   | Likely to occur monthly.                                   | Telemetry Noise: Sensor drift causing $Z$-score "nuisance" alerts due to environment. |
| 3     | Occasional | Once or twice per year.                                    | LLM Hallucination: Agent proposes a setpoint based on a low-confidence RAG lookup.    |
| 2     | Remote     | Every 3-5 years.                                           | Redis Cache Corruption: Data loss due to improper shutdown or memory overflow.        |
| 1     | Rare       | Extremely unlikely; requires multiple simultaneous faults. | TPM Key Compromise: Unauthorized signing of a register write-request.                 |

### Detection (D)**: 1 (Instant/Automatic) to 5 (Manual/Hidden)
Detection method is used to measure how effective the failure mode is at detecting the event.

| Score | Ranking  | Detection Criteria                                                      | Oversight ICS Implmentation Example                                                       |
|:------|:---------|:------------------------------------------------------------------------|:------------------------------------------------------------------------------------------|
| 5     | None     | Failure is invisible to the system until a physical trip occurs.        | **VFD Parameter Corruption**: VFD ignores speed ramp rate but reports **Healthy** status. |
| 4     | Low      | Requires manual review of forensic log audit to find the root cause     | **Consciousness Node Drift**: AI slowly changes its own baseline over months.             |
| 3     | Moderate | Detection based on statistical deviation ($Z$-score) from the baseline. | **Ingest Pipeline**: Anomaly engine flags telemetry as "Corrupt" if $>3\sigma$ from mean. |
| 2     | High     | Algorithmic detection with high confidence.                             | **Semantic Gatekeeper**: Hard-coded range check blocks non-compliant JSON payloads.       |
| 1     | Certain  | Immediate, binary detection. The process cannot proceed.                | **Hardware Watchdog**: Physical MCR relay drops if the heartbeat stops.                   | 

### Risk Priority Number (RPN)

**S x O x D = RPN**: Higher RPN indicates higher risk and priority for mitigation.

### Revised RPN

$RRPN = (S + 2O + 3D) / 6$: Target score after mitigation.

## 3. FMEA Matrix

| Component               | Failure Mode                | Local Effect                                                  | End Effect                                            | S | O | D | RPN | Mitigation / Control                                                                          | Detection Method                   | Revised RPN | SOP Reference |
|:------------------------|:----------------------------|:--------------------------------------------------------------|:------------------------------------------------------|:-:|:-:|:-:|:----|:----------------------------------------------------------------------------------------------|:-----------------------------------|:-----------:|:--------------|
| **Infrastructure**      | RabbitMQ Command TTL Expiry | Command discarded by Broker due to lag or partition.          | Intent fails; physical state remains unchanged.       | 4 | 2 | 2 | 16  | **Auto-Revert**: SBD inhibits AI writes; triggers **Incomplete Transition** alarm.            | RabbitMQ `basic.nack` / SBD Logs   |      4      | SOP-SYS-027   |
| **Infrastructure**      | Kafka UNS Blackout          | Agent loses real-time telemetry from UNS.                     | AI is "blinded"; cannot validate state before action. | 5 | 2 | 1 | 10  | **Inhibit Mode**: Immediate severance of write-access; fallback to Manual-Only.               | Heartbeat / Kafka Consumer Timeout |      5      | SOP-OPS-014   |
| **Infrastructure**      | Flink Stream Failure        | Loss of windowed $Z$-score math and hysteresis.               | Agent receives raw noisy data; risk of chattering.    | 4 | 3 | 2 | 24  | **Read-Only Mode**: Agent locked in "Observation" until stateful math is restored.            | Flink Checkpoint / Stream Silence  |      8      | SOP-OPS-015   |
| **Control**             | Bumpless Transfer Failure   | Process spike/dip during failover.                            | Mechanical stress; asset trip or pipe rupture.        | 4 | 3 | 2 | 24  | **Dynamic Gain Scaling**: Standby PID pre-loads active I-Term from Primary.                   | PID Error Delta Monitor            |      8      | SOP-OPS-017   |
| **Reasoning Agent**     | LLM Hallucination           | Agent suggests out-of-bounds setpoint.                        | Safety boundary breach; process excursion.            | 4 | 2 | 2 | 16  | **Semantic Gatekeeper**: Hard-coded range validation in FastAPI blocks non-compliant JSON.    | 403 Forbidden / Pydantic Error     |      4      | SOP-AI-007    |
| **Southbound Driver**   | Heartbeat Timeout           | Asset loses sync with the Edge Gateway.                       | Uncontrolled state; SIS takeover.                     | 5 | 2 | 1 | 10  | **Hardware Watchdog**: PLC logic triggers "Safe State" if toggle bit stops >500ms.            | Watchdog Timer Trip                |      5      | SOP-MAINT-006 |
| **Audit & Forensic**    | Trace ID Collision/Loss     | Inability to link action to reasoning.                        | Regulatory non-compliance; inability to perform RCA.  | 4 | 2 | 2 | 16  | **Header Propagation**: Mandatory link of `trace_id` across RabbitMQ/Kafka/WORM logs.         | UUID Integrity Check               |      8      | SOP-SYS-030   |
| **Audit & Forensic**    | WORM Storage Exhaustion     | System cannot write new audit trails.                         | Security/Audit blackout.                              | 4 | 1 | 2 | 8   | **Retention Purge**: FIFO deletion; `AUDIT_CAPACITY_CRITICAL` alarm at 90%.                   | Disk I/O Watchdog                  |      4      | SOP-SYS-029   |
| **RAG Engine**          | Stale Documentation         | Agent suggests fix on outdated manual.                        | Sub-optimal tuning; hardware damage.                  | 3 | 3 | 3 | 27  | **Metadata Check**: Mandatory `last_verified_date` check; citations shown to operator.        | pgvector Metadata Version          |      9      | SOP-AI-019    |
| **Telemetry DB**        | TimescaleDB Disk Full       | Anomaly Engine loses historical context.                      | Loss of long-term drift detection.                    | 4 | 1 | 2 | 8   | **Retention Policy**: Automated chunk dropping and Prometheus alerts >85%.                    | Disk Usage Alarm                   |      4      | SOP-SYS-025   |
| **Human Interface**     | Alarm Fatigue               | Operator ignores valid "Drift" alert.                         | Uncorrected drift leads to hardware failure.          | 4 | 4 | 2 | 32  | **Z-Score Rationalization**: Only σ>1.5 triggers urgency (ISA-18.2 alignment).                | Alarm Management Audit             |      8      | SOP-OPS-006   |
| **Consciousness Node**  | Threshold Desensitization   | AI raises $Z$-triggers to stop nuisance alerts.               | Critical failures missed (Under-reporting).           | 5 | 2 | 4 | 40  | **5/15 Rule**: Hard-coded cap of 15% delta + Human Two-Key approval.                          | Config Drift Watchdog              |     10      | SOP-AI-003    |
| **Consciousness Node**  | Approval Fatigue Loop       | Too many proposals cause blind clicking.                      | Accidental authorization of risky changes.            | 3 | 5 | 3 | 45  | **Auto-Authorization**: Low-impact (≤5%) changes auto-approved to focus human attention.      | Approval Rate Monitor              |     15      | SOP-AI-013    |
| **AI Logic**            | RAG Hallucination           | Agent proposes intent on incorrect manual.                    | Incorrect setpoint application.                       | 4 | 2 | 2 | 16  | **Citation Verification**: Operator must click URI/Hash link to unlock 'Commit' button.       | HMI Event Listener                 |      4      | SOP-AI-015    |
| **AI Logic**            | Consciousness Node Drift    | AI self-patches Flink thresholds too low.                     | Nuisance suppression; critical failures missed.       | 5 | 2 | 3 | 30  | **Max Delta Guard**: Cumulative drift capped at ±15%; requires Two-Key approval.              | Config Drift Watchdog              | SOP-AI-003  |
| **Gatekeeper**          | Protocol Mismatch           | JSON payload fails Protobuf schema.                           | Command drop; system command-chain stall.             | 3 | 2 | 2 | 12  | **Strict Schema Validation**: Gatekeeper rejects non-proto-compliant payloads.                | Validation Logic Error             | SOP-SYS-028 |
| **Ingest Pipe**         | Buffer Saturation           | Ingest exceeds CPU/Memory (Backpressure).                     | Delayed detection; "Blind" telemetry window.          | 4 | 2 | 2 | 16  | **Linux Cgroups**: Priority-weighted processing; drop low-priority logs first.                | CPU Pressure Stall (PSI)           | SOP-SYS-031 |
| **Human Interface**     | Alarm Fatigue               | Operator ignores valid "Drift" alert.                         | Uncorrected drift leads to hardware failure.          | 4 | 4 | 2 | 32  | **Z-Score Rationalization**: Only σ>1.5 triggers urgency (ISA-18.2).                          | Alarm Management Audit             | SOP-OPS-006 |
| **Asset: PLC**          | Logic Engine Stall          | Loss of all sensor telemetry for area.                        | Blind operation; loss of control.                     | 5 | 2 | 2 | 10  | **Watchdog Relay**: Trips the Master Control Relay (MCR) to de-energize outputs.              | MCR Status Feedback                |      5      | SOP-OPS-003   |
| **Asset: VFD**          | DC Bus Overvoltage          | Motor trip; potential water hammer.                           | Downtime; process pressure spikes.                    | 4 | 2 | 1 | 8   | **Braking Resistors**: Plus software-level "Surge Prediction" in the Agent.                   | VFD Fault Register                 |      4      | SOP-OPS-010   |
| **Asset: VFD**          | Param Corruption            | VFD uses incorrect ramp rates.                                | Erratic motor behavior; asset wear.                   | 3 | 1 | 3 | 9   | **Cyclic Param. Audit**: SBD verifies MAX_FREQ and ACCEL_TIME every 600s.                     | Parameter Hash Check               |      3      | SOP-SYS-020   |
| **AI Layer**            | Semantic/Context Drift      | Agent utilizes outdated RAG context.                          | Efficiency loss; sub-optimal tuning.                  | 3 | 3 | 3 | 27  | **Context Refresh**: Mandatory TTL on RAG citations; Agent pulls fresh data every 24h.        | RAG Metadata Version               |      9      | SOP-AI-016    |
| **Semantic Gatekeeper** | False Positive Rejection    | Safe intent blocked by rules.                                 | Efficiency loss; manual override required.            | 2 | 3 | 2 | 12  | **Simulation Mode**: Test new gatekeeper rules against historical telemetry.                  | False-Reject Event Log             |      4      | SOP-SEC-020   |
| **Approval Sys**        | Prompt Injection            | User bypasses intent guardrails.                              | Unauthorized setpoint change.                         | 5 | 1 | 4 | 20  | **Pessimistic Confirmation**: Human-in-the-loop required for all commands outside Safe Range. | Semantic Anomaly Alarm             |      5      | SOP-SEC-020   |
| **Approval Sys**        | Token Replay Attack         | Old approval token used for new intent.                       | Unauthorized process state change.                    | 5 | 1 | 5 | 25  | **Nonce-Based Tokens**: Tokens are single-use and tied to a specific intent hash.             | Token Integrity Mismatch           |      5      | SOP-SEC-015   |
| **Approval Sys**        | Stale Approval Token        | Operator approves but fails to "commit"; state is fragmented. | Logic desync between HMI and Edge.                    | 2 | 3 | 2 | 12  | **Proposal TTL**: All Adaptation Proposals expire after 24h.                                  | Token Expiry Handler               |      4      | SOP-AI-014    |

## 4. High-Risk Action Items (RPN > 20)

### 4.1 Alarm Fatigue Mitigation
- **Action**: Implement **State-Based Suppression.**
- **Logic**: If a field asset is in maintenance mode, suppress all non-critical telemetry alerts from that `asset_id` 
  to reduce operator cognitive load.

### 4.2 RAG Verification (Anti-Hallucination)
- **Action**: Implement **Mandatory Citation Verification.**
- **Logic**: The Reasoning Agent must provide a direct URI or file-hash link to the technical manual for every drafted 
  intent. The operator must click the link to enable the **Commit** button in the HMI.

### 4.3 Drift Control & HITL
- **Action**: Implement **Delta Logic Constraints.**
- **Logic**: The **Consciousness Node** is prohibited from modifying $Z$-Score thresholds by more than 15% from the 
  engineering baseline without a formal Change Request (CR) and manual reset.

### 4.4 Dynamic Gain Scaling (Bumpless Transfer)
- **Action**: Formalize **ST-14** (FAT) and **SAT-04** (Commissioning).
- **Logic**: Pre-loading the Standby PID I-term must be physically simulated using the **Field Asset Simulator** 
  (PAS) to ensure pressure spikes remain below 2%.

### 4.5 Trace ID Enforcement
- **Action**: Implement **Mandatory Header Propagation.**
- **Logic**: The SBD must reject any command lacking a valid `trace_id`. Ensures every valve movement is forensically 
  linked to a specific AI reasoning cycle.

### 4.6 UNS "Stale State" Boot-Lock
- **Action**: Implement **Initial State Hydration Delay.**
- **Logic**: Upon restart, the Agent is locked in `READ_ONLY` for 10 seconds to allow the Kafka UNS to receive fresh 
  Sparkplug B **BIRTH** certificates.

### 4.7 Broker Backpressure Kill-Switch
- **Action**: Implement **Latency-Based Execution Inhibit.**
- **Logic**: If Kafka-to-Agent lag exceeds 500ms, the system autonomously revokes the Agent's write permissions to 
  RabbitMQ to prevent "Late-Action" hazards.

## Revision Governance

This FMEA must be updated following any major change to the **Semantic Gatekeeper** configuration or after an "actual"
incident recorded in the **Incident Response Plan**.

## Revision History
| Version | Date       | Author    | Description of Change                                                                               | Approved By |
|:--------|:-----------|:----------|:----------------------------------------------------------------------------------------------------|:------------|
| 1.0     | 2026-04-10 | armummert | Initial creation of the FMEA Matrix document.                                                       | armummert   |
| 1.1     | 2026-04-24 | armummert | **Audit Fixes**: Added infra failures (MQTT / REDIS), Audit Integrity (SIEM/WORM), and Drift Gates. | armummert   |
| 1.3     | 2026-04-27 | armummert | Added drift guards, WORM exhaustion, and Dynamic Gain Scaling                                       | armummert   |
| 1.4     | 2026-04-30 | armummert | Full IEC 60812 compliance: Added local/end effect, detection method, and revised RPN                | armummert   |