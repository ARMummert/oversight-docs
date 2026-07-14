# OverSight ICS: Technical Contracts & Data Handshakes

## 1. The Transactional Outbox Schema (PostgreSQL)
**AUTHORITATIVE SOURCE**: The SQL definition below is the master schema for the Oversight system. All references in
`TELEMETRY_DATABASE_SCHEMA` or other documentation are non-authoritative and must mirror the section exactly.

The `outbox` table is the single source of truth for the **Southbound Driver**. All AI intents must be persisted here 
before execution.

```sql
CREATE TABLE intent_outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id UUID NOT NULL,               -- Link to LangGraph Reasoning Trace
    asset_id VARCHAR(50) NOT NULL,        -- e.g., 'PUMP_001'
    status VARCHAR(20) DEFAULT 'PENDING', -- PENDING, EXECUTING, SUCCESS, FAILED
    
    -- The Command Contract
    payload JSONB NOT NULL,               -- Detailed JSON (See Section 1.1)
    
    -- Security & Timing
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,      -- Command TTL (Default: created_at + 2000ms)
    
    -- Forensic Traceability
    justification_hash TEXT NOT NULL,     -- SHA-256 of the RAG citation context
    operator_approval_id UUID             -- Required for 'High Impact' actions
);
```

### 1.1 Intent Payload Structure
Every JSON entry in the payload column must adhere to this contract to pass the Semantic Gatekeeper.

```json
{
  "command_type": "FAILOVER_SEQUENCE",
  "priority": "HIGH",
  "sequence": [
    {
      "step": 1,
      "action": "WRITE_REGISTER",
      "address": "40001",
      "value": 1,
      "description": "Start Standby Asset"
    },
    {
      "step": 2,
      "action": "WAIT_FOR_VALUE",
      "address": "40005",
      "value": 60.0,
      "tolerance": 0.5,
      "timeout_ms": 5000,
      "description": "Wait for Standby RPM >= 60Hz"
    }
  ],
  "safety_envelope": {
    "max_pressure": 120.0,
    "max_current": 45.0
  }
}
```
### 1.2 Asset-Specific Payload Schemas

### 1.2.1 PLC Payload
For standard I/O and logic solvers, the payload focuses on discrete memory addresses:

```json
{
  "asset_type": "PLC",
  "action": "WRITE_REGISTER",
  "address": "40012",
  "value_type": "INT16",
  "value": 1
}
```

### 1.2.2 VFD Payload
For motor drives, the payload uses engineering-unit abstractions to prevent scaling errors in the AI:

```json
{
  "asset_type": "VFD",
  "action": "SET_SPEED",
  "units": "Hz",
  "target_value": 45.5,
  "ramp_rate_sec": 10.0,
  "torque_limit_pct": 110
}
```

*Note: The Southbound Driver is responsible for translating the "Hz" value into the specific raw hex value required 
by the VFD manufacturer's protocol.*

## 2. Sequence of Operations (SOOs): Bumpless Transfer

### Phase A: Dry Run & Isolation Check

1. **Pre-Condition Check** Before initiating the 20Hz spin, the Agent must verify the physical isolation state.
2. **Common Header Logic**: If the suction/discharge valves are shared with the primary active circuit (Common
   Header), the Dry Run is **FORBIDDEN** unless a bypass or min-flow path is confirmed **OPEN**. If no isolation is
   possible, skip to Phase B.
3. **Execution**: Command the Standby asset to its minimum frequency (e.g., 20Hz for a VFDs) with zero load.

### Phase B: Synchronization

To guarantee **Process Stability**, the Agent must follow this hard-coded sequence when generating a failover intent.

| Phase       | Action                                                          | Success Criteria                   | Failure Action             | 
|:------------|:----------------------------------------------------------------|:-----------------------------------|:---------------------------|
| 1. Sync     | Push PID State: Sync Primary Output_CV and `I-Term` to Standby. | ACK from Standby; **Delta < 0.5%** | Abort Failover; Alarm      |
| 2. Initiate | Command Standby to `START` at minimum ramp                      | Standby `Running` bit=TRUE         | Retry (Max 2); Alarm       |
| 3. Equalize | Ramp Standby until Discharge Pressure matches Primary           | Pressure Delta < 2%                | Hold State; Operator Veto  |
| 4. Handover | Command Primary to `STOP` while Standby takes load              | Primary `Current` = 0              | Alarm: **Incomplete Stop** |
| 5. Verify   | Monitor UNS for 30s stability                                   | $Z$-Score < 1.5.                   | Revert to Phase 1          |

