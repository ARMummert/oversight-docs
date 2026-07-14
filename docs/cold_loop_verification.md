## CP-01: Cold-Loop Verification (Pre-Commissioning)
* **Objective**: Confirm physical wiring integrity and instrument scaling before software ingestion [cite: Auditor's Findings].
* **Step 1: P&ID Verification**: Compare `asset_id` in the UNS against the physical tag on the instrument [cite: Auditor's Findings].
* **Step 2: Point-to-Point Check**: Force a 50% signal at the field transmitter; verify raw value in the Southbound Driver [cite: COMMISSIONING.md].
* **Step 3: Grounding Audit**: Measure resistance between Edge Gateway chassis and plant common ground (must be < 1 Ohm) [cite: COMMISSIONING.md].
* **Acceptance Criteria**: Physical tags match P&ID; raw signal scaling is accurate within ±0.5%; grounding is < 1 Ohm [cite: Auditor's Findings].