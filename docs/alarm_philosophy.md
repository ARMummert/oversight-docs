## Oversight ICS: Alarm Philosophy 

## Purpose & Scope

The purpose of this document is to provide a consistent framework for the identification, prioritization, and 
management of alarms within the Oversight system. The alarms are presented to the operator in an actionable, 
relevant, and timely manner. The goal is to prevent **Alarm Fatigue** while also maximizing operator situational 
awareness. This document aligns with ISA-18.2 standards for alarm management.

## Roles & Responsibilities

In accordance with ISA-18.2, the alarm management lifecycle is governed by strict Role-Based Access Control (RBAC):

| Role                     | Responsibility                                                                                                                                                                                                                                                         |
|:-------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Field Operator**       | Acts as the primary responder. Authorized to acknowledge alarms, execute primary intent authorization (First Key Auth), and shelve specific nuisance alarms according to standard operating procedures.                                                                |
| **Plant Manager**        | Oversees compliance metrics and provides Secondary Auth (Second Key Auth) for high-level strategic alarm overrides.                                                                                                                                                    |
| **Lead Safety Engineer** | Responsible for the overall alarm management lifecycle and provides Secondary Auth if Plant Manager is unable to provide. Handles statistical alarm rationalization, manages $Z$-Score tuning, and is the **ONLY** role authorized to approve alarm threshold changes. |

## Definition of an Alarm 

In accordance with ISA-18.2, a notification is only classified as an alarm if it is:

1. **Abnormal**: Represents a process deviation or equipment malfunction.
2. **Actionable**: Operators are required to perform an action to resolve the alarm within the defined time limit.
3. **Urgent**: Requires immediate attention to prevent safety incidents, equipment damage, or process disruption.
4. **Consequential**: A defined consequence if the operator ignores the alarm, within the defined time limit.
5. **Unique**: The alarm is not a duplicate of another alarm.

> [!Note]
> Notifications that do not require an immediate response are classified as **Advisories** or **Events** and do not
> trigger audio/visual urgency.

## Alarm Priority & Response SLA Matrix

To ensure functional safety and clear lines of control authority, the following operator response thresholds are 
strictly enforced. If a human operator does not acknowledge and initiate corrective action within the defined time 
limit, the Agent will automatically revoke manual control authority and execute the fail-safe protocol. The following 
Service Level Agreement (SLA) defines the boundaries between human operation and machine intervention:

| Priority Level      | $Z$-Score Trigger       | Operator Response Time Allowance | Autonomous Fallback Action                                                                 |
|:--------------------|:------------------------|:---------------------------------|:-------------------------------------------------------------------------------------------|
| **Advisory/Event**  | $< 1.5\sigma$           | No immediate action required     | None. Background monitoring.                                                               |
| **Warning (Amber)** | $1.5\sigma - 2.5\sigma$ | 15 Minutes                       | Auto-log to Maintenance Management System.                                                 |
| **High**            | $2.5\sigma - 3.0\sigma$ | 5 Minutes                        | Propose mitigation intent; wait for Operator Commit.                                       |
| **Critical (Red)**  | $> 3.0\sigma$           | 60 Seconds                       | Execute predictive failover. If unauthorized, shed authority to local PLC (`ORPHAN_MODE`). |

## Master Alarm Database Details

The Master Alarm Database is maintained in the Level 3.5 IDMZ using TimescaleDB and pgVector. It serves as the 
single source of truth for all system alarm configurations. Every registered tag includes its baseline deviation limit 
($\sigma$), active $Z$-Score thresholds, response time SLAs, and its cryptographically linked RAG justification hash 
referencing the vendor manuals.

## Alarm Design Techniques
Oversight replaces `if-then` logic with statistical rationalization and signal conditioning to minimize nuisance alarms
and prevent **Chattering**. 

- **Statistical Rationalization**: Evaluates signal drift against a rolling 30-day baseline using the $Z$-Score 
  anomaly engine.
- **Signal Conditioning**: Applies Deadband, Hysteresis Filtering, and On/Off Delays at the Bytewax stream processor 
  level to filter out process noise before it reaches the reasoning layer. It applies these to raw Sparkplug B 
  telemetry before it ever reaches the Reasoning Agent, which sheds the process noise.

*By applying Deadband and Hysteresis filtering at the Bytewax stream processor level, nominal process noise is shed
before it reaches the reasoning layer, which drastically reduces false-positives.*

## Management of Change

Changing an alarm threshold or deadband directly impacts functional process safety and the anomaly engine.

- Any modification to the Master Alarm Database requires formal MOC approval.
- Any adjustment to the $Z$-Score baselines or sensitivity parameters requires Two-Key Authorization  (Operator + Lead
  Safety Engineer or Plant Manager) and must be accompanied by a documented rationale citing the specific manufacturer 
  manual section justifying the change.
- All changes must be explicitly recorded, justified, and signed off in the `change_management_log.md` document before 
  being pushed to production.

*An annual Alarm System Audit will be conducted to compare the Master Alarm Database against the physical P&ID 
baselines and review the effectiveness of the $Z$-Score anomaly engine.*

