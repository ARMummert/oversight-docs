# FAT-01: Factory Acceptance Test – Hardware & Communication Protocol

### Project: 
OverSight ICS | Version: 1.1 | **Lead Engineer**: armummert 

### Scope: 
This document defines the protocol for validating the physical and logical network integrity, clock synchronization, 
and telemetry ingest stability of the Industrial IPC and Edge Gateway hardware.

### 1. Test Environment Setup

#### 1.1 Hardware Requirements
- **Hardware**: Industrial PC (IPC) with fanless chassis and Intel Core i5 (or equivalent) @ 2.4GHz+
- **Storage**: 256GB Industrial Grade SSD (SLC/pSLC)
- **Memory**: 16GB ECC DDR4 RAM (Minimum)
- **Digital I/O**: 1x DIN-Rail mounted Digital Output Module (for HW-06/08 physical relay check)
- **Networking**: Dual 1GbE Intel i210/i211 NICs (Required for hardware-level PTP support)
- **Network Isolation**: NIC 1 (Northbound / IT) and NIC 2 (Southbound / OT) must be physically connected to independent 
  subnets.
- **Security**: TPM 2.0 Module (Active and Provisioned)

#### 1.2 Core Software Stack
The following software versions must be verified and locked before testing begins:
- **Edge Gateway**: v1.0 (TLS 1.3 Enabled)
- **Southbound Driver**: v1.0.0 (Deterministic C++ Build)
- **OS**: Linux Mint / Debian 12 (Hardened Kernel - Real-time Kernel `PREEMPT_RT` preferred)
- **Containerization**: Docker 24.0+ with Compose v2.20+ for service isolation and virtual networking.

### 1.3 Communication Protocol & Simulation
- **Communication Protocol**: Standardized on **JSON over TLS 1.3** (Northbound); **Sparkplug B** / **MQTT** 
  (Southbound).
- **Northbound (NIC 1)**: JSON over TLS 1.3.
- **Southbound (NIC 2)**:
    - **MQTT**: Sparkplug B for Unified Namespace (UNS).
    - **OPC-UA**: Binary TCP with Sign & Encrypt (Basic256Sha256).
    - **EtherNet/IP**: CIP over TCP/UDP for Allen-Bradley asset integration.
    - **Modbus TCP**: Legacy register access for secondary instrumentation.
- **Simulation**: Oversight Field Asset Simulator (Fault Injection Module enabled) 
- **Clock Sync**: PTPd / chrony (configured for L2 PTP Grandmaster)

### 2. Communication & Hardware Test Cases
| ID        | Test Category          | Description                                                   | Expected Result                                             | Pass/Fail |
|:----------|:-----------------------|:--------------------------------------------------------------|:------------------------------------------------------------|:----------|
| **HW-01** | Port Integrity         | Run `nmap -sT -p-` against NIC 1 and NIC 2.                   | Only ports 443 (N) and 1883/502/44818 (S) are responsive.   | [ ]       |
| **HW-02** | PTP Stability          | Monitor clock offset for 30 mins via `pmc`.                   | Jitter remains < 1μs; zero synchronization drift.           | [ ]       |
| **HW-03** | Ingest Volume          | 50 tags at 100ms intervals for 1 hour.                        | 100% packet success; zero buffer overflows in logs.         | [ ]       |
| **HW-04** | Comms Fault            | Physically disconnect NIC 2 (Southbound) cable.               | System triggers `COMM_LOSS` alarm within < 2s.              | [ ]       |
| **HW-05** | Cert Security          | Connect via Gateway with expired/self-signed cert.            | Connection Rejected; log shows `AUTH_INVALID_CERT`.         | [ ]       |
| **HW-06** | Watchdog Trip          | Execute `kill -STOP` on Southbound Driver process.            | Physical DO-0 Relay opens (Safe State) within 500ms.        | [ ]       |
| **HW-07** | Modbus Security        | Spoof Modbus traffic from NIC 1 (Northbound).                 | Driver drops packet; Firewall logs `UNAUTHORIZED_SOURCE`.   | [ ]       |
| **HW-08** | Power Recovery         | Perform Hard Power Cut; Restore Power.                        | DO-0 remains in Safe State until OS/Driver are initialized. | [ ]       |
| **HW-09** | NIC Isolation          | Initiate UDP Broadcast Storm on NIC 1 (IT side).              | NIC 2 control latency/jitter (HW-02) remains unaffected.    | [ ]       |
| **HW-10** | OPC-UA Secure          | Attempt connection with untrusted client cert.                | Connection rejected; Audit log records security violation.  | [ ]       |
| **HW-11** | EtherNet/IP Stability  | Simulate 5% packet loss on EtherNet/IP I/O traffic.           | Driver triggers Safe-State within RPI threshold.            | [ ]       |
| **HW-12** | Thermal Stress         | Run CPU at 100% load for 30 minutes.                          | No thermal throttling; PTP jitter (HW-02) remains < 5μs.    | [ ]       |
| **HW-13** | Disk Health            | Verify SSD S.M.A.R.T. status after 24hr soak.                 | Zero "Reallocated Sector" counts; "Wear Leveling" < 1%.     | [ ]       |
| **HW-14** | Shielding              | Check continuity between NIC housing and ground.              | Resistance < 1 Ohm (Ensures EMI protection for OT comms).   | [ ]       |
| **HW-15** | Secure Boot Validation | Alter OS kernel hash to simulate tampering and initiate boot. | Boot process is blocked by TPM 2.0; OS load is prevented.   | [ ]       |

### 3. Hardware Signature (To be completed on-site)
| Component          | Manufacturer / Model      | Serial Number / Asset ID | Firmware/BIOS Version |
|:-------------------|:--------------------------|:-------------------------|:----------------------|
| **Industrial IPC** | (e.g., Advantech/OnLogic) |                          |                       |
| **NIC 1 (North)**  | Intel i210/211            | MAC:                     | (Driver Version)      |
| **NIC 2 (South)**  | Intel i210/211            | MAC:                     | (Driver Version)      |
| **Digital I/O**    | (Modbus/GPIO Mod)         |                          |                       |
| **TPM Module**     | (Integrated/Discrete)     | Version: 2.0             |                       |
| **Power Supply**   | (DIN-rail DC/DC)          |                          |                       |
| **SSD / Storage**  |                           |                          |                       |

### 4. Sign-off
**Test Performed By:** ___________________________ **Date:** _______________  
**Witnessed By:** _______________________________ **Date:** _______________

## Revision History
| Version | Date       | Author    | Description of Change                      | Approved By |
|:--------|:-----------|:----------|:-------------------------------------------|:------------|
| 1.0     | 2026-04-12 | armummert | Initial Version release 1.0.               | armummert   |
| 1.1     | 2026-04-27 | armummert | Added HW-15 for TPM Secure Boot Validation | armummert   |
