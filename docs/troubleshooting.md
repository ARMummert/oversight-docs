
# Oversight ICS Troubleshooting Guide

## 1. Purpose

This document provides **step-by-step troubleshooting procedures** for diagnosing and resolving failures within the 
OverSight ICS system across:

* AI Reasoning Layer
* Southbound Control Interface
* Telemetry Ingestion Pipeline
* Network & Time Synchronization
* Safety & Watchdog Systems

Intended for:

* Control Engineers
* OT Engineers
* System Integrators
* On-call Support Personnel

---

## 2. Safety Notice (MANDATORY)

**Before ANY action:**

1. Confirm system is in a **safe operational state**
2. Verify:

   * No active failover is in progress
   * No critical alarms are unacknowledged
3. If unsure:

   * Switch to **MONITOR-ONLY MODE**
   * OR initiate **AI LOCKOUT (see SOP-03)**

**NEVER:**

* Override safety interlocks
* Bypass LOTO protections
* Force-write to forbidden registers

**DO NOT restart services if:**

* Active failover is in progress
* System is in **CONTROL_ENABLED** with live actuation
* Watchdog instability is present

Switch to **MONITOR-ONLY** before restarting any control-related service.

---

# 3. System Health Quick Check (MANDATORY FIRST STEP)

## 3.1 Container & Service Health

```bash
docker ps
docker stats --no-stream
```

Verify:

* All containers are **UP**
* CPU < 80%, Memory < 85%

```bash
curl http://localhost:8000/health
```

Expected:

```json
{
  "status": "healthy",
  "services": {
    "ingest": "up",
    "reasoning": "up",
    "southbound": "up",
    "database": "up",
    "cache": "up"
  }
}
```

---

## 3.2 System Mode Verification (CRITICAL)

```bash
curl http://localhost:8000/system/mode
```

Possible values:

* `CONTROL_ENABLED`
* `ADVISORY_MODE`
* `MONITOR_ONLY`
* `AI_LOCKOUT`

**Always verify mode before taking action**

---

## 3.3 HMI Quick Indicators

| Indicator    | Meaning                    | Immediate Action                    |
| ------------ | -------------------------- | ----------------------------------- |
| Red Strobe   | SECURITY_VIOLATION_LOCKOUT | Check Gatekeeper logs + verify LOTO |
| Amber Pulse  | AGENT_STALL                | Restart reasoning agent (if safe)   |
| Grey Overlay | DATA_STALE                 | Check Southbound NIC + time sync    |
| Blue Border  | ADVISORY_MODE              | Manual confirmation required        |

---

## 3.4 Critical Error Codes

| Code            | Meaning              | Action                     |
| --------------- | -------------------- | -------------------------- |
| 403_LOTO        | Hardware lock active | Set HOA switch to AUTO     |
| 408_PESSIMISTIC | No feedback          | Verify power + mapping     |
| 503_TPM_ERR     | Signature failure    | Check TPM + rotate keys    |
| Z_SCORE_NAN     | No baseline          | Wait for Observation Phase |

---

# 4. Common Issues & Resolutions

---

## 4.1 No Telemetry Data

Diagnostics:

```bash
docker logs mqtt-broker
docker logs ingest-service
```

```sql
SELECT * FROM telemetry ORDER BY ts DESC LIMIT 10;
```

Resolution:

* Restart ingest service (if safe)
* Verify PLC connectivity
* Validate tag mapping
* Check firewall ports

---

## 4.2 AI Not Producing Decisions

Look for:

* STATE_OF_GRACE_ACTIVE
* NO_ANOMALY_DETECTED
* RAG_QUERY_FAILED

Resolution:

* Wait for **Observation Phase (≤600s)**
* Restart reasoning agent (if safe)
* Verify telemetry + RAG ingestion

---

## 4.3 Failover Not Executing

Check:

```sql
SELECT * FROM intent_outbox WHERE status='PENDING';
```

Resolution:

* Complete Two-Key confirmation
* Reissue command if TTL expired (>2s)
* Check safety rejection logs

---

## 4.4 Alarm Flood

Diagnostics:

```sql
SELECT priority, COUNT(*) FROM alarms GROUP BY priority;
```

