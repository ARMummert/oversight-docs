# OverSight ICS: Infrastructure & Network Specification

**Security Level:** `Level 3.5 IDMZ`
**Scope:** Edge Gateway, Network Synchronization, and Failover Hardware

## 1. Network Synchronization (PTP)
To ensure the integrity of the WORM (Write Once Read Many) telemetry storage and the precision of $Z$-Score calculations, the system requires sub-microsecond clock synchronization.

### 1.1 PTP Grandmaster Failover
- **Primary Clock**: The system syncs to a Stratum 1 GPS-disciplined PTP Grandmaster.
- **Failover Protection**: In the event of GPS signal loss or hardware failure, the Edge Gateway must automatically failover to a local Stratum 2 rubidium oscillator [cite: 3.1].
- **Holdover Stability**: The local oscillator must maintain < 1ms drift over a 24-hour period to prevent "Time-Smearing" in the telemetry database [cite: 3.1].
- **Heartbeat Monitoring**: The `/api/v1/system/heartbeat` endpoint reports PTP sync status. A `PTP_SYNC_LOSS` advisory is issued if drift exceeds 500ns [cite: 3.1].

## 2. Edge Gateway Hardware
- **Processing**: Industrial PC with Cgroup-limited CPU allocation to prevent "Zombie Agent" stall scenarios [cite: 3.1].
- **Storage**: Encrypted NVMe with a WORM-compliant filesystem for the immutable Audit Log.
- **Watchdog**: A hardware-based watchdog timer (PVP-02) is required. If the Edge Agent fails to pet the watchdog for 500ms, the system defaults to a "Safe State" (Hardware-bypass) [cite: 3.1].

## 3. Power Recovery & Degraded Modes
- **UPS Integration**: The Edge Gateway is backed by an industrial UPS.
- **Power Loss Protocol**: Upon loss of main power, the system enters "Minimalist Mode"—disabling LangGraph reasoning and focusing exclusively on 1:1 Safety Heartbeats to maintain the field asset link [cite: 3.1].
- **Cold Boot Recovery**: Following power restoration, the system performs a SHA-256 integrity check on all log blocks before resuming autonomous orchestration [cite: 3.1].
