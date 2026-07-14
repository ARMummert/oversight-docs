# OverSight ICS: API & Message Schema Specification (v1)

**Security Level:** `Level 3.5 IDMZ`
**Version**: 1.3 (N-1 Deprecation Policy) See [Versioning Policy](/docs/versioning_policy.md)

## 1. Authentication & Authorization
All API requests must be authenticated using RS256-signed JWTs.

### 1.1 Mandatory Headers
| Header                      | Value          | Description                                                      |
|:----------------------------|:---------------|:-----------------------------------------------------------------|
| `X-Site-ID`                 | `UUID`         | Identifies the physical plant location.                          |
| `Authorization`             | `Bearer <JWT>` | **Primary Operator JWT** (8-hour shift expiry).                  |
| `X-Secondary-Authorization` | `Bearer <JWT>` | **Secondary Operator JWT** required for High-Impact or Override. |

### 1.2 Security & Token Strategy
- **Dual-Auth Enforcement**: For `/orchestration/override`, the system verifies `primary_jwt.sub != secondary_jwt.sub`.
- **Role Requirements**: The secondary token must contain a `Supervisor` or `Safety Lead` claim.
- **Grace Window**: Expired tokens during high-impact events are cached for 60 seconds to allow for re-authentication 
  without command loss.

### 1.3 JWT Validation & Grace Period
The Gateway enforces the following logic to maintain cognitive reliability during failover operations:

- **Signature Verification**: All tokens are validated using RS256 against the IdP's(Identity Provider) public key.
- **Claim Validation**: The system strictly validates `exp` (expiry), `nbf` (not before), and `iss` (issuer) claims.
- **Refresh Strategy**: Refresh tokens are **disabled** for Level 3.5 IDMZ. A session expiration requires fill
  reauthentication.
- **Operational Persistence**: If a token expires during an active `/api/v1/incidents/{id}/confirm` request, the
  **60-second grace window** allows the handshake to complete, preventing a safety trip due to administrator timeout.

### 1.4 Dual-Authorization (Two-Key Protocol)
The `/orchestration/override` endpoint requires two distinct authorized users to be physically or logically present.

- **Secondary JWT Obtainment**: The secondary JWT is generated via a separate HMI challenge-response process. This
  requires a different set of credentials than the primary operator to guarantee true two-person integrity.
- **Anti-Self-Authorization**: The Gateway cross-references the `sub` (subject) claims of both the primary and secondary
  JWTs to prevent self-authorization. If `primary_jwt.sub == secondary_jwt.sub`, the request is rejected with a 
  `401 ERROR_INVALID_DUAL_AUTH` error.
- **Role Separation**: While the primary JWT requires a `Operator` role, the secondary JWT must contain a 
  `Supervisor` or `Safety Lead` claim.

### 1.5 OAuth2/OIDC Token Scope Mappings
To enforce Least Privilege across the Enterprise, the Gateway validates specific OAuth2 scopes embedded within the JWT. 

| Endpoint Family                    | Required Scope          | Permitted Roles                          |
|:-----------------------------------|:------------------------|:-----------------------------------------|
| `/api/v1/telemetry/*`              | `telemetry:read`        | Operator, Supervisor, Safety Lead        |
| `/api/v1/telemetry/ingest`         | `telemetry:write`       | Service_Account (Northbound Ingest)      |
| `/api/v1/orchestration/intent-log` | `orchestration:read`    | Operator, Supervisor, Safety Lead        |
| `/api/v1/orchestration/override`   | `orchestration:execute` | Supervisor, Safety Lead (Dual-Auth Only) |
| `/api/v1/incidents/*/confirm`      | `incident:confirm`      | Operator, Supervisor                     |
| `/api/v1/assets/*/mode`            | `asset:maintenance`     | Supervisor, Safety Lead                  |
| `/api/v1/alarms/*`                 | `alarms:acknowledge`    | Operator, Supervisor, Safety Lead        |

## 2. API Endpoints

### 2.1 Asset Management
| Endpoint                       | Method   | Description                                                 |
|:-------------------------------|:---------|:------------------------------------------------------------|
| `/api/v1/assets`               | `GET`    | Retrieve all monitored assets and current Z-Scores.         |
| `/api/v1/assets/{id}`          | `GET`    | Get health metrics, RAG-indexed manuals, and asset history. |
| `/api/v1/assets/{id}/mode`     | `PATCH`  | Places an asset into Maintenance or Out of Service Mode.    |
| `/api/v1/assets/{id}/lockout`  | `POST`   | Manually Tag-Out an asset to prevent AI-driven failover.    |
| `/api/v1/assets/{id}/lockout`  | `DELETE` | Unlock / Reverse a manual tag-out state on an asset.        |

