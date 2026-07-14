# Telemetry Database Schema

### 1. Objective
This document defines the relational and time-series database schema for the Oversight ICS data layer. It is optimized
for high-ingest rates (> 10,000 points/sec) and sub-second similarity searches for AI Grounding.

### 2. Infrastructure Requirements

- **Engine** PostgreSQL 16+
- **Extensions**: 
  - `TimescaleDB` (time-series optimization) for telemetry
  - `pgVector` (semantic vector storing) for RAG embeddings
  - `pgcrypto` (UUID generation) and cryptographic hashing for intent justification

```POSTGRESQL
-- Initial Setup
CREATE EXTENSION IF NOT EXISTS "timescaledb" CASCADE; -- TimescaleDB extension for time-series optimizations
CREATE EXTENSION IF NOT EXISTS "pgcrypto"; -- Provides UUID generation logic
CREATE EXTENSION IF NOT EXISTS "vector"; -- Provides the VECTOR data type for RAG
```

### 3. Telemetry Hypertable
This is the core time-series table for storing raw telemetry data from field assets. Timescale's hypertable
architecture allows Oversight ICS to efficiently manage high-velocity data as the database grows to millions of rows.

```POSTGRESQL
-- Main Telemetry Table
CREATE TABLE telemetry (
    time TIMESTAMPTZ NOT NULL,
    asset_id VARCHAR(50) NOT NULL,        -- Links to the UNS Asset Name
    tag_name VARCHAR(100) NOT NULL,       -- e.g., 'discharge_pressure'
    value DOUBLE PRECISION NOT NULL,      -- Numeric value of the telemetry point
    quality INT DEFAULT 192               -- OPC-UA Quality Bit (192 = Good)
);

-- Convert to Hypertable partitioned by time
SELECT create_hypertable('telemetry', 'time');

-- Create index for rapid retrieval
CREATE INDEX ON telemetry (asset_id, time DESC);
```

### 4. RAG Vector Table (Knowledge Base)
This table stores the embedded manual fragments from the technical manuals. Oversight uses `pgcrypto's` unique 
constraint capabilities for the integrity of the justification hash.

```POSTGRESQL
CREATE TABLE knowledge_vector (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_type  VARCHAR(50) NOT NULL,
    content     TEXT NOT NULL,  
    justification_hash CHAR(64) UNIQUE NOT NULL, 
    metadata    JSONB,       
    embedding   VECTOR(384) NOT NULL        -- Optimized for MiniLM-L6-v2
);

-- Semantic search index
CREATE INDEX ON knowledge_vector USING ivfflat (embedding vector_cosine_ops);
```

### 5. Intent Outbox Table
This table service as the **Technical Contract** between the **Reasoning Agent** and the **Southbound Driver**. Every 
command must be persisted here before exection.

```POSTGRESQL

CREATE TABLE intent_outbox (
    -- Identity & Status
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id UUID NOT NULL,
    schema_version VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    -- Safety Deadlines & Time Stamps
    created_at TIMESTAMPTZ DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL,
    processed_at TIMESTAMPTZ,
    -- Target & Action
    asset_id VARCHAR(50) NOT NULL,
    proposed_intent JSONB NOT NULL,
    metadata JSONB,
    -- Safety, Authority & RAG
    justification_hash TEXT NOT NULL,
    operator_approval_id VARCHAR(50),
    signature_tpm TEXT,
    -- Execution State
    status VARCHAR(20) DEFAULT 'PENDING',
    attempt_count INT DEFAULT 0,
    error_message TEXT,
    -- Auditing
    created_by VARCHAR(50),
    updated_by VARCHAR(50)                           
);
-- Index 1: Driver Queue Performance (Finds next valid command)
CREATE INDEX idx_outbox_driver_queue ON intent_outbox (status, expires_at) WHERE status = 'PENDING';

-- Index 2: Forensic Audit Trail (Reconstructs a specific reasoning path)
CREATE INDEX idx_outbox_audit_trail ON intent_outbox (trace_id, asset_id);
CREATE INDEX idx_intent_outbox_pending ON intent_outbox (status) WHERE status = 'PENDING';

```

### 5.1 Proposed Intent Payload Structure
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
### 6. Intent Archive Table
This table is used to store the historical intents that have been successfully executed. It inherits the same schema as
the `intent_outbox` table.

