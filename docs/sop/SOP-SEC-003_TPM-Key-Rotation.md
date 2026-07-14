# SOP-08: TPM 2.0 Key Rotation Procedure

### 1. Objective
This SOP defines the procedure for rotating the RSA private keys stored within the Edge Gateway's **Trusted Platform 
Module (TPM 2.0)** every 180 days.

### 2. Pre-Requisites
* System must be in **Passive Monitor** mode.
* Primary and Secondary (Safety Lead) Operator JWTs must be present.
* `maintenance_console` access on NIC 1.

### 3. Step-by-Step Procedure
1.  **Initiation**: Trigger the "Key Re-Gen" command via the maintenance console.
2.  **Dual-Auth**: Primary Operator and Safety Lead must provide their JWT signatures to authorize the TPM write.
3.  **Generation**: The TPM generates a new RSA-4096 key pair; the private key is locked to the hardware.
4.  **Grace Period**: The Southbound Driver caches the *previous* public key for 60 seconds to allow in-flight intents to clear.
5.  **Activation**: After 60 seconds, the new public key is pushed to the Southbound Driver for all future `Signature_Verify` calls.
6.  **Verification**: Confirm successful rotation via the **Execution Layer** of the Audit Log.