### 2.2 Telemetry Ingest (Southbound)
| Endpoint                   | Method | Rate Limit | Description                                                                 |
|:---------------------------|:-------|:-----------|:----------------------------------------------------------------------------|
| `/api/v1/telemetry/ingest` | `POST` | 1000 req/s | Receive telemetry; strictly requires **Integer** Quality field.             |
| `/api/v1/telemetry/query`  | `GET`  | 100 req/s  | Retrieve telemetry data from WORM for a specific asset.                     |

#### Telemetry Payload Schema
```json
{
  "tag_id": "PUMP_01_FLOW",
  "value": 45.2,
  "quality": 192, 
  "timestamp": "2026-04-24T11:53:00Z"
}
```

### 2.3 Incident Orchestration & Context
| Endpoint                           | Method | Rate Limit   | Description                                                      |
|:-----------------------------------|:-------|:-------------|:-----------------------------------------------------------------|
| `/api/v1/incidents/active`         | `GET`  | 10 req/s     | List all open anomalies requiring operator attention.            |
| `/api/v1/incidents/{id}/trace`     | `GET`  | 10 req/s     | Retrieve the LangGraph reasoning trace for an incident.          |
| `/api/v1/incidents/{id}/confirm`   | `POST` | **1 req/2s** | Execute **Pessimistic Confirmation** (Rate-limited).             |
| `/api/v1/orchestration/override`   | `POST` | 10 req/s     | **Two-Key Bypass**. Requires secondary JWT for High Impact.      |
| `/api/v1/orchestration/intent-log` | `GET`  | 10 req/s     | Retrieves the immutable reasoning trace before confirmation.     |

#### Orchestration Payload Schema (Northbound)
`/api/v1/incidents/{id}/confirm (POST)` 
```json
{
  "operator_intent": "CONFIRM_TRIP",
  "reasoning_trace_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "comments": "Confirmed anomalous vibration signature on secondary bearing."
}
```

`/api/v1/orchestration/override (POST)`
```json
{
  "target_asset_id": "PUMP_01",
  "override_action": "MANUAL_RESTART",
  "justification": "Cleared physical obstruction on Line 1. Ready for startup.",
  "secondary_token_present": true
}
```

### 2.4 System Health & Integrity
| Endpoint                    | Method | Description                                                        |
|:----------------------------|:-------|:-------------------------------------------------------------------|
| `/api/v1/system/heartbeat`  | `POST` | Driver reports latency, PTP sync, and CPU usage.                   |
| `/api/v1/system/heartbeat`  | `GET`  | Exposes PTP sync status and watchdog latency for the HMI and SIEM. |
| `/api/v1/audit/verify`      | `GET`  | Initiates SHA-256 integrity check across WORM log blocks.          |
| `/api/v1/system/compliance` | `GET`  | Returns current Continuous Compliance Monitoring metrics.          |

#### System Heartbeat Payload Schema (`GET`)
```json
{
  "ptp_sync_status": "LOCKED",
  "clock_drift_ns": 120,
  "watchdog_pet_latency_ms": 15
}
```

### 2.5 Alarm Management
| Endpoint                          | Method | Description             |
|:----------------------------------|:-------|:------------------------|
| `/api/v1/alarms/active`           | `GET`  | List all active alarms. |
| `/api/v1/alarms/{id}/acknowledge` | `POST` | Acknowledge an alarm.   |
| `/api/v1/alarms/{id}/shelve`      | `POST` | Shelve an alarm.        |
| `/api/v1/alarms/{id}/unshelve`    | `POST` | Unshelve an alarm.      |

#### Alarm Management Payload Schemas
`/api/v1/alarms/{id}/acknowledge (POST)`
```json
{
  "operator_comment": "Validating sensor drift per SOP-01"
}
```

`/api/v1/alarms/{id}/shelve (POST)`
```json
{
  "max_shelf_duration_minutes": 240, 
  "reason_code": "MAINTENANCE"
}
```

### 2.6 Agentic & Cognitive Engine (LangGraph / RAG)
| Endpoint                           | Method | Description                                                                                                                           |
|:-----------------------------------|:-------|:--------------------------------------------------------------------------------------------------------------------------------------|
| `/api/v1/agent/execute`            | `POST` | Execute an agentic task (e.g., RAG query, manual snippet retrieval). Payload must include `task_type`.                                |
| `/api/v1/agent/analyze`            | `POST` | Requests a reasoning trace and a RAG-proposed action from the AI based on a process deviation. Retrieves current snapshot from Redis. |
| `/api/v1/agent/task-log`           | `GET`  | Retrieve the immutable log of agentic task executions, including inputs, outputs, and reasoning traces.                               |
| `/api/v1/agent/forbidden-list`     | `GET`  | Retrieve the current list of forbidden registers or commands that the AI is not allowed to interact with.                             |
| `/api/v1/agent/confidence`         | `GET`  | Retrieves the statistical correlation score. If <85%, routes to Sensor Validation SOP.                                                |
| `/api/v1/agent/test/adversarial`   | `POST` | Runs the LG-05 prompt injection FAT test to ensure the agent correctly neutralizes adversarial input.                                 |