```sql
SELECT * FROM telemetry WHERE quality < 192;
```

Resolution:

* Switch to MONITOR-ONLY
* Check sensors
* Recalculate baseline (SOP-09)
* Review recent changes

---

## 4.5 Time Sync Failure

```bash
ptp4l -m
phc2sys -m
```

Resolution:

* Restart PTP
* Verify Grandmaster
* If unstable → MONITOR-ONLY

---

## 4.6 Database Performance

```sql
EXPLAIN ANALYZE SELECT * FROM intent_outbox WHERE status='PENDING';
```

Resolution:

* Ensure index on `status`
* Run VACUUM ANALYZE
* Archive old data

---

## 4.7 Redis Failure

```bash
docker logs redis
```

Resolution:

* Restart Redis
* UI will degrade (system still safe)

---

## 4.8 Watchdog Trip

### Thresholds

* Soft fault: >3s heartbeat loss
* Hard trip: >1500ms loss (3 cycles)
* Critical: >60s no recovery

Resolution:

1. Confirm plant stable
2. Restart southbound service
3. Re-establish heartbeat
4. Follow recovery sequence

---

## 4.9 Auth Failures

* Re-authenticate
* Verify RBAC
* Ensure distinct second JWT

---

## 4.10 RAG Issues

Resolution:

* Re-run SOP-AI-01
* Validate metadata
* Rollback knowledge version

---

## 4.11 Heartbeat Flatlining

Diagnostics:

* Check NIC link
* Verify PTP
* Monitor CPU starvation

Resolution:

* Verify subnet config
* Check clock quality
* Enforce CPU limits (≤40%)

---

## 4.12 AI Hallucination / Stall

Diagnostics:

* Check vector DB
* Check recursion depth
* Inspect the forbidden list

Resolution:

* Rebuild vector index
* Analyze trace logs

 **If >3 rejected intents:**

* Force system to **ADVISORY_MODE**
* Require operator intervention

---

# 5. Degraded Mode Behavior

| Failure         | Behavior                         |
|-----------------|----------------------------------|
| Redis           | UI degraded only                 |
| RAG             | AI loses context (advisory only) |
| Reasoning Agent | Passive monitoring               |

System MUST NOT issue control actions in degraded mode

---

# 6. Standard Recovery Sequence

1. Verify plant stability
2. Switch to MONITOR-ONLY
3. Restart the failed component
4. Validate telemetry
5. Enable ADVISORY_MODE
6. Require manual confirmation before CONTROL_ENABLED

---

# 7. Log Priority Order

1. Southbound Driver (execution truth)
2. Reasoning Agent (decision trace)
3. Audit Logs (compliance)
4. Database (historical)

---

# 8. Escalation Matrix

| Severity | Action                            |
|----------|-----------------------------------|
| P1       | AI Lockout + notify plant manager |
| P2       | Monitor mode + notify engineer    |
| P3       | Log + schedule fix                |
| P4       | Monitor                           |

---

## 8.1 Escalation Path

| Level | Action                      |
|-------|-----------------------------|
| L1    | Operator checks alarms      |
| L2    | OT Engineer reviews mapping |
| L3    | Systems Engineer debugging  |
| L4    | Vendor support              |

---

# 9. Log Locations

| Component  | Command                       |
|------------|-------------------------------|
| Ingest     | docker logs ingest-service    |
| Reasoning  | docker logs reasoning-agent   |
| Southbound | docker logs southbound-driver |
| DB         | docker logs postgres          |
| Cache      | docker logs redis             |

---

# 10. When to STOP

STOP if:

* Safety cannot be guaranteed
* Safety systems involved
* Repeated watchdog trips
* AI behavior unstable

---

# 11. Related Documents

* SOP-02 Alarm Response
* SOP-03 AI Lockout
* SOP-04 Rollback
* SOP-05 Watchdog Test
* SOP-AI-01 RAG Ingestion
* INCIDENT_RESPONSE_PLAN.md
* SAFETY_INTERFACE.md

---

# 12. Final Rule

> If troubleshooting requires bypassing safety — you are solving the wrong problem.

Always preserve:

* Safety
* Determinism
* Auditability

---