## HMI Design Guide

The Oversight ICS dashboard strictly adheres to the ISA-101 High-Performance HMI standard:

- **Grayscale**: The interface utilizes a high-contrast grayscale layout to minimize cognitive load.
- **State-Based Color**: Color is strictly reserved for abnormal process states. Amber indicates Drift/Advisory mode, 
  while Red indicates Critical Anomaly/Predictive Failover.
- **Safety Delays**: Includes visual countdown timers (Yellow Banners) during high-impact rate-limiting to prevent 
  operator **blind clicking** during crises.

## Performance Metrics (KPIs)

System performance is measured using automated KPIs tracking the rolling 24-hour distribution.

- **Critical Distribution Cap**: Critical alarms must not exceed 10% of the total alarm distribution in any 4-hour 
  period. If breached, the system issues a `KPI_DISTRIBUTION_DEVIATION`.
- **Alarm Flood Threshold**: Triggers an automatic review if $> 10$ alarms occur within 10 minutes.
- **Stale Alarm Threshold**: Any alarm active beyond 24 hours is flagged for mandatory engineering intervention.

## Alarm Storage

All alarm triggers, operator acknowledgments, and AI reasoning traces are permanently logged to a WORM storage volume. 
These logs include high-precision IEEE 1588v2 (PTP) hardware timestamps and a SHA-256 payload hash to ensure absolute 
forensic integrity for incident post-mortem and compliance auditing.

## Training Requirements

Operators must complete the certification checklist outlined in the `operator_training_guide.md`. This includes 
- Understanding the "Human-In-The-Loop" (HITL) model
- Interpreting $Z$-Scores correctly (Nominal vs. Drift vs. Critical),
- Locating trace IDs, and successfully executing a manual AI Lockout (Global Kill Switch).

## Alarm Rationalization and Nuisance Suppression

### Signal Conditioning Matrix
To maintain the integrity of the anomaly detection engine, all incoming telemetry is subjected to pre-processing 
filters before being evaluated by the $Z$-Score engine. This prevents **Chattering** alarms caused by process noise or
signal oscillation.

1. **Deadband Tolerance**: This is a precision buffer zone around the deadband limit. A signal must exceed the deadband
     plus the tolerance before a state change is registered in the UNS.
2. **On-Delay**: The duration that a signal must remain outside the deadband before the system acknowledges it is a 
   valid state change.
3. **Off-Delay**: The duration that the system maintains a tripped or active status after the signal returns within
   the deadband. 

| Signal Type | Deadband (% of Range) | Deadband Tolerance | On-Delay (Confirm) | Off-Delay (Retain) | Rationale                                                                  |
|:------------|:----------------------|:-------------------|:-------------------|:-------------------|:---------------------------------------------------------------------------|
| Temperature | 1%                    | $\pm$0.2%          | 60s                | 0s                 | Reflects the high thermal inertia of industrial processes.                 |
| Pressure    | 2%                    | $\pm$0.5%          | 15s                | 5s                 | Allows for momentary "hammer" or valve-stroke spikes                       |
| Flow Rate   | 5%                    | $\pm$1.0%          | 15s                | 15s                | Filters out pump turbulence and transient surges.                          |
| Level       | 5%                    | $\pm$1.0%          | 60s                | 60s                | Suppresses "slosh" effects in tanks and vessels.                           |
| VFD Current | 3%                    | $\pm$0.5%          | 10s                | 2s                 | Filters startup inrush noise while reacting quickly to mechanical binding. |

### Detection Logic ($Z$-Score)

1. **Deviation Threshold**: An alarm is only triggered if a process tag maintains a $Z$-Score drift of > 3.0 for a 
   sustained period (default: 10 cycles).
2. **State-Based Suppression**: Alarms that are related to a specific asset are automatically suppressed if that asset
   is currently in "Maintenance Mode" or "Out of Service Mode" in the Digital Twin.
3. **Hysteresis**: Once an alarm is active, the value must return to a $Z$-Score of < 2.0 before the alarm can be
   cleared. This prevents rapid toggling on a noisy signal.

## Alarm Prioritization & Response Target Matrix 
Oversight uses a targeted priority distribution to prevent **High-Priority Saturation** and **Alarm Fatigue**.

| Priority     | HMI Color (ISA-101) | Icon | Audio Pattern | SLA / Action Allowance                                                                    | Target Distribution |
|:-------------|:--------------------|:-----|:--------------|:------------------------------------------------------------------------------------------|:--------------------|
| **Critical** | Red                 | ◇    | Rapid Pulse   | **60 Seconds.** Immediate safety threat. Autonomous failover triggers if SLA is breached. | ~5%                 |
| **High**     | Orange              | ▽    | Fast Tone     | **5 Minutes.** Process drift. AI proposes intent; awaits Two-Key Auth.                    | ~10%                |
| **Warning**  | Amber               | △    | Steady Tone   | **15 Minutes.** Sustained drift. Auto-logs to Maintenance Management System.              | ~15%                |
| **Advisory** | Blue/Gray           | □    | Visual Only   | **No immediate action.** AI detected subtle drift; background tracking.                   | ~70%                |

