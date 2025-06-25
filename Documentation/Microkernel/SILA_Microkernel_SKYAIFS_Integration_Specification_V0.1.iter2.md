# SILA Microkernel & SKYAIFS Integration Specification - V0.1 - Iteration 2

**Based on:** `Documentation/Microkernel/SILA_Microkernel_SKYAIFS_Integration_Specification_V0.1.iter1.md`
**Key References:** 
*   `Documentation/SILA_Language/SILA_Specification_V0.2.md`
*   `Documentation/SKYAIFS_Filesystem/SKYAIFS_Framework_V0.2.md`
*   `Documentation/Microkernel/SILA_Microkernel_Internals_Design_V0.1.md` (and its evolution)
*   Relevant sections from `Documentation/OS_Architecture_Synthesis/Skyscope_OS_Integrated_Core_Component_Design_V0.1_SILA_Edition.md`
**Iteration Focus:** Detailing SILA-based fault reporting/handling, PQC key management interactions, concurrency control for shared resources, and refining atomicity protocols.

## 1. Introduction
Retained from Iteration 1. This iteration deepens the SILA-based interaction protocols between the Microkernel and SKYAIFS, focusing on robustness, security (PQC min 4096-bit), and advanced operational scenarios, all expressed using SILA V0.2 constructs.

## 2. Detailed SILA IPC Protocols

### 2.1. Raw Block I/O Protocol
Retained and refined from Iteration 1. The SILA Record structures for `SILA_BlockRead_Request_Record`, `SILA_BlockRead_Response_Record`, `SILA_BlockWrite_Request_Record`, and `SILA_BlockWrite_Response_Record` are considered stable for now, emphasizing PQC-aware types and capability passing.

### 2.2. Fault Reporting and Handling Protocols (SILA IPC)

#### 2.2.1. Microkernel to SKYAIFS Fault Reporting (Storage Device Issues)
*   **Scenario:** The Microkernel's Storage Service ASA detects an uncorrectable error on a physical storage device (e.g., during a read/write operation initiated by SKYAIFS).
*   **SILA Message Structure (`SILA_Microkernel_StorageDevice_Fault_Event_Record` - a SILA Record type, sent as a SILA Event):**
    `{
      event_id: SILA_UniqueID_String,
      faulting_storage_device_cap_fingerprint: SILA_CapFingerprint_Type, // Verifiable fingerprint of the device capability SKYAIFS was using
      faulting_logical_block_address_opt: SILA_Optional<SILA_LBA_Integer_Type>, // LBA if applicable
      reported_error_code_enum: SILA_StorageDeviceError_Enum { // SILA Enum
        UncorrectableReadError_EnumVal, 
        WriteVerificationError_EnumVal, 
        DeviceNotRespondingError_EnumVal, 
        MediaCorruptedSector_EnumVal 
      },
      timestamp_of_detection: SILA_Timestamp_Record, // PQC-signed timestamp from trusted time source
      original_skyaifs_request_id_opt: SILA_Optional<SILA_UniqueID_String>, // From the SKYAIFS original I/O request, if related
      diagnostic_info_sila_cap_opt: SILA_Optional<SILA_CapToken<SILA_KernelDiagnosticReport_Record>> // Capability to more detailed (PQC-signed) kernel diagnostics
    }`
*   **SILA Interaction Graph (Conceptual):**
    1.  Microkernel Storage Service ASA detects fault.
    2.  Constructs `SILA_Microkernel_StorageDevice_Fault_Event_Record`.
    3.  Sends this SILA event via secure SILA IPC to a pre-registered SKYAIFS "StorageHealthMonitor_ASA_EP_Cap" (an endpoint capability provided by SKYAIFS during its initialization).
*   **SKYAIFS ASA Response (Conceptual - within StorageHealthMonitor_ASA):**
    1.  Receives the SILA event. Logs it using `SILA_SKYAIFS_LogAuditEvent_Operation`.
    2.  Initiates internal SILA operations: Marks the affected block/device region as "suspect" or "degraded" in SKYAIFS metadata (this itself is a PQC-signed, versioned SILA graph update).
    3.  May trigger `SKYAIFS_DataRelocationBot_ASA` (via SILA IPC command) to proactively move data from the suspect area, if policy dictates.
    4.  Optionally acknowledges receipt to the Microkernel via a `SILA_IPC_Reply_Operation` if the protocol requires it (e.g., `SILA_FaultNotification_Acknowledgement_Record`).

#### 2.2.2. SKYAIFS to Microkernel Fault/Suspected Integrity Reporting
*   **Scenario:** An SKYAIFS Integrity Verification Bot ASA (or other SKYAIFS component) detects a metadata inconsistency or data corruption that it suspects might be due to an underlying storage issue not yet caught by the Microkernel/hardware (e.g., bit rot subtly corrupting a PQC hash).
*   **SILA Message Structure (`SILA_SKYAIFS_SuspectedStorageIntegrityIssue_Report_Record` - a SILA Record type):**
    `{
      report_id: SILA_UniqueID_String,
      reporting_skyaifs_agent_id_cap: SILA_CapToken<SKYAIFS_Bot_ASA_Type>, // Authenticates the reporter
      suspected_storage_device_cap_fingerprint: SILA_CapFingerprint_Type, // Device SKYAIFS believes is affected
      affected_lba_range_opt: SILA_Optional<SILA_LBARange_Struct_Record>, // LBA range if known
      description_of_inconsistency_sila_string: SILA_String_Record,
      pqc_evidence_payload_cap_opt: SILA_Optional<SILA_CapToken<SILA_Memory_Region_Type>> // e.g., copy of corrupted metadata block, expected vs actual PQC hashes
    }`