### 2.7 Control Authority & Autonomous Override
| Endpoint                                   | Method | Description                                                                                                              |
|:-------------------------------------------|:-------|:-------------------------------------------------------------------------------------------------------------------------|
| `/api/v1/control/status`                   | `GET`  | Retrieves the current control authority status. Possible values: `HUMAN`, `AI`, `LOCKED`.                                |
| `/api/v1/control/override/safe-state`      | `POST` | Executed by the predictive failover agent when a critical alarm times out (< 60 seconds). Initiates Emergency Trip/Halt. |
| `/api/v1/control/override/load-reduce`     | `POST` | Executed when a 'High' alarm times out (< 5 minutes). Agent safely reduces the system load to baseline parameters.       |
| `/api/v1/control/authority`                | `PUT`  | Toggles the control authority between manual (human) and autonomous (agent) modes using **Bumpless Transfer**.           |

#### Authority Payload Schema
```json
{
  "requested_mode": "AUTONOMOUS",
  "write_access_token_req": true
}
```

### 2.8 Hardware Root of Trust 
| Endpoint                  | Method | Description                                                                                                                               |
|:--------------------------|:-------|:------------------------------------------------------------------------------------------------------------------------------------------|
| `/api/v1/security/attest` | `GET`  | Retrieves the current PCR (Platform Configuration Register) values from the TPM 2.0 and the active SHA-256 hash of the Southbound Driver. |

#### Hardware Root of Trust Payload Schema
```json
{
  "tpm_pcr_measurements": { "pcr0": "...", "pcr7": "..." },
  "southbound_driver_hash": "4f8a2b9c3d1e7f6a5b0d8e9c2a1b3f4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b",
  "attestation_signature": "<tpm_signed_payload>"
}
```

### 2.9 RAG Knowledge Integrity
| Endpoint              | Method | Description                                                                   |
|:----------------------|:-------|:------------------------------------------------------------------------------|
| `/api/v1/rag/ingest`  | `POST` | Uploads the new SOP/Manual into the vector database.                          |
| `/api/v1/rag/context` | `GET`  | Internal Endpoint for the LangGraph Agent to retrieve verified manual chunks. |

#### Query Params
`?asset_id=PUMP_001&semantic_hash=<hash>`

#### RAG Ingest Payload Schema
```json
{
  "knowledge_id": "MAN-PLC-01",
  "knowledge_type": "MANUAL",
  "software_revision": "v1.0.0",
  "document_payload": "<base64_encoded_pdf>"
}
```

### 2.10 UNS Metadata Management 
| Endpoint                       | Method  | Description                                           |
|:-------------------------------|:--------|:------------------------------------------------------|
| `/api/v1/assets/{id}/metadata` | `PATCH` | Updates the static asset metadata payload in the UNS. |

#### UNS Metadata Payload Schema
```json
{
  "next_audit_due": "2027-01-15",
  "service_manual_ref": "VFD-AB-04-A",
  "owner": "Lead_OT_Engineer"
}
```

