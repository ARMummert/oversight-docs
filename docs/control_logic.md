# OverSight ICS: Control Logic & PID Specifications

**Scope:** Real-Time Control Loops and Autonomous Mitigation

## 1. PID Loop Management
The OverSight ICS supervises existing Field Asset PID loops to detect subtle drift before they trigger physical limit alarms.

### 1.1 Bumpless Transfer & Integral Windup
- **Transition State**: During a "Bumpless Transfer" (e.g., when the AI assumes control or updates a setpoint), the PID integral term must be initialized to the current output value [cite: 3.1].
- **Windup Protection**: To prevent "Overshoot" during autonomous mitigation, all supervised loops must implement **Back-Calculation** or **Conditional Integration** to clamp the integral term when the actuator reaches its physical limit [cite: 3.1].
- **Reset Logic**: If the AI detects a `KPI_DISTRIBUTION_DEVIATION`, it will force a PID reset to "Nominal" factory parameters to eliminate accumulated error [cite: 3.1].

## 2. Autonomous Mitigation Cadence
- **Confirmation Window**: High-impact actions require a **5-second Pessimistic Confirmation** via the `/api/v1/incidents/{id}/confirm` endpoint [cite: 3.1].
- **Stability Lock**: Following any autonomous change, a **15-minute suppression lock** is applied to the tag's alarms to allow the process variable to settle within the new $Z$-Score window [cite: 3.1].