*   **SILA Interaction Graph (Conceptual):**
    1.  SKYAIFS Bot ASA constructs the report.
    2.  Sends this report via SILA IPC to a Microkernel "StorageSystem_Diagnostics_ASA_EP_Cap".
*   **Microkernel ASA Response (Conceptual - within StorageSystem_Diagnostics_ASA):**
    1.  Logs the report. Validates the `reporting_skyaifs_agent_id_cap`.
    2.  May schedule low-level device diagnostic operations (e.g., read-after-write tests, surface scans if supported via SILA device control operations on the suspected device). These are SILA-orchestrated tasks.
    3.  Reports findings (as a `SILA_KernelDiagnosticResponse_Record`) back to SKYAIFS via a reply SILA IPC call.

### 2.3. PQC Key Management Interactions (Conceptual SILA)

Assuming a Microkernel-protected "System_KeyVault_ASA" manages or provides derivation services for high-entropy master PQC keys (as per `SILA_Specification_V0.2.md` and security policies).

#### 2.3.1. SKYAIFS Request for Per-File/Block Derived PQC Key
*   **Scenario:** SKYAIFS needs a new unique PQC key (e.g., for ML-KEM, adhering to min 4096-bit security like Kyber-1024) for encrypting a new file's content blocks.
*   **SILA Message (`SILA_SKYAIFS_RequestDerived_PQCKey_Operation_Record` - a SILA Record for a Call):**
    `{
      request_id: SILA_UniqueID_String,
      // Optional: Capability to a SKYAIFS-managed sub-master key if using a key hierarchy.
      // parent_key_material_for_derivation_cap_opt: SILA_Optional<SILA_CapToken<SILA_PQC_Key_Object_Type>>, 
      derivation_input_unique_salt_sila_blob: SILA_Blob_Record, // e.g., PQC hash of unique file ID + version + block_index
      key_specification_policy_cap: SILA_CapToken<SILA_PQC_KeyGenerationPolicy_Record>, // Specifies MLKEM_1024, usage flags (encrypt/decrypt), lifetime
      reply_to_skyaifs_key_manager_ep_cap: SILA_CapToken<SILA_IPC_Endpoint_Type>
    }`
*   **SILA Interaction Graph (Conceptual):**
    1.  SKYAIFS Key Management ASA sends this request to `System_KeyVault_ASA_EP_Cap`.
*   **`System_KeyVault_ASA` Response (`SILA_Derived_PQCKey_Response_Record` - SILA Record for Reply/Event):**
    `{
      original_request_id_ref: SILA_UniqueID_String,
      key_generation_status_enum: SILA_KeyGenerationStatus_Enum { Success_EnumVal, AuthorizationError_EnumVal, PolicyViolation_KeySpec_EnumVal, KeyVaultInternalError_EnumVal },
      derived_pqc_key_object_cap_opt: SILA_Optional<SILA_CapToken<SILA_PQC_Key_Object_Type>>, // Capability to the newly derived (and securely stored by KeyVault) key
      error_details_sila_record_opt: SILA_Optional<SILA_CapToken<SILA_Error_Record>>
    }`
    *   The `derived_pqc_key_object_cap_opt` is a SILA capability. SKYAIFS can use this capability with SILA PQC cryptographic primitive operations (e.g., `SILA_PQC_Encrypt_Operation(data_cap, derived_key_cap, params_cap)`). SKYAIFS cannot directly read or export the raw key bits from this capability. The KeyVault ASA manages the actual key material.

## 3. Shared SILA Data Structures & Capability Management
Retained and refined from Iteration 1. The `SILA_Shared_IO_Buffer_Record` and associated SILA capability flow (Microkernel allocates, SKYAIFS receives main cap, passes derived restricted caps for I/O) is crucial.

## 4. Atomicity & Consistency Considerations

### 4.1. Refined Two-Phase Commit (2PC) SILA Protocol (Conceptual)
Building on Iteration 1's concept for atomic metadata update + kernel resource allocation:
*   **Timeout Handling (SILA Timer Services & Contracts):**
    *   The initiating SKYAIFS ASA (acting as 2PC Coordinator) uses a `SILA_TimerService_ASA` to set timeouts after sending `SILA_PrepareResourceAllocation_Request` to the Microkernel (Participant).
    *   If a timeout SILA event is received before the Microkernel's `SILA_PrepareResource_Response`, the SKYAIFS Coordinator sends `SILA_AbortResourceAllocation_Request` and rolls back its own state. This is part of its SILA state machine logic.
    *   The Microkernel Participant also has an internal timeout. If it has "prepared" a resource (e.g., locked it, pending final commit) and doesn't receive a `SILA_CommitResourceAllocation_Request` or `SILA_AbortResourceAllocation_Request` from the SKYAIFS Coordinator within a defined window (specified in its `SILA_Module_Contract_Record`), it may unilaterally abort the transaction, release the resource, and log this event.