## 3. Standard Error Responses & HMI Logic
| Status Code | Error Message                 | Scenario                                                                                            |
|:------------|:------------------------------|:----------------------------------------------------------------------------------------------------|
| `400`       | `ERROR_INVALID_PAYLOAD`       | Payload is missing or malformed.                                                                    |
| `401`       | `ERROR_INVALID_DUAL_AUTH`     | Primary and Secondary operators are the same user.                                                  |
| `403`       | `FORBIDDEN_HARDWARE_LOTO`     | Command rejected; physical Local/Hand switch is active.                                             |
| `403`       | `FORBIDDEN_SOFTWARE_TAGOUT`   | Command rejected; asset is logically tagged out.                                                    |
| `404`       | `ASSET_NOT_FOUND`             | Asset not found.                                                                                    |
| `408`       | `TIMEOUT_PESSIMISTIC_CONFIRM` | Feedback not received within the 5s window.                                                         |
| `409`       | `CONFLICT_ACTIVE_INCIDENT`    | Action conflicts with current asset state (e.g., trying to override a system that is already halted |
| `422`       | `SCHEMA_TYPE_MISMATCH`        | Quality bit is not an Integer.                                                                      |
| `423`       | `OPERATIONAL_LOCK`            | 15-minute stability lock after autonomous action. HMI renders the red "locked" icon.                |
| `429`       | `RATE_LIMIT_EXCEEDED`         | Request throttled by safety rate-limiter.                                                           |
| `429`       | `ERROR_SAFETY_DELAY_ACTIVE`   | Too many requests on high-impact commands. HMI displays countdown timer and disables input.         |
| `500`       | `INTERNAL_SERVER_ERROR`       | Internal server error.                                                                              |
| `503`       | `SERVICE_UNAVAILABLE`         | Gateway WORM database or AI Agent offline. HMI must default to degraded manual control mode.        |

## 4. Traffic Management & Rate Limiting
To prevent command saturation and **blind clicking** during high-impact events, the Gateway enforces the following:

- **Safety Throttling**: High-impact endpoints (e.g., `/incidents/{id}/confirm`, `/orchestration/override`) are rate-limited to one request per 2 seconds.
- **DDoS Protection**: Northbound API endpoints are rate-limited per Site-ID to prevent resource exhaustion.
- **HMI Feedback**: When a `429` (Rate-Limit Exceeded) error is triggered, the HMI must display a **Safety Delay Active** notification.

### 4.1 Safety Delay Active State
When a `429` status code is received, the HMI must immediately initiate the **Safety Delay Active** notification. This state enforces a mandatory pause to ensure the **Pessimistic Confirmation** window (5s) or the **Safety Throttling limit** (1 req/2s) is respected.

**Behavioral Requirements:**
- **State Trigger**: Triggered immediately upon receiving a `429` error from high-impact endpoints.
- **Visual Notification**: The HMI must display a **Safety Delay Active** overlay on the primary dashboard.
- **Command Lock**: All orchestration and confirmation buttons must be programmatically disabled (greyed out) to prevent further command stack-up.
- **Countdown Timer**: A visual countdown timer must be displayed, derived from the `Retry-After` header or the 2s throttling period.

### 4.2 Countdown Timer
- **Initialization**: The timer starts the moment a `429` error is trapped by the HMI client.
- **Duration**: Set to a minimum of 2 seconds for orchestration endpoints to match the Gateway's safety rate limit.
- **Resolution**: Upon reaching 0.0s, the **Safety Delay Active** notification is dismissed and the command buttons are re-enabled for a single attempt.

### 4.3 Operational Impact during Failover
| Command Buffering     |        HMI Behavior        | Purpose                                                             |
|:----------------------|:--------------------------:|:--------------------------------------------------------------------|
| **Command Buffering** | **Rejected (No queueing)** | Prevents delayed **phantom** commands from executing after a delay. |
| **Auditory Feedback** |  Single **Beep-Low** tone  | Alerts the operator that the input was blocked by safety logic.     |
| **Status Retention**  |    Maintain Trace View     | The LangGraph reasoning trace must remain visible during the delay. |

### 4.4 HMI State Table 
| HMI State               | Visual Element            | Function                                                 | 
|:------------------------|:--------------------------|:---------------------------------------------------------|
| **Nominal**             | **Grey Scale Dashboard**  | Standard monitoring per ISA-101                          |
| **Safety Delay Active** | **Yellow Banner + Timer** | Enforces 2s throttling on high-impact commands           |
| **Operational Lock**    | **Red "locked" Icon**     | 15-minute stability lock active after autonomous action. |

## 5. Unified Namespace (UNS) Schema
Data is expected in the following hierarchy: `Enterprise/Site/Area/Line/Asset/Tag`.

**Example Telemetry Payload:**
```json
{
  "timestamp": "2026-04-24T12:00:00Z",
  "asset_id": "PUMP_001",
  "metrics": {
    "vibration_rms": 4.5,
    "current_draw": 12.2,
    "discharge_pressure": 48.5
  },
  "metadata": {
    "units": "SI",
    "quality": 192 
  }
}
```

## Revision History
| Version | Date       | Author    | Description of Change                                                                                                                                                   | Approved By           |
|:--------|:-----------|:----------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:----------------------|
| 1.0     | 2026-04-24 | armummert | Comprehensive baseline; corrected schema, auth, and rate limits.                                                                                                        | Lead Systems Engineer |
| 1.1     | 2026-04-27 | armummert | Added Security & Token Strategy, Added detailed JWT validation and grace period, Added dual-auth (two-key protocol) and RBAC, and And implemented safety rate limiting. | Lead Systems Engineer |
| 1.4     | 2026-05-14 | armummert | Realigned endpoints with FAT testing, fixed HTTP methods (POST agent/analyze), added missing schemas, and repaired structural document formatting.                      | Lead Systems Engineer |