```POSTGRESQL
CREATE TABLE intent_archive (
    -- Inherits all columns from intent_outbox
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id UUID NOT NULL,
    schema_version VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    created_at TIMESTAMPTZ DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL,
    processed_at TIMESTAMPTZ,
    asset_id VARCHAR(50) NOT NULL,
    proposed_intent JSONB NOT NULL,
    metadata JSONB,
    justification_hash TEXT NOT NULL,
    operator_approval_id VARCHAR(50),
    signature_tpm TEXT,
    status VARCHAR(20) DEFAULT 'PENDING',
    attempt_count INT DEFAULT 0,
    error_message TEXT,
    created_by VARCHAR(50),
    updated_by VARCHAR(50)                           
);

-- Enable TimescaleDB compression for the archive to save Edge storage
SELECT create_hypertable('intent_archive', 'created_at');
ALTER TABLE intent_archive SET (timescaledb.compress, timescaledb.compress_segmentby = 'asset_id');
```

### 6. Analytical Views
To reduce the CPU load on the Reasoning Agent, $Z$-Score logic is moved to a database view.

```POSTGRESQL
CREATE OR REPLACE VIEW asset_health AS
SELECT 
    time,
    asset_id,
    tag_name,
    value,
    -- Time-based window provides consistent Z-Score baselines
    (value - avg(value) OVER w) / NULLIF(stddev(value) OVER w, 0) AS z_score
FROM telemetry
WINDOW w AS (
    PARTITION BY asset_id, tag_name 
    ORDER BY time 
    RANGE BETWEEN INTERVAL '1 hour' PRECEDING AND CURRENT ROW
);
```

### 7. Forbidden Registers Table
This table is used to store the registers that are strictly forbidden from being written to.

```POSTGRESQL
CREATE TABLE forbidden_registers (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id         VARCHAR(50) NOT NULL, 
    register_address VARCHAR(100) NOT NULL, -- e.g., '40001' or 'PLC_A.Safety_Trip'
    reason           TEXT NOT NULL,         -- e.g., 'Emergency Stop Bypass'
    severity_level   INT DEFAULT 5,         -- 1-5 (5 = Absolute Block)
    created_at       TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_forbidden_lookup ON forbidden_registers (asset_id, register_address);
```

### 8. Operator Sessions Table
This table is used to store the operator sessions which are required to track who is currently authorized to approve
high-risk intents.

```POSTGRESQL
CREATE TABLE operator_sessions (
    operator_id      VARCHAR(50) PRIMARY KEY,
    session_token    TEXT NOT NULL,         -- Verified MFA Token
    role             VARCHAR(20) NOT NULL,  -- Lead Engineer, Safety Lead, Operator
    expires_at       TIMESTAMPTZ NOT NULL,  -- Short-lived authority (e.g., 4 hours)
    last_action_at   TIMESTAMPTZ DEFAULT now()
);
```

### 9. Audit Trail Table
This table is used to store the audit trail of all actions taken by the system.

```POSTGRESQL
CREATE TABLE audit_trail (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_time TIMESTAMPTZ DEFAULT now(),
    operator_id VARCHAR(50) NOT NULL,
    client_ip INET,  -- Network origin
    action_type VARCHAR(20), -- e.g., `INSERT, UPDATE, DELETE`
    table_name VARCHAR(50),
    record_id UUID, -- The ID of the row that was changed
    old_values JSONB,
    new_values JSONB,
    trace_id UUID
);

CREATE INDEX idx_audit_forensics ON audit_trail (event_time, table_name, record_id);
```

### 10. Asset Configuration Table
This table is used to store the configuration of each asset.

```POSTGRESQL
CREATE TABLE asset_config (
    asset_id         VARCHAR(50) PRIMARY KEY,
    unit             VARCHAR(10),
    min_setpoint     DOUBLE PRECISION,
    max_setpoint     DOUBLE PRECISION,
    critical_threshold DOUBLE PRECISION,
    updated_at       TIMESTAMPTZ DEFAULT now()
);
```

### 11. Data Retention & Archival Policy

Partitioning and automated data lifecycle management to prevent unbounded growth.

- **Telemetry**: TimescaleDB `retention_policy` set to 90 days. Older data is compressed or moved to cold storage.
- **Intent Outbox**: Rows with `status` of `SUCCESS` or `FAILED` older than 30 days are automatically moved to an
  `intent_archive` table to maintain polling performance.