*   **Recovery Paths & Persistent State (SILA Graph Logic & PQC-Signed Logs):**
    *   **Coordinator (SKYAIFS) Restart after Prepare Sent:** Upon restarting, the SKYAIFS Coordinator ASA checks its PQC-signed persistent log (stored in SKYAIFS itself). If a transaction `TX_ID_123` is found in "PrepareSent" state, it re-sends a query (`SILA_QueryTransactionStatus_Request { transaction_id: TX_ID_123 }`) to the Microkernel's Transactional Service endpoint.
        *   Microkernel responds with current status: `Prepared`, `Committed`, `Aborted`.
        *   SKYAIFS Coordinator then proceeds with commit (if its own log shows intent to commit and Microkernel is prepared) or abort.
    *   **Participant (Microkernel) Restart after Prepare Acknowledged:** Upon restarting, the Microkernel's Transactional Service ASA loads its list of "prepared" transactions from its PQC-signed persistent log. It then waits for the respective SKYAIFS Coordinators to re-contact it to drive resolution (commit/abort). If a coordinator doesn't re-contact within a longer recovery timeout, the Microkernel may unilaterally abort those specific prepared transactions to free resources, logging this decision extensively.
*   **SILA Contracts for 2PC Participants:** The `SILA_Module_Contract_Record` for both the SKYAIFS ASA initiating 2PC and the Microkernel's Transactional Service ASA must formally specify their roles, responsibilities, valid state transitions, timeout behaviors, and error reporting mechanisms within the 2PC protocol. The SILA Verifier checks these contracts for compatibility and completeness.

## 5. Concurrency Control for Shared Resources (SILA)

*   **Microkernel-Managed Locks via SILA Capabilities (Refined):**
    *   The Microkernel provides a "ResourceLockManager_ASA" service.
    *   SKYAIFS ASAs request locks on abstract resources (e.g., identified by a `SILA_Resource_URI_String` or a capability to a physical device region):
        `SILA_LockManager_RequestLock_Operation(
          target_resource_identifier_cap: SILA_CapToken, // Capability to the resource or its descriptor
          lock_type_enum: SILA_LockType_Enum { SharedRead_EnumVal, ExclusiveWrite_EnumVal, IntentToUpgrade_EnumVal },
          lock_timeout_ms_opt: SILA_Optional<SILA_Int>,
          reply_to_ep_cap: SILA_CapToken
        ) -> SILA_Async_JobID_Record`
    *   Response: `SILA_LockGrant_Status_Event_Record { original_request_id, status, granted_lock_object_cap_opt: SILA_Optional<SILA_CapToken<SILA_ResourceLock_Object_Type>> }`.
    *   The `SILA_ResourceLock_Object_Type` capability must be presented by SKYAIFS in subsequent Microkernel operations that require that lock on the target resource. The Microkernel SILA runtime verifies this lock capability.
    *   The `ResourceLockManager_ASA` enforces lock semantics (e.g., multiple shared reads allowed, only one exclusive write, deadlock prevention via lock ordering if complex resources are involved). Its internal logic is a target for formal verification.
*   **SILA Capability Rights for Fine-Grained Access:** As outlined in `SILA_Specification_V0.2.md`, SILA capabilities are parameterized by the type of object they reference and can carry specific rights. The Microkernel, when granting capabilities to SKYAIFS (e.g., for a device or memory region), will grant the most restrictive rights necessary by default. SKYAIFS may need to perform a separate, auditable SILA IPC call to request an elevation of rights on a capability, which is subject to strict policy checks by the Microkernel.

## Iteration 2 Conclusion
This iteration has substantially deepened the Microkernel-SKYAIFS integration specification by:
1.  Detailing SILA IPC protocols and message structures for fault reporting in both directions (Microkernel to SKYAIFS and SKYAIFS to Microkernel).
2.  Outlining a conceptual SILA IPC protocol for SKYAIFS to request derived PQC keys from a Microkernel-protected KeyVault ASA, emphasizing capability-based key access.
3.  Refining the Two-Phase Commit (2PC) SILA protocol with considerations for timeout handling and basic recovery paths, highlighting the role of SILA contracts.
4.  Elaborating on concurrency control mechanisms using a Microkernel-managed Lock Manager ASA and the inherent access control provided by SILA capability rights.
These enhancements pave the way for a more robust, secure, and resilient interaction model between these two critical Skyscope OS components.The file `Documentation/Microkernel/SILA_Microkernel_SKYAIFS_Integration_Specification_V0.1.iter2.md` has been successfully created with the specified content.

This completes the simulation of Iteration 2 for the Microkernel-SKYAIFS Integration Specification. The next step is to report this completion.
