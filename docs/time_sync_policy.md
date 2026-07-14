# Time Synchronization Policy

## 1. Objective
To ensure forensic alignment between AI reasoning and physical execution, OverSight ICS implements a tiered 
time synchronization model. This allows for precise correlation between the Reasoning Agent intents, the Southbound
Driver execution, and the field asset level telemetry.

## 2. Synchronization Tiers

| Network Zone            | Protocol            | Accuracy | Purpose                                                                 |
|:------------------------|:--------------------|:---------|:------------------------------------------------------------------------|
| **Enterprise (L4)**     | **NTP**             | ±50ms    | Syncing the Next.js Dashboard and User logs.                            |
| **Industrial DMZ (L3)** | **NTP**             | ±10ms    | Syncing the Edge Gateway to the Corporate Stratum 2 server.             |
| **Control Zone (L2)**   | **PTP (IEEE 1588)** | **<1μs** | Syncing the Southbound Driver and Field assets for $Z$-Score telemetry. |

### The Edge Gateway as a Time Bridge
The Edge Gateway acts as the **PTP Grandmaster** for the Southbound network (NIC 2).
1. It pulls the wall clock time via **NTP** from the corporate network (NIC 1).
2. It translates this to a high-precision **PTP hardware clock** on NIC 2.
3. This makes certain that the Level 2 telemetry is timestamped at the NIC level before it even reaches the Southbound 
   Driver's software.

### PTP Redundancy & BMCA
In the event of an Edge Gateway hardware failure, the Level 2 network must maintain synchronization to prevent telemetry 
jitter during recovery.

- **BMCA (Best Master Clock Algorithm)**: Oversight uses the standard IEEE 1588 BMCA to select the Grandmaster.

| Device                               | Priority 1 Value | Role                              |
|:-------------------------------------|:-----------------|:----------------------------------|
| Edge Gateway (PTP Grandmaster)       | 128              | Primary Master                    |
| Primary Field Asset (Passive Master) | 255              | Secondary Master / Passive Master |

*Per the IEEE 1588 standard, the BMCA selects the clock with the **lowest numerical value** as the preferred
Grandmaster.*

- **Configuration**: The Edge Gateway (Priority 128) is numerically lower than the Primary Field Asset (Priority 255),
  so it grants it higher priority.
- **Failure Logic**: If the Edge Gateway (Priority 128) stops announcing sync messages for >3 announce intervals, the 
  Primary PLC will autonomously take over as the Grandmaster.

## Revision History
| Version | Date       | Author       | Description of Change                                           | Approved By |
|:--------|:-----------|:-------------|:----------------------------------------------------------------|:------------|
| 1.0     | 2026-04-12 | [User]       | Initial Baseline for Version 1.0 Release.                       | [Manager]   |