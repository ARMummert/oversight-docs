# OverSight ICS: Document Control Index

## 1. Governance Overview
The Document Control Index serves as the master registry for all architectural, safety, and compliance documentation 
governing the OverSight ICS platform. Documents are categorized by their role in the system's **Chain of Trust**.

## 2. Core Architectural Documents
These files define the fundamental logic and authority structures of the system.

| Ref ID     | Document Name                                         | Current Version | Last Updated | Status      |
|:-----------|:------------------------------------------------------|:----------------|:-------------|:------------|
| **DOC-01** | [Control Authority Model](control_authority_model.md) | v1.0.2          | 2024-05-22   | **Active**  |
| **DOC-02** | [SBOM & Spares Inventory](sbom_spares_inventory.md)   | v0.1.0-alpha    | 2024-05-22   | **Draft**   |
| **DOC-03** | [System Operating Modes](operating_modes.md)          | v1.0.0          | 2024-05-21   | **Active**  |
| **DOC-04** | [Network Segmentation Plan](network_topology.md)      | TBD             | ---          | **Planned** |

## 3. Technical Specifications & Interface Docs
Detailed specifications for developers and systems integrators.

| Ref ID      | Document Name                    | Target Level | Format         |
|:------------|:---------------------------------|:-------------|:---------------|
| **SPEC-01** | LangGraph Agent Intent Schema    | Level 3.5    | JSON Schema    |
| **SPEC-02** | TPM 2.0 Command Signing Protocol | Level 1.5    | Markdown / C++ |
| **SPEC-03** | Field Asset Modbus Register Map  | Level 1.0    | CSV / XLSX     |

## 4. Compliance & Security Artifacts
Records required for hardware root-of-trust validation and forensic auditing.

| Artifact Type          | Storage Location        | Retention Period |
|:-----------------------|:------------------------|:-----------------|
| **TPM PCR Event Logs** | TimescaleDB (Immutable) | 7 Years          |
| **Write-Access Logs**  | TimescaleDB (Historian) | 7 Years          |
| **Firmware Manifests** | `/security/manifests/`  | Life of Asset    |

## 5. Change Control Procedure
To maintain the **Known Good State**, any changes to documents in Section 2 must follow the **Bumpless 
Documentation Update** protocol:
1. **Draft:** Changes are made in a separate branch.
2. **Review:** Peer review for safety implications (impact on `ORPHAN_MODE` or `SAFETY_LOCKOUT`).
3. **Attestation:** New SHA-256 hashes are calculated and updated in the **SBOM**.
4. **Release:** Document is merged and the version number is incremented.

---
*Last Index Audit: 2024-05-22*
*Authorized by: OverSight System Architect*