> [!IMPORTANT]
> Bumpless Transfer Requirement: To prevent pressure surges during handover, the Standby Controller's Integral term 
> must be preloaded with the Active Controller's current value. This prevents the "I-term" from starting at zero and 
> causing a massive initial correction (overshoot).

### 2.1 Performance-Scaled State Coherence
To guarantee true **Bumpless Transfer** with non-identical physical characteristics (e.g., impeller wear, motor 
efficiency, or pipe friction), the Reasoning Agent must perform **Dynamic Gain Scaling** on the PID state variables 
before injecting them into the Standby Controller.

### The Wear-Incompatibility Risk: 

If field asset A is wearing out and field asset B is new, asset A's **Integral (I-term)** will be much larger than 
asset B's **Integral (I-term)** and the Standby Controller will be unable to correctly compensate for the difference. 
Injecting asset A's raw **I-term** into asset B would result in an immediate overshoot and potential high-pressure
trip.

### Non-Identical Asset Mitigation (Warm-Up & Normalization)
To account for incompatibility risk between two field assets where the Primary and Standby controllers exhibit 
different wear patterns or mechanical characteristics, the Reasoning Agent must execute the following phased 
transition. This prevents surges caused by transferring control states between non-identical physical endpoints.

### Phase A: Zero-Load Profiling
Before the Standby field asset assumes the process load, the Reasoning Agent will initiate a **Dry Run** sequence.

1. **Command**: The Southbound Driver commands Asset B to its minimum stable operating speed (e.g., 20Hz for a VFD).
2. **Measurement**: The Agent captures the Baseline Amperage ($A_{base}$) and Power Consumption ($P_{base}$) under 
   zero-load conditions.
3. **Validation**: This baseline is compared against the Digital Twin’s "New" commissioning data to establish a current 
   **Degradation Offset.**

### Phase B: Scaling Factor Calculation ($R_p$)
The Agent shall not use raw output values ($CV$) from the Primary asset. Instead, it must calculate and apply a 
**Performance Ratio ($R_p$)**:$$R_p = \frac{\mu_{Power}(Asset_{Standby})}{\mu_{Power}(Asset_{Primary})}$$

1. **Logic**: If Asset B requires 10% more power than Asset A to achieve the same RPM/Flow due to impeller wear, 
   the Agent shall "up-scale" the output command string by 1.1x during the handover.
2. **Contract Requirement**: The $R_p$ value must be logged in the justification_hash for the failover intent.

### Phase C: Integral (I-term) Clamping & Equalization
During the transition window defined in the Sequence of Operations (SOO), the following safety constraints apply to the 
Standby Controller:

1. **Preloading**: The scaled I-term is injected into the Standby PID register.
2. **Safety Clamping**: The Agent commands the field asset to enforce a 120-second **Stability Lock**. During this 
   window, the I-term is restricted to a movement within $\pm 2\%$ of the injected value.
3. **Purpose**: This prevents **Integral Wind-up** surges if the physical equalization (pressure balancing) takes 
   longer than the software execution.

### Handover Gate: 
The "Handover" step (Step 4 in the SOO) is FORBIDDEN from executing until:

- The $R_p$ calculation is completed and verified.
- The Standby asset has maintained a stable discharge pressure within 5% of the Primary for a minimum of 15 seconds.
- **Standardized Tolerance**: The Standby asset has maintained a stable discharge pressure within 2% (aligned with
  FAT-03, ST-09) of the Primary for a minimum of 15 seconds.

## 3. State of Grace
When service startup initiates, the Agent is forbidden from issuing commands until the following warm-up is complete:

- **Discovery**: Query `GET /api/v1/assets` to map the current UNS hierarchy.
- **Observation Phase (600s)**: Ingest telemetry into TimescaleDB without active reasoning to stabilize the rolling 
  mean ($\mu$).
- **HMI Visibility**: The Agent shall broadcast a `SYSTEM_STATUS` message every 10 seconds during this window. The
  HMI **MUST** display: `**STATE OF GRACE: [x] SECONDS REMAINING**`.
  - `Agent_Authority_Ready`:
- **Integrity Sync**: Compare physical register values against the last known state in the Digital Twin.
- **Authority Handshake**: Set `Agent_Authority_Ready` bit to `TRUE` on the Southbound Driver only after the 600ms 
  timer expires.

## Revision History
| Version | Date       | Author    | Description of Change                          | Approved By |
|:--------|:-----------|:----------|:-----------------------------------------------|:------------|
| 1.0     | 2026-04-17 | armummert | Initial drafting of Outbox Schema and SOOs.    | armummert   |