## Operator Response

### Response Protocols
Every alarm in the Oversight Dashboard must be accompanied by a Reasoning Trace and a RAG-proposed action.

1. **Initial Response**: The AI must state, "I am alerting because [DATA POINT] has drifted by [Z-SCORE], which exceeds 
  the threshold of [THRESHOLD]."
2. **Recommended Action**: The AI must provide a recommended action, such as "I recommend checking the [SENSOR] for 
  potential fouling or recalibration."
3. **AI Confidence Level**: Every alarm is assigned a **Statistical Correlation Score** (SCC) calculated as 
  $1 - (p\text{-value of current } Z\text{-drift})$.
   - **Display**: Confidence is shown as a **Reliability Gauge** on the HMI.
   - **Low Confidence Protocol**: If confidence is < 85%, the gauge turns **Yellow**, and the alarm is flagged as a
     potential sensor fault. 
   - **Operator Requirement**: For alarms < 85%, the operator must verify the sensor health using 
     [SOP: Sensor Validation](/docs/sop/SOP-OPS-001_Sensor-Validation.md) before proceeding.
4. **Confirmation**: The operator must **ACK** the alarm to silence the audio and validate the AI's failover proposal to
  execute the recommended action.
5. **Resolution**: The AI must provide a resolution statement, such as "I have resolved the alarm and will no longer 
  be alerted for this sensor." 
6. **Auditability**: Operators must provide a comment when acknowledging "Critical" alarms.
7. **Explainability**: The system cites the specific manufacturer manual section justifying the recommended response.

## Advanced Alarm Management & Nuisance Suppression

### Alarm Suppression & Shelving
Operators are permitted to "Shelve" an alarm for a maximum of **4 hours** during maintenance. Shelving an alarm will 
suppress it from the dashboard for the remainder of the shelving period. However, the AI will continue to monitor the 
underlying data and will automatically "Unshelve" the alarm if the $Z$-Score increases by a factor of 1.5, indicating 
a critical escalation. The system will then force a re-trigger of the alarm and alert the operator of the worsening 
condition. **The system limits the number of concurrent shelved alarms to 5**. 

[!Advisory Response] If the limit of five concurrent shelved alarms is reached, the system issues a 
`SHELVED_ALARM_CAPACITY_EXCEEDED` advisory. Operators must immediately execute SOP-SHELVED-01 to audit and clear shelf 
slots to maintain system situational awareness.
- **Schema Enforcement**: Configuration files **MUST** include `max_shelf_duration_minutes:240`.
- **Capacity Limits**: The system limits the number of concurrent shelved alarms to **5**.
- **Capacity Advisory**: If the limit of five is reached, the system issues a `SHELVED_ALARM_CAPACITY_EXCEEDED` 
  advisory. Operators must execute `SOP-SHELVED-01` to audit and clear shelf slots to maintain plant situational 
  awareness.
- **Re-Trigger Logic**: If the shelved alarm's $Z$-Score increases by an additional factor of **1.5** (e.g., drifting
  from $3.0\sigma$ to $4.5\sigma$), the system overrides the shelf, forces an immediate re-trigger on the dashboard, 
  and alerts the operator to the worsening condition.

> [!NOTE]
> **Deadband Logic**: Once an alarm is triggered, the process variable must return past the setpoint by the deadband 
> percentage before the alarm state can be cleared. This creates a buffer zone around the setpoint to stabilize the 
> alarm status.

### Advanced Nuisance Suppression (FMEA Alignment)
- **Oscillation Dampening (Alarm Suppression)**: Post-Action Oscillation Dampening
  To prevent chattering immediately following an autonomous action (e.g., the AI updates a setpoint by 4%), the system 
  enforces a **15-minute Stability Lock** on new alarms for that specific tag, allowing the fluid dynamics or thermal 
  inertia to settle into the new baseline. *(Note: This lock NEVER suppresses Level 1 SIS interlocks).*
- **Command Timeout Distinction**: This is separate from the 5-second pessimistic confirmation timeout, which applies 
  only to the network handshake between the API and the Southbound Driver.
- **Sensor Integrity**: Alarms are suppressed if the telemetry `Quality` bit is $< 192$. The system will instead issue 
  a `SENSOR_FAULT_ADVISORY`.
- **Safety Override**: The 15-minute alarm stability lock **Never** suppresses Emergency Stops or manual Level 1 safety 
  interlocks.

## Revision History

| Version | Date       | Author    | Description of Change                                                                   | Approved By |
|:--------|:-----------|:----------|:----------------------------------------------------------------------------------------|:------------|
| 1.0     | 2026-04-18 | armummert | Initial baseline creation and Z-Score definitions.                                      | armummert   |
| 1.1     | 2026-06-17 | armummert | Added strict ISA-18.2 KPI targets, Response Windows, Shelving rules, and MOC alignment. | armummert   |